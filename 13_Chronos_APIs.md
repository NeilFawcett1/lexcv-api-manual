# Chronos APIs

All Chronos endpoints are `POST` and use the standard `RObj` envelope.

This manual includes:
- Implemented APIs (live in code now)
- Planned Chronos APIs (contract-first design)

Chronos hard canon:
- All time fields ending in `_ms` are Unix epoch milliseconds (UTC)
- Event time ranges are `[start_at_ms, end_at_ms)` with `end_at_ms > start_at_ms`

## Common

### Authentication
Use standard login envelope:
- `p1`: user login identifier
- `p2`: session/server token

### Success / failure
- `p1 = "Success"` on success
- `p1 = "Fail"` and `p2` error message on failure

### UUIDs
All UUID fields are canonical UUID strings.

### Time zone allowlist
Chronos accepts only the following timezone strings for `time_zone` and `event_tz`.
Aliases like `GMT` are rejected.

- `UTC`
- `Europe/London`
- `Europe/Dublin`
- `Europe/Lisbon`
- `Europe/Paris`
- `Europe/Berlin`
- `Europe/Amsterdam`
- `Europe/Brussels`
- `Europe/Zurich`
- `Europe/Madrid`
- `Europe/Rome`
- `Europe/Vienna`
- `Europe/Prague`
- `Europe/Warsaw`
- `Europe/Stockholm`
- `Europe/Copenhagen`
- `Europe/Helsinki`
- `Europe/Athens`
- `Europe/Bucharest`
- `Europe/Kyiv`
- `Europe/Istanbul`
- `America/New_York`
- `America/Chicago`
- `America/Denver`
- `America/Los_Angeles`
- `America/Phoenix`
- `America/Toronto`
- `America/Vancouver`
- `America/Mexico_City`
- `America/Bogota`
- `America/Lima`
- `America/Santiago`
- `America/Sao_Paulo`
- `America/Argentina/Buenos_Aires`
- `Asia/Dubai`
- `Asia/Riyadh`
- `Asia/Jerusalem`
- `Asia/Tehran`
- `Asia/Karachi`
- `Asia/Kolkata`
- `Asia/Dhaka`
- `Asia/Bangkok`
- `Asia/Singapore`
- `Asia/Jakarta`
- `Asia/Manila`
- `Asia/Hong_Kong`
- `Asia/Shanghai`
- `Asia/Taipei`
- `Asia/Seoul`
- `Asia/Tokyo`
- `Australia/Sydney`
- `Australia/Melbourne`
- `Australia/Brisbane`
- `Australia/Perth`
- `Pacific/Auckland`
- `Pacific/Fiji`
- `Africa/Cairo`
- `Africa/Johannesburg`
- `Africa/Nairobi`
- `Africa/Lagos`

### Calendar row contract (canonical)
Whenever a calendar row is returned, the row shape is:
- `[0]` `calendar_uuid`
- `[1]` `request_uuid`
- `[2]` `owner_user_uuid`
- `[3]` `group_uuid` (empty when none)
- `[4]` `name`
- `[5]` `description`
- `[6]` `time_zone` (Chronos allowlist; see Time zone allowlist)
- `[7]` `color_hex`
- `[8]` `is_primary` (`"0"|"1"`)
- `[9]` `visibility` (`private|shared|public_link`)
- `[10]` `status` (`active|archived|deleted`)
- `[11]` `version` (int string)
- `[12]` `created_at_ms`
- `[13]` `updated_at_ms`

---

## Implemented APIs

### Endpoint Paths
All implemented Chronos APIs are available on both path styles:
- `/api/chronos/<API_CODE>`
- `/api/<API_CODE>` (back-compat alias)

## CAPI_CreateCalendar
Purpose:
- Create a user-owned calendar.
- Emit Chronos create event through transactional outbox in the same DB transaction.

Authorization:
- Caller must be logged in.
- Calendar owner is always the authenticated user.

