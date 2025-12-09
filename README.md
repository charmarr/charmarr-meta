<p align="center">
  <img src="assets/charmarr-charmarr-meta.png" width="350" alt="Charmarr Meta">
</p>

# Charmarr Meta Repository

Central documentation and architecture repository for the Charmarr project.

## About This Repository

This meta repository serves as the source of truth for Charmarr's architecture, design decisions, and project documentation. It contains no executable code - only documentation, ADRs, and assets used across the Charmarr ecosystem.

## What is Charmarr?

Charmarr is a Kubernetes-native media automation platform built with Juju charms, providing:
- Automated media management (Radarr, Sonarr, Prowlarr)
- VPN-routed download clients (qBittorrent, SABnzbd)
- Media streaming (Plex)
- Content request management (Overseerr)
- Enterprise-grade networking (VPN egress, service mesh, Tailscale)

## Repository Contents

```
charmarr-meta/
├── adr/              Architecture Decision Records
├── assets/           Visual resources and images
├── docs/             General project documentation
└── .github/          GitHub workflows and automation
```

### [adr/](adr/)
Architecture Decision Records organized by domain (apps, interfaces, lib, networking, storage). Each ADR documents design choices with context, alternatives, and consequences.

### [assets/](assets/)
Images used in documentation and README files across Charmarr repositories.

### [docs/](docs/)
General documentation not specific to architectural decisions (user guides, tutorials, deployment documentation).
