# Authentication & Session APIs

APIs for user authentication, session management, and account lifecycle.

---

## API_FirebaseLogin

**Purpose:** Authenticates a user via Firebase, creates new accounts if needed, and triggers email verification.

**Authentication:** Firebase ID token (not server token)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"API_FirebaseLogin"` |
| `p1` | string | No | Not used |
| `p2` | string | Yes | Firebase ID token from client SDK |
| `p4[0][0]` | string (JSON) | No | Optional `user_meta` JSON object; if present, stored best-effort in `users.user_meta` |

### Example Request

```json
{
  "req": "API_FirebaseLogin",
  "p1": "",
  "p2": "<firebase_id_token>",
  "p3": [],
  "p4": [["{\"device_type\":\"web\",\"device_id\":\"...\",\"app_version\":\"1.0.0\"}"]],
  "p5": [],
  "p6": []
}
```

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message - see scenarios below |
| `p3[...]` | Success payload (varies by scenario) |

**Success `p3` conventions:**
- `user_meta` is always returned as the **last** `p3` element (JSON string).
- Verified users also receive:
  - `p3[0]` server token
  - `p3[1]` user flags JSON

### Response Scenarios

**New User Created (unverified):**
```json
{
  "p1": "Success",
  "p2": "new user",
  "p3": ["{\"app_platform\":\"ios\",\"app_version\":\"1.2.3\"}"]
}
```

**Existing Verified User:**
```json
{
  "p1": "Success",
  "p2": "logged in",
  "p3": [
    "server_token_here",
    "{\"user_state\":[\"verified\"],\"user_type\":[\"lawyer\"]}",
    "{\"app_platform\":\"ios\",\"app_version\":\"1.2.3\"}"
  ]
}
```

**Existing Unverified User (code already sent):**
```json
{
  "p1": "Success",
  "p2": "enter code",
  "p3": ["{}"]
}
```

**Existing Unverified User (code just sent):**
```json
{
  "p1": "Success",
  "p2": "code sent",
  "p3": ["{}"]
}
```

**Invalid Firebase Token:**
```json
{
  "p1": "Fail",
  "p2": "error message from Firebase"
}
```

---

## API_VerifyUser

**Purpose:** Verifies a user's email using the 6-digit code sent via email.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"API_VerifyUser"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token (access_token or server_token) |
| `p3[0]` | string | Yes | 6-digit verification code from email |

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message |

### Response Scenarios

**Verification Successful:**
```json
{
  "p1": "Success",
  "p2": "verified"
}
```

**Already Verified:**
```json
{
  "p1": "Fail",
  "p2": "already verified"
}
```

**Invalid Code:**
```json
{
  "p1": "Fail",
  "p2": "bad code"
}
```

---

## API_CheckLogin

**Purpose:** Validates current session and returns user info.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"API_CheckLogin"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token (access_token or server_token) |
| `p3[0]` | string | No | Optional `user_meta` JSON object (best-effort update). Must be a JSON **object** (not an array/string). |

### user_meta format (internal)

`user_meta` is stored in `users.user_meta` (MySQL `JSON NULL`) and is intended for internal diagnostics/security/UX.

- **Top-level must be an object** (e.g. `{...}`)
- Empty/whitespace is treated as “no update”
- The literal string `"null"` is treated as `{}`
- If the stored column is `NULL`, the server treats it as `{}` when reading
- Invalid JSON or non-object JSON is ignored (login still succeeds)

Common fields (see `data.UserMeta`):

```json
{
  "device_type": "ios",
  "device_id": "<stable-id-if-available>",
  "device_model": "iPhone14,3",
  "os": "iOS",
  "os_version": "17.2",
  "app_version": "1.2.3",
  "locale": "en-GB",
  "timezone": "Europe/London",
  "screen_w": 390,
  "screen_h": 844,
  "screen_scale": 3
}
```

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Username (echo back) |
| `p3[0]` | User flags JSON: `{"user_state":[], "user_type":[], "user_level":[]}` |
| `p3[1]` | Cato grants JSON (extensible): `{"grants":[{"resource":{"type":"system","id":"main"},"relations":[...]}]}` |
| `p3[2]` | User UUID |

**Note:** `user_meta` is not returned by this API response; it is consumed internally by the server (via `CheckLoginOrFail` / `users.CheckLogin`).

### Response Example

```json
{
  "p1": "Success",
  "p2": "user@email.com",
  "p3": [
    "{\"user_state\":[\"verified\"],\"user_type\":[\"lawyer\"]}",
    "{\"grants\":[{\"resource\":{\"type\":\"system\",\"id\":\"main\"},\"relations\":[\"admin\"]}]}",
    "550e8400-e29b-41d4-a716-446655440000"
  ]
}
```

---

## API_SignOut

**Purpose:** Signs out the current user by invalidating tokens and closing WebSocket connections.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"API_SignOut"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Current token |

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message |

### Response Example

```json
{
  "p1": "Success",
  "p2": "signed out"
}
```

---

## API_ProfileResetPassword

**Purpose:** Resets a user's password and emails them the new password. Does NOT require login.

**Authentication:** None required (public endpoint with rate limiting)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"API_ProfileResetPassword"` |
| `p1` | string | Yes | Email address of account to reset |
| `p2` | string | No | Not used |
| `p3[0]` | string | Yes | Confirmation string - must be `"RESET"` |

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message |

### Response Scenarios

**Password Reset Initiated:**
```json
{
  "p1": "Success",
  "p2": "password reset - check email"
}
```

**Missing Confirmation:**
```json
{
  "p1": "Fail",
  "p2": "confirmation required - send RESET in P3[0]"
}
```

**Rate Limited or Invalid:**
```json
{
  "p1": "Fail",
  "p2": "password reset failed - check email address"
}
```

### Rate Limits

- 5 minute cooldown between requests
- Maximum 3 resets per day per account
- Federated accounts (Google/Outlook) cannot use this API

---

## API_ProfileDelete

**Purpose:** Soft-deletes a user profile, preserving the UUID as a tombstone.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"API_ProfileDelete"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Confirmation string - must be `"DELETE"` |

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message |

### What Gets Deleted

- User bio, name, headline, firm info
- Profile pictures and banners (URLs cleared)
- Cases, education, highlights, recommendations
- DGraph search edges (countries, areas, courts, work types)
- Shortlist entries
- Messaging keys and blocks
- Alias entries
- Firebase Authentication account (async)

### What Gets Preserved

- UUID (as tombstone for relationship integrity)
- Follower/following relationships (edges point to tombstone)
- Posts and comments (remain attributed to user)

---

## API_ProfilePurge

**Purpose:** Completely purges a user profile including all content. **Destructive operation.**

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"API_ProfilePurge"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Confirmation string - must be `"PURGE"` |

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message |

### What Gets Purged (in addition to Delete)

- All posts (marked as deleted)
- All blog posts and folders (marked as deleted)
- All follower/following relationships (DGraph edges removed)
- All post reactions (deleted)
- All post comments by this user (marked as deleted)
- Conversation participation hidden (soft-delete)
- Recent messages by this user (soft-delete, subject to max age policy)
- Message receipts

**⚠️ WARNING:** This operation cannot be undone.