Request (`p3`):
- `[0]` `name` (required)
- `[1]` `description` (optional)
- `[2]` `time_zone` (optional, default `UTC`; must be in Time zone allowlist)
- `[3]` `color_hex` (optional; normalized to `#RRGGBB`; defaults to `#1F7A8C`)
- `[4]` `is_primary` (optional; truthy values become `1`, else `0`)
- `[5]` `visibility` (optional; `private|shared|public_link`; defaults to `private`)
- `[6]` `group_uuid` (optional)
- `[7]` `request_uuid` (required idempotency key)

Server behavior details:
- Name is sanitized and truncated to 200 chars.
- Description is sanitized and truncated to 4000 chars.
- Per-user active calendar max is enforced by dynamic config key `chronos_calendars_max_per_user`.

Response:
- `p1`: `Success|Fail`
- `p2`: error message on failure
- `p3`: single calendar row using canonical calendar row contract

Idempotency/replay semantics:
- Duplicate create request with same `(owner_user_uuid, request_uuid)` returns the existing row.
- Write + outbox are atomic (single DB transaction).

Common failure messages:
- `calendar name is required`
- `request_uuid is required`
- `unsupported time_zone; use one of the Chronos accepted time zones`
- `calendar limit reached (<n>)`

Example request:
```json
{
  "req": "CAPI_CreateCalendar",
  "p1": "user@example.com",
  "p2": "token",
  "p3": [
    "Work",
    "Main work calendar",
    "Europe/Brussels",
    "#1F7A8C",
    "1",
    "private",
    "",
    "2f6ff8b2-3ab2-4f48-9557-9cb9e94ad3db"
  ]
}
```

Example success response:
```json
{
  "p1": "Success",
  "p3": [
    "cal-uuid",
    "2f6ff8b2-3ab2-4f48-9557-9cb9e94ad3db",
    "user-uuid",
    "",
    "Work",
    "Main work calendar",
    "Europe/Brussels",
    "#1F7A8C",
    "1",
    "private",
    "active",
    "1",
    "1730000000000",
    "1730000000000"
  ]
}
```

## CAPI_ListCalendars
Purpose:
- List calendars owned by authenticated user.

Authorization:
- Caller must be logged in.
- Returns only owner calendars for current user.

Request (`p3`, optional):
- `[0]` `offset` (0-based)
- `[1]` `limit` (server-capped)

Server behavior details:
- Negative `offset` becomes `0`.
- Non-positive `limit` becomes server default.
- Max page size uses dynamic config key `chronos_calendars_list_max_per_page`.
- If `offset > total`, effective offset is clamped to `total`.
- Ordering is `is_primary DESC, updated_at_ms DESC`.

Response:
- `p1`: `Success|Fail`
- `p2`: error message on failure
- `p3[0]`: total rows
- `p3[1]`: effective offset
- `p3[2]`: effective limit
- `p4`: array of calendar rows using canonical calendar row contract

Example response shape:
```json
{
  "p1": "Success",
  "p3": ["2", "0", "20"],
  "p4": [
    ["cal-uuid-1", "req-uuid-1", "user-uuid", "", "Work", "...", "UTC", "#1F7A8C", "1", "private", "active", "1", "1730000000000", "1730000000000"],
    ["cal-uuid-2", "req-uuid-2", "user-uuid", "", "Family", "...", "UTC", "#4CAF50", "0", "shared", "active", "1", "1730000001000", "1730000001000"]
  ]
}
```

## CAPI_PatchCalendar
Status:
- Implemented.

Purpose:
- Update mutable calendar fields.
- Emit Chronos update event through transactional outbox.

Authorization:
- Owner OR Cato relation `owner|admin|writer` on calendar.

Request (`p3`):
- `[0]` `calendar_uuid` (required)
- `[1]` `name` (optional)
- `[2]` `description` (optional)
- `[3]` `time_zone` (optional)
- `[4]` `color_hex` (optional)
- `[5]` `visibility` (optional)
- `[6]` `status` (optional: `active|archived|deleted`)
- `[7]` `version` (required optimistic concurrency)
- `[8]` `request_uuid` (required by API contract)

