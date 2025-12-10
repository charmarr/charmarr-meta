# Recyclarr Integration in Arr Charms for Automated Quality Profile Management

## Context and Problem Statement

Charmarr aims to provide better automation than Docker Compose setups. A key pain point in media management is configuring quality profiles and custom formats in Radarr/Sonarr to automatically select high-quality releases while avoiding low-quality ones. Trash Guides provides community-maintained best practices, and Recyclarr syncs these configurations via API calls.

**Key questions:**
- Should Recyclarr be embedded in arr charms, run as a separate charm, or left to manual user configuration?
- How should quality profiles be configured - via Juju config, web UI, or automation?
- When should Recyclarr sync run - on startup, on schedule, or manually?

## Considered Options

* **Option A:** Embed Recyclarr in arr charms - Bundle binary, run during reconciliation
* **Option B:** Separate Recyclarr charm - Deploy as CronJob, relate to arr apps
* **Option C:** Manual user configuration - Users run Recyclarr themselves or configure profiles manually

## Decision Outcome

Chosen option: **Option A - Embed Recyclarr in arr charms**, because it provides massive UX improvement with trivial implementation complexity (~100 lines of code) while remaining optional for users who want manual control.

### Rationale

**Why embed (not separate charm):**
- Recyclarr is a simple CLI tool, not a complex service
- Separate charm adds maintenance burden, relation design, duplicate credentials
- Embedding is ~100 lines vs ~500+ lines for separate charm
- No architectural complexity - just a subprocess call

**Why automate (not manual):**
- This is a key differentiator vs Docker Compose (batteries included)
- Quality profiles are complex - Trash Guides community expertise valuable
- Optional feature - users can disable via config for manual control
- Profiles stay internal to each arr instance (no interface pollution)

**When Recyclarr Runs:**

1. **Every reconciliation** if `trash-profiles` config is set (idempotent operation)
   - This includes: install, upgrade, config-changed, relation changes, secret changes
   - Recyclarr is fast (~2-5 sec) and idempotent, so safe to run frequently
   - Ensures profiles stay in sync even if manually deleted via web UI
   - Profile data is published to relations after each sync

2. **Via manual action** (`juju run radarr sync-trash-profiles`)
   - Use when Trash Guides update their recommendations
   - Forces immediate sync without waiting for next reconcile event

3. **NOT on schedule** - No background cron job
   - Reduces unnecessary API calls to Radarr
   - User controls when updates happen via action or by triggering reconcile

**Rationale for "every reconcile"**: Juju spawns fresh Python process per event,
so state tracking ("has run once") is impossible without storing state in Juju
peer relation. Running on every reconcile is simpler than managing persistent state,
and the performance cost is negligible (~2-5 seconds per reconcile).

### Consequences

* Good: Massive UX win - auto-configured Trash Guides profiles
* Good: Trivial implementation - just run a binary, ~100 lines total
* Good: Optional - disable via config for manual control
* Good: No interface pollution - quality profiles stay internal to arr instance
* Good: Fits "for the love of the game" philosophy
* Bad: Charm package ~20MB larger (includes binary)
* Bad: Need to update charm to get new Recyclarr version (acceptable)
* Neutral: First profile becomes default in Overseerr (arbitrary but can be changed)

## Implementation Design

### Charm Configuration

```yaml
# Radarr charm config.yaml
trash-profiles:
  type: string
  default: ""
  description: |
    Comma-separated list of Trash Guide profile templates to sync.
    Examples: "hd-bluray-web", "uhd-bluray-web", "remux-web-1080p", "anime"
    
    Leave empty to skip auto-configuration (manual profile management).
    Profiles are synced on startup. Use 'sync-trash-profiles' action to re-sync.
    
    See: https://trash-guides.info/Radarr/
    
    Available templates:
    - hd-bluray-web: 1080p Bluray + WEB-DL
    - uhd-bluray-web: 4K Bluray + WEB-DL  
    - remux-web-1080p: 1080p Remux + WEB-DL
    - remux-web-2160p: 4K Remux + WEB-DL
    - anime: Anime-optimized profiles
```

