# Media Manager Charm Implementation (Radarr/Sonarr)

## Context and Problem Statement

Charmarr requires charm implementations for media manager applications (Radarr, Sonarr, and future Lidarr). These charms share ~95% identical logic, differing only in ports, media types, and Recyclarr templates. We need to define the implementation details including reconciler pattern, Pebble configuration, storage handling, relation management, config options, and actions while ensuring compliance with all previously established MADRs (storage, interfaces, authentication, Recyclarr integration).

## Considered Options

### Reconciler Event Handling
* **Option 1:** Single `observe_events()` for all events including custom interface events
* **Option 2:** `observe_events()` for standard K8s events + manual observation for custom interface events

### Workload Readiness
* **Option 1:** Poll API endpoint in reconciler loop
* **Option 2:** Pebble HTTP check with `level: ready`
* **Option 3:** Check service status only

### Recyclarr Execution
* **Option 1:** Track state to run once per lifecycle
* **Option 2:** Run every reconcile if configured (idempotent)

### Status Collection
* **Option 1:** Early returns on first issue found
* **Option 2:** Collect all statuses, let Juju aggregate

### External URL Configuration
* **Option 1:** Manual charm config option
* **Option 2:** Auto-configure from ingress relation data

## Decision Outcome

### Reconciler: Option 2 - Hybrid event observation

Standard K8s events via `observe_events()`, custom interface events manually observed. All funnel to single `_reconcile()` method.

```python
def __init__(self, framework: ops.Framework):
    # Standard K8s events
    observe_events(self, reconcilable_events_k8s, self._reconcile)

    # Custom interface events
    self.framework.observe(self.indexer.on.changed, self._reconcile)
    self.framework.observe(self.download_clients.on.changed, self._reconcile)
    self.framework.observe(self.storage.on.changed, self._reconcile)

    # Secret rotation
    self.framework.observe(self.on.secret_changed, self._reconcile)

    # Status (separate handler)
    self.framework.observe(self.on.collect_unit_status, self._collect_status)
```

**Rationale:** Custom interface events from charmarr-lib are charm-specific and not part of the standard ops event set. Explicit observation makes dependencies clear.

### Workload Readiness: Option 2 - Pebble HTTP check

```python
"checks": {
    "radarr-ready": {
        "override": "replace",
        "level": "ready",
        "http": {"url": f"http://localhost:{port}/ping"},
        "period": "10s",
        "timeout": "3s",
        "threshold": 3,
    }
}
```

**Rationale:** Pebble checks are the idiomatic way to handle readiness. The `/ping` endpoint requires no authentication and returns 200 when service is ready. `level: ready` ensures container is only marked ready when check passes.

### Recyclarr: Option 2 - Run every reconcile

```python
def _sync_trash_profiles(self) -> None:
    profiles_config = self.config.get("trash-profiles", "").strip()
    if not profiles_config:
        return  # Not configured, skip

    self._validate_trash_profiles(profiles_config)
    # ... generate config and run subprocess ...
```

**Rationale:**
- Juju spawns fresh Python process per event → instance variables reset → state tracking impossible
- Recyclarr is idempotent and cheap (~2-5 seconds)
- Simpler code, no state management complexity

### Status Collection: Option 2 - Collect all statuses

```python
def _collect_status(self, event: ops.CollectStatusEvent) -> None:
    container = self.unit.get_container(self._container_name)

    if not container.can_connect():
        event.add_status(ops.WaitingStatus("Waiting for Pebble"))
        return  # Only early-return here

    # Check all conditions, add all relevant statuses
    if not self.storage.is_ready():
        event.add_status(ops.BlockedStatus("Missing relation: media-storage"))

    if not self._workload_ready(container):
        event.add_status(ops.WaitingStatus("Waiting for workload"))

    # ... more checks ...

    event.add_status(ops.ActiveStatus())
```

**Rationale:** Juju aggregates statuses and shows the worst. Collecting all statuses gives operators complete visibility into charm state rather than hiding secondary issues.

### External URL: Option 2 - Auto-configure from ingress

```python
def _configure_external_url(self) -> None:
    if not self._ingress.is_ready():
        return

    ingress_data = self._ingress.get_data()
    scheme = "https" if ingress_data.tls else "http"
    external_url = f"{scheme}://{ingress_data.host}"

    # Configure via API if different
    # ...
```