Server behavior details:
- `name` max 200, `description` max 4000, `time_zone` allowlist enforced.
- `status` must be one of `active|archived|deleted` if provided.
- If resulting status is `deleted`, `deleted_at_ms` is set.
- If row is not updated and current state already matches requested values, API returns success (replay-safe no-op).
- If row is not updated and state differs, API returns `version conflict`.

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3`: updated calendar row

Replay/idempotency semantics:
- Write is emitted via Sacred Timeline transactional outbox.
- Replays are safe via optimistic concurrency (`version`) plus no-op state comparison.

Common failure messages:
- `calendar_uuid is required`
- `version is required`
- `calendar not found`
- `permission denied`
- `version conflict`

## CAPI_DeleteCalendar
Status:
- Implemented.

Purpose:
- Soft delete calendar by setting status to `deleted`.
- Emit Chronos delete event through transactional outbox.

Authorization:
- Owner OR Cato relation `owner|admin|writer` on calendar.

Request (`p3`):
- `[0]` `calendar_uuid` (required)
- `[1]` `request_uuid` (required)

Server behavior details:
- If calendar already deleted, returns success with existing deleted state (replay-safe).
- No-op replays do not emit duplicate state changes.

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3[0]`: `calendar_uuid`
- `p3[1]`: resulting `status` (`deleted`)
- `p3[2]`: `deleted_at_ms`

Replay/idempotency semantics:
- Write is emitted via Sacred Timeline transactional outbox.
- Replays are safe because delete query only applies when status is not already `deleted`.

Common failure messages:
- `calendar_uuid is required`
- `request_uuid is required`
- `calendar not found`
- `permission denied`

## CAPI_CreateEvent
Status:
- Implemented.

Purpose:
- Create an event in a calendar.
- Emit Chronos event-create outbox event transactionally.

Authorization:
- Calendar owner OR Cato relation `owner|admin|writer` on calendar.

Request (`p3`):
- `[0]` `calendar_uuid` (required)
- `[1]` `title` (required)
- `[2]` `description` (optional)
- `[3]` `location` (optional)
- `[4]` `start_at_ms` (required epoch ms)
- `[5]` `end_at_ms` (required epoch ms)
- `[6]` `all_day` (`0|1`, optional)
- `[7]` `event_tz` (optional, default `UTC`)
- `[8]` `visibility` (`default|public|private|confidential`, optional)
- `[9]` `transparency` (`opaque|transparent`, optional)
- `[10]` `recurrence_rrule` (optional)
- `[11]` `recurrence_exdates_json` (optional JSON string)
- `[12]` `recurring_event_uuid` (optional)
- `[13]` `original_start_at_ms` (optional epoch ms)
- `[14]` `metadata_json` (optional JSON string)
- `[15]` `request_uuid` (required idempotency)

