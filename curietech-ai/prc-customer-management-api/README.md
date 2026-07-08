# prc-customer-management-api

A **PROCESS-layer** Mule 4 API that orchestrates customer lifecycle business logic.
It exposes a REST/JSON interface via **APIkit** (RAML spec) and delegates all data
operations to the **SYSTEM layer** over HTTP. **This API does not own a data store** —
it orchestrates and enforces cross-domain business rules only.

## Stack
- Mule Runtime **4.11** (tested with 4.11.3)
- Java **17**
- Maven **3.9**
- DataWeave **2.0**
- APIkit (RAML 1.0), HTTP Connector, MUnit 3.x

## API base path & port
- Base path: `/api`
- HTTP listener: host `0.0.0.0`, port from property `http.port` (default `8083`)

## Endpoints

| Method | Path                       | Description                                                        | Success | Errors |
|--------|----------------------------|--------------------------------------------------------------------|---------|--------|
| GET    | `/customers`               | Proxy to `GET sys-customers /customers`                            | 200     | 502    |
| GET    | `/customers/{customerId}`  | Proxy to `GET sys-customers /customers/{id}`                       | 200     | 404, 502 |
| POST   | `/customers`               | Validate then `POST sys-customers /customers`                      | 201     | 400, 502 |
| PUT    | `/customers/{customerId}`  | Validate then `PUT sys-customers /customers/{id}`                  | 200     | 400, 404, 502 |
| DELETE | `/customers/{customerId}`  | Delete only if the customer has **no orders** (see business rule)  | 204     | 404, 409, 502 |

### Request validation (POST / PUT)
Basic validation is performed before delegating to the system layer:
- `firstName`, `lastName`, `email` are **required**
- `email` must match a basic email format

On failure the API returns **400**:
```json
{ "error": "VALIDATION_ERROR", "message": "email format is invalid" }
```

### DELETE business rule (customer must have no orders)
1. Call `GET sys-orders /orders?customerId={id}`.
2. If the response is a **non-empty array**, the customer still has orders → return **409**:
   ```json
   { "error": "CUSTOMER_HAS_ORDERS", "message": "Cannot delete customer with existing orders" }
   ```
3. If the array is **empty**, call `DELETE sys-customers /customers/{id}` and return **204**.
4. If the customer does not exist downstream, the `HTTP:NOT_FOUND` is propagated as **404**.

## Downstream dependencies (SYSTEM layer)

| System API        | Default base URL              | Property                 |
|-------------------|-------------------------------|--------------------------|
| sys-customers-api | `http://localhost:8081/api`   | `sys.customers.baseUrl`  |
| sys-orders-api    | `http://localhost:8082/api`   | `sys.orders.baseUrl`     |

## Error handling
A global error handler maps errors to JSON responses:

| Condition                                             | HTTP status | Body `error` value        |
|-------------------------------------------------------|-------------|---------------------------|
| `APP:VALIDATION_ERROR`                                | 400         | `VALIDATION_ERROR`        |
| `APIKIT:BAD_REQUEST` / `HTTP:BAD_REQUEST` (downstream)| 400         | `BAD_REQUEST`             |
| `APIKIT:NOT_FOUND` / `HTTP:NOT_FOUND` (downstream)    | 404         | `NOT_FOUND`               |
| `APIKIT:METHOD_NOT_ALLOWED`                           | 405         | `METHOD_NOT_ALLOWED`      |
| `APP:CUSTOMER_HAS_ORDERS`                             | 409         | `CUSTOMER_HAS_ORDERS`     |
| `HTTP:CONNECTIVITY` / `HTTP:TIMEOUT` (downstream)     | 502         | `BAD_GATEWAY`             |
| Any other error                                       | 500         | `INTERNAL_SERVER_ERROR`   |

## Configuration
Properties live in `src/main/resources/config/config-dev.yaml` and are loaded via
`configuration-properties` using `mule.env` (default `dev`):

```yaml
http:
  port: "8083"
sys:
  customers:
    baseUrl: "http://localhost:8081/api"
  orders:
    baseUrl: "http://localhost:8082/api"
```

To add another environment, create `config-<env>.yaml` and start with `-Dmule.env=<env>`.

## Project layout
```
src/main/resources/api/customer-management-api.raml   # API contract
src/main/resources/config/config-dev.yaml             # externalized config
src/main/mule/global.xml                               # listener, requester, apikit + global error handler
src/main/mule/customer-management-api.xml             # listener + APIkit router (main flow)
src/main/mule/customer-management-api-impl.xml        # endpoint implementation flows + validation
src/test/munit/customer-management-api-test-suite.xml # MUnit tests
```

## Build & run

Build (skip tests):
```bash
mvn clean package -DskipTests
```

Run MUnit tests + coverage:
```bash
mvn clean test
```

Run locally (Studio or standalone runtime). Then, with the two system APIs running on
ports 8081/8082, try:
```bash
# list
curl http://localhost:8083/api/customers

# get one
curl http://localhost:8083/api/customers/1

# create (valid)
curl -X POST http://localhost:8083/api/customers \
  -H "Content-Type: application/json" \
  -d '{"firstName":"Alice","lastName":"Walker","email":"alice.walker@example.com"}'

# create (validation error -> 400)
curl -X POST http://localhost:8083/api/customers \
  -H "Content-Type: application/json" \
  -d '{"firstName":"Bob","lastName":"Jones"}'

# delete (409 if the customer has orders, 204 if none)
curl -X DELETE http://localhost:8083/api/customers/1
```

## MUnit coverage
The suite mocks the downstream HTTP Request calls and covers: list customers, get one,
create (valid + validation error), delete blocked when orders exist (409), and delete
allowed when no orders (204).