**Rationale:** Ingress relation provides host and TLS status. Automation eliminates manual configuration. Follows Charmarr philosophy of "relate and it just works".

## Implementation Details

### Reconciler Phases

```
┌─────────────────────────────────────────────────────────────────┐
│                     RECONCILER FLOW                             │
├─────────────────────────────────────────────────────────────────┤
│  Phase 1: Container connection                                  │
│      └─► Early return if can't connect                         │
│                                                                 │
│  Phase 2: Shared storage mounting  [ORDER: Before Phase 3]     │
│      └─► lightkube patch StatefulSet                           │
│      └─► Early return if not ready                             │
│                                                                 │
│  Phase 3: Pebble layer configuration                           │
│      └─► Start workload                                        │
│                                                                 │
│  Phase 4: Workload readiness check  [ORDER: After Phase 3]     │
│      └─► Pebble check status                                   │
│      └─► Early return if not ready                             │
│                                                                 │
│  Phase 5: API key + root folder  [ORDER: After Phase 4]        │
│      └─► Read from config.xml, store in Juju Secret            │
│      └─► Auto-configure root folder via API                    │
│      └─► Drift detection on update-status                      │
│                                                                 │
│  Phase 6: Recyclarr sync  [ORDER: After Phase 5]               │
│      └─► Skip if trash-profiles not configured                 │
│      └─► Idempotent, runs every reconcile                      │
│                                                                 │
│  Phase 7: External URL  [ORDER: After Phase 5]                 │
│      └─► Configure from ingress relation                       │
│                                                                 │
│  Phase 8: Publish to relations  [ORDER: After Phase 6]         │
│      └─► media-indexer: API URL + secret ID                    │
│      └─► media-manager: API URL + secret ID + profiles         │
│                                                                 │
│  Phase 9: Configure from relations  [ORDER: After Phase 5]     │
│      └─► Auto-add download clients from relation data          │
│                                                                 │
│  Phase 10: Ingress config  [Leader only]                       │
│      └─► Submit IstioIngressRouteConfig                        │
└─────────────────────────────────────────────────────────────────┘
```

### Pebble Layer

```python
def _build_pebble_layer(self) -> ops.pebble.LayerDict:
    return {
        "summary": "Radarr layer",
        "services": {
            "radarr": {
                "override": "replace",
                "command": "/app/bin/Radarr -nobrowser -data=/config",
                "startup": "enabled",
                "environment": {
                    "RADARR__LOG__LEVEL": self._log_level_map.get(
                        self.config.get("log-level", "info").lower(),
                        "Info"
                    ),
                    "RADARR__APP__INSTANCENAME": self.app.name,
                },
            }
        },
        "checks": {
            "radarr-ready": {
                "override": "replace",
                "level": "ready",
                "http": {"url": f"http://localhost:{self._service_port}/ping"},
                "period": "10s",
                "timeout": "3s",
                "threshold": 3,
            }
        },
    }
```

### Storage Patching

Per ADR storage/adr-003, use lightkube to patch StatefulSet:

```python
def _ensure_shared_storage_mounted(self) -> bool:
    if not self.storage.is_ready():
        return False

    provider = self.storage.get_provider_data()
    if not provider:
        return False

    # Check if already mounted (idempotency)
    if self._shared_storage_mounted():
        return True

    # Patch StatefulSet
    client = Client()
    sts = client.get(StatefulSet, name=self.app.name, namespace=self.model.name)

    # Add volume
    volume = Volume(
        name="charmarr-shared-data",
        persistentVolumeClaim=PersistentVolumeClaimVolumeSource(
            claimName=provider.pvc_name
        )
    )

    # Add volumeMount to container (use exact container name from charmcraft.yaml)
    mount = VolumeMount(name="charmarr-shared-data", mountPath=provider.mount_path)

    # ... apply patch ...
    return True
```

### API Key Management

Per ADR apps/adr-002:

