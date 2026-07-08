# sys-orders-api (System Layer)

A **SYSTEM-layer** Mule 4 API that exposes raw CRUD operations over an **in-memory mock order datastore**.
There are no business rules and no cross-domain calls — this layer simply "unlocks" order data.

The datastore is a **non-persistent Object Store** (`orderStore`). On startup it is seeded with 8 mock
orders so the API is immediately usable.

## Stack

| Component      | Version   |
|----------------|-----------|
| Mule Runtime   | 4.9.x     |
| Java           | 17        |
| Maven          | 3.9       |
| DataWeave      | 2.0       |
| APIkit         | 1.11.10   |
| HTTP Connector | 1.11.1    |
| Object Store   | 1.2.4     |

## Configuration

Properties live in `src/main/resources/config/config-dev.yaml` and are loaded via
`configuration-properties` (environment selected by the `mule.env` property, default `dev`).

```yaml
http:
  port: "8082"
  host: "0.0.0.0"
api:
  basePath: "/api"
```

- **Host:** `0.0.0.0`
- **Port:** from property `http.port`, default `8082`
- **Base path:** `/api`

## Data Model

```json
{
  "id": "ORD-001",
  "customerId": "CUST-001",
  "status": "NEW",
  "items": [
    { "productId": "P-100", "name": "Widget", "quantity": 2, "unitPrice": 19.99 }
  ],
  "totalAmount": 39.98,
  "createdAt": "2024-02-01T10:00:00Z",
  "updatedAt": "2024-02-01T10:00:00Z"
}
```

Valid `status` values: `NEW`, `CONFIRMED`, `SHIPPED`, `CANCELLED`.

## Seed Data

Seeded on startup (only when the store is empty). Customer **CUST-005 has no orders** so downstream
"delete customer only if no orders" rules can be demonstrated.

| id      | customerId | status    | totalAmount |
|---------|------------|-----------|-------------|
| ORD-001 | CUST-001   | NEW       | 39.98       |
| ORD-002 | CUST-001   | CONFIRMED | 149.99      |
| ORD-003 | CUST-002   | SHIPPED   | 78.97       |
| ORD-004 | CUST-002   | NEW       | 199.00      |
| ORD-005 | CUST-003   | CONFIRMED | 99.98       |
| ORD-006 | CUST-003   | CANCELLED | 25.00       |
| ORD-007 | CUST-004   | SHIPPED   | 350.00      |
| ORD-008 | CUST-004   | NEW       | 99.95       |

## Endpoints

Base URL: `http://localhost:8082/api`

### GET /orders
Return all orders. Optional `?customerId=` filter.

```bash
curl http://localhost:8082/api/orders
curl "http://localhost:8082/api/orders?customerId=CUST-001"
```
Response `200`:
```json
[ { "id": "ORD-001", "customerId": "CUST-001", "status": "NEW", "items": [ ... ], "totalAmount": 39.98, ... } ]
```

### GET /orders/{orderId}
Return one order.

```bash
curl http://localhost:8082/api/orders/ORD-001
```
`200` -> the order. `404`:
```json
{ "error": "ORDER_NOT_FOUND" }
```

### POST /orders
Create an order. `id` is generated (e.g. `ORD-009`); `totalAmount` is computed from items when not
provided; `status` defaults to `NEW`; `createdAt`/`updatedAt` set to now.

```bash
curl -X POST http://localhost:8082/api/orders \
  -H "Content-Type: application/json" \
  -d '{"customerId":"CUST-002","items":[{"productId":"P-1","name":"Alpha","quantity":2,"unitPrice":19.99},{"productId":"P-2","name":"Beta","quantity":1,"unitPrice":5.00}]}'
```
Response `201` -> created order (with `id: ORD-009`, `totalAmount: 44.98`).

### PUT /orders/{orderId}
Update an existing order (items/status/etc.). Recomputes `totalAmount` when items change; sets
`updatedAt`. `404` when missing.

```bash
curl -X PUT http://localhost:8082/api/orders/ORD-001 \
  -H "Content-Type: application/json" \
  -d '{"customerId":"CUST-001","status":"CONFIRMED","items":[{"productId":"P-100","name":"Widget","quantity":4,"unitPrice":10.00}]}'
```
Response `200` -> updated order.

### PATCH /orders/{orderId}/status
Update only the status (used for cancel). `404` when missing.

```bash
curl -X PATCH http://localhost:8082/api/orders/ORD-002/status \
  -H "Content-Type: application/json" \
  -d '{"status":"CANCELLED"}'
```
Response `200` -> updated order.

### DELETE /orders/{orderId}
Remove an order. `204` on success, `404` when missing.

```bash
curl -X DELETE http://localhost:8082/api/orders/ORD-003 -i
```

## Error Handling

A global error handler maps APIkit routing errors and provides a generic `500`:

| Error                          | HTTP | Body                                                            |
|--------------------------------|------|-----------------------------------------------------------------|
| `APIKIT:BAD_REQUEST`           | 400  | `{"error":"BAD_REQUEST","message":"..."}`                       |
| `APIKIT:NOT_FOUND`             | 404  | `{"error":"NOT_FOUND","message":"..."}`                         |
| `APIKIT:METHOD_NOT_ALLOWED`    | 405  | `{"error":"METHOD_NOT_ALLOWED","message":"..."}`                |
| any other                      | 500  | `{"error":"INTERNAL_SERVER_ERROR","message":"..."}`             |

A missing order (`GET/PUT/PATCH/DELETE /orders/{id}`) returns `404 {"error":"ORDER_NOT_FOUND"}`.

## Run

```bash
# build
mvn clean package

# run MUnit tests
mvn clean test

# deploy to a local Mule 4.9 runtime (Anypoint Studio or standalone)
# then hit http://localhost:8082/api/orders
```

Override the port at runtime:
```bash
-M-Dhttp.port=9090
```

## Project Layout

```
src/main/mule/global.xml              # HTTP listener, APIkit config, ObjectStore, global error handler, properties
src/main/mule/sys-orders-api.xml      # main APIkit flow, seeding, CRUD implementation flows
src/main/resources/api/sys-orders-api.raml
src/main/resources/config/config-dev.yaml
src/test/munit/sys-orders-api-test.xml
src/test/resources/test-data/*.json
```
