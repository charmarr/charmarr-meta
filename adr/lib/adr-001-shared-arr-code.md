# Shared Arr Infrastructure in charmarr-lib

## Context and Problem Statement

Charmarr's arr charms (Radarr, Sonarr, Lidarr) need to configure their workloads via API calls after receiving relation data. These applications share nearly identical APIs (`/api/v3/*` endpoints for download clients, root folders, quality profiles, etc.). Without shared infrastructure, we'd duplicate significant code across 3+ charms for HTTP handling, config transformation, and reconciliation logic.

**Key questions:**
- How should we structure shared code for arr API interactions?
- How should relation data be transformed into workload API payloads?
- What reconciliation semantics should we use (additive vs declarative)?
- How do we handle manually-configured items that weren't created via Juju relations?

## Considered Options

### Library Structure
* **Option 1:** Single monolithic `ArrApiClient` with all logic
* **Option 2:** Separate concerns: API client + reconcilers + config builders
* **Option 3:** No shared code - each charm implements its own

### Reconciliation Semantics
* **Option 1:** Additive only - add/update items from relations, never delete
* **Option 2:** Aggressive declarative - delete anything not in relations
* **Option 3:** Managed vs unmanaged - track what we created, only delete those

### Config Building Location
* **Option 1:** Inside ArrApiClient methods
* **Option 2:** Separate config builder classes
* **Option 3:** Inside reconciler functions

## Decision Outcome

**Library Structure: Option 2 - Separate concerns**
**Reconciliation: Option 2 - Aggressive declarative**
**Config Building: Option 2 - Separate config builder classes**

### Rationale

**Separate concerns** because:
- `ArrApiClient` stays reusable (even for non-Charmarr projects)
- Reconcilers contain Charmarr-specific business logic
- Config builders handle the messy transformation logic
- Clear boundaries, easier to test each layer

**Aggressive declarative reconciliation** because:
- Juju relations become single source of truth
- No drift, no orphaned configs accumulating
- Stateless - no need to track "items I've created"
- Aligns with Charmarr's fully-automated philosophy

**Separate config builders** because:
- Keeps reconcilers focused on orchestration logic
- Transformation logic is complex (secret retrieval, URL parsing, API format)
- Enables reuse if same config format needed elsewhere

### Why NOT Managed vs Unmanaged?

We considered only deleting items that Charmarr created while leaving user-managed items alone. **This is fundamentally broken without state:**

1. Deploy qbittorrent-k8s → relate to radarr → "qbittorrent" DC created
2. User manually adds "my-seedbox" via web UI
3. Later: `juju remove-application qbittorrent-k8s`
4. Next reconcile sees two DCs, neither in relation data
5. **Impossible to distinguish** orphaned Charmarr DC from user-managed DC

Without tracking "things I've ever created" (which requires state), we can't implement this safely. The aggressive approach is the only clean stateless option.

**User expectation:** All workload configuration is managed via Juju relations. Manual additions will be removed on next reconciliation. Users who need additional download clients should charm them.

## Library Structure

```
charmarr-lib/
├── interfaces/
│   ├── media_indexer.py        # Relation interface classes + data models
│   ├── download_client.py      # + MEDIA_TYPE_DOWNLOAD_PATHS constant
│   ├── media_manager.py
│   ├── media_storage.py
│   └── vpn_gateway.py
├── arr/
│   ├── api_client.py           # ArrApiClient - clean HTTP wrapper
│   ├── reconcilers.py          # Shared reconciler functions
│   └── config_builders.py      # Transform relation data → API payloads
└── reconciler.py               # observe_events utilities
```

### Why Arr-Specific Code in Shared Lib?

Normally, app-specific code belongs in the charm, not the shared library. We make an exception for arr infrastructure because:

- Radarr, Sonarr, Lidarr share ~95% identical APIs
- Same reconciliation patterns apply to all three
- Significant code savings (hundreds of lines)
- Well-defined, stable API contract (Servarr project)

**Counter-example:** Overseerr's API is unique and only used by the Overseerr charm. Its API client stays in the charm, NOT in charmarr-lib.

**Rule:** Only add to charmarr-lib if code is actually reused across multiple charms.

## Shared Constants

### MEDIA_TYPE_DOWNLOAD_PATHS

This constant maps media manager types to their download folder names. It's used by:
- **Download client charms** (qBittorrent, SABnzbd) - to create categories with correct save paths
- **Arr charms** - to know where downloads will land (for import path configuration)

