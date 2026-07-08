# sys-customers-api

A **SYSTEM-layer** Mule 4 API that exposes raw CRUD operations over an **in-memory mock customer datastore**.
There is no real database — an Object Store (`persistent=false`) named `customerStore` is used as the backing store
and is seeded with sample data on application startup.

This is the system layer: it only unlocks data. There are **no business rules and no cross-domain calls**.

## Stack

| Component      | Version        |
| -------------- | -------------- |
| Mule Runtime   | 4.11.x (4.11.3)|
| Java           | 17             |
| Maven          | 3.9            |
| DataWeave      | 2.0            |
| HTTP Connector | 1.10.4         |
| APIkit         | 1.11.1         |
| Object Store   | 1.2.5          |

## Configuration

Configuration lives in `src/main/resources/config/config-<env>.yaml` (default env = `dev`) and is loaded via a
`<configuration-properties>` element. The environment is selected with the `mule.env` property (default `dev`).

```yaml
http:
  host: "0.0.0.0"
  port: "8081"
  basePath: "/api"
```

- **Host:** `0.0.0.0`
- **Port:** from property `http.port`, defaults to `8081`
- **Base path:** `/api`

Override at runtime, e.g.: `-Dmule.env=dev -Dhttp.port=8081`

## Data Model — Customer

```json
{
  "id": "CUST-001",
  "firstName": "Alice",
  "lastName": "Johnson",
  "email": "alice.johnson@example.com",
  "phone": "+1-202-555-0101",
  "createdAt": "2024-01-10T09:00:00Z"
}
```

## Seed Data

On startup, a one-time seeding flow (`sys-customers-seed-flow`) checks whether `CUST-001` already exists
(`os:contains` guard) and, if not, seeds the following 5 customers keyed by their `id`:

| id       | firstName | lastName | email                        | phone            | createdAt              |
| -------- | --------- | -------- | ---------------------------- | ---------------- | ---------------------- |
| CUST-001 | Alice     | Johnson  | alice.johnson@example.com    | +1-202-555-0101  | 2024-01-10T09:00:00Z   |
| CUST-002 | Bob       | Smith    | bob.smith@example.com        | +1-202-555-0102  | 2024-02-14T11:30:00Z   |
| CUST-003 | Carla     | Nguyen   | carla.nguyen@example.com     | +1-202-555-0103  | 2024-03-05T15:45:00Z   |
| CUST-004 | David     | Martinez | david.martinez@example.com   | +1-202-555-0104  | 2024-04-20T08:15:00Z   |
| CUST-005 | Emma      | Brown    | emma.brown@example.com       | +1-202-555-0105  | 2024-05-30T18:00:00Z   |

## Endpoints

Base URL: `http://localhost:8081/api`

| Method | Path                     | Description               | Success        | Errors                          |
| ------ | ------------------------ | ------------------------- | -------------- | ------------------------------- |
| GET    | `/customers`             | List all customers        | 200 (array)    | —                               |
| GET    | `/customers/{customerId}`| Get one customer          | 200 (object)   | 404 `CUSTOMER_NOT_FOUND`        |
| POST   | `/customers`             | Create a customer         | 201 (object)   | 400 bad request                 |
| PUT    | `/customers/{customerId}`| Update a customer         | 200 (object)   | 404 `CUSTOMER_NOT_FOUND`        |
| DELETE | `/customers/{customerId}`| Delete a customer         | 204 (no body)  | 404 `CUSTOMER_NOT_FOUND`        |

### GET /customers

```bash
curl http://localhost:8081/api/customers
```

Response `200`:

```json
[
  { "id": "CUST-001", "firstName": "Alice", "lastName": "Johnson", "email": "alice.johnson@example.com", "phone": "+1-202-555-0101", "createdAt": "2024-01-10T09:00:00Z" }
]
```

### GET /customers/{customerId}

```bash
curl http://localhost:8081/api/customers/CUST-001
```

Response `200`:

```json
{ "id": "CUST-001", "firstName": "Alice", "lastName": "Johnson", "email": "alice.johnson@example.com", "phone": "+1-202-555-0101", "createdAt": "2024-01-10T09:00:00Z" }
```

Response `404`:

```json
{ "error": "CUSTOMER_NOT_FOUND" }
```

### POST /customers

The `id` is auto-generated (next `CUST-00N`) and `createdAt` is set to the current time.

```bash
curl -X POST http://localhost:8081/api/customers \
  -H "Content-Type: application/json" \
  -d '{"firstName":"Frank","lastName":"Wilson","email":"frank.wilson@example.com","phone":"+1-202-555-0106"}'
```

Response `201`:

```json
{ "id": "CUST-006", "firstName": "Frank", "lastName": "Wilson", "email": "frank.wilson@example.com", "phone": "+1-202-555-0106", "createdAt": "2024-06-01T12:00:00Z" }
```

### PUT /customers/{customerId}

Updates provided fields. `id` and `createdAt` are immutable.

```bash
curl -X PUT http://localhost:8081/api/customers/CUST-002 \
  -H "Content-Type: application/json" \
  -d '{"firstName":"Bobby","phone":"+1-202-555-9999"}'
```

Response `200`:

```json
{ "id": "CUST-002", "firstName": "Bobby", "lastName": "Smith", "email": "bob.smith@example.com", "phone": "+1-202-555-9999", "createdAt": "2024-02-14T11:30:00Z" }
```

### DELETE /customers/{customerId}

```bash
curl -X DELETE http://localhost:8081/api/customers/CUST-003
```

Response `204` (no content). Returns `404 {"error":"CUSTOMER_NOT_FOUND"}` if not present.

## Error Handling

A global error handler maps APIkit errors to HTTP status codes and returns a JSON body `{"error":"...","message":"..."}`:

| Error type                  | HTTP status |
| --------------------------- | ----------- |
| `APIKIT:NOT_FOUND`          | 404         |
| `APIKIT:BAD_REQUEST`        | 400         |
| `APIKIT:METHOD_NOT_ALLOWED` | 405         |
| any other error             | 500         |

## How to Run

Run the tests:

```bash
mvn clean test
```

Run the application locally:

```bash
mvn mule:run
```

Then hit `http://localhost:8081/api/customers`.

## Project Layout

```
src/main/mule/global.xml               # HTTP listener, APIkit, ObjectStore, error handler, config props
src/main/mule/sys-customers-api.xml    # Seeding flow, main flow, CRUD implementation flows
src/main/resources/api/                # RAML specification
src/main/resources/config/             # Environment config (config-dev.yaml)
src/main/resources/dwl/                # Seed data
src/test/munit/                        # MUnit tests
```

## MUnit Tests

`src/test/munit/sys-customers-api-test.xml` covers GET all, GET one (found + 404), POST create,
PUT update (+ 404), and DELETE (+ 404). Each test clears and re-seeds the object store in a
`<munit:before-test>` block so results are deterministic.
