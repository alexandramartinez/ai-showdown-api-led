# API-Led Connectivity Network — Customers & Orders

A reference **API-led connectivity** implementation on MuleSoft, demonstrating the three
canonical layers — **Experience → Process → System** — for a Customers + Orders domain.

There is no real database. The **System layer mocks and seeds a "database"** using
file-backed JSON tables that behave like a system of record: creates, edits, deletes and
cancels **persist across requests** while the app runs.

---

## The 4 APIs (3 layers)

| Layer | Application | Port | Responsibility |
|-------|-------------|------|----------------|
| **Experience** | `exp-customer-orders-api` | `8081` | Channel-tailored for web/mobile. Delegates to the Process API and **aggregates** a customer + their orders into one payload. Adds `X-Correlation-Id`. No business logic. |
| **Process** | `prc-customer-orders-api` | `8082` | Orchestration + **all business rules**. Customer is the aggregate root; orders are a sub-resource. Owns the delete-guard, cancel semantics, existence/ownership checks. Reimplements no CRUD — it delegates. |
| **System** | `sys-customers-api` | `8083` | System of record for **Customers**. Rule-free CRUD over a file-backed mock table. |
| **System** | `sys-orders-api` | `8084` | System of record for **Orders**. Rule-free CRUD + `?customerId=` filter over a file-backed mock table. |

```
Client ──▶ exp-customer-orders-api ──▶ prc-customer-orders-api ──┬──▶ sys-customers-api ──▶ customers.json
 (8081)                        (8082)                            └──▶ sys-orders-api    ──▶ orders.json
```

### Why this topology (thinking like an integration architect)

- **One Process API, not two.** Every operation is "an order *attached to a customer*", so
  Customer is the natural **aggregate root** with orders as a sub-resource. The single
  cross-entity rule — *delete a customer only if it has no orders* — lives in one place
  instead of forcing chatty process-to-process calls.
- **System APIs split per entity.** Canonical API-led practice: one System API per
  system-of-record. Each is pure, reusable, rule-free CRUD. Swapping the file-mock for a
  real database later touches **only these two apps** — the whole point of the layering.
