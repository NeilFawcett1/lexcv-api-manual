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

### Calendar row contract (canonical)
Whenever a calendar row is returned, the row shape is:
- `[0]` `calendar_uuid`
- `[1]` `request_uuid`
- `[2]` `owner_user_uuid`
- `[3]` `group_uuid` (empty when none)
- `[4]` `name`
- `[5]` `description`
- `[6]` `time_zone` (IANA)
- `[7]` `color_hex`
- `[8]` `is_primary` (`"0"|"1"`)
- `[9]` `visibility` (`private|shared|public_link`)
- `[10]` `status` (`active|archived|deleted`)
- `[11]` `version` (int string)
- `[12]` `created_at_ms`
- `[13]` `updated_at_ms`

---

## Implemented APIs

## CAPI_CreateCalendar
Purpose:
- Create a user-owned calendar with transactional outbox + Sacred Timeline event.

Request (`p3`):
- `[0]` `name` (required)
- `[1]` `description` (optional)
- `[2]` `time_zone` (optional, default `UTC`)
- `[3]` `color_hex` (optional, default `#1F7A8C`)
- `[4]` `is_primary` (optional, `0|1`, default `0`)
- `[5]` `visibility` (optional, `private|shared|public_link`, default `private`)
- `[6]` `group_uuid` (optional)
- `[7]` `request_uuid` (required, idempotency key)

Response:
- `p1`: `Success|Fail`
- `p2`: error message on failure
- `p3`: single calendar row using canonical calendar row contract

Idempotency behavior:
- If `(owner_user_uuid, request_uuid)` already exists, existing row is returned.

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

## CAPI_ListCalendars
Purpose:
- List calendars owned by authenticated user.

Request:
- No additional `p3` parameters currently required.

Response:
- `p1`: `Success|Fail`
- `p2`: error message on failure
- `p3[0]`: total rows returned (count)
- `p4`: array of calendar rows using canonical calendar row contract

Example response shape:
```json
{
  "p1": "Success",
  "p3": ["2"],
  "p4": [
    ["cal-uuid-1", "req-uuid-1", "user-uuid", "", "Work", "...", "UTC", "#1F7A8C", "1", "private", "active", "1", "1730000000000", "1730000000000"],
    ["cal-uuid-2", "req-uuid-2", "user-uuid", "", "Family", "...", "UTC", "#4CAF50", "0", "shared", "active", "1", "1730000001000", "1730000001000"]
  ]
}
```

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

## CAPI_PatchCalendar
Request (`p3`):
- `[0]` `calendar_uuid` (required)
- `[1]` `name` (optional)
- `[2]` `description` (optional)
- `[3]` `time_zone` (optional)
- `[4]` `color_hex` (optional)
- `[5]` `visibility` (optional)
- `[6]` `status` (optional: `active|archived|deleted`)
- `[7]` `version` (required optimistic concurrency)
- `[8]` `request_uuid` (required idempotency)

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3`: updated calendar row

## CAPI_DeleteCalendar
Request (`p3`):
- `[0]` `calendar_uuid` (required)
- `[1]` `request_uuid` (required)

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3[0]`: `calendar_uuid`
- `p3[1]`: resulting `status` (`deleted`)
- `p3[2]`: `deleted_at_ms`

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

## CAPI_CreateEvent
Request (`p3`):
- `[0]` `calendar_uuid` (required)
- `[1]` `title` (required)
- `[2]` `description` (optional)
- `[3]` `location` (optional)
- `[4]` `start_at_ms` (required epoch ms)
- `[5]` `end_at_ms` (required epoch ms)
- `[6]` `all_day` (`0|1`, optional)
- `[7]` `event_tz` (optional, default `UTC`)
- `[8]` `visibility` (`default|private|public`, optional)
- `[9]` `transparency` (`opaque|transparent`, optional)
- `[10]` `recurrence_rrule` (optional)
- `[11]` `recurrence_exdates_json` (optional JSON string)
- `[12]` `recurring_event_uuid` (optional)
- `[13]` `original_start_at_ms` (optional epoch ms)
- `[14]` `metadata_json` (optional JSON string)
- `[15]` `request_uuid` (required idempotency)

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3`: event core row
- `p4[0]`: attendees rows (if provided)

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

## CAPI_PatchEvent
Request (`p3`):
- `[0]` `event_uuid` (required)
- `[1]` `title` (optional)
- `[2]` `description` (optional)
- `[3]` `location` (optional)
- `[4]` `start_at_ms` (optional)
- `[5]` `end_at_ms` (optional)
- `[6]` `all_day` (optional)
- `[7]` `event_tz` (optional)
- `[8]` `visibility` (optional)
- `[9]` `transparency` (optional)
- `[10]` `recurrence_rrule` (optional)
- `[11]` `recurrence_exdates_json` (optional)
- `[12]` `metadata_json` (optional)
- `[13]` `version` (required)
- `[14]` `request_uuid` (required)

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3`: updated event row

## CAPI_DeleteEvent
Request (`p3`):
- `[0]` `event_uuid` (required)
- `[1]` `request_uuid` (required)

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3[0]`: `event_uuid`
- `p3[1]`: resulting `status` (`cancelled` or soft deleted)
- `p3[2]`: `updated_at_ms`

## CAPI_ListEventsRange
Request (`p3`):
- `[0]` `calendar_uuid` (required)
- `[1]` `range_start_at_ms` (required)
- `[2]` `range_end_at_ms` (required)
- `[3]` `offset` (optional)
- `[4]` `limit` (optional)
- `[5]` `expand_recurrence` (`0|1`, optional, default `1`)

Response:
- `p1`: `Success|Fail`
- `p2`: error text
- `p3[0]`: total
- `p3[1]`: offset
- `p3[2]`: limit
- `p4`: event row list

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

