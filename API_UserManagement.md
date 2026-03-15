# User Management API Documentation

This document provides comprehensive documentation for user creation, verification, deletion, purge, and password reset APIs.


---

## Table of Contents


1. [RObj Structure](#robj-structure)
2. [API_FirebaseLogin](#api_firebaselogin)
3. [API_VerifyUser](#api_verifyuser)
4. [API_ProfileResetPassword](#api_profileresetpassword)
5. [API_ProfileDelete](#api_profiledelete)
6. [API_ProfilePurge](#api_profilepurge)
7. [User Lifecycle Flow](#user-lifecycle-flow)
8. [Database Schema](#database-schema)


---

## RObj Structure

All APIs use the standard RObj JSON structure for request/response communication:

```json
{
  "req": "API_Name",
  "p1": "string - typically username/email",
  "p2": "string - typically token or message",
  "p3": ["array", "of", "strings"],
  "p4": [["2d", "array"], ["of", "strings"]],
  "p5": [[["3d"], ["array"]], [["of"], ["strings"]]],
  "p6": [{"nested": "RObj objects"}]
}
```

### Field Conventions

| Field | Common Usage |
|----|----|
| `req` | API name (set by client) |
| `p1` | Username/email (input) or "Success"/"Fail" (output) |
| `p2` | Token (input) or message (output) |
| `p3` | Confirmation strings, additional data |
| `p4` | 2D data arrays |
| `p5` | 3D data arrays |
| `p6` | Nested RObj structures |


---

## API_FirebaseLogin

**Purpose:** Authenticates a user via Firebase, creates new accounts if needed, and triggers email verification.

**Endpoint:** `POST /api/API_FirebaseLogin`

**Authentication:** Firebase ID token

### Input RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_FirebaseLogin"` |
| `p1` | string | No | Not used for login |
| `p2` | string | Yes | Firebase ID token from client |
| `p4[0][0]` | string (JSON) | No | Optional `user_meta` JSON object (best-effort store) |

### Example Input

```json
{
  "req": "API_FirebaseLogin",
  "p1": "",
  "p2": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "p3": [],
  "p4": [["{\"device_type\":\"web\",\"device_id\":\"...\"}"]],
  "p5": [],
  "p6": []
}
```

### Output RObj

| Field | Type | Description |
|----|----|----|
| `p1` | string | `"Success"` or `"Fail"` |
| `p2` | string | Status message |
| `p3[0]` | string | Server token (only for verified users) |
| `p3[1]` | string | User flags JSON (only for verified users) |
| `p3[last]` | string (JSON) | `user_meta` JSON string (always present on success; last element) |

### Success Responses

**New User Created (unverified):**

```json
{
  "p1": "Success",
  "p2": "new user",
  "p3": []
}
```

> Verification email sent automatically. User must call API_VerifyUser with code.

**Existing User - Verification Code Already Sent:**

```json
{
  "p1": "Success",
  "p2": "enter code",
  "p3": []
}
```

**Existing User - Verification Code Sent Now:**

```json
{
  "p1": "Success",
  "p2": "code sent",
  "p3": []
}
```

**Existing Verified User - Logged In:**

```json
{
  "p1": "Success",
  "p2": "logged in",
  "p3": ["server_token_abc123...", "{\"user_type\":[\"lawyer\"],\"user_state\":[\"verified\"]}"]
}
```

### Error Responses

**Invalid Firebase Token:**

```json
{
  "p1": "Fail",
  "p2": "invalid firebase token"
}
```

**Failed to Create User:**

```json
{
  "p1": "Fail", 
  "p2": "unable to create user: [error message]"
}
```


---

## API_VerifyUser

**Purpose:** Verifies a user's email address using a 6-digit verification code sent via email.

**Endpoint:** `POST /api/API_VerifyUser`

**Authentication:** Firebase ID token (unverified users cannot use server token yet)

### Input RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_VerifyUser"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Firebase access token |
| `p3[0]` | string | Yes | 6-digit verification code from email |

### Example Input

```json
{
  "req": "API_VerifyUser",
  "p1": "user@example.com",
  "p2": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "p3": ["123456"],
  "p4": [],
  "p5": [],
  "p6": []
}
```

### Output RObj

| Field | Type | Description |
|----|----|----|
| `p1` | string | `"Success"` or `"Fail"` |
| `p2` | string | Status message |

### Success Response

```json
{
  "p1": "Success",
  "p2": "verified"
}
```

> User can now login and receive server token.

### Error Responses

**Not Logged In:**

```json
{
  "p1": "Fail",
  "p2": "no login"
}
```

**Already Verified:**

```json
{
  "p1": "Fail",
  "p2": "already verified"
}
```

**Wrong Verification Code:**

```json
{
  "p1": "Fail",
  "p2": "bad code"
}
```


---

## API_ProfileResetPassword

**Purpose:** Resets a user's password to a random 8-character string, logs them out of all devices, and emails the new password. This is for users who have **forgotten their password** and cannot log in.

**Endpoint:** `POST /api/API_ProfileResetPassword`

**Authentication:** ⚠️ **NONE REQUIRED** - This API does not require login

> This is one of the rare APIs that does not require authentication, since users who forgot their password cannot log in.

### Restrictions

* **Local accounts only:** Federated accounts (Google, Outlook, Microsoft) cannot reset password via this API - they must use their provider's password reset
* **Rate limiting:**
  * Minimum 5 minutes between resets
  * Maximum 3 resets per 24-hour period
* **Transactional:** Uses Sacred Timeline for multi-node replication

### Input RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileResetPassword"` |
| `p1` | string | Yes | Username (email) - the account to reset |
| `p2` | string | No | Not used (no login required) |
| `p3[0]` | string | Yes | Confirmation string: `"RESET"` |

### Example Input

```json
{
  "req": "API_ProfileResetPassword",
  "p1": "user@example.com",
  "p2": "",
  "p3": ["RESET"],
  "p4": [],
  "p5": [],
  "p6": []
}
```

### Output RObj

| Field | Type | Description |
|----|----|----|
| `p1` | string | `"Success"` or `"Fail"` |
| `p2` | string | Status message |

### Success Response

```json
{
  "p1": "Success",
  "p2": "password reset - check email"
}
```

### Error Responses

**Missing Email:**

```json
{
  "p1": "Fail",
  "p2": "email required in P1"
}
```

**Missing Confirmation:**

```json
{
  "p1": "Fail",
  "p2": "confirmation required - send RESET in P3[0]"
}
```

**User Not Found / Federated Account / Rate Limited:**

```json
{
  "p1": "Fail",
  "p2": "password reset failed - check email address"
}
```

> Note: For security, the same generic error is returned for all failure cases to avoid leaking information about whether an account exists.

### What Happens


1. Looks up user by email address
2. Validates user exists and has local auth (not Google/Outlook)
3. Checks rate limits (5 min cooldown, 3/day max)
4. Generates random 8-character password
5. Invalidates all current tokens (logs out all devices)
6. Updates Firebase password
7. Emails new password to user
8. Replicates via Sacred Timeline


---

## API_ProfileDelete

**Purpose:** Soft-deletes a user profile, clearing personal data while preserving the UUID tombstone for existing relationships.

**Endpoint:** `POST /api/API_ProfileDelete`

**Authentication:** Server token (verified users only)

### What Gets Cleared

| Category | Action |
|----|----|
| Personal data | Name, surname, headline, bio, firm info → cleared |
| Profile images | Pic, banner, small pic → deleted |
| Index tables | Cases, education, highlights, recommendations → cleared |
| DGraph edges | Search edges removed, user node becomes tombstone |
| User flags | "deleted" flag set |
| E2E keys | `message_keys_initialized` → reset to 0 |

### What Gets Preserved

* UUID (for tombstone/reuse)
* Existing relationships (followers can still reference the UUID)
* Email associated with account

### Input RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileDelete"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Server token |
| `p3[0]` | string | Yes | Confirmation string: `"DELETE"` |

### Example Input

```json
{
  "req": "API_ProfileDelete",
  "p1": "user@example.com",
  "p2": "server_token_abc123...",
  "p3": ["DELETE"],
  "p4": [],
  "p5": [],
  "p6": []
}
```

### Output RObj

| Field | Type | Description |
|----|----|----|
| `p1` | string | `"Success"` or `"Fail"` |
| `p2` | string | Status message |

### Success Response

```json
{
  "p1": "Success",
  "p2": "profile deleted"
}
```

### Error Responses

**Not Logged In:**

```json
{
  "p1": "Fail",
  "p2": "unauthorized"
}
```

**Missing Confirmation:**

```json
{
  "p1": "Fail",
  "p2": "confirmation required - send DELETE in P3[0]"
}
```

**Delete Failed:**

```json
{
  "p1": "Fail",
  "p2": "delete failed"
}
```

### UUID Reuse

If a deleted user re-registers with the same email:

* The original UUID is reused
* "deleted" flag is cleared
* Profile starts fresh but maintains relationship references


---

## API_ProfilePurge

**Purpose:** Completely purges a user profile, permanently removing all data including posts, messages, and relationships.

**Endpoint:** `POST /api/API_ProfilePurge`

**Authentication:** Server token (verified users only)

### ⚠️ WARNING: DESTRUCTIVE OPERATION

This operation cannot be undone. All user data is permanently removed.

### What Gets Purged

Everything from `API_ProfileDelete` PLUS:

| Category | Action |
|----|----|
| Posts | All posts marked as deleted |
| Blog | All blog posts and folders marked as deleted |
| Relationships | All follow/following edges removed |
| Lawyer data | Country/area/court/lawyertype/worktype relationships removed |
| Messages | All conversations and messages purged |
| Reactions | All post reactions and comments removed |

### Input RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfilePurge"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Server token |
| `p3[0]` | string | Yes | Confirmation string: `"PURGE"` |

### Example Input

```json
{
  "req": "API_ProfilePurge",
  "p1": "user@example.com",
  "p2": "server_token_abc123...",
  "p3": ["PURGE"],
  "p4": [],
  "p5": [],
  "p6": []
}
```

### Output RObj

| Field | Type | Description |
|----|----|----|
| `p1` | string | `"Success"` or `"Fail"` |
| `p2` | string | Status message |

### Success Response

```json
{
  "p1": "Success",
  "p2": "profile purged"
}
```

### Error Responses

**Not Logged In:**

```json
{
  "p1": "Fail",
  "p2": "unauthorized"
}
```

**Missing Confirmation:**

```json
{
  "p1": "Fail",
  "p2": "confirmation required - send PURGE in P3[0]"
}
```

**Purge Failed:**

```json
{
  "p1": "Fail",
  "p2": "purge failed"
}
```


---

## User Lifecycle Flow

```
                    ┌──────────────────────┐
                    │  Firebase Login      │
                    │  (new email)         │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  User Created        │
                    │  (unverified)        │
                    │  Email code sent     │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  API_VerifyUser      │
                    │  Enter 6-digit code  │
                    └──────────┬───────────┘
                               │
                               ▼
          ┌────────────────────┴────────────────────┐
          │                                         │
          ▼                                         ▼
┌──────────────────┐                    ┌──────────────────┐
│  Normal Usage    │                    │  Password Reset  │
│  (verified)      │◄──────────────────►│  (if needed)     │
└────────┬─────────┘                    └──────────────────┘
         │
         ├─────────────────────────────────────────┐
         │                                         │
         ▼                                         ▼
┌──────────────────┐                    ┌──────────────────┐
│  API_ProfileDelete│                   │  API_ProfilePurge│
│  (soft delete)   │                    │  (hard delete)   │
│  Tombstone UUID  │                    │  Complete removal│
└────────┬─────────┘                    └──────────────────┘
         │
         │ (same email re-registers)
         ▼
┌──────────────────┐
│  UUID Reused     │
│  Fresh profile   │
└──────────────────┘
```


---

## Database Schema

### Users Table (Key Columns)

```sql
CREATE TABLE `users` (
  `uuid` CHAR(36) NOT NULL,                    -- Primary key, preserved on delete
  `uname` varchar(100) NOT NULL,               -- Username (typically email)
  `email` varchar(100) NOT NULL,               -- Email address
  
  -- Authentication
  `access_token` varchar(2000) NOT NULL,       -- Firebase access token
  `session_token` varchar(2000) NOT NULL,      -- Session token (legacy)
  `server_token` varchar(80) NOT NULL,         -- Server-issued token
  `access_provider` varchar(100) DEFAULT 'local', -- 'local', 'google', 'microsoft'
  
  -- Verification
  `verified` int(11) NOT NULL DEFAULT '0',     -- DEPRECATED, use user_flags
  `vsent` int(11) NOT NULL DEFAULT '0',        -- 1 = verification email sent
  `vcode` int(11) NOT NULL DEFAULT '0',        -- 6-digit verification code
  
  -- Unified flags system (JSON)
  `user_flags` JSON,  -- {"user_state": ["verified", "deleted"], "user_type": ["lawyer"]}
  
  -- Password reset tracking
  `lastpwdreset` BIGINT DEFAULT NULL,          -- Unix timestamp of last reset
  `pwdresetcount` INT DEFAULT 0,               -- Resets in current 24h window
  
  -- E2E Messaging
  `message_keys_initialized` TINYINT(1) NOT NULL DEFAULT 0,
  
  PRIMARY KEY (`uuid`),
  INDEX `uname` (`uname`),
  INDEX `email_idx` (`email`)
);
```

### Messaging Tables (Key Columns)

```sql
-- User encryption keys (for E2E messaging)
CREATE TABLE `user_keys` (
  `uuid` CHAR(36) NOT NULL,                    -- User UUID (NOT user_uuid!)
  `identity_public_key` VARCHAR(200) NOT NULL,
  `identity_private_key_enc` VARCHAR(500) NOT NULL,
  `key_version` INT NOT NULL DEFAULT 1,
  PRIMARY KEY (`uuid`, `key_version`)
);

-- User blocks
CREATE TABLE `user_blocks` (
  `blocker_uuid` CHAR(36) NOT NULL,            -- User who is blocking
  `blocked_uuid` CHAR(36) NOT NULL,            -- User who is blocked
  UNIQUE KEY `uk_block` (`blocker_uuid`, `blocked_uuid`)
);
```

### Rate Limiting Logic

```
Password Reset Rate Limits:
├── 5-minute cooldown between resets
│   └── If (now - lastpwdreset) < 300 seconds → REJECT
│
├── Maximum 3 resets per 24 hours
│   └── If (now - lastpwdreset) < 86400 seconds AND pwdresetcount >= 3 → REJECT
│
└── Counter reset after 24 hours
    └── If (now - lastpwdreset) >= 86400 seconds → pwdresetcount = 0
```


---

## Sacred Timeline Replication

All destructive operations (delete, purge, password reset) use the Sacred Timeline transactional outbox pattern:


1. **Transaction Start**
2. **Execute SQL changes**
3. **Insert outbox event**
4. **Transaction Commit**
5. **Async replication to other nodes**

This ensures:

* Atomic operations (all or nothing)
* Idempotent replay on other nodes
* Eventual consistency across all servers

### Sacred Timeline API Codes

| Constant | Value | Description |
|----|----|----|
| `APIDeleteProfile` | `"DeleteProfile"` | Soft-delete profile |
| `APIPurgeProfile` | `"PurgeProfile"` | Hard-delete profile |
| `APIResetPassword` | `"ResetPassword"` | Password reset |


