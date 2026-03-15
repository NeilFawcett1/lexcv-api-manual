# AppAdmin API Manual

This document provides complete reference documentation for all AppAdmin APIs in lexcv-server. These APIs provide administrative functionality for health monitoring, log access, database statistics, Sacred Timeline management, and recovery operations.

---

## Table of Contents

1. [Authentication](#authentication)
2. [RObj Structure Reference](#robj-structure-reference)
3. [Login APIs](#login-apis)
   - [APPADMIN_CHECKLOGIN](#appadmin_checklogin)
4. [User Flags APIs](#user-flags-apis)
   - [APPADMIN_USERFLAGS](#appadmin_userflags)
5. [Profile APIs](#profile-apis)
   - [APPADMIN_PROFILE_GET](#appadmin_profile_get)
   - [APPADMIN_PROFILE_SEARCH](#appadmin_profile_search)
   - [APPADMIN_PROFILE_EDIT](#appadmin_profile_edit)
   - [APPADMIN_PROFILE_VCODE](#appadmin_profile_vcode)
   - [APPADMIN_PROFILE_MEDIA](#appadmin_profile_media)
   - [APPADMIN_PROFILE_POSTS](#appadmin_profile_posts)
   - [APPADMIN_PROFILE_COMMENTS](#appadmin_profile_comments)
   - [APPADMIN_PROFILE_FOLLOWS](#appadmin_profile_follows)
6. [Health APIs](#health-apis)
   - [APPADMIN_HEALTH](#appadmin_health)
   - [APPADMIN_HEALTHCHECK](#appadmin_healthcheck)
7. [Log APIs](#log-apis)
   - [APPADMIN_LOGS](#appadmin_logs)
  - [APPADMIN_CLIOAUDIT](#appadmin_clioaudit)
   - [APPADMIN_LOGSDOWNLOAD](#appadmin_logsdownload)
8. [Timeline APIs](#timeline-apis)
   - [APPADMIN_TIMELINE](#appadmin_timeline)
9. [Database APIs](#database-apis)
   - [APPADMIN_DBSTATS](#appadmin_dbstats)
   - [APPADMIN_DBQUERY](#appadmin_dbquery)
10. [Recovery APIs](#recovery-apis)
    - [APPADMIN_RECOVERY](#appadmin_recovery)
    - [APPADMIN_MAINTENANCE](#appadmin_maintenance)
11. [Vox APIs](#vox-apis)
  - [APPADMIN_VOXBROADCAST](#appadmin_voxbroadcast)
  - [APPADMIN_VOXEVENT](#appadmin_voxevent)
12. [Cato APIs](#cato-apis)
  - [APPADMIN_CATO_EVALUATE](#appadmin_cato_evaluate)
  - [APPADMIN_CATO_EXPLAIN](#appadmin_cato_explain)
  - [APPADMIN_CATO_EFFECTIVE_FOR_USER](#appadmin_cato_effective_for_user)
  - [APPADMIN_CATO_EFFECTIVE_FOR_RESOURCE](#appadmin_cato_effective_for_resource)
  - [APPADMIN_CATO_EFFECTIVE_FOR_USER_DISTINCT](#appadmin_cato_effective_for_user_distinct)
  - [APPADMIN_CATO_EFFECTIVE_FOR_RESOURCE_DISTINCT](#appadmin_cato_effective_for_resource_distinct)
  - [APPADMIN_CATO_TUPLE_GRANT](#appadmin_cato_tuple_grant)
  - [APPADMIN_CATO_TUPLE_REVOKE](#appadmin_cato_tuple_revoke)
  - [APPADMIN_CATO_TUPLE_LIST](#appadmin_cato_tuple_list)
  - [APPADMIN_CATO_POLICY_RESOURCE](#appadmin_cato_policy_resource)
  - [APPADMIN_CATO_GRANTS_FOR_SUBJECT](#appadmin_cato_grants_for_subject)
  - [APPADMIN_CATO_RELATIONS_FOR_RESOURCE](#appadmin_cato_relations_for_resource)
  - [APPADMIN_CATO_RELATIONS_FOR_SUBJECT](#appadmin_cato_relations_for_subject)
  - [APPADMIN_CATO_TUPLE_BULK_GRANT](#appadmin_cato_tuple_bulk_grant)
  - [APPADMIN_CATO_TUPLE_BULK_REVOKE](#appadmin_cato_tuple_bulk_revoke)
  - [APPADMIN_CATO_GROUP_CREATE](#appadmin_cato_group_create)
  - [APPADMIN_CATO_GROUP_DELETE](#appadmin_cato_group_delete)
  - [APPADMIN_CATO_GROUP_LIST](#appadmin_cato_group_list)
  - [APPADMIN_CATO_GROUP_MEMBERS](#appadmin_cato_group_members)
  - [APPADMIN_CATO_GROUP_ADD_MEMBER](#appadmin_cato_group_add_member)
  - [APPADMIN_CATO_GROUP_REMOVE_MEMBER](#appadmin_cato_group_remove_member)
  - [APPADMIN_CATO_GROUP_GRANT_RESOURCE](#appadmin_cato_group_grant_resource)
  - [APPADMIN_CATO_GROUP_REVOKE_RESOURCE](#appadmin_cato_group_revoke_resource)
  - [APPADMIN_CATO_GROUP_GRANTS](#appadmin_cato_group_grants)

---

## Authentication

All AppAdmin APIs require **RoleRoot** authentication, except for `APPADMIN_HEALTHCHECK` which is public (for load balancer health probes).

Standard authentication uses:
- **P1**: Username
- **P2**: Access token

---

## RObj Structure Reference

All APIs use the RObj structure for request/response:

```go
type RObj struct {
    Req string       `json:"req"` // Request identifier (optional)
    P1  string       `json:"p1"`  // Primary parameter / Response status
    P2  string       `json:"p2"`  // Secondary parameter / Response message
    P3  []string     `json:"p3"`  // Additional parameters / JSON response data
    P4  [][]string   `json:"p4"`  // 2D string array (rarely used)
    P5  [][][]string `json:"p5"`  // 3D string array (rarely used)
    P6  []RObj       `json:"p6"`  // Nested RObj array (rarely used)
}
```

**Standard Response Pattern:**
- `P1`: `"Success"` or `"Fail"`
- `P2`: Human-readable message
- `P3[0]`: JSON-encoded response data (when applicable)

---

## Login APIs

### APPADMIN_CHECKLOGIN

Validates admin login credentials. Used by the C++ admin app at LexCV HQ to verify access before making other API calls.

**Endpoint:** `POST /api/APPADMIN_CHECKLOGIN`  
**Authentication:** Admin bypass (`admin@lexcv.co` with configured access code)

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Username (`admin@lexcv.co`) |
| P2 | string | Access code (64-char hex from `admin.access.code` config) |

#### Response

**Success:**
| Field | Value |
|-------|-------|
| P1 | `"Success"` |
| P2 | `"Login valid"` |

**Failure:**
| Field | Value |
|-------|-------|
| P1 | `"Fail"` |
| P2 | `"no login"` |

#### Example

```bash
curl -X POST https://api.lexcv.co/api/APPADMIN_CHECKLOGIN \
  -H "Content-Type: application/json" \
  -d '{"p1":"admin@lexcv.co","p2":"a1b2c3d4...128charhex..."}'
```

---

## User Flags APIs

### APPADMIN_USERFLAGS

Manages user flags including roles, states, types, and levels. Supports both querying and modifying flags.

**Endpoint:** `POST /api/APPADMIN_USERFLAGS`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Username (`admin@lexcv.co`) |
| P2 | string | Access code |
| P3[0] | string | action: `get`, `set`, `add_state`, `remove_state`, `add_role`, `remove_role`, `set_type`, `set_level` |
| P3[1] | string | Target user UUID |
| P3[2] | string | Value (for set/add/remove actions) |

#### Actions

| Action | P3[2] Value | Description |
|--------|-------------|-------------|
| `get` | - | Returns current flags for user |
| `set` | JSON | Sets complete flags from JSON |
| `add_state` | state | Adds user_state: `deleted`, `deactivated`, `hidden`, `banned`, `verified` |
| `remove_state` | state | Removes a user_state value |
| `add_role` | role | Grants a Cato system role: `root`, `admin`, `moderator`, `billing` |
| `remove_role` | role | Revokes a Cato system role |
| `set_type` | type | Sets user_type: `lawyer`, `otherpro`, `company`, `student`, `public` |
| `set_level` | level | Sets user_level: `level1`, `level2`, `level3`, `level4` |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | JSON of updated flags |

---

## Profile APIs

### APPADMIN_PROFILE_GET

Retrieves complete profile data for a user including flags, follow counts, post/comment counts.

**Endpoint:** `POST /api/APPADMIN_PROFILE_GET`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Username (`admin@lexcv.co`) |
| P2 | string | Access code |
| P3[0] | string | lookup_type: `uuid`, `uname`, or `alias` |
| P3[1] | string | lookup_value |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | JSON of AdminUserProfile |

**AdminUserProfile JSON Structure:**
```json
{
  "uuid": "user-uuid",
  "uname": "user@email.com",
  "name": "John",
  "surname": "Doe",
  "email": "user@email.com",
  "telephone": "+1234567890",
  "website": "https://example.com",
  "headline": "Lawyer at Firm",
  "alias": "johndoe",
  "verified": "1",
  "vcode": "123456",
  "vsent": "2025-01-01 12:00:00",
  "pic_dataloc": "media-uuid",
  "banner_dataloc": "media-uuid",
  "small_pic": "thumbnail-url",
  "created_at": "2024-01-01 00:00:00",
  "updated_at": "2025-01-01 00:00:00",
  "flags": {"user_state": ["verified"], "user_type": ["lawyer"]},
  "follow_counts": {"followers": 100, "following": 50},
  "post_count": 25,
  "comment_count": 150
}
```

---

### APPADMIN_PROFILE_SEARCH

Searches for users by name, username, or email.

**Endpoint:** `POST /api/APPADMIN_PROFILE_SEARCH`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Username (`admin@lexcv.co`) |
| P2 | string | Access code |
| P3[0] | string | search_type: `name`, `uname`, `email`, `all` |
| P3[1] | string | search_query |
| P3[2] | string | offset (optional, default 0) |
| P3[3] | string | limit (optional, default 20, max 100) |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | Total count |
| P4 | Array of `[uuid, uname, name, surname, email, verified]` |

---

### APPADMIN_PROFILE_EDIT

Edits user profile fields.

**Endpoint:** `POST /api/APPADMIN_PROFILE_EDIT`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Username (`admin@lexcv.co`) |
| P2 | string | Access code |
| P3[0] | string | Target user UUID |
| P3[1] | string | field_name: `name`, `surname`, `email`, `headline`, `telephone`, `website`, `alias` |
| P3[2] | string | new_value |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |

---

### APPADMIN_PROFILE_VCODE

Manages user verification code - view, regenerate, or manually verify.

**Endpoint:** `POST /api/APPADMIN_PROFILE_VCODE`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Username (`admin@lexcv.co`) |
| P2 | string | Access code |
| P3[0] | string | action: `get`, `regenerate`, `verify` |
| P3[1] | string | Target user UUID |

#### Actions

| Action | Description |
|--------|-------------|
| `get` | Returns current vcode, vsent timestamp, and verified status |
| `regenerate` | Generates new 6-digit code and updates vsent |
| `verify` | Manually sets user as verified (sets flag and legacy column) |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | vcode (for get/regenerate) |
| P3[1] | vsent (for get) |
| P3[2] | verified (for get) |

---

### APPADMIN_PROFILE_MEDIA

Views and manages user profile media (pictures, banners).

**Endpoint:** `POST /api/APPADMIN_PROFILE_MEDIA`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Username (`admin@lexcv.co`) |
| P2 | string | Access code |
| P3[0] | string | action: `get`, `delete_pic`, `delete_banner` |
| P3[1] | string | Target user UUID |

#### Actions

| Action | Description |
|--------|-------------|
| `get` | Returns JSON with pic_dataloc, banner_dataloc, small_pic |
| `delete_pic` | Removes profile picture and small thumbnail |
| `delete_banner` | Removes banner image |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | JSON media data (for get action) |

---

### APPADMIN_PROFILE_POSTS

Retrieves posts by a specific user.

**Endpoint:** `POST /api/APPADMIN_PROFILE_POSTS`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Username (`admin@lexcv.co`) |
| P2 | string | Access code |
| P3[0] | string | Target user UUID |
| P3[1] | string | offset (optional, default 0) |
| P3[2] | string | limit (optional, default 20, max 100) |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | Total post count |
| P6 | Array of post RObj (same format as API_PostsListAll) |

**Post RObj Structure:**
- P3: `[type, uuid, author_uuid, text, creation_date, last_edited]`
- P4[0]: `[like_count, support_count, laugh_count, dislike_count]`
- P4[1]: `[comment_count, repost_count, impressions]` (impressions = unique impressions estimate from Clio)
- P4[2]: `[repost_of_uuid, best_answer_uuid]`
- P4[3]: `[author_uname, "", ""]`

---

### APPADMIN_PROFILE_COMMENTS

Retrieves comments by a specific user.

**Endpoint:** `POST /api/APPADMIN_PROFILE_COMMENTS`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Username (`admin@lexcv.co`) |
| P2 | string | Access code |
| P3[0] | string | Target user UUID |
| P3[1] | string | offset (optional, default 0) |
| P3[2] | string | limit (optional, default 20, max 100) |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | Total comment count |
| P6 | Array of comment RObj |

**Comment RObj Structure:**
- P3: `[type("1"), uuid, author_uuid, text, created_at, updated_at, post_uuid, parent_uuid]`
- P4[0]: `[like_count, support_count, laugh_count, dislike_count]`
- P4[1]: `[author_uname, "", ""]`

---

### APPADMIN_PROFILE_FOLLOWS

Retrieves follow relationships for a user.

**Endpoint:** `POST /api/APPADMIN_PROFILE_FOLLOWS`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Username (`admin@lexcv.co`) |
| P2 | string | Access code |
| P3[0] | string | action: `followers`, `following`, `counts` |
| P3[1] | string | Target user UUID |
| P3[2] | string | offset (optional, default 0) |
| P3[3] | string | limit (optional, default 20, max 100) |

#### Actions

| Action | Description |
|--------|-------------|
| `followers` | Returns list of users who follow target |
| `following` | Returns list of users target follows |
| `counts` | Returns follower and following counts only |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3 | For counts: `[followers_count, following_count]`; otherwise `[total_count]` |
| P4 | Array of `[uuid, uname, name, surname]` (for followers/following) |

---

## Health APIs

### APPADMIN_HEALTH

Returns comprehensive health status for all system components.

**Endpoint:** POST to standard API handler  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Username |
| P2 | string | Access token |
| P3[0] | string | Component filter (optional) |

**P3[0] Component Values:**
- `"all"` - Check all components (default)
- `"server"` - Server status only
- `"mysql"` - MySQL health only
- `"dgraph"` - DGraph health only
- `"redis"` - Redis health only
- `"redpanda"` - Redpanda health only
- `"nginx"` - Nginx health only

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Overall health status: `"healthy"`, `"degraded"`, or `"unhealthy"` |
| P3[0] | JSON `SystemHealth` object |

**P3[0] JSON Structure (`SystemHealth`):**

```json
{
  "overall": "healthy",
  "node_id": "belgium-1",
  "uptime": "2h30m15s",
  "components": {
    "server": {
      "component": "server",
      "status": "healthy",
      "message": "Running on linux, 4 CPUs",
      "last_checked": "2025-01-15T10:30:00Z"
    },
    "mysql": {
      "component": "mysql",
      "status": "healthy",
      "message": "Open: 10, InUse: 2, Idle: 8, WaitCount: 0",
      "latency": "1.234ms",
      "last_checked": "2025-01-15T10:30:00Z"
    },
    "dgraph": {
      "component": "dgraph",
      "status": "healthy",
      "message": "DGraph responding",
      "latency": "5.678ms",
      "last_checked": "2025-01-15T10:30:00Z"
    },
    "redis": {
      "component": "redis",
      "status": "healthy",
      "message": "Redis responding",
      "latency": "0.456ms",
      "last_checked": "2025-01-15T10:30:00Z"
    },
    "redpanda": {
      "component": "redpanda",
      "status": "healthy",
      "message": "Cluster healthy, 1 brokers",
      "last_checked": "2025-01-15T10:30:00Z"
    },
    "nginx": {
      "component": "nginx",
      "status": "healthy",
      "message": "nginx: active",
      "last_checked": "2025-01-15T10:30:00Z"
    }
  },
  "memory": {
    "alloc": "45.2 MB",
    "total_alloc": "123.4 MB",
    "sys": "78.9 MB",
    "num_gc": 42
  },
  "goroutines": 156
}
```

---

### APPADMIN_HEALTHCHECK

Simple health check endpoint for load balancer probes. Returns HTTP 200 with "OK" if healthy.

**Endpoint:** GET `/api/healthcheck`  
**Authentication:** None (public endpoint)

#### Request

No parameters required. This is a simple GET request.

#### Response

| HTTP Status | Body |
|-------------|------|
| 200 | `"OK"` |
| 503 | `"Service Unavailable"` |

---

## Log APIs

### APPADMIN_LOGS

Provides access to application and system logs with filtering and search capabilities.

**Endpoint:** POST to standard API handler  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Username |
| P2 | string | Access token |
| P3[0] | string | Action (required) |
| P3[1] | string | Log source (required for most actions) |
| P3[2] | string | Count / Search term / Start line |
| P3[3] | string | End line (for `"between"` action) / Max results (for `"search"` action) |

**P3[0] Action Values:**

| Action | Description | P3[2] | P3[3] |
|--------|-------------|-------|-------|
| `"list"` | List available log sources | - | - |
| `"tail"` | Get last N lines | Line count (default 100, max 5000) | - |
| `"head"` | Get first N lines | Line count (default 100, max 5000) | - |
| `"search"` | Search for text | Search term | Max results (default 200, max 1000) |
| `"between"` | Get lines between range | Start line | End line (max range 5000) |

**P3[1] Log Source Values:**
- `"app"` - Application log (nflog.log)
- `"system"` - System log (syslog/journald)
- `"nginx"` - Nginx access/error logs
- `"redpanda"` - Redpanda broker logs

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message with entry count |
| P3[0] | JSON `LogResult` object |

**P3[0] JSON Structure (`LogResult`):**

```json
{
  "log_file": "app",
  "total_lines": 15432,
  "from_line": 15332,
  "to_line": 15432,
  "entries": [
    {
      "line": 15332,
      "timestamp": "2025-01-15T10:30:00Z",
      "level": "INFO",
      "message": "Request processed successfully"
    },
    {
      "line": 15333,
      "timestamp": "2025-01-15T10:30:01Z",
      "level": "ERROR",
      "message": "Database connection timeout"
    }
  ],
  "query": "error"
}
```

#### Examples

**Tail last 50 lines of app log:**
```json
{
  "P1": "admin",
  "P2": "token123",
  "P3": ["tail", "app", "50"]
}
```

**Search for errors:**
```json
{
  "P1": "admin",
  "P2": "token123",
  "P3": ["search", "app", "ERROR", "100"]
}
```

**Get lines 1000-1100:**
```json
{
  "P1": "admin",
  "P2": "token123",
  "P3": ["between", "app", "1000", "1100"]
}
```

---

### APPADMIN_CLIOAUDIT

Provides head/tail/search/query access to the Clio audit log stored in MySQL (`clio_audit_log`).

**Endpoint:** POST `/api/APPADMIN_CLIOAUDIT`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Username |
| P2 | string | Access token |
| P3[0] | string | Action: `"tail"` \| `"head"` \| `"search"` \| `"query"` |
| P3[1] | string | Limit (optional, default 100, max 1000) |
| P3[2] | string | For `search`: search term (optional). For `query`: action name filter (optional) |
| P3[3] | string | `actor_uuid` filter (optional) |
| P3[4] | string | `subject_uuid` filter (optional) |
| P3[5] | string | `from_ts_ms` filter (optional, inclusive) |
| P3[6] | string | `to_ts_ms` filter (optional, inclusive) |

Notes:
- `search` runs a LIKE search across action name, UUIDs, IP, device, and `meta` JSON.
- `query` applies exact-match filters (action name / actor / subject) and optional time bounds.

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | `"ok"` or error message |
| P3[0] | JSON object containing results |

**P3[0] JSON Structure:**

```json
{
  "action": "tail",
  "count": 2,
  "entries": [
    {
      "event_id": "...",
      "action": "api.login.firebase",
      "ts_ms": 1736973600123,
      "actor_uuid": "...",
      "subject_uuid": "...",
      "ip": "1.2.3.4",
      "device": "Mozilla/5.0 ...",
      "source_node": "belgium-1",
      "meta_json": "{...}",
      "created_at": "2025-01-15T10:30:00Z"
    }
  ]
}
```

#### Examples

**Tail last 50 audit entries:**
```json
{
  "P1": "admin",
  "P2": "token123",
  "P3": ["tail", "50"]
}
```

**Search for a UUID:**
```json
{
  "P1": "admin",
  "P2": "token123",
  "P3": ["search", "200", "3f0c..."]
}
```

**Query by actor_uuid (exact match):**
```json
{
  "P1": "admin",
  "P2": "token123",
  "P3": ["query", "200", "", "actor-uuid-here"]
}
```

**Query by subject_uuid (exact match):**
```json
{
  "P1": "admin",
  "P2": "token123",
  "P3": ["query", "200", "", "", "subject-uuid-here"]
}
```

**Query by action name + time range:**
```json
{
  "P1": "admin",
  "P2": "token123",
  "P3": ["query", "200", "api.login.firebase", "", "", "1736900000000", "1736990000000"]
}
```

**Query by action + actor + subject (all exact match):**
```json
{
  "P1": "admin",
  "P2": "token123",
  "P3": ["query", "200", "api.login.firebase", "actor-uuid-here", "subject-uuid-here"]
}
```

---

### APPADMIN_LOGSDOWNLOAD

Downloads complete log files as attachments.

**Endpoint:** POST to standard API handler  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Username |
| P2 | string | Access token |
| P3[0] | string | Log source (required) |

**P3[0] Log Source Values:**
- `"app"` - Application log
- `"nginx"` - Nginx logs
- `"system"` - System logs
- `"redpanda"` - Redpanda logs

#### Response

Returns direct file download with headers:
- `Content-Disposition: attachment; filename=<source>_<timestamp>.log`
- `Content-Type: text/plain`
- `Content-Length: <file_size>`

---

## Timeline APIs

### APPADMIN_TIMELINE

Provides statistics and management for the Sacred Timeline (event sourcing system).

**Endpoint:** POST to standard API handler  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Username |
| P2 | string | Access token |
| P3[0] | string | Action (default: `"stats"`) |
| P3[1] | string | Additional parameter (varies by action) |

**P3[0] Action Values:**

| Action | Description |
|--------|-------------|
| `"stats"` | Get timeline consumer/producer statistics |
| `"outbox"` | Get outbox table statistics |
| `"projection"` | Get projection table statistics |
| `"full"` | Get all statistics combined |
| `"reset_stats"` | Reset runtime statistics counters |
| `"clear_outbox"` | Clear completed outbox entries |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Summary message |
| P3[0] | JSON statistics object |

**P3[0] JSON Structure for `"stats"` (`TimelineStats`):**

```json
{
  "initialized": true,
  "consumer": {
    "node_id": "belgium-1",
    "consumer_group_id": "lexcv-belgium-1",
    "total_lag": 0,
    "events_received": 12345,
    "events_processed": 12340,
    "events_skipped": 5,
    "events_failed": 0,
    "last_event_time": "2025-01-15T10:30:00Z",
    "uptime": "2h30m15s"
  },
  "producer": {
    "node_id": "belgium-1",
    "events_published": 6789,
    "events_failed": 2,
    "last_publish_time": "2025-01-15T10:29:55Z",
    "outbox_pending": 3,
    "outbox_failed": 1
  },
  "brokers": ["redpanda-belgium:9092"],
  "topic": "lexcv-timeline",
  "collected_at": "2025-01-15T10:30:00Z"
}
```

**P3[0] JSON Structure for `"outbox"` (`OutboxStats`):**

```json
{
  "total_pending": 5,
  "total_failed": 2,
  "oldest_pending_ms": 1736935500000,
  "oldest_pending": "2025-01-15T10:25:00Z",
  "recent_entries": [
    {
      "id": 123,
      "aggregate_id": "user_12345",
      "api_code": "API_POSTADDNEW",
      "retry_count": 3,
      "created_at_ms": 1736935500000,
      "created_at": "2025-01-15T10:25:00Z",
      "last_error": "connection refused"
    }
  ]
}
```

> **Schema Note:** The `sacred_timeline_outbox` table stores `created_at` as `BIGINT UNSIGNED` (Unix milliseconds). The API returns both the raw milliseconds (`created_at_ms`) and formatted time (`created_at`).

**P3[0] JSON Structure for `"projection"` (`ProjectionStats`):**

> **Schema Note:** The projection action queries the `sacred_timeline` table (not `sacred_timeline_projection`). Timestamps are stored as `BIGINT UNSIGNED` (Unix milliseconds).

```json
{
  "total_events": 98765,
  "events_by_api": {
    "API_POSTADDNEW": 45000,
    "API_PROFILEBASICEDIT": 30000,
    "API_PEERPROFILEADDFOLLOW": 15000
  },
  "events_by_source": {
    "belgium-1": 50000,
    "mumbai-1": 48765
  },
  "latest_event_ms": 1736938200000,
  "latest_event_time": "2025-01-15T10:30:00Z",
  "oldest_event_ms": 1733011200000,
  "oldest_event_time": "2024-12-01T00:00:00Z",
  "last_hour_events": 1234,
  "last_24_hour_events": 28000
}
```

**P3[0] JSON Structure for `"full"` (`TimelineFullStats`):**

```json
{
  "timeline": { /* TimelineStats */ },
  "outbox": { /* OutboxStats */ },
  "projection": { /* ProjectionStats */ },
  "collected_at": "2025-01-15T10:30:00Z"
}
```

---

## Database APIs

### APPADMIN_DBSTATS

Provides database statistics for MySQL and DGraph.

**Endpoint:** POST to standard API handler  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Username |
| P2 | string | Access token |
| P3[0] | string | Action (default: `"full"`) |
| P3[1] | string | Parameter (table/type name for detail actions) |

**P3[0] Action Values:**

| Action | Description | P3[1] |
|--------|-------------|-------|
| `"mysql"` | MySQL statistics only | - |
| `"dgraph"` | DGraph statistics only | - |
| `"full"` | Both MySQL and DGraph | - |
| `"mysql_table"` | Detailed stats for specific table | Table name (required) |
| `"dgraph_type"` | Detailed stats for specific type | Type name (required) |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Summary message |
| P3[0] | JSON statistics object |

**P3[0] JSON Structure for `"mysql"` (`MySQLStats`):**

```json
{
  "database_name": "lexcv",
  "version": "8.0.35",
  "total_tables": 25,
  "total_data_size": "156.7 MB",
  "total_index_size": "45.2 MB",
  "tables": [
    {
      "table_name": "users",
      "engine": "InnoDB",
      "row_count": 15000,
      "data_size": "25.4 MB",
      "index_size": "8.2 MB",
      "total_size": "33.6 MB",
      "data_size_bytes": 26634240,
      "columns": [
        {
          "name": "uuid",
          "data_type": "varchar",
          "column_type": "varchar(36)",
          "nullable": false,
          "key": "PRI",
          "ordinal_position": 1
        },
        {
          "name": "email",
          "data_type": "varchar",
          "column_type": "varchar(255)",
          "nullable": false,
          "key": "UNI",
          "ordinal_position": 5
        }
      ],
      "indexes": [
        { "name": "PRIMARY", "unique": true, "columns": ["uuid"] },
        { "name": "uk_email", "unique": true, "columns": ["email"] }
      ],
      "create_time": "2024-01-15T00:00:00Z",
      "update_time": "2025-01-15T10:25:00Z"
    }
  ],
  "collected_at": "2025-01-15T10:30:00Z"
}
```

**MySQL Table Schema Detail (`tables[].columns[]`)**

Each entry in `tables[].columns[]` is derived from `information_schema.COLUMNS`.

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Column name |
| `data_type` | string | Base SQL type (e.g. `varchar`, `bigint`, `tinyint`, `mediumtext`) |
| `column_type` | string | Full SQL type including length/unsigned/etc (e.g. `varchar(36)`, `bigint unsigned`) |
| `nullable` | bool | True if `IS_NULLABLE = YES` |
| `key` | string | Key type from `COLUMN_KEY` (`PRI`, `UNI`, `MUL`, or empty) |
| `default` | string (optional) | Default value (omitted if NULL) |
| `extra` | string (optional) | Extra attributes (e.g. `auto_increment`) |
| `ordinal_position` | number | 1-based ordinal position in the table |

**MySQL Index Detail (`tables[].indexes[]`)**

Each entry in `tables[].indexes[]` is derived from `information_schema.STATISTICS`.

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Index name (`PRIMARY`, `idx_*`, etc.) |
| `unique` | bool | True if index is unique |
| `columns` | []string | Columns in index order |

> Note: `APPADMIN_DBSTATS` now includes schema metadata for every table when using actions `mysql` and `full`. This is intended for tooling to validate production schema conformity without running ad-hoc SQL.

**P3[0] JSON Structure for "mysql_table" (`MySQLTableDetail`):**

The `mysql_table` action returns a focused view for one table (row count + size + full column/index listing). Table name is sanitized server-side to only allow letters, digits, and underscore.

```json
{
  "table_name": "job_applications",
  "row_count": 1234,
  "data_size": "5.1 MB",
  "columns": [
    {
      "name": "uuid",
      "type": "varchar",
      "column_type": "varchar(36)",
      "nullable": false,
      "key": "PRI",
      "ordinal_position": 1
    },
    {
      "name": "provisional_status_message",
      "type": "mediumtext",
      "column_type": "mediumtext",
      "nullable": true,
      "key": "",
      "ordinal_position": 8
    }
  ],
  "indexes": [
    { "name": "PRIMARY", "unique": true, "columns": ["uuid"] },
    { "name": "idx_job_provisional", "unique": false, "columns": ["jobuuid", "provisional_status", "created_at"] }
  ]
}
```

**P3[0] JSON Structure for `"dgraph"` (`DGraphStats`):**

```json
{
  "health": "healthy",
  "total_nodes": 250000,
  "type_stats": [
    { "type_name": "users", "node_count": 15000 },
    { "type_name": "nations", "node_count": 8 },
    { "type_name": "countries", "node_count": 195 },
    { "type_name": "geoareas", "node_count": 3500 },
    { "type_name": "courts", "node_count": 1200 },
    { "type_name": "lawyertype", "node_count": 45 },
    { "type_name": "worktype", "node_count": 120 },
    { "type_name": "organisationtype", "node_count": 25 },
    { "type_name": "professionaltype", "node_count": 30 },
    { "type_name": "location", "node_count": 8500 },
    { "type_name": "posts", "node_count": 85000 },
    { "type_name": "comment", "node_count": 42000 },
    { "type_name": "blog", "node_count": 3200 },
    { "type_name": "blogfolders", "node_count": 450 },
    { "type_name": "blogs", "node_count": 1800 },
    { "type_name": "conversation", "node_count": 5600 },
    { "type_name": "message", "node_count": 28000 }
  ],
  "predicate_stats": [
    { "predicate_name": "countryInNation", "value_type": "uid", "edge_count": 195, "is_index": false },
    { "predicate_name": "userFollows", "value_type": "uid", "edge_count": 18500, "is_index": false }
  ],
  "schema": [
    { "predicate": "uuid", "type": "string", "index": true, "tokenizer": ["exact"] },
    { "predicate": "email", "type": "string", "index": true, "tokenizer": ["exact"] },
    { "predicate": "userFollows", "type": "uid", "reverse": true, "count": true },
    { "predicate": "folderAllowFollowers", "type": "string", "index": true, "tokenizer": ["exact"] },
    { "predicate": "postText", "type": "string", "index": true, "tokenizer": ["fulltext"], "lang": true }
  ],
  "collected_at": "2025-01-15T10:30:00Z"
}
```

**Schema Field Information:**

The `schema` array contains all non-internal DGraph predicates (excludes `dgraph.*` predicates) with their definitions:

| Field | Type | Description |
|-------|------|-------------|
| `predicate` | string | Name of the predicate |
| `type` | string | Data type: `string`, `int`, `float`, `bool`, `datetime`, `geo`, `password`, `uid`, `[uid]` |
| `index` | bool | Whether the predicate has an index (omitted if false) |
| `tokenizer` | []string | Index tokenizers: `exact`, `hash`, `term`, `fulltext`, `trigram`, etc. (omitted if empty) |
| `reverse` | bool | Whether reverse edges are stored (for uid types) (omitted if false) |
| `count` | bool | Whether count index is enabled (omitted if false) |
| `list` | bool | Whether the predicate stores a list (omitted if false) |
| `upsert` | bool | Whether upsert directive is set (omitted if false) |
| `lang` | bool | Whether language tags are enabled (omitted if false) |

**DGraph Types Tracked (17 types):**

| Type | Description |
|------|-------------|
| `users` | User accounts |
| `nations` | Geographic nations/regions |
| `countries` | Countries |
| `geoareas` | Geographic areas within countries |
| `courts` | Courts/tribunals |
| `lawyertype` | Types of lawyers |
| `worktype` | Types of legal work |
| `organisationtype` | Types of organizations |
| `professionaltype` | Types of professionals |
| `location` | Location records |
| `posts` | Social posts |
| `comment` | Post comments |
| `blog` | Individual blog posts |
| `blogfolders` | Blog folders for organization |
| `blogs` | Blog containers |
| `conversation` | Message conversations |
| `message` | Individual messages |

**DGraph Predicates Tracked (29 predicates):**

| Category | Predicates |
|----------|------------|
| **Geographic** | `countryInNation`, `areaInCountry`, `courtInCountry`, `lawyerTypeInCountry` |
| **Lawyer Profile** | `lawyerIsType`, `lawyerInCountry`, `lawyerInArea`, `lawyerDoesWorkType`, `lawyerUsesCourt` |
| **Company/Professional** | `companyIsType`, `companyInCountry`, `professionalIsType` |
| **Social** | `userFollows`, `userShortlists` |
| **Posts** | `postAuthor`, `commentAuthor`, `commentOnPost`, `reactedBy`, `repostOf`, `bestAnswer` |
| **Blogs** | `blogAuthor`, `folderOwner`, `folderContainsBlog`, `userCanAccessFolder` |
| **Messaging** | `userBlocks`, `userInConversation`, `conversationHasUser`, `messageInConversation`, `messageAuthor` |
```

**P3[0] JSON Structure for `"full"` (`FullDBStats`):**

```json
{
  "mysql": { /* MySQLStats */ },
  "dgraph": { /* DGraphStats */ },
  "collected_at": "2025-01-15T10:30:00Z"
}
```

---

### APPADMIN_DBQUERY

Runs database queries for diagnostics and administrative operations.

**Endpoint:** POST to standard API handler  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Username |
| P2 | string | Access token |
| P3[0] | string | Database: `"mysql"`, `"mysql_exec"`, `"dgraph"`, `"dgraph_upsert"`, or `"dgraph_patch_schema"` |
| P3[1] | string | Query string |

**Database Options:**

| Database | Description | Safety |
|----------|-------------|--------|
| `mysql` | Read-only MySQL queries | Only `SELECT`, `SHOW`, `DESCRIBE` allowed |
| `mysql_exec` | MySQL write operations | **DANGEROUS** - Allows `DELETE`, `UPDATE`, `INSERT`, `ALTER` |
| `dgraph` | Read-only DGraph queries | Only read queries allowed |
| `dgraph_upsert` | DGraph mutations | **DANGEROUS** - Allows delete/set operations |
| `dgraph_patch_schema` | Patch Dgraph schema from server files | Safe (no `DropAll`); requires schema files present |

**Query Timeouts:**
- `mysql`: 30 seconds
- `mysql_exec`: 60 seconds
- `dgraph`: 30 seconds
- `dgraph_upsert`: 30 seconds
- `dgraph_patch_schema`: 30 seconds

#### DGraph Schema Patch (from server files)

If you set `P3[0]` to `dgraph_patch_schema`, the server will:

- Load `schemas/predicates/edges.graph` and `schemas/predicates/types.graph` from disk (relative to the server working directory or executable location).
- Apply them via Dgraph `AlterSchema` (predicates first, then types).
- **It does not drop data** and does not repopulate nodes.

If the schema files are not present on the server, this will return `Fail` with a message telling you to either deploy the files, or use `dgraph_upsert` with `schema:<...>`.

#### DGraph Upsert Formats

The `dgraph_upsert` option supports multiple formats:

**Format 1 - Delete by UID:**
```
delete:0x1234
```

**Format 2 - Schema Alteration:**
```
schema:predicateName: string @index(exact) .
anotherPredicate: [uid] @reverse .
```

**Format 3 - JSON with query/delete/set:**
```json
{
  "query": "{ conv as var(func: type(conversation)) msg as var(func: type(message)) }",
  "delete": "uid(conv) * * .\nuid(msg) * * .",
  "set": ""
}
```

**Format 4 - Raw upsert block:**
```
query { u as var(func: eq(uuid, "...")) }
mutation { set { uid(u) <predicate> "value" . } }
```

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message with row count or result |
| P3[0] | JSON with query results |

**P3[0] JSON Structure (MySQL):**

```json
{
  "columns": ["id", "email"],
  "rows": [
    {"id": 1, "email": "user@example.com"}
  ],
  "count": 1
}
```

**P3[0] JSON Structure (mysql_exec):**

```json
{
  "rows_affected": 5,
  "last_insert_id": 0
}
```

**P3[0] JSON Structure (DGraph):**

```json
{
  "users": [
    {
      "uid": "0x1234",
      "uuid": "abc-123"
    }
  ]
}
```

#### Examples

**MySQL Read Query:**
```json
{
  "P1": "admin",
  "P2": "token123",
  "P3": ["mysql", "SELECT COUNT(*) as count FROM users WHERE created_at > '2025-01-01'"]
}
```

**MySQL Delete (DANGEROUS):**
```json
{
  "P1": "admin",
  "P2": "token123",
  "P3": ["mysql_exec", "DELETE FROM messages WHERE conversation_uuid = 'abc-123'"]
}
```

**DGraph Read Query:**
```json
{
  "P1": "admin",
  "P2": "token123",
  "P3": ["dgraph", "{ users(func: type(users), first: 10) { uid uuid } }"]
}
```

**DGraph Delete by UID:**
```json
{
  "P1": "admin",
  "P2": "token123",
  "P3": ["dgraph_upsert", "delete:0x1234"]
}
```

**DGraph Schema Alteration (add predicates):**
```json
{
  "P1": "admin",
  "P2": "token123",
  "P3": ["dgraph_upsert", "schema:folderAllowFollowers: string @index(exact) .\nfolderAllowFollowing: string @index(exact) ."]
}
```

**DGraph Delete All of Type (DANGEROUS):**
```json
{
  "P1": "admin",
  "P2": "token123",
  "P3": ["dgraph_upsert", "{\"query\": \"{ conv as var(func: type(conversation)) }\", \"delete\": \"uid(conv) * * .\", \"set\": \"\"}"]
}
```

---

## Recovery APIs

### APPADMIN_RECOVERY

Provides recovery and maintenance operations. Some operations are dangerous and require confirmation codes.

**Endpoint:** POST to standard API handler  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Username |
| P2 | string | Access token |
| P3[0] | string | Action (required) |
| P3[1] | string | Parameter (varies by action) |
| P3[2] | string | Confirmation code (for dangerous actions) |

**P3[0] Action Values:**

| Action | Description | P3[1] | P3[2] | Dangerous |
|--------|-------------|-------|-------|-----------|
| `"status"` | Get server/recovery status | - | - | No |
| `"reload_config"` | Reload dynamic configuration | - | - | No |
| `"gc"` | Force garbage collection | - | - | No |
| `"generate_confirmation"` | Generate confirmation code | Action name | - | No |
| `"restart"` | Restart server process | - | Confirmation code | **Yes** |
| `"timeline_replay"` | Replay events from offset | Start offset | Confirmation code | **Yes** |
| `"db_reset"` | Purge DB and replay all events | - | Confirmation code | **Yes** |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | JSON result object |

**P3[0] JSON Structure for `"status"` (`RecoveryStatus`):**

```json
{
  "server_uptime": "2h30m15s",
  "memory_alloc": "45.2 MB",
  "num_goroutines": 156,
  "num_cpu": 4,
  "timeline_connected": true,
  "database_connected": true,
  "last_config_reload": "2025-01-15T09:00:00Z"
}
```

**P3[0] JSON Structure for actions (`RecoveryAction`):**

```json
{
  "action": "reload_config",
  "status": "success",
  "message": "Dynamic configuration reloaded successfully",
  "started_at": "2025-01-15T10:30:00Z",
  "completed_at": "2025-01-15T10:30:01Z",
  "duration": "1.234s"
}
```

#### Dangerous Operations Workflow

For dangerous operations (`restart`, `timeline_replay`, `db_reset`):

1. **Generate confirmation code:**
```json
{
  "P1": "admin",
  "P2": "token123",
  "P3": ["generate_confirmation", "restart"]
}
```
Response contains a confirmation code valid for a short time.

2. **Execute with confirmation:**
```json
{
  "P1": "admin",
  "P2": "token123",
  "P3": ["restart", "", "ABC12345"]
}
```

---

### APPADMIN_MAINTENANCE

Provides scheduled maintenance operations for database optimization.

**Endpoint:** POST to standard API handler  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Username |
| P2 | string | Access token |
| P3[0] | string | Action (required) |

**P3[0] Action Values:**

| Action | Description |
|--------|-------------|
| `"optimize_tables"` | Run OPTIMIZE TABLE on all InnoDB tables |
| `"analyze_tables"` | Run ANALYZE TABLE on all tables |
| `"check_integrity"` | Check database and DGraph integrity |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Summary message |
| P3[0] | JSON result object |

**P3[0] JSON Structure for `"optimize_tables"`:**

```json
{
  "action": "optimize_tables",
  "tables": {
    "users": "optimized",
    "posts": "optimized",
    "sessions": "failed: table locked"
  },
  "duration": "45.678s"
}
```

**P3[0] JSON Structure for `"check_integrity"`:**

```json
{
  "mysql": {
    "status": "healthy",
    "tables_checked": 25,
    "issues": []
  },
  "dgraph": {
    "status": "healthy",
    "schema_valid": true
  },
  "collected_at": "2025-01-15T10:30:00Z"
}
```

---

## Vox APIs

### APPADMIN_VOXBROADCAST

Creates a **global Vox broadcast** with a plain-text message and notifies all currently connected websocket sessions.

This API is intended for the AppAdmin console to push operational announcements.

**Endpoint:** `POST /api/APPADMIN_VOXBROADCAST`  
**Authentication:** RoleRoot

**Important naming note (re: api_code confusion):**
- The **HTTP API name** is `APPADMIN_VOXBROADCAST`.
- The **stored Vox record** is created via `VoxBroadcast(...)` and uses `api_code = "Vox_Broadcast"` in `vox_broadcasts`.
- The **websocket push** uses `Req = "SAPI_VoxEvent"`, with the embedded JSON field `apiCode = "Vox_Broadcast"`.

#### Request

| Field | Type | Required | Description |
|------|------|----------|-------------|
| P1 | string | Yes | Admin username |
| P2 | string | Yes | Admin access token |
| P3[0] | string | Yes | Message text |

#### Response

**Success:**
| Field | Value |
|------|-------|
| P1 | `"Success"` |
| P2 | `"Broadcast created"` |
| P3[0] | JSON `{ "voxUuid":"...", "apiCode":"Vox_Broadcast", "createdAt":"...", "deliveredLocal": N }` |

**Failure:**
| Field | Value |
|------|-------|
| P1 | `"Fail"` |
| P2 | Error message |

#### Websocket notification

Connected clients receive a websocket message:
- `Req = "SAPI_VoxEvent"`
- `P1 = "Success"`
- `P2 = JSON` containing `{ voxUuid, apiCode:"Vox_Broadcast", createdAt, payload:{message:"..."} }`

#### Example

```bash
curl -X POST https://api.lexcv.co/api/APPADMIN_VOXBROADCAST \
  -H "Content-Type: application/json" \
  -d '{
    "p1":"admin@lexcv.co",
    "p2":"<token>",
    "p3":["Scheduled maintenance at 22:00 UTC"]
  }'
```


### APPADMIN_VOXEVENT

Creates a **per-user Vox inbox event** (for testing) and notifies that user’s currently connected websocket sessions.

**Endpoint:** `POST /api/APPADMIN_VOXEVENT`  
**Authentication:** RoleRoot

**Vox api_code used:** `Vox_AdminMsg`

#### Request

| Field | Type | Required | Description |
|------|------|----------|-------------|
| P1 | string | Yes | Admin username |
| P2 | string | Yes | Admin access token |
| P3[0] | string | Yes | Target user UUID |
| P3[1] | string | Yes | Message text |

#### Response

**Success:**
| Field | Value |
|------|-------|
| P1 | `"Success"` |
| P2 | `"Vox event created"` |
| P3[0] | JSON `{ "voxUuid":"...", "apiCode":"Vox_AdminMsg", "createdAt":"..." }` |

**Failure:**
| Field | Value |
|------|-------|
| P1 | `"Fail"` |
| P2 | Error message |

#### Websocket notification

The target user will receive a websocket message:
- `Req = "SAPI_VoxEvent"`
- `P1 = "Success"`
- `P2 = JSON` containing `{ voxUuid, apiCode:"Vox_AdminMsg", createdAt, actorUuid, payload:{message:"..."} }`

#### Example

```bash
curl -X POST https://api.lexcv.co/api/APPADMIN_VOXEVENT \
  -H "Content-Type: application/json" \
  -d '{
    "p1":"admin@lexcv.co",
    "p2":"<token>",
    "p3":["<targetUserUuid>","Hello from AppAdmin"]
  }'
```


---

## Cato APIs

These APIs provide an AppAdmin management surface for Cato (tuple-based authorization). They are designed to support an “Active Directory style” permissions UI.

For Cato model semantics (tuple format, deny precedence, and external checks), see: [notes/API References/11_Cato_Authorization.md](notes/API%20References/11_Cato_Authorization.md)

**Authentication:** All `APPADMIN_CATO_*` endpoints require **RoleRoot**.

**Core concepts:**
- **Resource** is identified as `(resource_type, resource_id)`.
- **Relation** is a string like `view`, `edit`, `root`, etc.
- **Subject** is either a direct user (`subject_type="user"`, `subject_relation=""`) or a userset (`subject_type="group"`, `subject_relation="member"` meaning `group:<id>#member`).
- **Tuple ID** is deterministic (SHA-256 of canonical tuple key) and returned by grant/revoke APIs.

### APPADMIN_CATO_EVALUATE

Evaluates whether a user has a relation on a resource.

**Endpoint:** `POST /api/APPADMIN_CATO_EVALUATE`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Admin username |
| P2 | string | Admin access token / bypass code |
| P3[0] | string | user_uuid |
| P3[1] | string | resource_type |
| P3[2] | string | resource_id |
| P3[3] | string | relation |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | JSON `Decision` |

**Decision JSON structure:**
```json
{
  "kind": "allow|deny|external_check",
  "proven_allow": true,
  "proven_deny": false,
  "allow_conditions": [{"kind":"...","effect":"allow","params":{},"note":"..."}],
  "deny_conditions": [{"kind":"...","effect":"deny","params":{},"note":"..."}]
}
```

#### Example

```bash
curl -X POST https://api.lexcv.co/api/APPADMIN_CATO_EVALUATE \
  -H "Content-Type: application/json" \
  -d '{
    "p1":"admin@lexcv.co",
    "p2":"<token>",
    "p3":["<userUuid>","system","main","root"]
  }'
```

---

### APPADMIN_CATO_EXPLAIN

Returns the decision plus a proof path of tuples (when a tuple-only proof exists).

**Endpoint:** `POST /api/APPADMIN_CATO_EXPLAIN`  
**Authentication:** RoleRoot required

#### Request

Same as `APPADMIN_CATO_EVALUATE`.

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | JSON `ExplainResult` |

**ExplainResult JSON structure (high level):**
```json
{
  "user_id": "...",
  "resource": {"type":"...","id":"..."},
  "relation": "...",
  "decision": {"kind":"allow|deny|external_check", "proven_allow": true, "proven_deny": false},
  "deny_path": [{"kind":"tuple","tuple":{"resource":{},"relation":"...","subject":{}}}],
  "allow_path": [{"kind":"tuple","tuple":{"resource":{},"relation":"...","subject":{}}}]
}
```

#### Example

```bash
curl -X POST https://api.lexcv.co/api/APPADMIN_CATO_EXPLAIN \
  -H "Content-Type: application/json" \
  -d '{
    "p1":"admin@lexcv.co",
    "p2":"<token>",
    "p3":["<userUuid>","profile","<profileUuid>","view"]
  }'
```

---

### APPADMIN_CATO_EFFECTIVE_FOR_USER

Lists effective grants for a user, expanding internal group membership (`group:<id>#member`).

**Endpoint:** `POST /api/APPADMIN_CATO_EFFECTIVE_FOR_USER`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Admin username |
| P2 | string | Admin access token / bypass code |
| P3[0] | string | user_uuid |
| P3[1] | string | offset (optional, default 0) |
| P3[2] | string | limit (optional, default 50, max 200) |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | total_count |
| P4 | Rows: `[resource_type, resource_id, relation, source, group_id, tuple_id]` |

`source` is `direct` (explicit tuple to user) or `group` (via group membership).

---

### APPADMIN_CATO_EFFECTIVE_FOR_RESOURCE

Lists effective users for a given `resource#relation`, expanding internal group membership.

**Endpoint:** `POST /api/APPADMIN_CATO_EFFECTIVE_FOR_RESOURCE`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Admin username |
| P2 | string | Admin access token / bypass code |
| P3[0] | string | resource_type |
| P3[1] | string | resource_id |
| P3[2] | string | relation |
| P3[3] | string | offset (optional, default 0) |
| P3[4] | string | limit (optional, default 50, max 200) |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | total_count |
| P4 | Rows: `[user_uuid, source, group_id, grant_tuple_id, membership_tuple_id]` |

---

### APPADMIN_CATO_EFFECTIVE_FOR_USER_DISTINCT

Returns one row per distinct `(resource_type, resource_id, relation)` with provenance (`reasons_json`).

**Endpoint:** `POST /api/APPADMIN_CATO_EFFECTIVE_FOR_USER_DISTINCT`  
**Authentication:** RoleRoot required

#### Request

Same as `APPADMIN_CATO_EFFECTIVE_FOR_USER`.

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | total_count |
| P4 | Rows: `[resource_type, resource_id, relation, reasons_json]` |

`reasons_json` is a JSON array of the underlying effective rows:
```json
[
  {"resource":{"type":"...","id":"..."},"relation":"...","source":"direct","group_id":"","tuple_id":"..."},
  {"resource":{"type":"...","id":"..."},"relation":"...","source":"group","group_id":"<groupId>","tuple_id":"..."}
]
```

---

### APPADMIN_CATO_EFFECTIVE_FOR_RESOURCE_DISTINCT

Returns one row per distinct `user_uuid` with provenance (`reasons_json`).

**Endpoint:** `POST /api/APPADMIN_CATO_EFFECTIVE_FOR_RESOURCE_DISTINCT`  
**Authentication:** RoleRoot required

#### Request

Same as `APPADMIN_CATO_EFFECTIVE_FOR_RESOURCE`.

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | total_count |
| P4 | Rows: `[user_uuid, reasons_json]` |

`reasons_json` is a JSON array of the underlying effective rows:
```json
[
  {"user_id":"<userUuid>","source":"direct","group_id":"","grant_tuple_id":"...","membership_tuple_id":""},
  {"user_id":"<userUuid>","source":"group","group_id":"<groupId>","grant_tuple_id":"...","membership_tuple_id":"..."}
]
```

---

### APPADMIN_CATO_TUPLE_GRANT

Writes a single tuple.

**Endpoint:** `POST /api/APPADMIN_CATO_TUPLE_GRANT`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Admin username |
| P2 | string | Admin access token / bypass code |
| P3[0] | string | resource_type |
| P3[1] | string | resource_id |
| P3[2] | string | relation |
| P3[3] | string | subject_type |
| P3[4] | string | subject_id |
| P3[5] | string | subject_relation (optional) |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | tuple_id |

---

### APPADMIN_CATO_TUPLE_REVOKE

Revokes (soft-deletes) a single tuple.

**Endpoint:** `POST /api/APPADMIN_CATO_TUPLE_REVOKE`  
**Authentication:** RoleRoot required

#### Request

Same as `APPADMIN_CATO_TUPLE_GRANT`.

#### Response

Same response shape as `APPADMIN_CATO_TUPLE_GRANT` (returns deterministic `tuple_id`).

---

### APPADMIN_CATO_TUPLE_LIST

Lists tuples with filters and paging.

**Endpoint:** `POST /api/APPADMIN_CATO_TUPLE_LIST`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Admin username |
| P2 | string | Admin access token / bypass code |
| P3[0] | string | resource_type (optional) |
| P3[1] | string | resource_id (optional) |
| P3[2] | string | relation (optional) |
| P3[3] | string | subject_type (optional) |
| P3[4] | string | subject_id (optional) |
| P3[5] | string | subject_relation (optional; if provided it filters even if empty) |
| P3[6] | string | include_deleted (optional `"0"`/`"1"`, default `0`) |
| P3[7] | string | offset (optional, default 0) |
| P3[8] | string | limit (optional, default 50, max 200) |

**Subject relation filtering note:** to filter direct user grants you may include `P3[5]` as an empty string.

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | total_count |
| P4 | Rows: `[tuple_id, resource_type, resource_id, relation, subject_type, subject_id, subject_relation, deleted, created_at, updated_at]` |

`created_at` and `updated_at` are Unix milliseconds.

---

### APPADMIN_CATO_POLICY_RESOURCE

Lists subjects granted `resource#relation`.

**Endpoint:** `POST /api/APPADMIN_CATO_POLICY_RESOURCE`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Admin username |
| P2 | string | Admin access token / bypass code |
| P3[0] | string | resource_type |
| P3[1] | string | resource_id |
| P3[2] | string | relation |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P4 | Rows: `[subject_type, subject_id, subject_relation]` |

---

### APPADMIN_CATO_GRANTS_FOR_SUBJECT

Lists tuples for a subject.

**Endpoint:** `POST /api/APPADMIN_CATO_GRANTS_FOR_SUBJECT`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Admin username |
| P2 | string | Admin access token / bypass code |
| P3[0] | string | subject_type |
| P3[1] | string | subject_id |
| P3[2] | string | include_deleted (optional `"0"`/`"1"`, default `0`) |
| P3[3] | string | offset (optional, default 0) |
| P3[4] | string | limit (optional, default 50, max 200) |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | total_count |
| P4 | Rows: `[tuple_id, resource_type, resource_id, relation, subject_type, subject_id, subject_relation, deleted, created_at, updated_at]` |

---

### APPADMIN_CATO_RELATIONS_FOR_RESOURCE

Lists distinct relations used on a resource.

**Endpoint:** `POST /api/APPADMIN_CATO_RELATIONS_FOR_RESOURCE`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Admin username |
| P2 | string | Admin access token / bypass code |
| P3[0] | string | resource_type |
| P3[1] | string | resource_id |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3 | List of relation strings |

---

### APPADMIN_CATO_RELATIONS_FOR_SUBJECT

Lists distinct relations granted to a subject.

**Endpoint:** `POST /api/APPADMIN_CATO_RELATIONS_FOR_SUBJECT`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Admin username |
| P2 | string | Admin access token / bypass code |
| P3[0] | string | subject_type |
| P3[1] | string | subject_id |
| P3[2] | string | subject_relation (optional, default empty) |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3 | List of relation strings |

---

### APPADMIN_CATO_TUPLE_BULK_GRANT

Grants tuples in bulk.

**Endpoint:** `POST /api/APPADMIN_CATO_TUPLE_BULK_GRANT`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Admin username |
| P2 | string | Admin access token / bypass code |
| P4 | [][]string | Rows: `[resource_type, resource_id, relation, subject_type, subject_id, subject_relation]` |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | success_count |
| P3[1] | fail_count |
| P4 | Rows: `[tuple_id, status, message]` |

---

### APPADMIN_CATO_TUPLE_BULK_REVOKE

Revokes tuples in bulk.

**Endpoint:** `POST /api/APPADMIN_CATO_TUPLE_BULK_REVOKE`  
**Authentication:** RoleRoot required

#### Request

Same as `APPADMIN_CATO_TUPLE_BULK_GRANT`.

#### Response

Same as `APPADMIN_CATO_TUPLE_BULK_GRANT`.

---

### APPADMIN_CATO_GROUP_CREATE

Creates a group by granting an ownership marker tuple on `group:<group_id>`.

**Endpoint:** `POST /api/APPADMIN_CATO_GROUP_CREATE`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Admin username |
| P2 | string | Admin access token / bypass code |
| P3[0] | string | group_id (optional; generated if empty) |
| P3[1] | string | owner_user_uuid (required) |
| P3[2] | string | also_add_owner_as_member (optional `"0"`/`"1"`, default `1`) |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | group_id |

---

### APPADMIN_CATO_GROUP_DELETE

Deletes a group by revoking all tuples where `resource_type="group"` and `resource_id=<group_id>`.

**Endpoint:** `POST /api/APPADMIN_CATO_GROUP_DELETE`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Admin username |
| P2 | string | Admin access token / bypass code |
| P3[0] | string | group_id |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | revoked_count |

---

### APPADMIN_CATO_GROUP_LIST

Lists groups via ownership marker tuples (`group:<id>#owner@user:<uuid>`).

**Endpoint:** `POST /api/APPADMIN_CATO_GROUP_LIST`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Admin username |
| P2 | string | Admin access token / bypass code |
| P3[0] | string | offset (optional, default 0) |
| P3[1] | string | limit (optional, default 50, max 200) |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | total_count |
| P4 | Rows: `[group_id, owner_user_uuid]` |

---

### APPADMIN_CATO_GROUP_MEMBERS

Lists group members (direct `group:<id>#member@user:<uuid>` tuples).

**Endpoint:** `POST /api/APPADMIN_CATO_GROUP_MEMBERS`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Admin username |
| P2 | string | Admin access token / bypass code |
| P3[0] | string | group_id |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P4 | Rows: `[user_uuid]` |

---

### APPADMIN_CATO_GROUP_ADD_MEMBER

Adds a user to a group.

**Endpoint:** `POST /api/APPADMIN_CATO_GROUP_ADD_MEMBER`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Admin username |
| P2 | string | Admin access token / bypass code |
| P3[0] | string | group_id |
| P3[1] | string | user_uuid |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |

---

### APPADMIN_CATO_GROUP_REMOVE_MEMBER

Removes a user from a group.

**Endpoint:** `POST /api/APPADMIN_CATO_GROUP_REMOVE_MEMBER`  
**Authentication:** RoleRoot required

#### Request

Same as `APPADMIN_CATO_GROUP_ADD_MEMBER`.

#### Response

Same response shape as `APPADMIN_CATO_GROUP_ADD_MEMBER`.

---

### APPADMIN_CATO_GROUP_GRANT_RESOURCE

Grants `resource#relation` to the group userset (`group:<id>#member`).

**Endpoint:** `POST /api/APPADMIN_CATO_GROUP_GRANT_RESOURCE`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Admin username |
| P2 | string | Admin access token / bypass code |
| P3[0] | string | group_id |
| P3[1] | string | resource_type |
| P3[2] | string | resource_id |
| P3[3] | string | relation |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | tuple_id |

---

### APPADMIN_CATO_GROUP_REVOKE_RESOURCE

Revokes `resource#relation` from the group userset (`group:<id>#member`).

**Endpoint:** `POST /api/APPADMIN_CATO_GROUP_REVOKE_RESOURCE`  
**Authentication:** RoleRoot required

#### Request

Same as `APPADMIN_CATO_GROUP_GRANT_RESOURCE`.

#### Response

Same response shape as `APPADMIN_CATO_GROUP_GRANT_RESOURCE` (returns deterministic `tuple_id`).

---

### APPADMIN_CATO_GROUP_GRANTS

Lists tuples where the subject is `group:<id>#member` (i.e., resources granted to the group).

**Endpoint:** `POST /api/APPADMIN_CATO_GROUP_GRANTS`  
**Authentication:** RoleRoot required

#### Request

| Field | Type | Description |
|-------|------|-------------|
| P1 | string | Admin username |
| P2 | string | Admin access token / bypass code |
| P3[0] | string | group_id |
| P3[1] | string | offset (optional, default 0) |
| P3[2] | string | limit (optional, default 50, max 200) |

#### Response

| Field | Description |
|-------|-------------|
| P1 | `"Success"` or `"Fail"` |
| P2 | Message |
| P3[0] | total_count |
| P4 | Rows: `[tuple_id, resource_type, resource_id, relation]` |

---

## Error Handling

All APIs follow consistent error handling:

**Authentication Failure:**
```json
{
  "P1": "Fail",
  "P2": "Root access required"
}
```

**Missing Parameters:**
```json
{
  "P1": "Fail",
  "P2": "Missing action or log source"
}
```

**Unknown Action:**
```json
{
  "P1": "Fail",
  "P2": "Unknown action: xyz"
}
```

**Database Error:**
```json
{
  "P1": "Fail",
  "P2": "Database not initialized"
}
```

---

## Notes

1. **Rate Limiting:** Admin APIs are not rate-limited but should be used responsibly.
2. **Timeouts:** Database queries have a 30-second timeout; maintenance operations have longer timeouts (up to 5 minutes per table).
3. **Log Limits:** Maximum 5000 lines per log request; maximum 1000 search results.
4. **Confirmation Codes:** Valid for a short time period and single use.
5. **Platform Differences:** Some features (nginx status, system logs) may not be available on Windows.

---

## Schema Reference

The admin APIs reference the following database tables:

### sacred_timeline (Event Store)
```sql
CREATE TABLE sacred_timeline (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    event_id VARCHAR(36) NOT NULL,           -- UUID, idempotency key
    api_code VARCHAR(100) NOT NULL,          -- e.g., "PROFILE_BASIC_UPDATE"
    query_type VARCHAR(20) NOT NULL,         -- "mysql", "mysql_tx", "dgraph"
    queries JSON NOT NULL,                   -- Array of query strings
    params JSON,                             -- Array of param arrays
    metadata JSON,                           -- actor, source node, correlation ID
    created_at BIGINT UNSIGNED NOT NULL,     -- Unix ms timestamp
    partition_key VARCHAR(100),              -- for sharding
    sequence_num BIGINT UNSIGNED,            -- Redpanda offset
    partition_num INT UNSIGNED,              -- Redpanda partition
    source_node VARCHAR(100),                -- Which server published
    processed_at BIGINT UNSIGNED,            -- Unix ms when received
    UNIQUE KEY uk_event_id (event_id)
);
```

### sacred_timeline_outbox (Transactional Outbox)
```sql
CREATE TABLE sacred_timeline_outbox (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    event_json TEXT NOT NULL,                -- Serialized event
    aggregate_id VARCHAR(64) NOT NULL,       -- For partitioning
    api_code VARCHAR(64) NOT NULL,           -- For debugging
    created_at BIGINT UNSIGNED NOT NULL,     -- Unix ms timestamp
    retry_count INT DEFAULT 0,               -- Publish attempts
    last_error TEXT NULL                     -- Last error message
);
```

### sacred_timeline_watermarks (Consumer Offsets)
```sql
CREATE TABLE sacred_timeline_watermarks (
    partition_num INT UNSIGNED NOT NULL PRIMARY KEY,
    last_offset BIGINT UNSIGNED NOT NULL DEFAULT 0,
    updated_at BIGINT UNSIGNED NOT NULL      -- Unix ms timestamp
);
```

---

*Last updated: December 2025*
