# Admin APIs

Administrative APIs for system management. These require elevated privileges.

---

## Overview

Admin APIs are divided into categories:

| Category | Prefix | Required Role | Description |
|----------|--------|---------------|-------------|
| App Admin | `APPADMIN_` | Root | User/flag management |
| Admin | `AdminAPI_` | Deployment | System setup/config |
| Standard Admin | `API_Admin*` | Admin | Monitoring/stats |

---

## Permission Levels

### System Roles (Cato tuples on `system:main`)

| Role | Level | Description |
|------|-------|-------------|
| `root` | Highest | Full system access, user flag modification |
| `admin` | High | Rate limit management, monitoring |
| `moderator` | Medium | Content moderation |
| (none) | Standard | Regular user |

### Checking Roles

```go
hasRoot, _ := HasSystemRole(loginResult, users.RoleRoot) // true if root
isAdmin, _ := IsSystemAdmin(loginResult)                 // true if admin or root
```

---

## APPADMIN_USERFLAGS

**Purpose:** Manages user flags for any user in the system.

**Authentication:** Requires ROOT privileges

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"APPADMIN_USERFLAGS"` |
| `p1` | string | Yes | Admin username (email) |
| `p2` | string | Yes | Admin token |
| `p3[0]` | string | Yes | Action (see below) |
| `p3[1]` | string | Yes | Target user UUID |
| `p3[2+]` | string | Varies | Action-specific parameters |

### Actions

#### GET - Get user's current flags

**Request:**
```json
{
  "p3": ["get", "target-user-uuid"]
}
```

**Response:**
```json
{
  "p1": "Success",
  "p3": ["{\"user_state\":[\"verified\"],\"user_type\":[\"lawyer\"],\"user_level\":[\"premium\"]}"]
}
```

#### SET - Replace all flags

**Request:**
```json
{
  "p3": ["set", "target-user-uuid", "{\"user_state\":[\"verified\"],\"user_type\":[\"lawyer\"],\"user_level\":[\"level1\"]}"]
}
```

#### ADD_STATE - Add a user_state value

**Request:**
```json
{
  "p3": ["add_state", "target-user-uuid", "verified"]
}
```

**Valid States:** `verified`, `suspended`, `deleted`, `premium`

#### REMOVE_STATE - Remove a user_state value

**Request:**
```json
{
  "p3": ["remove_state", "target-user-uuid", "suspended"]
}
```

#### ADD_ROLE - Grant a system role (Cato)

**Request:**
```json
{
  "p3": ["add_role", "target-user-uuid", "admin"]
}
```

**Valid Roles:** `root`, `admin`, `moderator`

#### REMOVE_ROLE - Revoke a system role (Cato)

**Request:**
```json
{
  "p3": ["remove_role", "target-user-uuid", "admin"]
}
```

#### SET_TYPE - Set user_type

**Request:**
```json
{
  "p3": ["set_type", "target-user-uuid", "lawyer"]
}
```

**Valid Types:** `lawyer`, `otherpro`, `company`, `student`, `public`

#### SET_LEVEL - Set user_level

**Request:**
```json
{
  "p3": ["set_level", "target-user-uuid", "level2"]
}
```

**Valid Levels:** `level1`, `level2`, `level3`, `level4`

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message or error |
| `p3[0]` | Updated flags JSON (on success) |

---

## API_AdminRateLimits

**Purpose:** Manages rate limiting statistics and violations.

**Authentication:** Requires Admin or Root privileges

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"API_AdminRateLimits"` |
| `p1` | string | Yes | Admin username (email) |
| `p2` | string | Yes | Admin token |
| `p3[0]` | string | Yes | Action: `"stats"`, `"violators"`, `"clear"` |
| `p3[1+]` | string | Varies | Action-specific parameters |

### Actions

#### STATS - Get overall statistics

**Request:**
```json
{
  "p3": ["stats"]
}
```

**Response:**
```json
{
  "p1": "Success",
  "p2": "{\"totalRequests\":15234,\"blockedRequests\":42,\"activeTokens\":567}"
}
```

#### VIOLATORS - List rate limit violators

**Request:**
```json
{
  "p3": ["violators", "5"]
}
```

Where `p3[1]` is minimum violation count (default: 1)

**Response:**
```json
{
  "p1": "Success",
  "p2": "[{\"id\":\"192.168.1.1\",\"type\":\"ip\",\"violations\":23},{\"id\":\"user-uuid\",\"type\":\"user\",\"violations\":12}]"
}
```

#### CLEAR - Clear violations for user/IP

**Request:**
```json
{
  "p3": ["clear", "192.168.1.1", "ip"]
}
```

Or for user:
```json
{
  "p3": ["clear", "user-uuid", "user"]
}
```

---

