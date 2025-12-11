# Multimeter Test Utility Charm

**Status:** Accepted

**Related:** [adr-001-testing-strategy.md](adr-001-testing-strategy.md) — overall testing approach using pytest-bdd and Gherkin

## Context and Problem Statement

Integration testing Charmarr charms requires relating them to other charms. For example, testing Radarr's `download-client` relation requires a charm implementing the provider side (qBittorrent or SABnzbd).

This creates dependencies:
- Can't test Radarr until qBittorrent charm exists
- Testing one charm requires deploying multiple real charms
- Harder to isolate what's being tested

We need a test utility that can stand in for any Charmarr charm during integration tests.

## Considered Options

* **Option 1:** Always test with real charms (full stack)
* **Option 2:** Mock charms per interface (one mock per interface)
* **Option 3:** Single charmarr-multimeter charm implementing all interfaces

## Decision Outcome

**Option 3 — Single charmarr-multimeter charm implementing all interfaces**

charmarr-multimeter is a test utility charm that:
- Implements all Charmarr interfaces as both provider and requirer
- Provides valid dummy data when acting as provider
- Accepts and stores relation data when acting as requirer
- Exposes received data via Juju actions for test assertions

## Interface Implementation

Multimeter implements both sides of every Charmarr interface:

| Interface | Provider Endpoint | Requirer Endpoint |
|-----------|-------------------|-------------------|
| media-indexer | `media-indexer-provider` | `media-indexer-requirer` |
| download-client | `download-client-provider` | `download-client-requirer` |
| media-manager | `media-manager-provider` | `media-manager-requirer` |
| media-storage | `media-storage-provider` | `media-storage-requirer` |
| media-server | `media-server-provider` | `media-server-requirer` |
| vpn-gateway | `vpn-gateway-provider` | `vpn-gateway-requirer` |

## Behavior

### Passive by Default

Multimeter is passive — it provides/accepts data but doesn't validate or assert. Test assertions happen in pytest step definitions, not in the charm.

### Provider Mode

When acting as provider, charmarr-multimeter publishes configurable dummy data:

```python
# Configure via Juju config or action
juju config charmarr-multimeter media-indexer-url="http://fake-prowlarr:9696"
juju config charmarr-multimeter media-indexer-api-key="test-api-key"
```

### Requirer Mode

When acting as requirer, charmarr-multimeter stores received relation data and exposes it via action:

```bash
juju run charmarr-multimeter/0 get-relation-data endpoint=download-client-requirer
```

Returns JSON with all data received on that endpoint.

## Usage in Tests

```gherkin
Feature: Radarr download client relation

  Scenario: Radarr configures download client from relation
    Given charmarr-multimeter is deployed as download-client provider
    And radarr is deployed
    When radarr is related to charmarr-multimeter via download-client
    Then radarr should have the download client configured
```

```python
@given("charmarr-multimeter is deployed as download-client provider")
def deploy_charmarr_multimeter_as_provider(juju):
    # Deploy and configure charmarr-multimeter to provide download-client data
    juju.config("charmarr-multimeter", {
        "download-client-host": "fake-qbittorrent.test.svc",
        "download-client-port": "8080",
        "download-client-type": "torrent",
    })

@then("radarr should have the download client configured")
def verify_radarr_config(juju):
    # Use Radarr API or action to verify download client was added
    ...
```

## Location

Lives in the charmarr monorepo alongside other charms:

```
charmarr/
├── charms/
│   ├── radarr-k8s/
│   ├── qbittorrent-k8s/
│   └── charmarr-multimeter/    # test utility
└── ...
```

This allows editing charmarr-multimeter while developing charms in the same PR.

## Charm Structure

```
charms/charmarr-multimeter/
├── src/
│   └── charm.py
├── charmcraft.yaml
├── pyproject.toml
└── tests/
    └── unit/              # unit tests for charmarr-multimeter itself
```

No integration tests for charmarr-multimeter — it is the integration test utility.

## Consequences

**Good:**
- Test any charm in isolation without full stack
- Single utility to maintain, not one mock per interface
- Unblocks parallel charm development
- Clear separation: charmarr-multimeter provides data, tests do assertions
- Lives in monorepo for easy co-development

**Bad:**
- Must keep charmarr-multimeter updated when interfaces change
- Dummy data may not catch all edge cases real charms would expose
- Adds one more charm to maintain

**Mitigations:**
- Interface changes require updating both real charms and charmarr-multimeter — enforce via CI
- Supplement charmarr-multimeter tests with occasional full-stack integration tests
