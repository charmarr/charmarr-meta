# Download Client VPN Integration Method

**Status:** Accepted

## Context and Problem Statement

Download client pods need to be configured to route their traffic through the VPN gateway via VXLAN. This requires injecting init containers, sidecars, adding NET_ADMIN capability, and configuring routing rules. How do we automate this configuration in a way that aligns with Juju's charm-based orchestration model?

## Considered Options

* **Mutating webhook (admission controller)** - Kubernetes webhook automatically injects containers when pods are created
* **Juju relations + lightkube StatefulSet patching** - Download client charms relate to gluetun, then patch their own StatefulSet
* **VPN charm patches consumers** - gluetun-k8s charm patches consumer StatefulSets
* **Manual pod labels + external controller** - Label pods, separate controller watches and patches

## Decision Outcome

Chosen option: **"Juju relations + lightkube StatefulSet patching"**, because it leverages Juju's existing relation system to communicate configuration, keeps modification logic in the consuming charms (single responsibility), avoids webhook infrastructure complexity (certificates, admission controller deployment), maintains clean resource ownership (each charm manages its own resources), and makes VPN routing topology visible via `juju status --relations`.

### Implementation Approach

Consumer charms use `VPNGatewayRequirer` from vpn-k8s-lib, which provides a `reconcile()` method that handles StatefulSet patching. The library reads `routing_method` from relation data and applies the appropriate patching strategy (currently pod-gateway).

The library uses **strategic merge patch** to add containers without replacing Juju's existing containers. This is the same pattern validated for storage patching.

**Resource ownership principle**: Each charm patches its own resources. The gluetun-k8s charm patches itself with gateway containers; consumer charms patch themselves with client containers. Neither modifies resources it doesn't own.

### What Gets Patched (Consumer Side)

When a download client relates to gluetun-k8s, vpn-k8s-lib patches the consumer's StatefulSet to add:

1. **Init container** (`pod-gateway:client_init.sh`):
   - Creates VXLAN interface pointing to gateway pod
   - Gets IP via DHCP from gateway
   - Modifies routing table (default route via VXLAN, except cluster CIDRs)
   - Requires `NET_ADMIN` capability

2. **Sidecar container** (`pod-gateway:client_sidecar.sh`):
   - Monitors gateway connectivity via periodic pings
   - Detects when gateway pod restarts (gets new K8s IP)
   - Resets VXLAN to discover new gateway IP
   - ~3MB RAM overhead

3. **NetworkPolicy** (kill switch):
   - Blocks all egress except cluster CIDRs
   - If VXLAN routing fails, traffic blocked at K8s level
   - Created as separate resource, owned by consumer charm

### Why Consumer Self-Patching (Not Webhook)

**Mutating webhook would require:**
- Webhook server deployment (another thing to manage)
- TLS certificates (cert-manager dependency or self-signed PKI)
- MutatingWebhookConfiguration (cluster-scoped resource)
- Failure mode complexity (`failurePolicy: Fail` blocks pods, `Ignore` leaks traffic)

**Consumer self-patching provides:**
- Topology visible in `juju status --relations`
- Clear which client uses which gateway
- No certificate management
- Failure mode is clear (charm goes to error status)
- Same pattern as storage patching (already validated)

### Consequences

* Good, because declarative - `juju relate qbittorrent gluetun` makes intent clear
* Good, because topology visible - `juju status --relations` shows VPN routing relationships
* Good, because simpler - no webhook certificates, admission controller, or ValidatingWebhookConfiguration
* Good, because charm-native - uses Juju's relation system rather than fighting it
* Good, because clean ownership - each charm manages its own Kubernetes resources
* Good, because strategic merge patch safely coexists with Juju's container management
* Good, because consumer charms already require `--trust` for storage patching (no new permissions)
* Good, because routing method communicated via relation data (extensible)
* Bad, because requires lightkube dependency in download client charms (via vpn-k8s-lib)
* Bad, because requires RBAC permissions for charms to patch their own StatefulSets
* Bad, because client sidecar adds ~3MB RAM per consumer pod (negligible)

**Why NOT "VPN charm patches consumers":**
- Breaks resource ownership principle
- gluetun-k8s would need cluster-wide RBAC to patch arbitrary StatefulSets
- Ordering dependencies unclear (what if consumer deployed first?)
- Cleanup responsibility fuzzy (who removes containers when relation breaks?)

**Rejected: Mutating webhook:**
- Adds significant infrastructure complexity
- Less visible in Juju topology
- Certificate management burden
- Privacy-critical feature shouldn't depend on webhook availability
