# Media Storage Interface Design

## Context and Problem Statement

The media-storage interface connects the charmarr-storage charm (which creates and manages a shared PVC) with all media applications (Radarr, Sonarr, qBittorrent, Plex, Overseerr, etc.). We need to define data models and interface classes that enable all apps to mount the same shared PVC for hardlinks to work across the entire media stack, as required by Trash Guides best practices.

Key requirements:
- Single shared PVC mounted by all applications
- Hardlinks must work across all apps (Radarr, Sonarr, qBittorrent, Plex)
- Storage provider is completely passive (just creates PVC and publishes details)
- Apps need PVC name and mount path to patch their StatefulSets
- Provider doesn't need to react to requirers joining/leaving

## Considered Options

### Data Exchange Options
**Option 1: Include provisioning mode (NFS vs StorageClass)**
- Provider publishes how storage is provisioned
- Requirers see implementation details
- Extra information requirers don't need

**Option 2: Minimal - only PVC name and mount path**
- Provider publishes only what requirers need
- Provisioning mode is internal implementation detail
- Simpler data model

### Requirer Data Options
**Option 1: No requirer data**
- Requirers don't publish anything
- Provider can't track who's mounting

**Option 2: Publish instance_name only**
- Provider can track connected apps for metrics/logging
- Minimal overhead

### Event Handling Options
**Option 1: Both Provider and Requirer emit events**
- Symmetric design
- Provider doesn't need to react to anything

**Option 2: Only Requirer emits events**
- Provider is passive, just publishes data
- Requirer reacts when storage becomes available
- Simpler implementation

## Decision Outcome

**Data Exchange: Minimal (Option 2)**
**Requirer Data: instance_name only (Option 2)**
**Events: Only Requirer (Option 2)**

### Data Exchange Overview

```mermaid
graph LR
    P[Provider: charmarr-storage]
    R[Requirer: Radarr/Sonarr/qBittorrent/Plex/etc]

    P -->|pvc_name<br/>mount_path| R
    R -->|instance_name| P

    style P fill:#e1f5ff
    style R fill:#fff4e1
```

### Data Models (Pydantic 2.0)

```python
from pydantic import BaseModel, Field

class MediaStorageProviderData(BaseModel):
    """Data published by charmarr-storage."""
    pvc_name: str = Field(
        description="Name of the shared PVC to mount"
    )
    mount_path: str = Field(
        description="Mount path for the shared storage (e.g., /data)"
    )

class MediaStorageRequirerData(BaseModel):
    """Data published by apps mounting storage."""
    instance_name: str = Field(
        description="App instance name (for metrics/logging)"
    )
```

### Provider/Requirer Classes

```python
class MediaStorageProvider(Object):
    """Provider side - used by charmarr-storage charm.

    Note: No custom events - provider is completely passive.
    """

    def __init__(self, charm, relation_name: str = "media-storage"):
        super().__init__(charm, relation_name)
        # No event observation - provider just publishes data

    def publish_data(self, data: MediaStorageProviderData) -> None:
        """Publish provider data to all relations."""
        ...

class MediaStorageRequirerEvents(ObjectEvents):
    """Custom events for MediaStorageRequirer."""
    changed = EventSource(MediaStorageChangedEvent)

class MediaStorageRequirer(Object):
    """Requirer side - used by all media apps."""
    on = MediaStorageRequirerEvents()

    def __init__(self, charm, relation_name: str = "media-storage"):
        super().__init__(charm, relation_name)
        events = charm.on[relation_name]
        # Requirer observes relation events and emits custom event
        self.framework.observe(events.relation_changed, self._emit_changed)
        self.framework.observe(events.relation_broken, self._emit_changed)

    def publish_data(self, data: MediaStorageRequirerData) -> None:
        """Publish requirer data to relation."""
        ...

    def get_provider_data(self) -> Optional[MediaStorageProviderData]:
        """Get storage provider data if available.

        Note: Returns single provider data, not a list.
        There is only one storage provider (charmarr-storage).
        """
        ...

    def is_ready(self) -> bool:
        """Check if requirer has published data and provider is available."""
        ...
```

### Charm Usage

```python
# charmarr-storage charm (Provider - no event observation)
class CharmarrStorageCharm(CharmBase):
    def __init__(self, *args):
        super().__init__(*args)
        self.provider = MediaStorageProvider(self, "media-storage")
        # Observe own events, not relation events
        self.framework.observe(self.on.config_changed, self._reconcile)

    def _reconcile(self, event):
        # Check if PVC is created
        if not self._pvc_exists():
            self._create_pvc()

        # Publish storage details
        data = MediaStorageProviderData(
            pvc_name="charmarr-shared-storage",
            mount_path="/data",
        )
        self.provider.publish_data(data)

# Radarr charm (Requirer - observes storage availability)
class RadarrCharm(CharmBase):
    def __init__(self, *args):
        super().__init__(*args)
        self.storage_requirer = MediaStorageRequirer(self, "media-storage")
        self.framework.observe(self.storage_requirer.on.changed, self._reconcile)

    def _reconcile(self, event):
        if not self.storage_requirer.is_ready():
            self.unit.status = WaitingStatus("Waiting for shared storage")
            return

        # Get storage details
        provider = self.storage_requirer.get_provider_data()

        # Patch StatefulSet to add volume mount
        self.patch_statefulset_with_pvc(
            pvc_name=provider.pvc_name,
            mount_path=provider.mount_path,
        )
```

### Consequences

**Good:**
- Extremely simple data model (minimal necessary information)
- Provider is completely passive (no event handling overhead)
- Only requirer reacts to storage availability
- Single provider model (not a list) - clearer API than download-client
- Provisioning mode hidden from requirers (implementation detail)
- instance_name enables provider to track connected apps for metrics/status
- All apps mount same PVC at same path (enables hardlinks)

**Bad:**
- Requirers must use lightkube to patch their own StatefulSets
- No validation that apps actually mounted the storage
- Provider has no way to enforce which apps can mount (open to all)
- Breaking storage relation requires manual StatefulSet cleanup

## Implementation Notes

- Models and classes live in `charmarr-lib`, shared by all charms
- Provider observes NO relation events (completely passive)
- Requirer observes relation-changed and relation-broken, emits single `changed` event
- Only one storage provider exists (charmarr-storage charm)
- All requirers mount the same PVC at the same mount_path
- Requirers use lightkube to patch StatefulSet volumeMounts and volumes
- charmarr-storage charm creates PVC with ReadWriteMany access mode
- Provisioning mode (NFS vs StorageClass) is internal to charmarr-storage charm
- Hardlinks work because all apps see the same filesystem