```yaml
# actions.yaml
sync-trash-profiles:
  description: |
    Manually sync quality profiles and custom formats from Trash Guides.
    Run this when Trash Guides update their recommendations.
```

### Charmcraft Parts (Binary Download)

```yaml
# charmcraft.yaml
parts:
  recyclarr:
    plugin: dump
    source: https://github.com/recyclarr/recyclarr/releases/download/v7.4.0/recyclarr-linux-x64.tar.xz
    override-pull: |
      curl -L https://github.com/recyclarr/recyclarr/releases/latest/download/recyclarr-linux-x64.tar.xz \
        -o recyclarr.tar.xz
      tar -xf recyclarr.tar.xz
      rm recyclarr.tar.xz
    organize:
      recyclarr: bin/recyclarr
```

### Charm Implementation

```python
class RadarrCharm(CharmBase):
    def __init__(self, *args):
        super().__init__(*args)
        # ... other initialization ...
        self.framework.observe(self.on.sync_trash_profiles_action, self._on_sync_trash_profiles)
    
    def _reconcile(self, event):
        """Main reconciler - handles all events."""
        # ... other reconciliation logic ...
        
        # Sync profiles if configured
        trash_profiles = self.config.get("trash-profiles", "").strip()
        if trash_profiles:
            self._sync_trash_profiles(trash_profiles)
        
        # Publish quality profiles to media-manager relation
        self._publish_quality_profiles()
    
    def _sync_trash_profiles(self, profiles_config: str):
        """Run Recyclarr to sync Trash Guides profiles."""
        # Generate Recyclarr config
        profiles_list = [p.strip() for p in profiles_config.split(",") if p.strip()]
        recyclarr_config = self._generate_recyclarr_config(profiles_list)
        
        # Write config to temp file
        config_path = "/tmp/recyclarr.yml"
        Path(config_path).write_text(recyclarr_config)
        
        # Run recyclarr (binary in charm container, API calls to localhost)
        recyclarr_bin = Path(self.charm_dir) / "bin" / "recyclarr"
        result = subprocess.run(
            [str(recyclarr_bin), "sync", "--config", config_path],
            capture_output=True,
            text=True,
            timeout=120,
        )
        
        if result.returncode != 0:
            logger.error(f"Recyclarr sync failed: {result.stderr}")
            raise Exception(f"Recyclarr sync failed: {result.stderr}")
        
        logger.info(f"Recyclarr sync successful: {result.stdout}")
    
    def _generate_recyclarr_config(self, profiles: list[str]) -> str:
        """Generate Recyclarr YAML config from profile list."""
        # Recyclarr calls Radarr at localhost:7878 (same pod)
        return f"""
radarr:
  radarr:
    base_url: http://localhost:7878
    api_key: {self.radarr_api_key}
    
    quality_profiles:
      - trash_ids:
{chr(10).join(f"          - {profile}" for profile in profiles)}
"""
    
    def _publish_quality_profiles(self):
        """Query Radarr API and publish profiles to relation."""
        # Query Radarr for current profiles
        response = requests.get(
            "http://localhost:7878/api/v3/qualityprofile",
            headers={"X-Api-Key": self.radarr_api_key}
        )
        profiles = [
            QualityProfile(id=p["id"], name=p["name"])
            for p in response.json()
        ]
        
        # Update relation data
        provider_data = MediaManagerProviderData(
            quality_profiles=profiles,
            # ... other fields ...
        )
        self.media_manager_provider.publish_data(provider_data)
    
    def _on_sync_trash_profiles(self, event):
        """Action handler for manual profile sync."""
        trash_profiles = self.config.get("trash-profiles", "").strip()
        if not trash_profiles:
            event.fail("trash-profiles config is not set")
            return
        
        try:
            self._sync_trash_profiles(trash_profiles)
            self._publish_quality_profiles()
            event.set_results({"result": "Profiles synced successfully"})
        except Exception as e:
            event.fail(str(e))
```

