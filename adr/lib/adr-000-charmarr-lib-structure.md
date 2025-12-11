# Charmarr-Lib Monorepo Structur# Charmarr-Lib Monorepo Structure

**Status:** Accepted

## Context and Problem Statement

Charmarr requires two shared libraries: `charmarr-core` (interfaces, API clients, reconcilers for arr applications) and `charmarr-vpn` (VPN gateway interface and StatefulSet patching). How should these libraries be organized for development, packaging, and distribution?

Key considerations:
- Clean, consistent public API across packages
- Independent installability (`pip install charmarr-core` vs `pip install charmarr-vpn`)
- Shared `charmarr` namespace for discoverability
- Freedom to reorganize internals without breaking consumers
- `charmarr-vpn` should remain reusable outside Charmarr ecosystem

## Considered Options

* **Option 1: Separate repositories** — `github.com/adi/charmarr-core` and `github.com/adi/charmarr-vpn`
* **Option 2: Monorepo with separate namespace packages** — single repo, two pip-installable packages under `charmarr.*` namespace
* **Option 3: Single package with extras** — `pip install charmarr-lib[vpn]`

## Decision Outcome

Chosen option: **Option 2 (Monorepo with namespace packages)**, because it provides development convenience while maintaining distribution flexibility and clear naming.

### Repository Structure

```
github.com/adi/charmarr-lib/
├── packages/
│   ├── core/
│   │   ├── pyproject.toml
│   │   └── src/
│   │       └── charmarr/
│   │           └── core/
│   │               ├── __init__.py
│   │               ├── interfaces/
│   │               │   ├── __init__.py
│   │               │   ├── _media_indexer.py
│   │               │   ├── _download_client.py
│   │               │   ├── _media_manager.py
│   │               │   ├── _media_storage.py
│   │               │   └── _media_server.py
│   │               ├── _clients/
│   │               │   ├── _base.py
│   │               │   ├── _arr.py
│   │               │   ├── _prowlarr.py
│   │               │   ├── _plex.py
│   │               │   └── _overseerr.py
│   │               ├── _reconcilers/
│   │               │   ├── _download_clients.py
│   │               │   ├── _root_folders.py
│   │               │   └── _applications.py
│   │               └── _models/
│   │                   └── _config_builders.py
│   │
│   └── vpn/
│       ├── pyproject.toml
│       └── src/
│           └── charmarr/
│               └── vpn/
│                   ├── __init__.py
│                   ├── interfaces/
│                   │   ├── __init__.py
│                   │   └── _gateway.py
│                   ├── _patching/
│                   │   ├── _models.py
│                   │   ├── _gateway.py
│                   │   ├── _client.py
│                   │   └── _network_policy.py
│                   └── _constants.py
```

### Installation

```bash
pip install charmarr-core
pip install charmarr-vpn
```

### Public API

Two import paths per package — `interfaces` for relation classes/models, root for everything else:

```python
# Core
from charmarr.core.interfaces import (
    MediaIndexerProvider,
    MediaIndexerRequirer,
    MediaIndexerProviderData,
    DownloadClientProvider,
    DownloadClientRequirer,
    DownloadClientProviderData,
    # ...
)
from charmarr.core import (
    ArrApiClient,
    ProwlarrApiClient,
    reconcile_download_clients,
    reconcile_root_folder,
    # ...
)

# VPN
from charmarr.vpn.interfaces import (
    VPNGatewayProvider,
    VPNGatewayRequirer,
    VPNGatewayProviderData,
    VPNGatewayRequirerData,
)
from charmarr.vpn import (
    patch_gateway_statefulset,
    patch_client_statefulset,
    create_killswitch_network_policy,
    # ...
)
```

### Conventions

1. **PEP 420 namespace packages** — no `__init__.py` in `src/charmarr/`, only in subpackages
2. **Underscore-prefixed internals** — all implementation files and directories prefixed with `_`
3. **Re-export via `__init__.py`** — public API defined explicitly in `__init__.py` files
4. **Consistent structure** — both packages follow same pattern (`interfaces/` + everything else)

### pyproject.toml

**packages/core/pyproject.toml:**
```toml
[project]
name = "charmarr-core"
version = "0.1.0"
description = "Core charm libraries for Charmarr media automation"
dependencies = [
    "ops>=2.0",
    "pydantic>=2.0",
    "lightkube>=0.15",
    "httpx>=0.27",
]

[tool.setuptools.packages.find]
where = ["src"]
```

**packages/vpn/pyproject.toml:**
```toml
[project]
name = "charmarr-vpn"
version = "0.1.0"
description = "VPN gateway charm library for Kubernetes"
dependencies = [
    "ops>=2.0",
    "pydantic>=2.0",
    "lightkube>=0.15",
]
```

### Testing Strategy

**Library repo: unit tests only.** The library produces data structures and implements decision logic — unit tests verify correctness. No real K8s or Juju integration happens within the library itself.

**Charm repo: unit + integration tests.** Charms test against real infrastructure using pytest-operator / jubilant. Integration tests are the source of truth — if unit tests pass but integration fails, the mock was wrong.

```
packages/
├── core/
│   └── tests/
│       └── unit/
│           ├── interfaces/
│           │   └── test_media_indexer.py
│           ├── clients/
│           │   └── test_arr.py
│           └── reconcilers/
│               └── test_download_clients.py
│
└── vpn/
    └── tests/
        └── unit/
            ├── interfaces/
            │   └── test_gateway.py
            └── patching/
                ├── test_gateway_patch.py
                └── test_client_patch.py
```

**Focus unit tests on:**
- Reconciler decision logic (add/update/delete)
- K8s patch generation (easy to mess up specs)
- Pydantic model validation

**Skip:**
- Trivial re-exports
- `__init__.py` wiring

### Consequences

**Good:**
- Single repo for coordinated development, PRs can span both packages
- Independent versioning and releases when needed
- Clear `charmarr.*` namespace signals ecosystem membership
- Underscore convention prevents accidental imports of internals
- `charmarr-vpn` has no dependency on `charmarr-core`, remains independently reusable
- Consistent import patterns reduce cognitive load

**Bad:**
- PEP 420 namespace packages require no `__init__.py` in shared parent — easy to mess up
- Two `pyproject.toml` files to maintain
- Local development requires editable installs of both packages
