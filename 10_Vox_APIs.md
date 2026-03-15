# Vox APIs

APIs for the Vox notification inbox.

---

## Overview

Vox is LexCV’s canonical **user notification inbox**.

Key properties:

- Vox is **generic**: a Vox event can represent any user-visible event in the system (jobs, posts, messages, admin actions, etc.).
- Vox is **inbox-only (v1)**: clients list events via API calls (no WebSocket/push required for correctness).
- Vox is **replicated** using the **Sacred Timeline transactional outbox**.

---

## Canon & Design

### Canonical identity

- Every Vox event has a unique UUID (`uuid`).
- Replay must not duplicate events (inserts are idempotent).

### Canonical naming (`api_code`)

- Vox uses the existing API language as its canonical event naming scheme.
- `api_code` should typically be an existing API code string.

### Canonical subject (`subject_uuid`)

`api_code` answers: **what happened**.

`subject_uuid` answers: **what is this event about** (the primary entity UUID).

Examples:

- A reaction notification:
  - `api_code = "API_PostAddReact"`
  - `subject_uuid = <post_uuid>`
- A new message notification:
  - `api_code = "API_MessageSend"`
  - `subject_uuid = <conversation_uuid>`

`subject_uuid` is optional because not every Vox event has a single obvious primary entity.

### Canonical time (`created_at`)

- `created_at` is **unix epoch milliseconds**.
- UI ordering is newest-first.

### Generic payload (`payload_json`)

- `payload_json` is optional JSON for lightweight client rendering.
- Payload must remain **small** and **non-sensitive**.
- If the client needs more information, it should fetch the relevant entity via its dedicated API.

---

## Data Model (MySQL)

Table: `vox_events`

| Column | Type | Description |
|---|---|---|
| `uuid` | `CHAR(36)` PK | Vox event UUID |
| `user_uuid` | `CHAR(36)` | Recipient user UUID |
| `api_code` | `VARCHAR(100)` | Canonical event name (API code language) |
| `created_at` | `BIGINT` | Unix ms |
| `actor_uuid` | `CHAR(36) NULL` | Optional actor UUID (avoids parsing `payload_json` for common filters) |
| `subject_uuid` | `CHAR(36) NULL` | Optional subject UUID (primary entity this event is about) |
| `payload_json` | `JSON NULL` | Optional payload |
| `sent` | `TINYINT` | Push flag (0=not pushed, 1=pushed) |
| `read_at` | `BIGINT NULL` | Unix ms read receipt |
| `deleted` | `TINYINT` | Soft delete flag |
| `deleted_at` | `BIGINT NULL` | Unix ms deletion time |

Recommended indexes (as implemented in schema):

- (`user_uuid`, `created_at`)
- (`user_uuid`, `read_at`)
- (`user_uuid`, `deleted`)

---

## Broadcasts (Stateless)

Vox also supports **global broadcasts** (system announcements) that are delivered to all users without per-user fanout.

Broadcasts are **stateless**:

- No per-user read receipts
- No deletion
- Clients may locally decide when a broadcast is considered “read” (e.g. after 24 hours)

### Data Model (MySQL)

Table: `vox_broadcasts`

| Column | Type | Description |
|---|---|---|
| `uuid` | `CHAR(36)` PK | Broadcast UUID |
| `api_code` | `VARCHAR(100)` | Typically `Vox_Broadcast` |
| `created_at` | `BIGINT` | Unix ms |
| `actor_uuid` | `CHAR(36) NULL` | Optional actor UUID |
| `subject_uuid` | `CHAR(36) NULL` | Optional subject UUID (primary entity this broadcast is about) |
| `delivery` | `JSON` | JSON array of delivery flags (e.g. `["Vox_All","Vox_Nation_UK"]`) |
| `sent` | `TINYINT` | Push flag (0=not pushed, 1=pushed) |
| `payload_json` | `JSON NULL` | Optional payload |

### Listing Behavior

`API_VoxList` returns a **merged** newest-first stream of:

- user inbox items from `vox_events`
- global broadcasts from `vox_broadcasts`

Broadcast items will appear with:

- `apiCode = "Vox_Broadcast"`
- `readAt = 0` (always; stateless)

Filtering rules (current): `API_VoxList` includes broadcasts matching any of:

- `Vox_All`
- `Vox_Nation_<X>` where X is a nation short code (user->country->nation)
- `Vox_Country_<X>` where X matches the user’s linked country name/short
- `Vox_Usertypes_<X>` where X is the user_flags user type (e.g. lawyer, company)

### State Behavior

`API_VoxState` operations (`read`, `delete`) do **not** apply to broadcasts and will fail if called with a broadcast UUID.

---

## Replication, Ordering, Replay (Sacred Timeline)

### Transactional outbox rule

All Vox mutations MUST use the transactional outbox pattern:

1. Execute the MySQL write in a transaction
2. Insert the Sacred Timeline outbox event in the same transaction

This makes Vox writes replayable and replicable across nodes.

### Partitioning rule (ordering guarantee)

To preserve per-user ordering during reset/replay:

- The outbox `aggregate_id` MUST be the recipient `user_uuid`.

### UI ordering

Vox list ordering is newest → oldest:

- `ORDER BY created_at DESC, uuid DESC`

The UUID tie-breaker ensures deterministic ordering when timestamps collide.

### Replay behavior

