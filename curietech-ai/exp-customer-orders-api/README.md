# exp-customer-orders-api

Experience-layer Mule 4 API for web/mobile clients. It exposes a single client-friendly REST/JSON surface and routes only to PROCESS-layer APIs over HTTP. It owns no data and does not call SYSTEM-layer APIs directly.

## Client-facing endpoints

Base URL locally: `http://localhost:8080/api`

| Method | Endpoint | Downstream process call |
|---|---|---|
| GET | `/customers` | `prc-customer-management-api GET /customers` |
| GET | `/customers/{customerId}` | `prc-customer-management-api GET /customers/{id}` |
| POST | `/customers` | `prc-customer-management-api POST /customers` |
| PUT | `/customers/{customerId}` | `prc-customer-management-api PUT /customers/{id}` |
| DELETE | `/customers/{customerId}` | `prc-customer-management-api DELETE /customers/{id}` |
| GET | `/customers/{customerId}/orders` | `prc-order-management-api GET /customers/{id}/orders` |
| GET | `/customers/{customerId}/orders/{orderId}` | `prc-order-management-api GET /orders/{orderId}` |
| POST | `/customers/{customerId}/orders` | `prc-order-management-api POST /customers/{id}/orders` |
| PUT | `/customers/{customerId}/orders/{orderId}` | `prc-order-management-api PUT /orders/{orderId}` |
| POST | `/customers/{customerId}/orders/{orderId}/cancel` | `prc-order-management-api POST /orders/{orderId}/cancel` |
| DELETE | `/customers/{customerId}/orders/{orderId}` | `prc-order-management-api DELETE /orders/{orderId}` |

The experience API is intentionally thin: payloads are passed through, downstream HTTP statuses such as `400`, `404`, and `409` are propagated, and the `x-correlation-id` header is passed through when supplied.

## Call chain

```text
Web/mobile client
  -> exp-customer-orders-api (Experience, port 8080, base path /api)
    -> prc-customer-management-api (Process, port 8083, base path /api)
      -> customer system API (System, port 8081)
    -> prc-order-management-api (Process, port 8084, base path /api)
      -> order system API (System, port 8082)
```

This API must not call the system APIs directly.

## Downstream dependencies and ports

| Layer | API | Local URL |
|---|---|---|
| Experience | `exp-customer-orders-api` | `http://localhost:8080/api` |
| Process | `prc-customer-management-api` | `http://localhost:8083/api` |
| Process | `prc-order-management-api` | `http://localhost:8084/api` |
| System | customer system API | `http://localhost:8081/api` |
| System | order system API | `http://localhost:8082/api` |

## Configuration

Default development configuration is in `src/main/resources/config/config-dev.yaml`:

```yaml
http:
  host: "0.0.0.0"
  port: "8080"

prc:
  customer:
    baseUrl: "http://localhost:8083/api"
  order:
    baseUrl: "http://localhost:8084/api"
```

The application loads `config/config-${mule.env}.yaml`; `mule.env` defaults to `dev` in `global.xml` and can be overridden at runtime with `-Dmule.env=<env>`.

## Error handling

| Condition | Client response |
|---|---|
| Downstream HTTP `400`, `404`, `409` | Same status and JSON body propagated from process API |
| Downstream connectivity/timeout | `502` with `{"error":"BAD_GATEWAY"}` |
| Unhandled error | `500` with `{"error":"INTERNAL_SERVER_ERROR","message":"Unexpected error while processing the request"}` |

## Run locally

Start the stack from the bottom up so each layer has its dependencies available:

1. Start customer system API on port `8081`.
2. Start order system API on port `8082`.
3. Start `prc-customer-management-api` on port `8083`.
4. Start `prc-order-management-api` on port `8084`.
5. Start this experience API on port `8080`:

```bash
mvn clean package
mvn mule:run -Dmule.env=dev
```

## Example requests

```bash
curl -H "x-correlation-id: demo-001" http://localhost:8080/api/customers
curl http://localhost:8080/api/customers/CUST-10421/orders
curl -X POST http://localhost:8080/api/customers/CUST-10421/orders/ORD-001/cancel
```

## Tests

MUnit tests mock all process-layer HTTP requests, so tests do not require any downstream APIs to be running:

```bash
mvn test -Dmule.env=dev
```