- **No "one big API"** (logic isn't collapsed into a monolith) and **no over-splitting**
  (CRUD isn't duplicated across layers). Shared transforms and RAML fragments keep it DRY.

---

## The mock "database"

Each System API ships a **seed** file on its classpath
(`src/main/resources/data/*.seed.json`). On the first write it copies the seed into a
mutable working file under `./data-store/` (configurable) and thereafter **reads and writes
that file** using the Mule File connector + DataWeave — no Object Store, no Java, no DB.

- State persists for the life of the running app (a real DB-like feel for demos/tests).
- Delete the `./data-store/` directory (or restart with it removed) to re-seed.
- **Limitation:** single-node, coarse file writes — not concurrency-hardened. This is a
  mock; it is deliberately simple.

### Seed data (designed to exercise every rule)

4 customers, 6 orders:

| Customer | Id suffix | Orders | Demonstrates |
|----------|-----------|--------|--------------|
| Ada Lovelace | `…111` | 2 | order sub-resource ops; **409** on customer delete |
| Alan Turing | `…222` | 2 (one already `CANCELLED`) | cancel semantics |
| Katherine Johnson | `…333` | 2 | filtered listing |
| Edsger Dijkstra | `…444` | **0** | **successful** customer delete (204) |

---

## Operations → endpoints (all 11)

Exposed identically by the **Experience** (`:8081`) and **Process** (`:8082`) layers:

| # | Operation | Method & path |
|---|-----------|---------------|
| 1 | List all customers | `GET /api/customers` |
| 2 | One customer's details | `GET /api/customers/{customerId}` *(Experience embeds orders)* |
| 3 | A customer's orders | `GET /api/customers/{customerId}/orders` |
| 4 | One order's details | `GET /api/customers/{customerId}/orders/{orderId}` |
| 5 | Create a customer | `POST /api/customers` |
| 6 | Edit a customer | `PUT /api/customers/{customerId}` |
| 7 | Delete a customer *(only if no orders)* | `DELETE /api/customers/{customerId}` → **409** if orders exist |
| 8 | Create an order for a customer | `POST /api/customers/{customerId}/orders` |
| 9 | Edit an order *(stays attached)* | `PUT /api/customers/{customerId}/orders/{orderId}` |
| 10 | Cancel an order *(not deleted)* | `POST /api/customers/{customerId}/orders/{orderId}/cancel` |
| 11 | Delete an order | `DELETE /api/customers/{customerId}/orders/{orderId}` |

The **System** APIs expose the underlying flat CRUD (`/api/customers`, `/api/orders?customerId=`).

---

## Tech stack / versions

Targets the latest generally-available releases (as of 2026-01):

| Component | Version |
|-----------|---------|
| Mule runtime | **4.9.x** (`minMuleVersion` 4.9.0) |
| Java | **17** |
| DataWeave | **2.x** (`%dw 2.0`, ships with the 4.9 runtime) |
| Maven | 3.9+ |
| `mule-maven-plugin` | 4.3.0 |
| APIkit (`mule-apikit-module`) | 1.11.x |
| HTTP connector | 1.10.x |
| File connector | 1.5.x |
| MUnit | 3.x |

Versions are centralized in the parent [pom.xml](pom.xml) `<properties>`. If a newer
release exists, bump it there.

---

## Prerequisites (read before building)

This scaffold is complete but this machine is **not yet provisioned to build/run Mule**:

1. **Maven is not installed.** Install Maven 3.9+ (`brew install maven`).
2. **MuleSoft artifacts are not on Maven Central.** They resolve from the MuleSoft
   repositories (declared in [pom.xml](pom.xml)) and generally require **Enterprise
   credentials** in `~/.m2/settings.xml`:
   ```xml
   <servers>
     <server>
       <id>anypoint-exchange-v3</id>
       <username>~~~Client~~~</username>
       <password>YOUR_CONNECTED_APP_CLIENT_ID~?~YOUR_CONNECTED_APP_CLIENT_SECRET</password>
     </server>
     <server>
       <id>mulesoft-releases</id>
       <username>YOUR_NEXUS_USER</username>
       <password>YOUR_NEXUS_PASSWORD</password>
     </server>
   </servers>
   ```
3. **An EE runtime is required** (the apps declare `requiredProduct: MULE_EE` because they
   use MUnit and `ee:transform`). Run via Anypoint Studio's embedded runtime, a standalone
   EE runtime, or `mvn ... -Dmule.runtime=...` with a licensed runtime.

The build/run commands below are correct once the above are in place.

---

## Build

From the repo root (reactor build, all 4 modules):

```bash
mvn clean package
```

Or build a single app:

```bash
cd sys-customers-api && mvn clean package
```

### Run tests (MUnit)

```bash
mvn clean test
```

Each module's tests cover the happy paths **and** the key rules:
- System: create→get persistence, filtered listing, 404 on missing, delete removes.
- Process: **delete-with-orders → 409**, delete-without-orders → 204,
  create-order-for-missing-customer → 404, **cancel sets `CANCELLED` via PUT (not DELETE)**.
- Experience: **GET one customer aggregates orders** (`orderCount` + embedded `orders`),
  404 propagation, and 409 relay from the delete-guard.

---

## Run locally (bottom-up)

The layers call each other over HTTP, so start them **bottom-up**. Deploy each
`target/*.jar` to a Mule runtime, or run each project from Anypoint Studio.

```
1. sys-customers-api   (:8083)   ─┐  independent — start first
   sys-orders-api      (:8084)   ─┘
2. prc-customer-orders-api (:8082)   needs both System APIs up
3. exp-customer-orders-api (:8081)   needs the Process API up
```

Each API also serves its live spec console at `http://localhost:<port>/api/console/`.

---

## End-to-end walkthrough (through the Experience API on :8081)

```bash
BASE=http://localhost:8081/api

# 1) List seeded customers
curl -s $BASE/customers | jq

# 2) One customer WITH orders embedded (Experience aggregation)
curl -s $BASE/customers/11111111-1111-1111-1111-111111111111 | jq

# 5) Create a customer -> capture the new id
NEW=$(curl -s -X POST $BASE/customers -H 'Content-Type: application/json' \
  -d '{"firstName":"Grace","lastName":"Hopper","email":"grace.hopper@example.com"}')
echo "$NEW" | jq
CID=$(echo "$NEW" | jq -r .id)

# 8) Create an order attached to the new customer
ORD=$(curl -s -X POST $BASE/customers/$CID/orders -H 'Content-Type: application/json' \
  -d '{"items":[{"sku":"MG-002","name":"Coffee Mug","qty":2,"price":9.50}]}')
echo "$ORD" | jq                       # total computed = 19.0
OID=$(echo "$ORD" | jq -r .id)

# 3) The customer now has one order
curl -s $BASE/customers/$CID/orders | jq

# 10) Cancel it — still present, status CANCELLED (NOT deleted)
curl -s -X POST $BASE/customers/$CID/orders/$OID/cancel | jq

# 7a) Delete guard: a customer WITH orders -> 409 Conflict
curl -s -o /dev/null -w "%{http_code}\n" \
  -X DELETE $BASE/customers/11111111-1111-1111-1111-111111111111    # -> 409

# 11) Delete the order, then...
curl -s -o /dev/null -w "%{http_code}\n" -X DELETE $BASE/customers/$CID/orders/$OID  # -> 204

# 7b) ...the customer with no orders can now be deleted
curl -s -o /dev/null -w "%{http_code}\n" -X DELETE $BASE/customers/$CID              # -> 204

# 7c) Edsger (seeded with zero orders) deletes cleanly too
curl -s -o /dev/null -w "%{http_code}\n" \
  -X DELETE $BASE/customers/44444444-4444-4444-4444-444444444444    # -> 204
```

---

## Project layout

```
.
├── pom.xml                       # parent/aggregator: versions + reactor + Mule repos
├── sys-customers-api/            # System API — customers table
├── sys-orders-api/               # System API — orders table (+ customerId filter)
├── prc-customer-orders-api/      # Process API — orchestration + business rules
└── exp-customer-orders-api/      # Experience API — channel-tailored aggregation
```

Every module follows the standard Mule 4 layout:

```
<app>/
├── pom.xml                       # inherits versions from the parent
├── mule-artifact.json            # minMuleVersion / requiredProduct
└── src/
    ├── main/
    │   ├── mule/                 # global.xml (listener/apikit/error-handler) + flows
    │   └── resources/
    │       ├── api/              # RAML 1.0 spec + dataTypes/ + examples/ fragments
    │       ├── config-local.yaml # ports + downstream hosts (externalized config)
    │       └── data/             # SYSTEM APIs only: *.seed.json mock table
    └── test/munit/               # MUnit tests
```

---

## Cross-cutting design

- **Contract-first**: RAML drives APIkit routing and auto-validation (400/404/405/406/415).
- **Consistent error envelope** everywhere: `{ "error": { "code", "message", "correlationId" } }`.
- **Externalized config**: ports and downstream URLs come from `config-local.yaml` — nothing hardcoded.
- **Correlation ID** flows Experience → Process → System (`X-Correlation-Id`) for traceability.
- **DRY**: reusable sub-flows (`*-readTable`/`*-writeTable`, `ensureCustomerExists`,
  `getOrderVerifyingOwnership`, `relayResponse`) and shared RAML dataType fragments; no CRUD
  duplicated across layers.
```
