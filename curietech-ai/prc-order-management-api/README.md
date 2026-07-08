# prc-order-management-api

Process-layer Mule 4 API that orchestrates order lifecycle business logic and delegates persistence to SYSTEM APIs over HTTP. It does not own a data store.

## Runtime

- Mule Runtime: 4.11.3
- Java: 17
- Maven: 3.9
- API style: REST/JSON with APIkit and RAML

## Local configuration

Configuration is loaded from `src/main/resources/config/config-dev.yaml`.

| Property | Default | Description |
| --- | --- | --- |
| `http.port` | `8084` | Listener port for this process API |
| `sys.orders.baseUrl` | `http://localhost:8082/api` | Base URL for `sys-orders-api` |
| `sys.customers.baseUrl` | `http://localhost:8081/api` | Base URL for `sys-customers-api` |

The HTTP listener binds to `0.0.0.0:${http.port}` with base path `/api`.

## Endpoints

| Method | Path | Behavior |
| --- | --- | --- |
| `GET` | `/api/customers/{customerId}/orders` | Calls `GET sys-orders /orders?customerId={id}` and returns the order array. |
| `GET` | `/api/orders/{orderId}` | Proxies `GET sys-orders /orders/{id}` and propagates 404. |
| `POST` | `/api/customers/{customerId}/orders` | Validates customer exists via `sys-customers`, validates payload, sets `customerId` from path, then calls `POST sys-orders /orders`. Returns 201. |
| `PUT` | `/api/orders/{orderId}` | Validates payload. If `customerId` is present, validates it exists before calling `PUT sys-orders /orders/{id}`. |
| `POST` | `/api/orders/{orderId}/cancel` | Fetches order, returns 409 if already `CANCELLED`; otherwise calls `PATCH sys-orders /orders/{id}/status` with `{ "status": "CANCELLED" }`. |
| `DELETE` | `/api/orders/{orderId}` | Calls `DELETE sys-orders /orders/{id}` and returns 204. |

## Business rules

- Customer existence is validated before order creation.
- Missing customer during create returns `404 { "error": "CUSTOMER_NOT_FOUND" }`.
- Order payload must include at least one item and every `quantity` must be greater than 0.
- Create always sets `customerId` from the URI path; a body-supplied `customerId` cannot override it.
- Cancel is a status update, not a delete.
- Cancelling an already cancelled order returns `409 { "error": "ALREADY_CANCELLED" }`.

## Error handling

A global error handler maps downstream/system errors to JSON bodies:

- `HTTP:NOT_FOUND` -> 404 `{ "error": "NOT_FOUND" }`
- `HTTP:BAD_REQUEST` and APIkit bad requests -> 400 `{ "error": "BAD_REQUEST" }`
- `HTTP:CONNECTIVITY` / `HTTP:TIMEOUT` -> 502 `{ "error": "BAD_GATEWAY" }`
- Generic errors -> 500 `{ "error": "INTERNAL_SERVER_ERROR" }`

## Run locally

```bash
mvn clean package -DskipTests
mvn mule:run -Dmule.env=dev
```

Base URL: `http://localhost:8084/api`

## Run MUnit

```bash
mvn clean test -Dmule.env=dev
```
