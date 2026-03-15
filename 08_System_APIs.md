# Utility & System APIs

APIs for application configuration, WebSocket connections, and system utilities.

---

## API_GetConfig

**Purpose:** Retrieves application configuration and reference data.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"API_GetConfig"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Country count (as string) |
| `p3[0]` | Search results per page |
| `p3[1]` | Application title |
| `p3[2]` | Message of the day (MOTD) |
| `p3[3]` | Post text maximum length (default 500) |
| `p4` | Countries list: `[[id, name, short, "0"], ...]` |
| `p5[0]` | Organisation types: `[[id, name], ...]` (for company profiles) |
| `p5[1]` | Professional types: `[[id, name], ...]` (for individual profiles) |

### P5 Lookup Data

**p5[0] - Organisation Types** (used for company profile dropdowns):
| Index | Field | Description |
|-------|-------|-------------|
| 0 | id | Organisation type ID |
| 1 | name | Organisation type name (e.g., "Law Firm", "Chambers", "In-House Legal", "Consultancy") |

**p5[1] - Professional Types** (used for individual profile dropdowns):
| Index | Field | Description |
|-------|-------|-------------|
| 0 | id | Professional type ID |
| 1 | name | Professional type name (e.g., "Solicitor", "Barrister", "Paralegal", "Legal Executive") |

### Response Example

```json
{
  "p1": "Success",
  "p2": "195",
  "p3": ["20", "LexCV", "Welcome to the professional legal network!", "500"],
  "p4": [
    ["1", "Afghanistan", "AF", "0"],
    ["2", "Albania", "AL", "0"],
    ["185", "United Kingdom", "GB", "0"]
  ],
  "p5": [
    [["1", "Law Firm"], ["2", "Chambers"], ["3", "In-House Legal"], ["4", "Consultancy"]],
    [["1", "Solicitor"], ["2", "Barrister"], ["3", "Paralegal"], ["4", "Legal Executive"]]
  ]
}
```

### Use Cases

- Load countries list on app startup
- Get current MOTD for display
- Retrieve pagination settings
- Populate organisation type dropdown for company profiles
- Populate professional type dropdown for individual profiles

---

## API_WebSocket

**Purpose:** Establishes a WebSocket connection for real-time events.

**Authentication:** Via URL parameters, cookies, or headers

**Server endpoint:** `GET /api/w/API_WebSocket` (HTTP upgrade to WebSocket)

### Connection Methods

#### URL Parameters (GET)

```
wss://api.lexcv.co/api/w/API_WebSocket?p1=user@email.com&p2=token
```

Or using alternate parameter names:

```
wss://api.lexcv.co/api/w/API_WebSocket?uname=user@email.com&token=abc123
```

#### Cookies

```
Cookie: p1=user@email.com; p2=token
```

#### Headers

```
X-Lexcv-User: user@email.com
X-Lexcv-Token: abc123
```

### Connection Response

On successful connection, the server sends an RObj ack:

```json
{
  "req": "API_WebSocket",
  "p1": "Success",
  "p2": "connected",
  "p3": ["<socketUuid>"]
}
```

`p3[0]` is the socket UUID and must be echoed back in client pings.

### Client → Server Messages

The server accepts `data.RObj` JSON messages.

#### Ping (keepalive)

Send:

```json
{
  "req": "PING",
  "p1": "<socketUuid>"
}
```

Response:

```json
{
  "req": "PING",
  "p1": "Success",
  "p2": "PONG"
}
```

If `p1` does not match the active socket UUID, the server replies `Fail` and closes.

### Server → Client Push Events (SAPI)

All server push notifications are also sent as `data.RObj` payloads.

- `req` identifies the push event type (usually starts with `SAPI_`).
- Numeric values are commonly sent as strings.

Examples you can receive:

- `SAPI_NewMessage` (Mercury)
- `SAPI_ProfilePicEdit`, `SAPI_ProfileBannerEdit`
- `SAPI_VoxEvent` (Vox realtime inbox event)

The exact fields vary by push type; clients should route by `req`.

### How Delivery Works (Two Global Servers)

There is **no “ping globally”** step.

In LexCV there are two separate concepts:

- **When** to emit a realtime event (trigger)
- **How** to deliver it to whichever server currently hosts the user’s sockets (routing)

#### Trigger: local vs Sacred Timeline

Many realtime events are triggered from **Sacred Timeline post-execute hooks** on each node.