Server behavior details:
- `start_at_ms` and `end_at_ms` are validated as Chronos epoch ms range with `end_at_ms > start_at_ms`.
- `title` max 255, `description` max 4000, `location` max 255 after sanitization.
- Empty optional JSON fields are persisted as `NULL`.

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3`: event core row

Replay/idempotency semantics:
- Create writes use transactional outbox.
- Request idempotency key is `request_uuid` (unique per creator+calendar in schema).

Common failure messages:
- `calendar_uuid is required`
- `title is required`
- `start_at_ms is required`
- `end_at_ms is required`
- `request_uuid is required`
- `calendar not found`
- `permission denied`

Event row contract (`p3`):
- `[0]` `event_uuid`
- `[1]` `calendar_uuid`
- `[2]` `ical_uid`
- `[3]` `sequence_int`
- `[4]` `status`
- `[5]` `title`
- `[6]` `description`
- `[7]` `location`
- `[8]` `start_at_ms`
- `[9]` `end_at_ms`
- `[10]` `all_day`
- `[11]` `event_tz`
- `[12]` `transparency`
- `[13]` `visibility`
- `[14]` `recurrence_rrule`
- `[15]` `recurrence_exdates_json`
- `[16]` `recurring_event_uuid`
- `[17]` `original_start_at_ms`
- `[18]` `metadata_json`
- `[19]` `version`
- `[20]` `created_by_uuid`
- `[21]` `updated_by_uuid`
- `[22]` `created_at_ms`
- `[23]` `updated_at_ms`

## CAPI_PatchEvent
Status:
- Implemented.

Purpose:
- Update mutable fields on an existing event.
- Emit Chronos event-update outbox event transactionally when a mutation occurs.

Authorization:
- Calendar owner OR Cato relation `owner|admin|writer` on calendar.

Request (`p3`):
- `[0]` `event_uuid` (required)
- `[1]` `calendar_uuid` (required)
- `[2]` `title` (optional)
- `[3]` `description` (optional)
- `[4]` `location` (optional)
- `[5]` `start_at_ms` (optional)
- `[6]` `end_at_ms` (optional)
- `[7]` `all_day` (optional)
- `[8]` `event_tz` (optional)
- `[9]` `visibility` (optional)
- `[10]` `transparency` (optional)
- `[11]` `recurrence_rrule` (optional)
- `[12]` `recurrence_exdates_json` (optional)
- `[13]` `metadata_json` (optional)
- `[14]` `version` (required)
- `[15]` `request_uuid` (required)

Server behavior details:
- `event_uuid` must belong to provided `calendar_uuid`.
- `version` is optimistic concurrency guard.
- If DB update affects 0 rows and current row already equals requested state, returns success as replay-safe no-op.
- If DB update affects 0 rows and state differs, returns `version conflict`.

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3`: updated event row

Replay/idempotency semantics:
- Patch writes use transactional outbox.
- Replays are safe via optimistic concurrency (`version`) and state compare on no-op retries.

Common failure messages:
- `event_uuid is required`
- `calendar_uuid is required`
- `version is required`
- `event not found`
- `event does not belong to calendar`
- `permission denied`
- `version conflict`

## CAPI_DeleteEvent
Status:
- Implemented.

Purpose:
- Soft delete event by setting status to `cancelled`.
- Emit Chronos event-delete outbox event transactionally when state changes.

Authorization:
- Calendar owner OR Cato relation `owner|admin|writer` on calendar.

Request (`p3`):
- `[0]` `event_uuid` (required)
- `[1]` `calendar_uuid` (required)
- `[2]` `request_uuid` (required)

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3[0]`: `event_uuid`
- `p3[1]`: resulting `status` (`cancelled`)
- `p3[2]`: `updated_at_ms`

Server behavior details:
- If event already `cancelled`, returns success with existing state (replay-safe).
- If event is active, state changes to `cancelled` and outbox event is inserted in same transaction.

Replay/idempotency semantics:
- Delete writes use transactional outbox.
- Replays are safe because delete only applies when status is not already `cancelled`.

Common failure messages:
- `event_uuid is required`
- `calendar_uuid is required`
- `request_uuid is required`
- `event not found`
- `event does not belong to calendar`
- `permission denied`

## CAPI_ListEvents
Status:
- Implemented.

Purpose:
- List events for multiple calendars in one call.
- Return independent pagination state per calendar.
- Best fit for clients that render calendar-specific lanes, tabs, or grouped sections.

Authorization:
- Caller must be able to view each calendar (`owner` or Cato relation `owner|admin|writer|reader|freebusy`).

Request (`p3`):
- `[0]` `range_start_ms` (required)
- `[1]` `range_end_ms` (required)
- `[2]` `default_limit_per_calendar` (optional, capped to `200`, default `20`)

Request (`p4`):
- One row per calendar page request.
- Row `[0]`: `calendar_uuid` (required)
- Row `[1]`: `offset` (optional)
- Row `[2]`: `limit` (optional, capped to `200`)
- If `p4` is empty, server scopes to all active calendars owned by caller and starts each with offset `0`.

Server behavior details:
- Event selection uses overlap semantics against request range: `start_at_ms < range_end_ms` AND `end_at_ms > range_start_ms`.
- Event status filter: only non-`cancelled` events.
- Sorting per calendar: `start_at_ms ASC, uuid ASC`.
- Duplicate calendar UUIDs in request are deduplicated (first occurrence wins).
- Negative offset becomes `0`.
- If offset exceeds total, effective offset is clamped to total.
- Unknown, deleted, invalid UUID, or unauthorized calendars are silently skipped.

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3[0]`: count of visible calendar groups returned
- `p6`: grouped calendar results (list-of-lists), each item is an `RObj`
- Child `RObj.p3[0]`: `calendar_uuid`
- Child `RObj.p3[1]`: `total_in_range`
- Child `RObj.p3[2]`: `effective_offset`
- Child `RObj.p3[3]`: `effective_limit`
- Child `RObj.p3[4]`: `has_more` (`0|1`)
- Child `RObj.p4`: event rows for that calendar page, using event row contract