```python
def _ensure_api_key_secret(self) -> None:
    """Read API key from config.xml, store in Juju Secret."""
    container = self.unit.get_container(self._container_name)

    # Read from config.xml
    config_content = container.pull(f"{self._config_path}/config.xml").read()
    root = ET.fromstring(config_content)
    api_key = root.find("ApiKey").text

    # Create or update secret
    try:
        secret = self.model.get_secret(label=f"{self.app.name}-api-key")
        # Check drift on update-status
    except SecretNotFoundError:
        secret = self.app.add_secret(
            content={"api-key": api_key},
            label=f"{self.app.name}-api-key",
        )

    # Grant to relations
    for relation in self.model.relations["media-indexer"]:
        secret.grant(relation)
    for relation in self.model.relations["media-manager"]:
        secret.grant(relation)

def _check_credential_drift(self) -> None:
    """Detect if user changed API key via web UI."""
    current_key = self._read_api_key_from_config()
    secret = self.model.get_secret(label=f"{self.app.name}-api-key")
    stored_key = secret.get_content()["api-key"]

    if current_key != stored_key:
        logger.warning("Credential drift detected - syncing")
        secret.set_content({"api-key": current_key})
```

### Root Folder Configuration

Opinionated, auto-configured:

```python
# Constants
_root_folder = "/data/media/movies"  # Radarr
# _root_folder = "/data/media/tv"    # Sonarr

def _ensure_root_folder_configured(self) -> None:
    api_key = self._get_api_key()

    response = requests.get(
        f"http://localhost:{self._service_port}/api/v3/rootfolder",
        headers={"X-Api-Key": api_key}
    )
    existing = [rf["path"] for rf in response.json()]

    if self._root_folder not in existing:
        requests.post(
            f"http://localhost:{self._service_port}/api/v3/rootfolder",
            headers={"X-Api-Key": api_key},
            json={"path": self._root_folder}
        )
```

### Recyclarr Integration

Per ADR apps/adr-003:

```python
def _sync_trash_profiles(self) -> None:
    profiles_config = self.config.get("trash-profiles", "").strip()
    if not profiles_config:
        return

    self._validate_trash_profiles(profiles_config)
    recyclarr_config = self._generate_recyclarr_config(profiles_config)

    config_path = "/tmp/recyclarr.yml"
    Path(config_path).write_text(recyclarr_config)

    result = subprocess.run(
        [str(Path(self.charm_dir) / "bin" / "recyclarr"), "sync", "--config", config_path],
        capture_output=True,
        text=True,
        timeout=120,
    )

    if result.returncode != 0:
        raise RecyclarrSyncError(result.stderr)

def _validate_trash_profiles(self, profiles_config: str) -> None:
    profiles = [p.strip().lower() for p in profiles_config.split(",")]

    indicators_1080p = {"1080p", "hd-", "hd_"}
    indicators_4k = {"2160p", "uhd-", "uhd_", "4k"}

    has_1080p = any(any(ind in p for ind in indicators_1080p) for p in profiles)
    has_4k = any(any(ind in p for ind in indicators_4k) for p in profiles)

    if has_1080p and has_4k:
        raise ValueError("Cannot mix 1080p and 4K profiles")
```

## charmcraft.yaml

```yaml
name: radarr-k8s
type: charm
title: Radarr
summary: Movie collection manager for Usenet and BitTorrent users
description: |
  Radarr is a movie collection manager for Usenet and BitTorrent users.

  This charm provides automated deployment on Kubernetes with:
  - Automatic indexer sync via Prowlarr integration
  - Automatic download client configuration
  - Trash Guides quality profiles via embedded Recyclarr
  - Shared media storage for hardlinks
  - Istio ingress integration

links:
  documentation: https://github.com/charmarr/radarr-k8s
  source: https://github.com/charmarr/radarr-k8s
  issues: https://github.com/charmarr/radarr-k8s/issues

assumes:
  - k8s-api
  - juju >= 3.6

platforms:
  amd64:
    - name: ubuntu
      channel: "24.04"

charm-libs:
  - lib: charms.istio_ingress_k8s.v0.istio_ingress_route

parts:
  charm:
    source: .
    plugin: uv
    build-packages: [git]
    build-snaps: [astral-uv]
    override-build: |
      craftctl default
      git describe --always > $CRAFT_PART_INSTALL/version

  recyclarr:
    plugin: dump
    source: https://github.com/recyclarr/recyclarr/releases/latest/download/recyclarr-linux-x64.tar.xz
    source-type: tar
    organize:
      recyclarr: bin/recyclarr

containers:
  radarr:
    resource: radarr-image

resources:
  radarr-image:
    type: oci-image
    description: OCI image for Radarr
    upstream-source: lscr.io/linuxserver/radarr:latest

storage:
  config:
    type: filesystem
    location: /config
    minimum-size: 1G

provides:
  media-manager:
    interface: media-manager

requires:
  media-indexer:
    interface: media-indexer
    limit: 1
  download-client:
    interface: download-client
  media-storage:
    interface: media-storage
    limit: 1
  ingress:
    interface: istio_ingress_route
    limit: 1

config:
  options:
    trash-profiles:
      type: string
      default: ""
      description: |
        Comma-separated list of Trash Guide profile templates to sync via Recyclarr.

        Leave empty to skip auto-configuration (manual profile management).

        Examples:
          - hd-bluray-web (1080p Bluray + WEB-DL)
          - uhd-bluray-web (4K Bluray + WEB-DL)
          - remux-web-1080p (1080p Remux + WEB-DL)
          - remux-web-2160p (4K Remux + WEB-DL)
          - anime (Anime-optimized)

        WARNING: Do not mix 1080p and 4K profiles. Deploy separate instances instead.

        See: https://trash-guides.info/Radarr/

    log-level:
      type: string
      default: "info"
      description: |
        Application log level.
        One of: trace, debug, info, warn, error

actions:
  sync-trash-profiles:
    description: |
      Manually sync quality profiles and custom formats from Trash Guides via Recyclarr.
      Requires trash-profiles config to be set.

  rotate-api-key:
    description: |
      Rotate the Radarr API key.
      Generates new key, restarts Radarr, updates Juju secret.
      Related applications update automatically via secret-changed event.
```

