<p align="center">
  <img src="../../assets/charmarr-charmarr-adr.png" width="450" alt="Charmarr ADR">
</p>

# Charmarr Networking Architecture

This directory contains the architectural decision records (ADRs) for Charmarr's networking design, covering ingress, egress, service mesh integration, and deployment topology from the networking perspective.

## Architecture Overview

Charmarr's networking architecture provides:
- **Ingress**: Three-gateway design with Tailscale for remote access, segmented by security domain
- **Egress**: VPN routing for download clients via VXLAN overlay with two-layer kill switch
- **Service Mesh**: Istio Ambient for mTLS and observability, coexisting with VPN routing
- **Topology**: Multi-model separation (core + downloads) for operational clarity

## ADRs

### Ingress
- **[ADR-001: Ingress Architecture](adr-001-ingress.md)** - Three Istio gateways (arr/media/admin) with Tailscale integration

### VPN Egress
- **[ADR-002: VPN Gateway Solution](adr-002-vpn-gateway.md)** - pod-gateway + gluetun centralized gateway with VXLAN overlay (includes DNS privacy consideration)
- **[ADR-003: Download Client Integration](adr-003-download-client-egress.md)** - Juju relations + lightkube StatefulSet patching (no webhook)
- **[ADR-004: VPN Kill Switch](adr-004-vpn-kill-switch.md)** - Two-layer defense: NetworkPolicy + gluetun firewall

### Service Mesh
- **[ADR-005: Istio Mesh Integration](adr-005-istio-mesh-vpn-integration.md)** - Download clients IN mesh, gateway OUT; routing layer separation enables coexistence

### Topology
- **[ADR-006: Multi-Model Topology](adr-006-model-topology.md)** - charmarr-core + charmarr-downloads models with cross-model relations

## Key Design Principles

1. **Defense in Depth**: Multiple enforcement points (NetworkPolicy, gluetun, Istio AuthorizationPolicy)
2. **Layer Separation**: Routing decisions before ztunnel interception (VXLAN ↔ Istio coexistence)
3. **Declarative Topology**: Juju relations make VPN routing visible via `juju status --relations`
4. **Security Segmentation**: Three ingress domains, namespace-level policies, mesh enrollment control

## Traffic Flow Summary

**External → Services**: Tailscale → MetalLB → Istio Gateway → Service
**Download Client → Tracker**: eth0 → vxlan0 → pod-gateway → gluetun → VPN tunnel
**Download Client → Radarr**: eth0 → ztunnel (mTLS) → ztunnel → Radarr
**Kill Switch**: NetworkPolicy (routing failures) + gluetun firewall (VPN disconnects)

## Explicitly Deferred

- Custom domain configuration (future enhancement)

## Reading Order

Suggested order for understanding the complete architecture:
1. ADR-001 (ingress foundation)
2. ADR-002 → ADR-003 → ADR-004 (VPN egress stack)
3. ADR-005 (mesh integration)
4. ADR-006 (topology)
