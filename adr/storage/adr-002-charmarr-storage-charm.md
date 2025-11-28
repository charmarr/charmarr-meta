# Storage Charm as PVC Lifecycle Manager

**Status:** Accepted

**Related ADRs:**
- [ADR-001: Shared PVC Architecture](adr-001-shared-pvc-architecture.md) - Establishes the need for a shared PVC
- [ADR-003: Storage Backends](adr-003-storage-backends.md) - Defines backend-specific management logic
- [ADR-004: PVC Patching in Arr Charms](adr-004-pvc-patching-in-arr-charms.md) - Defines how consuming charms receive and use PVC information

## Context and Problem Statement

With the decision to use a shared PVC for all arr applications (see [ADR-001](adr-001-shared-pvc-architecture.md)), we need to determine who is responsible for creating and managing this PVC's lifecycle, including initial provisioning, expansion, and configuration. The PVC needs to be created before any arr charms can mount it, and expansion operations need to be coordinated. Should this PVC be managed by infrastructure tooling (Terraform), by each consuming charm independently, or by a dedicated storage management charm?

## Considered Options

* Create PVC directly in Terraform and pass the name to charms via configuration
* Have each arr charm attempt to create/claim the shared PVC independently
* Create a dedicated charmarr-storage charm that owns and manages the PVC lifecycle
* Use Juju storage pools and let Juju handle PVC creation per standard patterns

## Decision Outcome

Chosen option: "Create a dedicated charmarr-storage charm that owns and manages the PVC lifecycle", because this keeps all operational concerns within the Juju model rather than split between Terraform and Juju. A storage charm can expose the PVC details through Juju relations, handle expansion through standard charm configuration, implement proper reconciliation patterns, and maintain observable state through Juju status. This provides a clean separation where the storage charm owns storage concerns and arr charms own application concerns.

### Implementation Details

**Charm Configuration:**

The storage charm exposes the following configuration options:
- `backend-type`: Required, either "local" or "nfs", determines which resources the charm manages
- `storage-class`: For local backend, which Kubernetes StorageClass to use (e.g., "topolvm-provisioner", "local-path")
- `nfs-server`: For NFS backend, the NFS server IP address or hostname
- `nfs-path`: For NFS backend, the export path on the NFS server
- `size`: Storage capacity to provision, semantic meaning differs by backend type (see [ADR-003](adr-003-storage-backends.md))

**Backend Selection Guidance:**

The choice of storage backend depends on your deployment scenario:

**Use local storage with simpler provisioners (local-path, hostPath) when:**
- Running single-node MicroK8s for homelab/testing
- You already have filesystem-level redundancy (ZFS, BTRFS, hardware RAID)
- You prioritize simplicity over enterprise features
- Your storage is managed at the filesystem layer, not the Kubernetes layer

**Use NFS storage when:**
- **Going multi-node** - NFS provides ReadWriteMany access so pods can be scheduled on any node
- You have a dedicated NAS appliance (Synology, TrueNAS, etc.) that handles redundancy
- Your media storage is already on a network file server
- You want storage to survive complete cluster rebuilds
- Network latency is acceptable for your workload (media files are large, not latency-sensitive)

**Use local storage with TopoLVM when:**
- You need ReadWriteOnce block storage distributed across multiple nodes (not applicable to Charmarr's shared media use case)
- You're running databases or other workloads that need node-local storage with capacity tracking
- You want LVM features like thin provisioning and snapshots across a cluster

**Important Note on Multi-Node Media Workloads:** For Charmarr specifically, if you go multi-node, **NFS is the right choice**, not TopoLVM. TopoLVM provides ReadWriteOnce access, which means all pods mounting the shared media PVC would still be stuck on one node anyway. NFS's ReadWriteMany support lets you truly distribute your arr application pods across nodes while still accessing the same shared media storage.

**Important Note on Redundancy:** Kubernetes and TopoLVM do NOT provide data redundancy. If you need redundancy (and you should for media libraries), implement it at the filesystem or hardware level:
- **ZFS** with mirrored or RAIDZ pools
- **BTRFS** with RAID1/RAID10
- **Hardware RAID** controllers
- **mdadm** software RAID on Linux
- **NAS appliances** with built-in redundancy (TrueNAS, Synology)

The Kubernetes layer (TopoLVM, local-path, etc.) sits on top of your redundant filesystem and provides provisioning, not protection.

**Reconciler Pattern:**

The charm implements a reconciler that continuously ensures actual state matches desired state. The reconciler is triggered by:
- Config-changed events when the user modifies charm configuration
- Update-status events for periodic health checks
- Relation events when consuming charms connect or disconnect

The reconciler workflow:
1. Check if required Kubernetes resources exist (PVC for local, PV and PVC for NFS)
2. Compare current resource specifications with desired configuration
3. Create missing resources or patch existing ones to match desired state
4. Watch for resource conditions indicating success or failure
5. Update charm status and relation data accordingly

**Reactive Error Handling:**

The charm uses a reactive rather than proactive validation approach. Instead of attempting to verify that the underlying storage backend has sufficient capacity before requesting resources, the charm requests what the user configured and handles failures when they occur.

For local storage, if the LVM volume group lacks sufficient space, the TopoLVM CSI driver will fail the expansion and report this through PVC conditions. The charm reconciler detects this failure condition and transitions to error status with the message from the CSI driver.

For NFS storage, the Kubernetes-level updates succeed since they're just metadata, but applications may encounter disk full errors when attempting to use the space. The charm relies on observability integration with the COS stack to monitor actual filesystem usage and alert when capacity is reached.

This reactive approach is chosen because proactive validation would require the charm to inspect infrastructure details (LVM volume groups, NFS server capacity) that are outside its proper scope and would require additional permissions and complexity. Users are expected to ensure their infrastructure has sufficient capacity before requesting expansions.

**Charm Status State Machine:**

- **Maintenance**: During initial setup or expansion operations while resources are being created/updated
- **Active**: Resources are created, bound, and ready for use; relation data is current
- **Error**: Resource creation or expansion failed (e.g., insufficient storage capacity, invalid configuration)
- **Blocked**: Cannot proceed due to missing required configuration (e.g., backend-type not set)

**Relation Interface:**

The charm provides a relation endpoint called `shared-storage` that exposes:
```python
{
    "pvc-name": "charmarr-shared-media",
    "size": "2Ti",
    "mount-path": "/data",
    "backend-type": "local"  # or "nfs"
}
```

When the storage size is expanded, the charm updates this relation data, triggering relation-changed events in all connected consuming charms. Consuming charms don't need to take action on size changes since their existing volume mounts automatically reflect the expanded capacity, but they can use this information to update status displays or trigger other logic.

### Consequences

* Good, because storage lifecycle is managed through standard Juju patterns (deploy, configure, relate)
* Good, because expansion is triggered through charm configuration rather than manual Terraform reapply or kubectl commands
* Good, because storage state is visible through Juju status and relations rather than requiring kubectl inspection
* Good, because the storage charm can implement reconciliation logic to handle failures and ensure desired state
* Good, because relation data provides a clean interface for arr charms to discover PVC name and configuration
* Good, because reactive error handling keeps the charm implementation simple and focused
* Good, because charm status clearly communicates the state of storage operations to users
* Bad, because adds one more charm to deploy compared to having Terraform create the PVC directly
* Bad, because storage charm needs backend-specific logic for different storage types (local vs NFS)
* Bad, because reactive error handling means users may not discover capacity issues until after requesting expansion
* Bad, because the charm requires RBAC permissions to create and modify cluster-level resources like PersistentVolumes