Authorization/filtering:
- Results are returned only for calendars the caller can view (`owner` or Cato view relations).
- Unknown/unauthorized/deleted calendars are filtered out.

Example request:
```json
{
  "req": "CAPI_ListEvents",
  "p1": "user@example.com",
  "p2": "token",
  "p3": ["1733011200000", "1735693200000", "25"],
  "p4": [
    ["calendar-a-uuid", "0", "25"],
    ["calendar-b-uuid", "50", "10"]
  ]
}
```

Example success response shape:
```json
{
  "p1": "Success",
  "p3": ["2"],
  "p6": [
    {
      "p3": ["calendar-a-uuid", "41", "0", "25", "1"],
      "p4": [
        ["event-1", "calendar-a-uuid", "event-1", "0", "confirmed", "Title", "", "", "1733200000000", "1733203600000", "0", "UTC", "opaque", "default", "", "", "", "0", "", "1", "creator-uuid", "creator-uuid", "1733190000000", "1733190000000"]
      ]
    },
    {
      "p3": ["calendar-b-uuid", "88", "50", "10", "1"],
      "p4": []
    }
  ]
}
```

Common failure messages:
- `range_start_ms and range_end_ms are required`
- `invalid range_start_ms`
- `invalid range_end_ms`

## CAPI_ListEventsUnified
Status:
- Implemented.

Purpose:
- Return a single merged event feed across calendar scope.
- Best fit for infinite-scroll month/week agenda views.

Authorization:
- Caller must be logged in.
- Calendar scope is filtered by view permissions (`owner|admin|writer|reader|freebusy`).

Request (`p3`):
- `[0]` `range_start_ms` (required)
- `[1]` `range_end_ms` (required)
- `[2]` `limit` (optional, default `100`, max `300`)
- `[3]` `cursor_start_at_ms` (optional, keyset cursor)
- `[4]` `cursor_event_uuid` (optional, keyset tie-breaker)

Request (`p4`):
- Optional calendar scope rows.
- Row `[0]`: `calendar_uuid`
- If `p4` is empty, scope defaults to all active calendars owned by caller.

Server behavior details:
- Range overlap filter: `start_at_ms < range_end_ms` AND `end_at_ms > range_start_ms`.
- Only non-cancelled events are returned.
- Sort order is globally merged: `start_at_ms ASC, uuid ASC`.
- Keyset pagination resumes strictly after `(cursor_start_at_ms, cursor_event_uuid)`.
- Server fetches `limit + 1` rows internally to compute `has_more`.

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3[0]`: returned event count
- `p3[1]`: `has_more` (`0|1`)
- `p3[2]`: `next_cursor_start_at_ms` (empty if none)
- `p3[3]`: `next_cursor_event_uuid` (empty if none)
- `p3[4]`: effective limit
- `p3[5]`: calendar scope count after permission filtering
- `p4`: merged event rows using event row contract

Common failure messages:
- `range_start_ms and range_end_ms are required`
- `invalid range_start_ms`
- `invalid range_end_ms`
- `invalid cursor_start_at_ms`

Client integration pattern (infinite vertical month scroll):
- Use a fixed initial window (for example, current month start to +90 days).
- First request uses empty cursor fields.
- While user scrolls near bottom, request next page with returned cursor.
- Append rows to in-memory feed and dedupe by `event_uuid`.
- When `has_more=0`, extend date window forward (for example +60/+90 days), reset cursor, and continue.

Recommended request cadence:
- Keep one in-flight request at a time for this feed.
- Trigger next page when viewport reaches ~70% of current loaded items.
- Use `limit` in range 80-150 for smooth UX (server max 300).

Cursor loop pseudocode:
```javascript
let rangeStart = monthStartUtcMs(now)
let rangeEnd = addDays(rangeStart, 90)
let cursorStart = ""
let cursorEvent = ""
let hasMore = true
const seen = new Set() // event_uuid
const feed = []

