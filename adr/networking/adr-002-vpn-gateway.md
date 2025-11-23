# VPN Gateway Solution for Download Clients

## Context and Problem Statement

Download clients (qBittorrent, SABnzbd) need to route all external traffic through a VPN for privacy and to avoid exposing the user's real IP address to torrent trackers and Usenet providers. How do we implement VPN egress routing in a Kubernetes environment that integrates with Charmarr's Juju-based architecture?

## Considered Options

* **pod-gateway + gluetun (centralized gateway)** - VXLAN overlay routing to a single VPN gateway pod
* **Istio egress gateway** - Use Istio's built-in egress gateway with VPN configuration
* **Per-pod gluetun sidecars** - Each download client runs gluetun as a sidecar container
* **VPN CNI plugin** - Custom CNI plugin for VPN routing at the network layer

## Decision Outcome

Chosen option: **"pod-gateway + gluetun (centralized gateway)"**, because it provides battle-tested VXLAN-based routing that is proven by the k8s@home community, supports multiple download clients through a single VPN connection, and integrates cleanly with Juju's charm-based lifecycle management.

### Consequences

* Good, because single VPN connection serves multiple download clients (cost-effective, simpler management)
* Good, because VXLAN overlay operates below service mesh layer, enabling coexistence with Istio Ambient
* Good, because pod-gateway is maintained by k8s@home community with extensive media stack experience
* Good, because gluetun supports multiple VPN providers (NordVPN, Mullvad, ProtonVPN, etc.) with unified configuration
* Good, because centralized gateway simplifies VPN credential management and provider switching
* Bad, because creates single point of failure (mitigated by deploying multiple separate gateway instances if needed)
* Bad, because requires NetworkPolicy for kill switch defense-in-depth (additional complexity)
* Bad, because gluetun's firewall operates inside pod, theoretically bypassable if pod is compromised

**DNS Privacy Consideration:**
* Download clients using cluster DNS (default Kubernetes behavior) leak domain queries to upstream DNS resolvers, potentially exposing tracker/provider domains to ISP even though traffic is VPN'd
* **Most secure**: Configure gluetun to override DNS (`DOT=on` for DNS-over-TLS to VPN provider) - queries go through VPN tunnel
* **Alternative**: Custom DNS (Pi-hole, AdGuard) for ad-blocking + privacy, but requires additional infrastructure
* **Implementation**: vpnarr charm should provide configuration option for DNS strategy (default: cluster DNS, options: vpn-provider, custom)
* Privacy-conscious users should explicitly configure VPN provider DNS via charm config

**Rejected alternatives:**
* Istio egress gateway: Operates at wrong layer (Layer 7 HTTP/TLS), doesn't support BitTorrent protocols, requires specific destination configuration
* Per-pod sidecars: Wasteful VPN connections, complicates telemetry (traffic disappears into individual pods), harder to observe overall VPN health
* VPN CNI plugin: Too invasive, conflicts with existing CNI (Calico/Cilium), limited provider support, difficult to charm

**Technical details:**
* pod-gateway image: `ghcr.io/angelnu/pod-gateway:v1.11.1`
* gluetun image: `ghcr.io/qdm12/gluetun:latest`
* VXLAN ID: Configurable (default 42)
* VPN interface naming: `tun0` (must coordinate between containers)