```python
# In charmarr_lib/interfaces/download_client.py

from enum import Enum

class MediaManager(str, Enum):
    """Media manager applications."""
    RADARR = "radarr"
    SONARR = "sonarr"
    LIDARR = "lidarr"
    READARR = "readarr"
    WHISPARR = "whisparr"

MEDIA_TYPE_DOWNLOAD_PATHS: dict[MediaManager, str] = {
    MediaManager.RADARR: "movies",
    MediaManager.SONARR: "tv",
    MediaManager.LIDARR: "music",
    MediaManager.READARR: "books",
    MediaManager.WHISPARR: "xxx",
}
```

**Usage in download client charms:**

```python
from charmarr_lib.interfaces.download_client import (
    MEDIA_TYPE_DOWNLOAD_PATHS,
    MediaManager,
)

def _get_category_path(self, manager: MediaManager) -> str:
    """Get save path for a media manager's category."""
    type_folder = MEDIA_TYPE_DOWNLOAD_PATHS.get(manager, "other")

    # qBittorrent: absolute path
    return f"/data/torrents/{type_folder}"

    # SABnzbd: relative path (in separate method)
    # return type_folder
```

**Why in charmarr-lib, not duplicated:**
- Single source of truth for folder naming convention
- Ensures Trash Guides compliance across all charms
- Adding a new media type (e.g., Whisparr) updates all charms automatically