async function loadNextPage() {
  if (!hasMore) {
    rangeStart = rangeEnd
    rangeEnd = addDays(rangeEnd, 90)
    cursorStart = ""
    cursorEvent = ""
    hasMore = true
  }

  const req = {
    req: "CAPI_ListEventsUnified",
    p1: authUser,
    p2: authToken,
    p3: [
      String(rangeStart),
      String(rangeEnd),
      "120",
      cursorStart,
      cursorEvent
    ],
    p4: [] // optional: provide specific calendar UUID rows if needed
  }

  const res = await postRObj(req)
  if (res.p1 !== "Success") throw new Error(res.p2 || "ListEventsUnified failed")

  const rows = res.p4 || []
  for (const row of rows) {
    const eventUUID = row[0]
    if (!seen.has(eventUUID)) {
      seen.add(eventUUID)
      feed.push(row)
    }
  }

  hasMore = (res.p3?.[1] === "1")
  cursorStart = res.p3?.[2] || ""
  cursorEvent = res.p3?.[3] || ""
}
```

React hook pattern (race-safe):
```javascript
const [feed, setFeed] = useState([])
const [loading, setLoading] = useState(false)
const [hasMore, setHasMore] = useState(true)
const [rangeStart, setRangeStart] = useState(monthStartUtcMs(Date.now()))
const [rangeEnd, setRangeEnd] = useState(addDays(monthStartUtcMs(Date.now()), 90))
const cursorRef = useRef({ start: "", event: "" })
const seenRef = useRef(new Set())
const reqSeqRef = useRef(0)
const abortRef = useRef(null)

async function loadNext() {
  if (loading) return
  setLoading(true)

  const mySeq = ++reqSeqRef.current
  if (abortRef.current) abortRef.current.abort()
  abortRef.current = new AbortController()

  try {
    let start = rangeStart
    let end = rangeEnd
    if (!hasMore) {
      start = end
      end = addDays(end, 90)
      setRangeStart(start)
      setRangeEnd(end)
      cursorRef.current = { start: "", event: "" }
      setHasMore(true)
    }

    const req = {
      req: "CAPI_ListEventsUnified",
      p1: authUser,
      p2: authToken,
      p3: [
        String(start),
        String(end),
        "120",
        cursorRef.current.start,
        cursorRef.current.event
      ],
      p4: []
    }

    const res = await postRObj(req, { signal: abortRef.current.signal })
    if (mySeq !== reqSeqRef.current) return // ignore stale response
    if (res.p1 !== "Success") throw new Error(res.p2 || "ListEventsUnified failed")

    const rows = res.p4 || []
    const append = []
    for (const row of rows) {
      const eventUUID = row[0]
      if (!seenRef.current.has(eventUUID)) {
        seenRef.current.add(eventUUID)
        append.push(row)
      }
    }
    if (append.length > 0) setFeed(prev => [...prev, ...append])

    const nextHasMore = res.p3?.[1] === "1"
    cursorRef.current = {
      start: res.p3?.[2] || "",
      event: res.p3?.[3] || ""
    }
    setHasMore(nextHasMore)
  } finally {
    if (mySeq === reqSeqRef.current) setLoading(false)
  }
}
```

React-specific tips:
- Call `loadNext()` from an `IntersectionObserver` sentinel near list bottom.
- Reset `feed`, `seenRef`, cursor, and request sequence when major filters or timezone context change.
- Keep server sort order; do not re-sort appended pages client-side.

Notes for month UI rendering:
- Group feed rows by local day using `start_at_ms`/`end_at_ms` after fetch.
- Keep the canonical server order intact for stable scroll append.
- Recurrence instances are not expanded server-side in this endpoint; if recurrence expansion is needed, handle expansion in client or add a dedicated expansion API.

---

## Planned APIs (Contract)

Note:
- These are design contracts for implementation.
- Authorization decisions are Cato-backed.

## CAPI_GetCalendar
Request (`p3`):
- `[0]` `calendar_uuid` (required)

Response:
- `p1`: `Success|Fail`
- `p2`: error message
- `p3`: calendar row (canonical contract)

## CAPI_ListSharedCalendars
Request (`p3`):
- `[0]` `offset` (optional, default `0`)
- `[1]` `limit` (optional, server-capped)

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3[0]`: total
- `p3[1]`: offset
- `p3[2]`: limit
- `p4`: calendar rows visible by Cato permissions

