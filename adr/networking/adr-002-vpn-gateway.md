# VPN Gateway Solution for Download Clients

**Status:** Accepted

## Context and Problem Statement

Download clients (qBittorrent, SABnzbd) need to route all external traffic through a VPN for privacy and to avoid exposing the user's real IP address to torrent trackers and Usenet providers. How do we implement VPN egress routing in a Kubernetes environment that integrates with Charmarr's Juju-based architecture?

## Considered Options

* **gluetun as centralized gateway** - VXLAN overlay routing to a single VPN gateway pod running gluetun
* **Istio egress gateway** - Use Istio's built-in egress gateway with VPN configuration
* **Per-pod gluetun sidecars** - Each download client runs gluetun as a sidecar container
* **VPN CNI plugin** - Custom CNI plugin for VPN routing at the network layer

## Decision Outcome

Chosen option: **"gluetun as centralized gateway"**, because it provides a battle-tested VPN client that supports multiple providers, enables VXLAN-based routing proven by the k8s@home community, supports multiple download clients through a single VPN connection, and integrates cleanly with Juju's charm-based lifecycle management.

### Architecture

**gluetun-k8s charm**: Runs gluetun container (VPN client, creates `tun0` interface). At runtime, `VPNGatewayProvider` from vpn-k8s-lib patches the charm's StatefulSet to add pod-gateway gateway containers (`gateway_init.sh` + `gateway_sidecar.sh`) that handle the VXLAN endpoint.

**Consumer pods**: Download client charms use `VPNGatewayRequirer` from vpn-k8s-lib, which patches their StatefulSets with pod-gateway client containers (`client_init.sh` + `client_sidecar.sh`) that configure VXLAN routing to the gluetun pod.

**vpn-k8s-lib**: Separate Python package providing the `vpn-gateway` interface classes and routing helpers. Both provider and requirer sides use the library for StatefulSet patching. The routing method (currently pod-gateway) is communicated via relation data, allowing future extensibility without charm code changes.

### Why pod-gateway is Required

Gluetun alone only provides VPN for its own pod's network namespace. It does not:
- Create VXLAN interfaces for accepting traffic from other pods
- Run a DHCP server for client IP assignment
- Handle routing/NAT for traffic from other network namespaces

pod-gateway provides the VXLAN overlay infrastructure:
- **Gateway side**: Creates VXLAN endpoint, runs DHCP server, forwards traffic to `tun0`
- **Client side**: Creates VXLAN interface, manipulates routing table, monitors gateway health

### pod-gateway Components

**Gateway containers (patched into gluetun-k8s):**
- `gateway_init.sh`: Creates VXLAN tunnel, sets up iptables forwarding rules, blocks non-VPN traffic
- `gateway_sidecar.sh`: Runs DHCP and DNS server for client pods

**Client containers (patched into consumer pods):**
- `client_init.sh`: Creates VXLAN interface, gets IP via DHCP, changes default route
- `client_sidecar.sh`: Monitors gateway connectivity, resets VXLAN if gateway pod IP changes (handles gateway restarts gracefully)

### Consequences

* Good, because single VPN connection serves multiple download clients (cost-effective, simpler management)
* Good, because VXLAN overlay operates below service mesh layer, enabling coexistence with Istio Ambient
* Good, because gluetun supports multiple VPN providers (NordVPN, Mullvad, ProtonVPN, etc.) with unified configuration
* Good, because centralized gateway simplifies VPN credential management and provider switching
* Good, because interface abstraction (vpn-k8s-lib) allows future replacement of gluetun or pod-gateway with alternatives
* Good, because both provider and consumer use same library for patching (consistent approach)
* Good, because routing method is communicated via relation data (extensible without charm changes)
* Bad, because creates single point of failure (mitigated by kill switch; deploy separate gateways if HA needed)
* Bad, because requires NetworkPolicy for kill switch defense-in-depth (handled by vpn-k8s-lib)
* Bad, because consumer pods need both init container AND sidecar (minimal resource overhead ~3MB RAM)

**DNS Privacy Consideration:**
* Download clients using cluster DNS (default) leak domain queries to upstream DNS resolvers
* **Recommended**: Configure gluetun's DNS-over-TLS to VPN provider (`DOT=on`)
* gluetun-k8s charm should provide configuration option for DNS strategy

**Scaling constraint:**
* gluetun-k8s charm MUST block scaling beyond 1 unit (multiple gateway pods cause VXLAN ID conflicts)
* HA approach: Deploy multiple separate gluetun-k8s applications with different names

**Rejected alternatives:**
* Istio egress gateway: Operates at wrong layer (Layer 7 HTTP/TLS), doesn't support BitTorrent protocols
* Per-pod sidecars: Wasteful VPN connections, complicates telemetry, harder to manage
* VPN CNI plugin: Too invasive, conflicts with existing CNI, difficult to charm

**Image versions:**
* gluetun: `ghcr.io/qdm12/gluetun:latest`
* pod-gateway: `ghcr.io/angelnu/pod-gateway:v1.13.0`