## User Experience

```bash
# Deploy with Trash Guides profiles
juju deploy radarr radarr-1080p --config trash-profiles="hd-bluray-web"
juju deploy radarr radarr-4k --config trash-profiles="uhd-bluray-web"

# Profiles auto-created on startup:
# - radarr-1080p: "HD-Bluray+WEB" profile created
# - radarr-4k: "UHD-Bluray+WEB" profile created

# User can add additional profiles via web UI (charm doesn't interfere)

# Later, Trash Guides update their recommendations:
juju run radarr-1080p sync-trash-profiles

# Profiles updated, relation-changed triggers Overseerr to pick up changes
```

## Quality Profile Validation

**Resolution mixing prevention:**

```python
def _validate_trash_profiles(self, profiles_config: str) -> None:
    """Ensure all profiles are same resolution (prevent mixing 1080p and 4K)."""
    profiles_list = [p.strip() for p in profiles_config.split(",")]
    
    has_1080p = any("1080p" in p or "hd-" in p for p in profiles_list)
    has_4k = any("2160p" in p or "uhd-" in p or "4k" in p for p in profiles_list)
    
    if has_1080p and has_4k:
        raise ValueError(
            "Cannot mix 1080p and 4K profiles in same instance. "
            "Deploy separate instances: radarr-1080p and radarr-4k"
        )
```

## Available Trash Guides Templates

**Radarr:**
- `hd-bluray-web` - 1080p Bluray + WEB-DL (most common)
- `uhd-bluray-web` - 4K Bluray + WEB-DL (4K instance)
- `remux-web-1080p` - 1080p Remux + WEB-DL (high quality)
- `remux-web-2160p` - 4K Remux + WEB-DL (highest quality)
- `anime` - Anime-optimized with group tiers

**Sonarr:**
- `web-1080p` - 1080p WEB-DL
- `web-2160p` - 4K WEB-DL  
- `anime` - Anime-optimized

**Profile naming:**
Templates create profiles with standardized Trash Guides names:
- `hd-bluray-web` → "HD-Bluray+WEB"
- `uhd-bluray-web` → "UHD-Bluray+WEB"
- `remux-web-1080p` → "Remux-1080p"

## Integration with Media-Manager Interface

After Recyclarr creates profiles in Radarr:

```
1. Recyclarr creates profiles in Radarr
   └─> Radarr API: POST /api/v3/qualityprofile

2. Radarr charm queries profiles  
   └─> Radarr API: GET /api/v3/qualityprofile
   
3. Radarr charm publishes to relation
   └─> See [Interfaces ADR 006](../interfaces/adr-006-media-manager.md) for how profiles flow to Overseerr
```

## Key Technical Details

**Why Recyclarr runs in charm container (not workload):**
- Charm and workload share same Kubernetes pod
- Both can access `localhost:7878` (Radarr API)
- No need to push binary into workload container
- Cleaner separation: charm handles automation, workload runs service

**Binary location:**
- Downloaded during charm build via Charmcraft parts
- Located at: `{charm_dir}/bin/recyclarr`
- Executable directly from charm code

**API access:**
- Recyclarr calls: `http://localhost:7878/api/v3/*`
- Same pod networking enables this
- Uses Radarr API key from charm config

## Implementation Complexity

```
Total implementation: ~100 lines of code
- Charmcraft parts config: ~10 lines
- Config YAML generation: ~20 lines
- Subprocess execution: ~15 lines
- Error handling: ~15 lines
- Profile publishing: ~20 lines
- Action handler: ~20 lines
```

Compare to alternatives:
- Separate charm: ~500+ lines + relation interface design
- Manual configuration: Documentation burden, user toil

## Related Decisions

- [Interfaces ADR 006](../interfaces/adr-006-media-manager.md) - How quality profiles flow from Radarr to Overseerr via relation data, including query-first default selection logic