## CAPI_GetEvent
Request (`p3`):
- `[0]` `event_uuid` (required)

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3`: event row contract
- `p4[0]`: attendees rows

Attendee row contract:
- `[0]` `attendee_uuid`
- `[1]` `event_uuid`
- `[2]` `attendee_user_uuid`
- `[3]` `attendee_email` (required)
- `[4]` `display_name`
- `[5]` `comment_text`
- `[6]` `additional_guests`
- `[7]` `response_status`
- `[8]` `is_optional`
- `[9]` `is_organizer`
- `[10]` `is_resource`
- `[11]` `is_self`
- `[12]` `created_at_ms`
- `[13]` `updated_at_ms`

## CAPI_SetEventAttendees
Request:
- `p3[0]`: `event_uuid`
- `p3[1]`: `request_uuid`
- `p4`: attendee rows to upsert

Attendee upsert row (`p4[i]`):
- `[0]` `attendee_user_uuid` (optional)
- `[1]` `attendee_email` (required)
- `[2]` `display_name` (optional)
- `[3]` `comment_text` (optional)
- `[4]` `additional_guests` (optional, default `0`)
- `[5]` `response_status` (`needsAction|accepted|tentative|declined`)
- `[6]` `is_optional` (`0|1`)
- `[7]` `is_organizer` (`0|1`)
- `[8]` `is_resource` (`0|1`, optional)
- `[9]` `is_self` (`0|1`, optional)

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3[0]`: `event_uuid`
- `p3[1]`: attendee count

## CAPI_WorkgroupCreate
Request (`p3`):
- `[0]` `name` (required)
- `[1]` `description` (optional)
- `[2]` `request_uuid` (required)

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3`: workgroup row (`group_uuid`, owner/company uuid, name, description, status, created_at_ms, updated_at_ms)

## CAPI_WorkgroupPatch
Request (`p3`):
- `[0]` `group_uuid` (required)
- `[1]` `name` (optional)
- `[2]` `description` (optional)
- `[3]` `status` (optional)
- `[4]` `version` (required)
- `[5]` `request_uuid` (required)

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3`: updated workgroup row

## CAPI_WorkgroupMemberAdd
Request (`p3`):
- `[0]` `group_uuid`
- `[1]` `user_uuid`
- `[2]` `role` (`manager|member`)
- `[3]` `request_uuid`

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3`: membership row (`membership_uuid`, `group_uuid`, `user_uuid`, `role`, `status`, `created_at_ms`, `updated_at_ms`)

## CAPI_WorkgroupMemberRemove
Request (`p3`):
- `[0]` `group_uuid`
- `[1]` `user_uuid`
- `[2]` `request_uuid`

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3[0]`: `group_uuid`
- `p3[1]`: `user_uuid`
- `p3[2]`: resulting status (`removed`)