## Radarr vs Sonarr Differences

| Aspect | Radarr | Sonarr |
|--------|--------|--------|
| Charm name | `radarr-k8s` | `sonarr-k8s` |
| Container name | `radarr` | `sonarr` |
| Service port | 7878 | 8989 |
| Root folder | `/data/media/movies` | `/data/media/tv` |
| Manager enum | `MediaManager.RADARR` | `MediaManager.SONARR` |
| Trash profile examples | hd-bluray-web, uhd-bluray-web, anime | web-1080p, web-2160p, anime |
| Image | `lscr.io/linuxserver/radarr:latest` | `lscr.io/linuxserver/sonarr:latest` |
| Trash Guides URL | https://trash-guides.info/Radarr/ | https://trash-guides.info/Sonarr/ |

All other implementation (reconciler, relations, config, actions, storage, libraries) is identical.

## Deferred to v1.x/v2

- **Observability integrations:** grafana-dashboard, metrics-endpoint, logging relations
- **PostgreSQL support:** Native SQLite only in v1

## Consequences

### Good

* Single MADR covers both Radarr and Sonarr (95% identical)
* Reconciler pattern ensures idempotent, self-healing behavior
* Pebble checks provide robust readiness detection
* Auto-configuration minimizes user setup (root folder, external URL, download clients)
* Drift detection handles users changing credentials via web UI
* Recyclarr integration provides Trash Guides quality profiles out of the box
* Compliant with all existing MADRs (storage, interfaces, auth)

### Bad

* Requires `--trust` for RBAC permissions (StatefulSet patching)
* lightkube dependency for storage patching adds complexity
* Recyclarr runs every reconcile (acceptable cost for simplicity)
* No observability in v1 (deferred)

### Neutral

* LinuxServer images chosen over alternatives (well-maintained, matches legacy stack)
* Two config options only (minimal, opinionated)
* Two actions only (sync-trash-profiles, rotate-api-key)

## Related MADRs

- [storage/adr-001](storage/adr-001-shared-pvc-architecture.md) - Shared PVC architecture
- [storage/adr-003](storage/adr-003-pvc-patching-in-arr-charms.md) - StatefulSet patching via lightkube
- [storage/adr-004](storage/adr-004-config-storage.md) - Config storage (native SQLite)
- [interfaces/adr-003](interfaces/adr-003-media-indexer.md) - media-indexer interface
- [interfaces/adr-004](interfaces/adr-004-download-client.md) - download-client interface
- [interfaces/adr-005](interfaces/adr-005-media-storage.md) - media-storage interface
- [interfaces/adr-006](interfaces/adr-006-media-manager.md) - media-manager interface
- [apps/adr-002](apps/adr-002-cross-app-auth.md) - Authentication and credential management
- [apps/adr-003](apps/adr-003-recyclarr-integration.md) - Recyclarr integration
