# g2p-bridge-connectors

Pluggable **connectors** for the OpenG2P [G2P Bridge](https://github.com/OpenG2P/g2p-bridge).
This repo holds the **reference/standard** implementations of the connector
interfaces the Bridge defines — bank, mapper, notification, geo, agency and
warehouse — as independent Python packages:

| Package | Interface | Reference impl | Dispatch |
| --- | --- | --- | --- |
| `bank-connectors` | `BankConnectorInterface` | `ExampleBankConnector` | by sponsor **bank code** |
| `mapper-connectors` | `MapperInterface` | `SPARMapper` | single (config) |
| `notification-connectors` | `NotificationInterface` | `NovuNotifier` | single (config) |
| `geo-resolver` | `GeoResolver` | `FarmerGeoResolverImpl` | by **target registry** |
| `agency-allocator` | `AgencyAllocator` | `AgencyAllocatorRefImpl` | single (config) |
| `warehouse-allocator` | `WarehouseAllocator` | `WarehouseAllocatorRefImpl` | single (config) |

## Why this repo is separate

The G2P Bridge **core/platform** (`g2p-bridge`) is a standard, versioned,
open-source product. We do **not** want adopters to fork or modify it just to run a
different bank/notifier/registry. Connectors are the one part that varies per
deployment, so they live here, apart from core.

- **Core is consumed as a library.** At build time the Bridge's Celery image pulls
  the reference connectors from this repo by git ref (`G2P_BRIDGE_CONNECTORS_REF`).
  Core is never forked.
- **Connectors are selected by config, not code.** Each factory resolves the active
  implementation from a config value (a dotted class path), so adding a connector is
  additive — no edit to core or to this repo's factories.

## How the Bridge picks a connector (config-driven)

Every factory returns whatever implementation the package's `Initializer` registered,
chosen from config:

- **Single-impl** connectors read `…_IMPL` (one dotted path).
- **Keyed** connectors (bank, geo) read `…_IMPLS` (a `{key: dotted-path}` map); the
  factory dispatches by key (bank code / target registry).

| Connector | Config env var | Default |
| --- | --- | --- |
| bank | `G2P_BRIDGE_BANK_CONNECTORS_IMPLS` | `{"EXAMPLE-BANK": "…bank_connectors.ExampleBankConnector"}` |
| mapper | `G2P_BRIDGE_MAPPER_CONNECTORS_IMPL` | `…implementations.SPARMapper` |
| notification | `G2P_BRIDGE_NOTIFICATION_CONNECTORS_IMPL` | `…implementations.NovuNotifier` |
| geo | `G2P_BRIDGE_GEO_RESOLVER_IMPLS` | `{"farmer": "…implementations.FarmerGeoResolverImpl"}` |
| agency | `G2P_BRIDGE_AGENCY_ALLOCATOR_IMPL` | `…implementations.AgencyAllocatorRefImpl` |
| warehouse | `G2P_BRIDGE_WAREHOUSE_ALLOCATOR_IMPL` | `…implementations.WarehouseAllocatorRefImpl` |

## Writing your own connector

You do **not** fork core, and you do **not** need to fork this repo. Write one small
package and layer it onto the published Bridge image.

**1. Implement the interface** in your own package, depending on the relevant
connector package here for the interface + DTOs:

```python
# acme_bank_connector/acme_bank_connector.py
from openg2p_g2p_bridge_bank_connectors.bank_interface.bank_connector_interface import (
    BankConnectorInterface, CheckFundsResponse, BlockFundsResponse,  # ...
)

class AcmeBankConnector(BankConnectorInterface):
    def check_funds(self, ...) -> CheckFundsResponse: ...
    def block_funds(self, ...) -> BlockFundsResponse: ...
    # ...implement every abstract method
```

**2. Build a derived image** — a few lines, `FROM` the published Bridge Celery image:

```dockerfile
FROM openg2p/openg2p-bridge-celery:<version>
COPY acme-bank-connector /conn
RUN pip install /conn
```

**3. Select it via config** (env / Helm values). For a keyed connector, register it
under its dispatch key:

```
G2P_BRIDGE_BANK_CONNECTORS_IMPLS={"ACME-BANK":"acme_bank_connector.AcmeBankConnector"}
```

For a single-impl connector, point the `…_IMPL` var at your class.

**4. Deploy** with the standard Helm chart, overriding only the Celery `image.tag`
to your derived image. Core, the APIs, the chart and every other image stay the
Bridge's standard, unmodified builds — your connector is a thin, supportable overlay.

See the [design documentation](https://docs.openg2p.org) for the full split
rationale and the end-to-end connector guide.