## CAPI_ResourceCreate
Request (`p3`):
- `[0]` `resource_type` (`room|equipment|vehicle|other`)
- `[1]` `name`
- `[2]` `description`
- `[3]` `capacity` (optional int)
- `[4]` `metadata_json` (optional)
- `[5]` `request_uuid`

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3`: resource row

## CAPI_ResourcePatch
Request (`p3`):
- `[0]` `resource_uuid`
- `[1]` `name` (optional)
- `[2]` `description` (optional)
- `[3]` `capacity` (optional)
- `[4]` `status` (`active|maintenance|retired`, optional)
- `[5]` `metadata_json` (optional)
- `[6]` `version` (required)
- `[7]` `request_uuid` (required)

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3`: updated resource row

## CAPI_ResourceCalendarAttach
Request (`p3`):
- `[0]` `resource_uuid`
- `[1]` `calendar_uuid`
- `[2]` `is_primary` (`0|1`)
- `[3]` `request_uuid`

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3`: mapping row (`mapping_uuid`, `resource_uuid`, `calendar_uuid`, `is_primary`, `created_at_ms`)

## CAPI_ResourceList
Request (`p3`):
- `[0]` `resource_type` (optional filter)
- `[1]` `offset` (optional)
- `[2]` `limit` (optional)

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3[0]`: total
- `p3[1]`: offset
- `p3[2]`: limit
- `p4`: resource rows

## CAPI_FreeBusyQuery
Request (`p3`):
- `[0]` `range_start_at_ms`
- `[1]` `range_end_at_ms`
- `[2]` `granularity_minutes` (optional)

Request (`p4`):
- each row is principal selector:
  - `[0]` `principal_type` (`user|group|resource|calendar`)
  - `[1]` `principal_uuid`

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p4`: busy interval rows:
  - `[0]` `principal_type`
  - `[1]` `principal_uuid`
  - `[2]` `busy_start_at_ms`
  - `[3]` `busy_end_at_ms`

## CAPI_CompositeView
Request (`p3`):
- `[0]` `range_start_at_ms`
- `[1]` `range_end_at_ms`

Request (`p4`):
- selectors (`calendar_uuid` rows or principal rows, implementation choice)

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p4[0]`: raw event rows
- `p4[1]`: merged busy interval rows
- `p4[2]`: conflict rows

## CAPI_SuggestSlots
Request (`p3`):
- `[0]` `range_start_at_ms`
- `[1]` `range_end_at_ms`
- `[2]` `duration_minutes`
- `[3]` `max_results` (optional)

Request (`p4`):
- participant/resource selectors

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p4`: suggestion rows:
  - `[0]` `slot_start_at_ms`
  - `[1]` `slot_end_at_ms`
  - `[2]` `score`
  - `[3]` `reason_codes_json`

## CAPI_ConflictCheck
Request (`p3`):
- `[0]` `candidate_start_at_ms`
- `[1]` `candidate_end_at_ms`

Request (`p4`):
- participant/resource selectors

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3[0]`: hard conflict count
- `p3[1]`: soft conflict count
- `p4`: conflict rows:
  - `[0]` `conflict_type` (`hard|soft|policy`)
  - `[1]` `entity_type`
  - `[2]` `entity_uuid`
  - `[3]` `event_uuid`
  - `[4]` `start_at_ms`
  - `[5]` `end_at_ms`

---

## Cato-backed permission model (Chronos)

Chronos authorization is Cato-backed.

Role semantics (resource-scoped in Cato):
- `owner`
- `admin`
- `writer`
- `reader`
- `freebusy`

Expected checks by API category:
- Calendar create/list mine: owner context on own resources
- Shared reads: `reader` or `freebusy` depending endpoint
- Event mutations: `writer` or above
- Calendar admin actions: `admin` or `owner`

Chronos does not require a local ACL authority table for permissions.

---

## Versioning

- `2026-03-15`: initial Chronos API manual created as chapter 13.

