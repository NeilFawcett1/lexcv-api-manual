# Products API Manual (PAPI)

This document describes the `/api/products/*` endpoints used by **non-core LexCV family apps** to register a device and query when it was first registered.

These endpoints:
- Use the standard request/response JSON envelope (`RObj`).
- **Do not** use `CheckLoginOrFail` and are **not** linked to the `users` table.
- Are intended to support “first install time” tracking for free-trial calculations.

## Base

- **HTTP**: `POST`
- **Content-Type**: `application/json`
- **Base path**: `/api/products/`

## Data Model

### `customproducts` table

Device registrations are stored in the `customproducts` table (see `schemas/customproducts.sql`).

Each row contains:
- `uuid` (CHAR(36)) — unique row identifier (UUID)
- `device_id` (varchar(200)) — the device identifier provided by the client (unique)
- `registered_ms` (BIGINT) — epoch milliseconds when the device was first registered
- `product` (varchar(64)) — product identifier/name for the external app
- `meta` (JSON) — JSON object/flags associated with the device + product (e.g. `{ "price": 50 }`)
- `localPartition` (varchar(50)) — server instance ID that accepted the original registration

### Replication

When a device is registered for the first time, the server writes an event into the Sacred Timeline outbox and it will replicate to other nodes.

## Request/Response Envelope (`RObj`)

All requests and responses use the standard LexCV JSON object:

```json
{
  "req": "...",
  "p1": "...",
  "p2": "...",
  "p3": [],
  "p4": [],
  "p5": [],
  "p6": []
}
```

For these PAPI endpoints:
- Input `p1` is used for the **device ID**.
- Output `p2` is used for the **registered epoch ms** (on success).

## Endpoint: `PAPI_RegisterDevice`

Registers a device ID (idempotent). If the device already exists, it does not create a new row and returns the original `registered_ms`.

In multi-node deployments, registration is replicated via Sacred Timeline using an idempotent upsert that converges to the **earliest** observed `registered_ms` for that device ID.

- **Path**: `/api/products/PAPI_RegisterDevice`
- **Method**: `POST`

### Request

- `req`: must be `PAPI_RegisterDevice`
- `p1`: device ID (required; trimmed; max length 200)

Example:

```json
{
  "req": "PAPI_RegisterDevice",
  "p1": "device-123",
  "p2": "",
  "p3": [],
  "p4": [],
  "p5": [],
  "p6": []
}
```

### Success response

- `p1`: `Success`
- `p2`: epoch milliseconds (string) when the device was first registered

```json
{
  "req": "PAPI_RegisterDevice",
  "p1": "Success",
  "p2": "1737582585123",
  "p3": [],
  "p4": [],
  "p5": [],
  "p6": []
}
```

### Failure responses

- Missing/empty device ID:
  - `p1`: `Fail`
  - `p2`: `no device id`

- Invalid device ID (too long, etc.):
  - `p1`: `Fail`
  - `p2`: error message (e.g. `device id too long`)

### Notes

- **Idempotency**: calling multiple times with the same device ID always returns the original `registered_ms`.
- **Replication**: only the first successful insert emits a Sacred Timeline event.

Note: a registration call may also emit a timeline event if it discovers an earlier registration time during a race (so replicas converge).

### curl example

```bash
curl -sS -X POST "https://YOUR_HOST/api/products/PAPI_RegisterDevice" \
  -H "Content-Type: application/json" \
  -d '{"req":"PAPI_RegisterDevice","p1":"device-123","p2":"","p3":[],"p4":[],"p5":[],"p6":[]}'
```

## Endpoint: `PAPI_QueryDevice`

Queries whether a device ID exists and returns the `registered_ms`.

This endpoint also updates the device row’s `lastqueried_ms` to the current epoch milliseconds on every successful query, so usage can be observed across the cluster via Sacred Timeline replication.

- **Path**: `/api/products/PAPI_QueryDevice`
- **Method**: `POST`

### Request

- `req`: must be `PAPI_QueryDevice`
- `p1`: device ID (required; trimmed; max length 200)

```json
{
  "req": "PAPI_QueryDevice",
  "p1": "device-123",
  "p2": "",
  "p3": [],
  "p4": [],
  "p5": [],
  "p6": []
}
```

### Success response

- `p1`: `Success`
- `p2`: epoch milliseconds (string) when the device was first registered

```json
{
  "req": "PAPI_QueryDevice",
  "p1": "Success",
  "p2": "1737582585123",
  "p3": [],
  "p4": [],
  "p5": [],
  "p6": []
}
```

### Failure responses

- Missing/empty device ID:
  - `p1`: `Fail`
  - `p2`: `no device id`

- Not found:
  - `p1`: `Fail`
  - `p2`: `not found`

### curl example

```bash
curl -sS -X POST "https://YOUR_HOST/api/products/PAPI_QueryDevice" \
  -H "Content-Type: application/json" \
  -d '{"req":"PAPI_QueryDevice","p1":"device-123","p2":"","p3":[],"p4":[],"p5":[],"p6":[]}'
```

## Endpoint: `PAPI_UpdateDevice`

Updates (or creates) a device record with a **product name** and **meta JSON**.

- **Path**: `/api/products/PAPI_UpdateDevice`
- **Method**: `POST`

### Request

- `req`: must be `PAPI_UpdateDevice`
- `p1`: device ID (required; trimmed; max length 200)
- `p2`: product (required; trimmed; max length 64)
- `p3[0]`: meta JSON (optional; defaults to `{}`)

Example:

```json
{
  "req": "PAPI_UpdateDevice",
  "p1": "device-123",
  "p2": "pro_monthly",
  "p3": ["{\"price\":50,\"currency\":\"GBP\"}"],
  "p4": [],
  "p5": [],
  "p6": []
}
```

### Success response

- `p1`: `Success`
- `p2`: epoch milliseconds (string) when the device was first registered

### Failure responses

- Missing device ID: `p1=Fail`, `p2=no device id`
- Missing product: `p1=Fail`, `p2=no product`
- Invalid meta JSON: `p1=Fail`, `p2=invalid meta json`

### Notes

- If the device row does not exist yet, it will be created (`registered_ms` set to “now”).
- Replication uses Sacred Timeline with per-device ordering; updates converge in order for the same `device_id`.

### curl example

```bash
curl -sS -X POST "https://YOUR_HOST/api/products/PAPI_UpdateDevice" \
  -H "Content-Type: application/json" \
  -d '{"req":"PAPI_UpdateDevice","p1":"device-123","p2":"pro_monthly","p3":["{\"price\":50}"],"p4":[],"p5":[],"p6":[]}'
```

## Client guidance

- Choose a device ID that is stable for your app’s needs (and compliant with platform policies). The server treats the device ID as an opaque string.
- If you want “first install time”, call `PAPI_RegisterDevice` on first run (and optionally also on subsequent runs; it is safe).
- If you want to only check without creating a record, call `PAPI_QueryDevice`.

## Security notes

These endpoints are intentionally not tied to LexCV user login. If you need to prevent abuse, consider adding one or more of the following (future work):
- API key / shared secret for the external app
- tighter IP-based rate limits for `/api/products/*`
- server-side allowlist of expected app package IDs / attestation (Android Play Integrity)