## AdminAPI_SetCFG

**Purpose:** Populates/updates the configuration table with default values.

**Authentication:** None (deployment script endpoint)

**⚠️ WARNING:** This endpoint is for deployment automation only. It should be protected at the network/firewall level.

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"AdminAPI_SetCFG"` |

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message |

### Behavior

- Uses `INSERT ... ON DUPLICATE KEY UPDATE` for upsert
- Populates all default configuration values
- Safe to run multiple times (idempotent)

### Config Keys Set

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `web_root` | string | `/var/www` | Web root directory |
| `search_results_per_page` | int | 5 | Search pagination |
| `max_auto_complete` | int | 10 | Autocomplete limit |
| `app_title` | string | `"LexCV"` | Application title |
| `motd` | string | `""` | Message of the day |
| `profile_link_length_min` | int | 12 | Vanity URL min length |
| `profile_link_length_max` | int | 30 | Vanity URL max length |
| `posts_per_page` | int | 20 | Post feed pagination |
| `media_cdn_base_url` | string | `https://cdn.lexcv.co` | CDN base URL |
| `media_token_ttl_days` | int | 7 | CDN token lifetime |

---

## AdminAPI_CleanupGraph

**Purpose:** Removes orphaned nodes and invalid edges from DGraph.

**Authentication:** None (deployment script endpoint)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"AdminAPI_CleanupGraph"` |

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Cleanup summary |

---

## AdminAPI_ResetGraph

**Purpose:** Resets DGraph schema and data. **DESTRUCTIVE.**

**Authentication:** None (deployment script endpoint)

**⚠️ WARNING:** This deletes all graph data. Only use in development/testing.

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"AdminAPI_ResetGraph"` |
| `p3[0]` | string | Yes | Confirmation: `"CONFIRM_RESET"` |

---

## AdminAPI_SampleData

**Purpose:** Populates the system with sample/test data.

**Authentication:** None (deployment script endpoint)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"AdminAPI_SampleData"` |
| `p3[0]` | string | No | Data set: `"minimal"`, `"standard"`, `"full"` |

### Data Sets

| Set | Users | Posts | Comments | Description |
|-----|-------|-------|----------|-------------|
| `minimal` | 5 | 10 | 20 | Basic testing |
| `standard` | 50 | 200 | 500 | Feature testing |
| `full` | 500 | 2000 | 5000 | Load testing |

---

## AdminAPI_TimelineStatus

**Purpose:** Returns the catchup status for all Sacred Timeline partitions.

**Authentication:** None (deployment script endpoint)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"AdminAPI_TimelineStatus"` |

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"OK"` or `"ERROR"` |
| `p2` | Summary: events behind, partitions status |
| `p3[0]` | JSON detail per partition |

---

## AdminAPI_TimelineCatchup

**Purpose:** Triggers a manual catchup from watermarks. Replays events missed while the node was offline.

**Authentication:** None (deployment script endpoint)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"AdminAPI_TimelineCatchup"` |

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"OK"` or `"ERROR"` |
| `p2` | Result summary: events replayed, skipped, duration |

---

## AdminAPI_TimelineReconcile

**Purpose:** Triggers a FULL reconciliation (replay ALL events from start). **WARNING:** Can take a very long time.

**Authentication:** None (deployment script endpoint)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"AdminAPI_TimelineReconcile"` |

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"OK"` or `"ERROR"` |
| `p2` | Result summary: events replayed, skipped, duration |

---

## AdminAPI_ClearUsers

**Purpose:** Clears all users from the system. **DESTRUCTIVE.**

**Authentication:** None (deployment script endpoint)

**⚠️ WARNING:** This deletes all user data. Only use in development/testing.

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"AdminAPI_ClearUsers"` |

---

## AdminAPI_AddBlogSchema

**Purpose:** Adds the blog schema to DGraph.

**Authentication:** None (deployment script endpoint)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"AdminAPI_AddBlogSchema"` |

---

## AdminAPI_BlogCreateDefaultFolders

**Purpose:** Creates default blog folders for all users.

**Authentication:** None (deployment script endpoint)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"AdminAPI_BlogCreateDefaultFolders"` |

---

## Security Notes

### Admin API Protection

1. **Network Level:** Admin APIs should only be accessible from internal networks or specific IPs
2. **Authentication:** APPADMIN and API_Admin APIs require proper authentication
3. **Logging:** All admin actions are logged with user identification
4. **Rate Limiting:** Admin APIs have separate, more generous rate limits

### Audit Trail

Admin actions create audit log entries:

```
[APPADMIN_USERFLAGS] Root access by: admin-user-uuid
[APPADMIN_USERFLAGS] Added role 'admin' to user: target-user-uuid
```

Logs are stored in the application log and are searchable by action type.