- Inserts are idempotent (uuid primary key + `INSERT IGNORE`).
- State updates (read/delete) are safe to replay.

---

## API_VoxList

**Purpose:** Lists the logged-in user’s Vox inbox events (paged for infinite scroll), newest first.

**Authentication:** Requires valid login (p1=email, p2=token)

**Endpoint:** `POST /api/API_VoxList`

### Request RObj

| Field | Type | Required | Description |
|---|---:|---:|---|
| `req` | string | Yes | `"API_VoxList"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | No | Limit (JSON int string; default 50; max 200) |
| `p3[1]` | string | No | Offset (JSON int string; default 0) |

### Request Example

```json
{
  "req": "API_VoxList",
  "p1": "user@email.com",
  "p2": "token",
  "p3": ["50", "0"]
}
```

### Response RObj

On success, `p3[0]` contains a JSON object:

```json
{
  "p1": "Success",
  "p3": [
    "{\"total\":123,\"hasMore\":true,\"limit\":50,\"offset\":0,\"events\":[...]}"
  ]
}
```

Where each `events[]` item contains:

- `uuid` (string)
- `apiCode` (string)
- `createdAt` (number; unix ms)
- `actorUuid` (string; empty if unknown)
- `subjectUuid` (string; empty if unknown)
- `payloadJson` (string containing JSON; defaults to `{}` when empty)
- `readAt` (number; 0 if unread)

### Notes

- Infinite scroll: request next page by incrementing `offset += limit`.
- VoxList includes both inbox events and global broadcasts (see “Broadcasts (Stateless)” above).

---

## API_VoxState

**Purpose:** Mutates state for Vox events:

- Mark a single event as read
- Delete (soft delete) a single event
- Mark **all** events as read (bulk)

Note: Broadcasts are stateless and cannot be marked read or deleted via this API.

**Authentication:** Requires valid login (p1=email, p2=token)

**Endpoint:** `POST /api/API_VoxState`

### Request RObj

| Field | Type | Required | Description |
|---|---:|---:|---|
| `req` | string | Yes | `"API_VoxState"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Conditional | Vox UUID (required for `read`/`delete`; ignored for `read_all`) |
| `p3[1]` | string | Yes | Operation: `"read"`, `"delete"`, or `"read_all"` |
| `p3[2]` | string | No | Timestamp (unix ms as JSON number string; default now) |

### Request Example

```json
{
  "req": "API_VoxState",
  "p1": "user@email.com",
  "p2": "token",
  "p3": ["vox-event-uuid", "read", "1700000000000"]
}
```

### Request Example (Mark All Read)

```json
{
  "req": "API_VoxState",
  "p1": "user@email.com",
  "p2": "token",
  "p3": ["", "read_all", "1700000000000"]
}
```

### Response Example

```json
{
  "p1": "Success"
}
```

---

## API_VoxUnreadCount

**Purpose:** Returns the logged-in user’s unread Vox inbox count (badge count).

Important: This count is **inbox-only** (rows from `vox_events`). It **excludes broadcasts** because broadcasts are stateless and have no per-user read receipts.

**Authentication:** Requires valid login (p1=email, p2=token)

**Endpoint:** `POST /api/API_VoxUnreadCount`

### Request RObj

| Field | Type | Required | Description |
|---|---:|---:|---|
| `req` | string | Yes | `"API_VoxUnreadCount"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

On success, `p3[0]` contains a JSON object:

```json
{
  "p1": "Success",
  "p3": [
    "{\"unread\":12}"
  ]
}
```

### Replication

- `API_VoxState` writes are performed using the transactional outbox so they replicate safely.

---

## Server Helpers (Canon)

These helpers live in `lexcv-server/lib/notify`.

- `VoxSend(...)`: Inserts a Vox event + writes Sacred Timeline outbox in one MySQL transaction.
- `VoxSendWithSubject(...)`: Same, but stores `subject_uuid`.
- `VoxSendWithActor(...)`: Same, but stores `actor_uuid`.
- `VoxSendWithActorAndSubject(...)`: Same, but stores both.
- `VoxBroadcast(...)`: Inserts a stateless broadcast (default delivery `Vox_All`).
- `VoxBroadcastWithDelivery(...)`: Broadcast + explicit delivery flags.
- `VoxBroadcastWithDeliveryAndActor(...)`: Broadcast + actor column.
- `VoxBroadcastWithDeliveryAndActorAndSubject(...)`: Broadcast + actor + subject columns.
- `VoxUnreadCount(...)`: Returns unread inbox count (excludes broadcasts).
- `VoxSetReadTx(...)`: Transactional read receipt update + outbox.
- `VoxSetReadAllTx(...)`: Transactional bulk mark-all-read + outbox.
- `VoxDeleteTx(...)`: Transactional soft delete update + outbox.

---

## Payload Conventions

`payload_json` is intentionally flexible.

Recommended keys (when applicable):

- `actorUuid`: UUID of the user who caused the event
- `subjectUuid`: UUID of the primary subject (post UUID, job UUID, etc.)
- `summary`: short human-readable string (optional)
- `metadata`: small map of extra fields

Notes:

- `actor_uuid` and `subject_uuid` exist as first-class MySQL columns. Prefer setting those columns for common filtering/deduping.
- Including `actorUuid`/`subjectUuid` in `payload_json` is optional and should be treated as redundant convenience for clients.

Do NOT place sensitive data in `payload_json`.