- The node that *originated* the change usually does **not** replay its own event (`ShouldExecuteEvent` returns false), so it must emit realtime notifications directly in the request handler/background worker.
- Other nodes *do* execute the replicated event and can emit realtime notifications in `handlePostExecute`.

This is the “piggyback Sacred Timeline events to generate a local socket event” approach and it is used for several domains (e.g. Mercury messaging, some profile edit events, and Vox as it is being wired).

#### Routing + delivery: Redis IPC + in-memory sockets

Delivery is:

1) Each server keeps an **in-memory socket registry** (per process) keyed by `user_uuid → socket_uuid → conn`.
2) On connect/disconnect the server writes **socket metadata to Redis** (user/socket/server mapping).
3) When code wants to push to a user, it calls the notification layer (for example `nfnotify.NotifyNewMessage`).
4) Notification delivery uses Redis IPC:
   - It reads the user’s socket metadata from Redis to discover which **server instance IDs** currently host that user’s sockets.
   - It enqueues the JSON payload onto a **Redis list queue per server** (IPC).
5) Every server runs a background listener that consumes *its* queue and delivers the payload to local sockets only.

Practical implications:

- Redis does **not** automatically sync Belgium ↔ Mumbai. Whether “Redis routes between regions” depends entirely on deployment.
- In the current systemd deployment in this repo, `REDIS_ADDR` is not set, so Redis defaults to `127.0.0.1:6379` on each host.
  - That means Redis IPC/presence is **local to each server** (Belgium has its own Redis; Mumbai has its own Redis).
  - Cross-region realtime delivery therefore primarily relies on **Sacred Timeline replication**: both servers see the same replicated events and each emits its own local websocket pushes.
- If the user is connected to neither server, the push is best-effort and is dropped (no sockets to write to).
- **Redis is currently required by the notifier path** because the notifier routes via Redis IPC even for local delivery (unless you add a direct in-process fast-path).

If you removed Redis entirely, you’d need a replacement for:

- local IPC delivery (the notifier currently enqueues to the local server queue)
- optionally cross-server routing/presence *if* you want to deliver across multiple processes/instances without relying on Sacred Timeline triggers

With only two servers you *could* replace Redis IPC with a simple “broadcast to both servers” via HTTP/gRPC, but you’d still be reintroducing an IPC mechanism and service discovery in a different form.

### Connection Limits

| Limit | Value |
|-------|-------|
| Read limit | 1 MB per message |
| Read timeout | 90 seconds |
| Write timeout | 10 seconds |
| Max connections per user | 5 |

### Reconnection

- If connection drops, client should reconnect after 1-5 seconds
- Use exponential backoff for repeated failures
- Maximum reconnection delay: 30 seconds

---

## Rate Limiting

All APIs are subject to IP-based rate limiting:

### Default Limits

| Category | Limit |
|----------|-------|
| General API calls | 100 requests/minute |
| Login attempts | 10 attempts/5 minutes |
| Password reset | 3 requests/hour |
| Search queries | 30 requests/minute |
| WebSocket connections | 5 connections/user |

### Rate Limit Response

When rate limited, APIs return:

```json
{
  "p1": "Fail",
  "p2": "rate limited - try again later"
}
```

HTTP status code: `429 Too Many Requests`

---

## Error Handling

### Standard Error Response

```json
{
  "p1": "Fail",
  "p2": "error description"
}
```

### Common Error Messages

| Message | Description |
|---------|-------------|
| `"login failed"` | Invalid credentials |
| `"token expired"` | Session expired, re-login required |
| `"not authorized"` | Permission denied |
| `"not found"` | Resource doesn't exist |
| `"rate limited"` | Too many requests |
| `"validation failed"` | Invalid input data |
| `"server error"` | Internal error (check logs) |

---

## API Versioning

The API uses implicit versioning through the request field:

```json
{
  "req": "API_ProfileBasic"
}
```

Breaking changes result in new API names (e.g., `API_ProfileBasicV2`).

---

## CORS

The API supports CORS for browser-based clients:

| Header | Value |
|--------|-------|
| `Access-Control-Allow-Origin` | `*` or requesting origin |
| `Access-Control-Allow-Methods` | `GET, POST, OPTIONS` |
| `Access-Control-Allow-Headers` | `Content-Type, X-Lexcv-User, X-Lexcv-Token` |
| `Access-Control-Max-Age` | `86400` |

---

## Content Types

### Request

```
Content-Type: application/json
```

### Response

```
Content-Type: application/json
```

All API communication uses JSON encoding for RObj structures.
