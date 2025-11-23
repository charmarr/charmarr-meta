# Download Client VPN Integration Method

## Context and Problem Statement

Download client pods need to be configured to route their traffic through the VPN gateway via VXLAN. This requires injecting an init container, adding NET_ADMIN capability, and configuring routing rules. How do we automate this configuration in a way that aligns with Juju's charm-based orchestration model?

## Considered Options

* **Mutating webhook (admission controller)** - Kubernetes webhook automatically injects init container when pods are created
* **Juju relations + lightkube StatefulSet patching** - Download client charms relate to vpnarr, then patch their own StatefulSet
* **Manual pod labels + external controller** - Label pods, separate controller watches and patches
* **Helm post-renderer** - Use Helm's post-rendering to modify manifests (if using Helm)

## Decision Outcome

Chosen option: **"Juju relations + lightkube StatefulSet patching"**, because it leverages Juju's existing relation system to communicate configuration, keeps modification logic in the download client charms (single responsibility), avoids webhook infrastructure complexity (certificates, admission controller deployment), and makes VPN routing topology visible via `juju status --relations`.

### Consequences

* Good, because declarative - `juju relate qbittorrent vpnarr` makes intent clear
* Good, because topology visible - `juju status --relations` shows VPN routing relationships
* Good, because simpler - no webhook certificates, admission controller, or ValidatingWebhookConfiguration
* Good, because charm-native - uses Juju's relation system rather than fighting it
* Good, because download clients already need to relate to vpnarr for configuration anyway
* Bad, because requires lightkube dependency in download client charms
* Bad, because requires RBAC permissions for charms to patch their own StatefulSets
* Bad, because charm must handle StatefulSet recreation if patch fails

**Implementation pattern (shared in charmarr-lib):**
```python
def _on_vpn_gateway_relation_changed(self, event):
    vxlan_id = event.relation.data[event.app].get('vxlan_id')
    gateway_ip = event.relation.data[event.app].get('gateway_ip')
    cluster_cidrs = event.relation.data[event.app].get('cluster_cidrs')

    patch = {
        "spec": {
            "template": {
                "spec": {
                    "initContainers": [{
                        "name": "vpn-route-init",
                        "image": "ghcr.io/angelnu/pod-gateway:v1.11.1",
                        "command": ["/bin/client_init.sh"],
                        "env": [
                            {"name": "VXLAN_ID", "value": vxlan_id},
                            {"name": "VXLAN_GATEWAY", "value": gateway_ip},
                            {"name": "NOT_ROUTED_TO_GATEWAY_CIDRS", "value": cluster_cidrs}
                        ],
                        "securityContext": {"capabilities": {"add": ["NET_ADMIN"]}}
                    }]
                }
            }
        }
    }
    client = lightkube.Client()
    client.patch(StatefulSet, self.app.name, patch, namespace=self.model.name)
```

**Scaling constraint:**
* vpnarr charm MUST block scaling beyond 1 unit (multiple gateway pods cause VXLAN ID conflicts)
* Implementation: `if self.app.planned_units() > 1: self.unit.status = BlockedStatus("Scaling not supported")`
* HA approach: Deploy multiple separate vpnarr applications, not `juju scale-application`

**Rejected alternatives:**
* Mutating webhook: Adds significant infrastructure (webhook server, TLS certificates, admission controller), less visible in Juju topology, harder to debug
* Manual labels: Requires external controller, breaks Juju's declarative model, controller needs cluster-wide permissions
* Helm post-renderer: Only works if using Helm, Charmarr uses Juju charms natively