**Why NOT a full shared abstraction for download clients:**
- qBittorrent and SABnzbd have completely different APIs (~20% similar vs arr's ~95%)
- Category creation logic differs (absolute vs relative paths, different endpoints)
- Only the constant is truly shared; forcing more would create a leaky abstraction

## ArrApiClient

Clean HTTP wrapper with typed methods. **No business logic** - just thin wrappers around API endpoints.

```python
class ArrApiClient:
    """Generic API client for Radarr/Sonarr/Lidarr.

    Handles HTTP mechanics only. No Juju awareness, no config transformation.
    """

    def __init__(self, base_url: str, api_key: str):
        self._base_url = base_url.rstrip("/")
        self._api_key = api_key
        self._session = requests.Session()
        self._session.headers["X-Api-Key"] = api_key

    # Download Clients
    def get_download_clients(self) -> list[dict]: ...
    def get_download_client_by_name(self, name: str) -> dict | None: ...
    def add_download_client(self, config: dict) -> dict: ...
    def update_download_client(self, client_id: int, config: dict) -> dict: ...
    def delete_download_client(self, client_id: int) -> None: ...

    # Root Folders
    def get_root_folders(self) -> list[dict]: ...
    def add_root_folder(self, path: str) -> dict: ...

    # Host Config (for external URL)
    def get_host_config(self) -> dict: ...
    def update_host_config(self, config: dict) -> dict: ...

    # Quality Profiles (read-only for media-manager relation)
    def get_quality_profiles(self) -> list[dict]: ...

    # Indexers (for media-indexer integration)
    def get_indexers(self) -> list[dict]: ...
    def add_indexer(self, config: dict) -> dict: ...
    def update_indexer(self, indexer_id: int, config: dict) -> dict: ...
    def delete_indexer(self, indexer_id: int) -> None: ...
```

**Design principles:**
- Returns raw dicts (API response format)
- Raises exceptions on HTTP errors
- No retry logic (let caller handle)
- No caching (stateless)

## Shared Reconcilers

Declarative, idempotent operations. Each reconciler follows the pattern:
1. Get current state from API
2. Compare against desired state
3. Delete removed, update changed, add new

### reconcile_download_clients

```python
def reconcile_download_clients(
    api_client: ArrApiClient,
    desired_clients: list[DownloadClientProviderData],
    category: str,  # "radarr", "sonarr", "lidarr"
    model: Model,   # For Juju secret retrieval
) -> None:
    """Reconcile download clients to match desired state from relations.

    IMPORTANT: This is aggressive reconciliation. Any download client
    not in desired_clients will be DELETED, including manually-configured
    clients added via the web UI.

    Args:
        api_client: Configured ArrApiClient instance
        desired_clients: ALL DownloadClientProviderData from relations
        category: Download category for this arr app
        model: Juju Model for secret access
    """
    # 1. Get current state
    current = api_client.get_download_clients()
    current_by_name = {dc["name"]: dc for dc in current}

    # 2. Build desired configs
    desired_configs = {}
    for provider in desired_clients:
        config = DownloadClientConfig.from_provider_data(
            provider=provider,
            category=category,
            model=model,
        )
        desired_configs[provider.instance_name] = config

    # 3. Delete removed (AGGRESSIVE - deletes manual configs too!)
    for name, current_dc in current_by_name.items():
        if name not in desired_configs:
            logger.info(f"Removing download client: {name}")
            api_client.delete_download_client(current_dc["id"])

    # 4. Add or update
    for name, desired_config in desired_configs.items():
        existing = current_by_name.get(name)
        if existing:
            if _needs_update(existing, desired_config):
                logger.info(f"Updating download client: {name}")
                api_client.update_download_client(existing["id"], desired_config)
        else:
            logger.info(f"Adding download client: {name}")
            api_client.add_download_client(desired_config)
```

### reconcile_root_folder

```python
def reconcile_root_folder(
    api_client: ArrApiClient,
    path: str,
) -> None:
    """Ensure root folder exists. Idempotent.

    Note: Does NOT delete other root folders. Root folders are
    additive-only because users may have valid reasons for multiple.
    """
    existing = api_client.get_root_folders()
    if not any(rf["path"] == path for rf in existing):
        logger.info(f"Adding root folder: {path}")
        api_client.add_root_folder(path)
```

### reconcile_external_url

```python
def reconcile_external_url(
    api_client: ArrApiClient,
    external_url: str,
) -> None:
    """Configure external URL in host config. Idempotent."""
    current = api_client.get_host_config()

    # Parse URL to extract components
    parsed = urlparse(external_url)

    updates = {}
    if current.get("urlBase") != parsed.path:
        updates["urlBase"] = parsed.path
    # ... other host config fields if needed

    if updates:
        logger.info(f"Updating external URL config: {external_url}")
        api_client.update_host_config({**current, **updates})
```

### reconcile_indexers

```python
def reconcile_indexers(
    api_client: ArrApiClient,
    desired_indexers: list[IndexerConfig],  # Built from Prowlarr sync
) -> None:
    """Reconcile indexers. Aggressive - removes non-managed indexers.

    Note: Indexers typically come from Prowlarr sync, not direct relation.
    This handles the arr-side cleanup when indexers are removed from Prowlarr.
    """
    # Same pattern as download_clients
    ...
```

## Config Builders

Transform clean relation data models into messy workload API payloads. Handle all the complexity: secret retrieval, URL parsing, API format quirks.

```python
class DownloadClientConfig:
    """Build download client API payloads from relation data."""

    @staticmethod
    def from_provider_data(
        provider: DownloadClientProviderData,
        category: str,
        model: Model,
    ) -> dict:
        """Transform relation data into *arr API payload.

        Handles:
        - Secret retrieval from Juju
        - URL parsing (host, port extraction)
        - Client-specific field mapping (qBittorrent vs SABnzbd)
        - API payload format (fields array, implementation name, etc.)
        """
        if provider.client == DownloadClient.QBITTORRENT:
            return DownloadClientConfig._build_qbittorrent(provider, category, model)
        elif provider.client == DownloadClient.SABNZBD:
            return DownloadClientConfig._build_sabnzbd(provider, category, model)
        # ... other clients

    @staticmethod
    def _build_qbittorrent(
        provider: DownloadClientProviderData,
        category: str,
        model: Model,
    ) -> dict:
        # Retrieve secrets
        username = model.get_secret(id=provider.username_secret_id).get_content()["username"]
        password = model.get_secret(id=provider.password_secret_id).get_content()["password"]

        # Parse URL
        parsed = urlparse(str(provider.api_url))
        host = parsed.hostname
        port = parsed.port or 8080
        use_ssl = parsed.scheme == "https"

        # Build API payload
        return {
            "enable": True,
            "protocol": "torrent",
            "priority": 1,
            "name": provider.instance_name,
            "implementation": "QBittorrent",
            "configContract": "QBittorrentSettings",
            "fields": [
                {"name": "host", "value": host},
                {"name": "port", "value": port},
                {"name": "useSsl", "value": use_ssl},
                {"name": "urlBase", "value": provider.base_path or ""},
                {"name": "username", "value": username},
                {"name": "password", "value": password},
                {"name": "movieCategory", "value": category},
                {"name": "movieImportedCategory", "value": ""},
                {"name": "recentMoviePriority", "value": 0},
                {"name": "olderMoviePriority", "value": 0},
                {"name": "initialState", "value": 0},
                {"name": "sequentialOrder", "value": False},
                {"name": "firstAndLast", "value": False},
            ],
            "tags": [],
        }

    @staticmethod
    def _build_sabnzbd(
        provider: DownloadClientProviderData,
        category: str,
        model: Model,
    ) -> dict:
        # Retrieve API key
        api_key = model.get_secret(id=provider.api_key_secret_id).get_content()["api-key"]

        # Parse URL
        parsed = urlparse(str(provider.api_url))

        return {
            "enable": True,
            "protocol": "usenet",
            "priority": 1,
            "name": provider.instance_name,
            "implementation": "Sabnzbd",
            "configContract": "SabnzbdSettings",
            "fields": [
                {"name": "host", "value": parsed.hostname},
                {"name": "port", "value": parsed.port or 8080},
                {"name": "useSsl", "value": parsed.scheme == "https"},
                {"name": "urlBase", "value": provider.base_path or ""},
                {"name": "apiKey", "value": api_key},
                {"name": "movieCategory", "value": category},
                {"name": "recentMoviePriority", "value": -100},
                {"name": "olderMoviePriority", "value": -100},
            ],
            "tags": [],
        }
```

## Charm Usage

With this infrastructure, charm reconcilers become clean and declarative:

```python
class RadarrCharm(CharmBase):
    _root_folder = "/data/media/movies"
    _category = "radarr"

    def __init__(self, framework: ops.Framework):
        super().__init__(framework)

        # Interfaces from charmarr-lib
        self.download_clients = DownloadClientRequirer(self, "download-client")
        self.storage = MediaStorageRequirer(self, "media-storage")

        # API client created when workload ready
        self._api_client: ArrApiClient | None = None

        # Event observation
        observe_events(self, reconcilable_events_k8s, self._reconcile)
        self.framework.observe(self.download_clients.on.changed, self._reconcile)
        # ...

    def _reconcile(self, event: ops.EventBase) -> None:
        # ... container ready, storage mounted, workload ready checks ...

        # Initialize API client if needed
        if not self._api_client:
            self._api_client = ArrApiClient(
                base_url=f"http://localhost:{self._service_port}",
                api_key=self._get_api_key(),
            )

        # Workload configuration - clean, declarative calls
        reconcile_root_folder(self._api_client, self._root_folder)

        if self._ingress.is_ready():
            external_url = self._get_external_url_from_ingress()
            reconcile_external_url(self._api_client, external_url)

        # Download clients from relations - pass ALL, reconciler handles diff
        providers = self.download_clients.get_providers()
        reconcile_download_clients(
            api_client=self._api_client,
            desired_clients=providers,
            category=self._category,
            model=self.model,
        )

        # Publish to outgoing relations
        self._publish_media_manager_data()
```

## Bulk API Endpoints

Radarr/Sonarr have bulk endpoints:
- `PUT /api/v3/downloadclient/bulk` - apply same change to multiple items
- `DELETE /api/v3/downloadclient/bulk` - delete multiple by ID list

**Finding:** These are for "apply same change to many items" (e.g., add tag to all), NOT for reconciliation where each item differs. **Not useful for our use case.** Stick with individual API calls.

## Consequences

### Good

* **Clean separation**: API client reusable, reconcilers contain business logic, config builders handle transformation
* **Stateless**: No tracking of "items I created" - relations are single source of truth
* **Idempotent**: Safe to call reconcilers repeatedly
* **Declarative**: Charm says "make it this way", reconciler figures out how
* **Testable**: Each layer can be unit tested independently
* **Reusable**: ArrApiClient works for any arr app, even outside Charmarr
* **DRY**: Hundreds of lines shared across Radarr/Sonarr/Lidarr
* **Shared constants**: `MEDIA_TYPE_DOWNLOAD_PATHS` ensures consistent folder naming

### Bad

* **Aggressive deletion**: Users lose manually-configured items on reconcile
* **No escape hatch**: Can't mix Charmarr-managed + manual configs (except by not relating)
* **Library coupling**: Charms depend on charmarr-lib for core functionality
* **API format dependency**: Config builders must track arr API changes

### Neutral

* **Documentation requirement**: Must clearly document aggressive reconciliation behavior
* **Arr-specific code in lib**: Exception to "charm code stays in charm" rule, justified by reuse
* **Download client constants only**: Full abstraction not warranted for qBit/SAB due to API differences

## Documentation Requirements

### User-Facing

> **⚠️ Important: Managed Configuration**
>
> Charmarr manages all download client, indexer, and related configuration via Juju relations.
>
> **Any manually-added configuration (via web UI) will be removed** on the next reconciliation cycle. This includes download clients, indexers, and other settings managed by Charmarr relations.
>
> If you need additional download clients, deploy them as Juju charms and create relations. Do not add them manually via the web UI.

## Related MADRs

- [apps/adr-004-radarr-sonarr](../apps/adr-004-radarr-sonarr.md) - Media manager charm implementation
- [apps/adr-007-download-clients](../apps/adr-007-download-clients.md) - Download client charm implementation
- [interfaces/adr-004-download-client](../interfaces/adr-004-download-client.md) - Download client interface data models
- [interfaces/adr-003-media-indexer](../interfaces/adr-003-media-indexer.md) - Media indexer interface data models
- [apps/adr-002-cross-app-auth](../apps/adr-002-cross-app-auth.md) - Credential management and secret handling
