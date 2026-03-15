# User Flags System - Client Reference Guide

This document describes the unified user flags system and how client applications should handle user state/role/type information returned by the LexCV Server APIs.

## Overview

The server uses a unified JSON-based flags system to manage user state, types, and levels. This replaces the legacy `status1`, `status2`, `status3`, `ulevel`, and `enabled` columns.

## Flags Structure

All user flags are returned as a JSON object with the following structure:

```json
{
  "user_state": ["verified"],
  "user_type": ["lawyer"],
  "user_level": ["level1"]
}
```

### Flag Categories

#### user_state (array)

Controls account status. Multiple states can be active simultaneously.

| Value | Description |
|----|----|
| `deleted` | Account is permanently deleted |
| `deactivated` | Account is temporarily deactivated by user |
| `hidden` | Profile is hidden from public view |
| `banned` | Account is banned by admin |
| `verified` | Email/identity has been verified |

**Note:** Users with `deleted`, `deactivated`, `banned`, or `hidden` states are excluded from search results and public lists.

#### user_type (array)

Account type - typically exclusive (only one value).

| Value | Description |
|----|----|
| `lawyer` | Registered lawyer profile |
| `otherpro` | Other professional (non-lawyer) profile |
| `company` | Law firm or company profile |
| `student` | Law student profile |
| `public` | General public user |

#### user_level (array)

Subscription/access level - typically exclusive (only one value).

| Value | Description |
|----|----|
| `level1` | Basic/free tier (default) |
| `level2` | Premium tier 1 |
| `level3` | Premium tier 2 |
| `level4` | Enterprise tier |


---

## API Return Formats

### RObj Structure

All API responses use the standard RObj structure:

```json
{
  "req": "API_Name",
  "p1": "Success|Fail",
  "p2": "message or data",
  "p3": ["array", "of", "strings"],
  "p4": [["2D", "array"], ["of", "strings"]],
  "p5": [[["3D", "array"]]],
  "p6": [{"nested": "RObj"}]
}
```


---

## Core Authentication APIs

### API_CheckLogin

**Request:**

```json
{
  "p1": "username@email.com",
  "p2": "access_token"
}
```

**Success Response:**

```json
{
  "p1": "Success",
  "p2": "username@email.com",
  "p3": [
    "{\"user_state\":[\"verified\"],\"user_type\":[\"lawyer\"],\"user_level\":[\"level1\"]}",
    "{\"grants\":[{\"resource\":{\"type\":\"system\",\"id\":\"main\"},\"relations\":[\"admin\"]}]}"
  ]
}
```

**P3 Fields:**

* `p3[0]` - User flags as JSON string (human-readable)
* `p3[1]` - Cato grants JSON (extensible), e.g. `{ "grants": [{ "resource": {"type":"system","id":"main"}, "relations": ["admin"] }] }`

**Verified state:** determined from `p3[0].user_state` containing `"verified"`.

**Fail Response:**

```json
{
  "p1": "Fail",
  "p2": "username@email.com"
}
```

### API_FirebaseLogin

**Request:**

```json
{
  "p2": "firebase_id_token"
}
```

**Success Responses:**


1. **New user created:**

```json
{
  "p1": "Success",
  "p2": "new user"
}
```


2. **Existing verified user (includes flags):**

```json
{
  "p1": "Success",
  "p2": "logged in",
  "p3": [
    "server_token_here",
    "{\"user_state\":[\"verified\"],\"user_type\":[\"lawyer\"],\"user_level\":[\"level1\"]}"
  ]
}
```

* `p3[0]` - Server authentication token
* `p3[1]` - User flags as JSON string


3. **Existing unverified user:**

```json
{
  "p1": "Success",
  "p2": "enter code"
}
```

or

```json
{
  "p1": "Success",
  "p2": "code sent"
}
```

### API_VerifyUser

**Request:**

```json
{
  "p1": "username@email.com",
  "p2": "access_token",
  "p3": ["verification_code"]
}
```

**Success Response:**

```json
{
  "p1": "Success",
  "p2": "verified"
}
```


---

## List/Search APIs

All list APIs return user items with flags in a consistent position. Items are returned in `p4` as 2D arrays.

### Common Item Format

Each user item in search/list results:

| Position | Field | Description |
|----|----|----|
| 0 | uuid | User's unique identifier |
| 1 | fullname | Display name (first + last) |
| 2 | headline | User's professional headline |
| 3 | smallPicUrl | Signed URL for avatar image |
| 4 | **flagsJSON** | User flags as JSON string |
| 5 | position | Index position in results |
| 6 | followByViewer | "1" if logged-in user follows this user |
| 7 | followsViewer | "1" if this user follows logged-in user |
| 8 | shortlistFlag | "1" if on viewer's shortlist |
| 9 | shortlistRank | Shortlist rank (0-5) |
| 10 | smallPicToken | Auth token for avatar image |

**Position 4 (flagsJSON) Example:**

```json
"{\"user_state\":[\"verified\"],\"user_type\":[\"lawyer\"],\"user_level\":[\"level1\"]}"
```

### API_SearchMain

Main search endpoint.

**Request:**

```json
{
  "p1": "username@email.com",
  "p2": "access_token",
  "p3": ["search_term", "offset", "limit"],
  "p4": [["filter_type", "filter_id", "filter_value"]]
}
```

### API_SearchAuto

Autocomplete search.

**Request:**

```json
{
  "p1": "user_uuid",
  "p2": "access_token",
  "p3": ["partial_name", "lawyer,company"]
}
```

### API_ProfileShortlist

Get user's shortlist.

**Request:**

```json
{
  "p1": "username@email.com",
  "p2": "access_token",
  "p3": ["rank_filter", "offset", "limit"]
}
```

### API_Followers / API_Following

Get followers or following list.

**Request:**

```json
{
  "p1": "username@email.com",
  "p2": "access_token",
  "p3": ["target_uuid", "offset", "limit"]
}
```


---

## Posts & Comments APIs

### Post Object Structure

Posts in all list APIs (`API_PostsListAll`, `API_PostsListFollowing`, `API_GetPost`) return author flags in P4\[5\].

**Post RObj Structure:**

```json
{
  "p3": [
    "0",                    // [0] Type: "0"=post, "1"=comment
    "post-uuid",            // [1] Post UUID
    "author-uuid",          // [2] Author UUID
    "Post content text",    // [3] Post text
    "1702000000000",        // [4] Creation date (unix ms)
    "1702000000000"         // [5] Last edited (unix ms)
  ],
  "p4": [
    ["10", "5", "2", "1"],       // [0] Reaction counts [like, support, laugh, dislike]
    ["3", "0", "100"],           // [1] Engagement [comment_count, repost_count, impressions] (impressions = unique estimate)
    ["", ""],                    // [2] References [repost_of_uuid, best_answer_uuid]
    ["AuthorName", "Headline", "avatar-url", "avatar-token"],  // [3] Author info
    ["0"],                       // [4] Viewer's reaction (-1=none, 0=like, 1=support, 2=laugh, 3=dislike)
    ["{\"user_state\":[\"verified\"],\"user_type\":[\"lawyer\"],\"user_level\":[\"level1\"]}"]  // [5] Author flags JSON
  ],
  "p5": [[["1", "image-cdn-url"]]],  // Media: [[type, url]] where type: 1=image, 2=pdf, 3=video
  "p6": []                            // Reserved for nested comments
}
```

**P4\[5\] Author Flags:**

| Index | Field | Description |
|----|----|
| 0 | flagsJSON | Author's user flags as JSON string |

### Comment Object Structure

Comments in `API_PostGetComments` return author flags in P4\[6\].

**Comment RObj Structure:**

```json
{
  "p3": [
    "1",                    // [0] Type: always "1" for comment
    "comment-uuid",         // [1] Comment UUID
    "author-uuid",          // [2] Author UUID
    "Comment text",         // [3] Comment text
    "1702000000000",        // [4] Creation date (unix ms)
    "1702000000000"         // [5] Last edited (unix ms)
  ],
  "p4": [
    ["5", "0", "0", "0"],        // [0] Reaction counts [like_count, 0, 0, 0]
    ["1", "0", "50"],            // [1] Engagement [has_replies, 0, impressions] (impressions = unique estimate)
    ["post-uuid", "parent-uuid"], // [2] References [root_post_uuid, parent_uuid]
    ["AuthorName", "Headline", "avatar-url", "avatar-token"],  // [3] Author info
    ["-1"],                      // [4] Viewer's reaction
    ["0", "0"],                  // [5] Post author's engagement [author_reaction, is_favourite]
    ["{\"user_state\":[\"verified\"],\"user_type\":[\"lawyer\"],\"user_level\":[\"level1\"]}"]  // [6] Author flags JSON
  ],
  "p5": [],                      // Reserved for media
  "p6": [...]                    // Nested replies (recursive)
}
```

**P4\[6\] Author Flags:**

| Index | Field | Description |
|----|----|
| 0 | flagsJSON | Comment author's user flags as JSON string |

### Posts List APIs

#### API_PostsListAll

Returns combined timeline (followed users + nation posts).

**Request:**

```json
{
  "p1": "username@email.com",
  "p2": "access_token",
  "p3": ["0", "20"]  // [offset, limit]
}
```

**Response:**

```json
{
  "p1": "Success",
  "p3": ["150"],     // Total count
  "p6": [
    { /* Post RObj with P4[5] author flags */ },
    { /* Post RObj with P4[5] author flags */ }
  ]
}
```

#### API_PostsListFollowing

Returns posts only from followed users (no nation timeline).

Same request/response format as `API_PostsListAll`.

#### API_GetPost

Returns a single post by UUID.

**Request:**

```json
{
  "p1": "username@email.com",
  "p2": "access_token",
  "p3": ["post-uuid"]
}
```

**Response:**

```json
{
  "p1": "Success",
  "p6": [
    { /* Post RObj with P4[5] author flags */ }
  ]
}
```

#### API_PostGetComments

Returns paginated comments for a post or comment.

**Request:**

```json
{
  "p1": "username@email.com",
  "p2": "access_token",
  "p3": ["parent-uuid", "0", "0", "20"]  // [parent_uuid, parent_type, offset, limit]
}
```

**Response:**

```json
{
  "p1": "Success",
  "p3": ["25", "1"],  // [total_count, has_more]
  "p6": [
    { /* Comment RObj with P4[6] author flags */ },
    { /* Comment RObj with nested replies in P6 */ }
  ]
}
```


---

## Profile APIs

### API_ProfilePeerProfile

Returns full profile for viewing another user's profile.

**Response Structure:**

```json
{
  "p1": "Success",
  "p2": "",
  "p6": [{
    "sL": [
      "uuid",                  // 0 - User UUID
      "James",                 // 1 - First name
      "Smith",                 // 2 - Last name
      "Senior Partner",        // 3 - Headline
      "email@example.com",     // 4 - Email
      "+44 20 7000 1000",      // 5 - Telephone
      "https://website.com",   // 6 - Website
      "{...flags JSON...}",    // 7 - User flags (JSON string)
      "bio text",              // 8 - Biography
      "linkedin.com/in/user",  // 9 - Social link
      "https://pic-url",       // 10 - Profile pic URL
      "pic-token",             // 11 - Profile pic token
      "https://banner-url",    // 12 - Banner URL
      "banner-token",          // 13 - Banner token
      "0.5",                   // 14 - Banner X offset
      "0.3"                    // 15 - Banner Y offset
    ],
    "oL": [
      [[...countries...]],          // 0
      [[...areas OR orgType...]],   // 1 (lawyer: areas, company: organisation type)
      [[...types OR locations...]], // 2 (lawyer: lawyer types, company: locations)
      [[...courts OR workTypes...]],// 3 (lawyer: courts, company: work types)
      [[...work OR reserved...]],   // 4 (lawyer: work types, company: reserved empty)
      [[...cases...]],              // 5
      [[...education...]],          // 6
      [[...highlights...]],         // 7
      [[...recommendations...]],    // 8
      [[...experience...]],         // 9
      [[...publications...]]        // 10
    ]
  }]
}
```

**Note:** The target user's type is declared in `sL[7]` (flags JSON). When `user_type` is `company`, `oL[6]` (education) and `oL[9]` (experience) are empty arrays.

**sL\[7\] Flags Example:**

```json
"{\"user_state\":[\"verified\"],\"user_type\":[\"lawyer\"],\"user_level\":[\"level1\"]}"
```

### API_ProfileBasic

Returns basic profile information.

**Response P3 Fields:**

| Index | Field |
|----|----|
| 0 | username |
| 1 | name |
| 2 | surname |
| 3 | headline |
| 4 | email |
| 5 | telephone |
| 6 | website |
| 7 | profileViews |
| 8 | profileRank |

**Note:** Profile APIs do not currently return user flags. Use `API_CheckLogin` for the logged-in user's flags, or check list APIs for other users' flags.


---

## Client Implementation Guide

### Parsing Flags

```dart
// Dart example
class UserFlags {
  final List<String> userState;
  final List<String> userType;
  final List<String> userLevel;
  
  UserFlags.fromJson(Map<String, dynamic> json) 
    : userState = List<String>.from(json['user_state'] ?? []),
      userType = List<String>.from(json['user_type'] ?? []),
      userLevel = List<String>.from(json['user_level'] ?? []);
  
  bool get isVerified => userState.contains('verified');
  bool get isLawyer => userType.contains('lawyer');
  bool get isActive => !userState.any((s) => 
      ['deleted', 'deactivated', 'banned', 'hidden'].contains(s));
}

// Usage from API_CheckLogin response
final flagsJson = jsonDecode(response['p3'][0]);
final flags = UserFlags.fromJson(flagsJson);
```

```javascript
// JavaScript example
class UserFlags {
  constructor(json) {
    this.userState = json.user_state || [];
    this.userType = json.user_type || [];
    this.userLevel = json.user_level || [];
  }
  
  get isVerified() { return this.userState.includes('verified'); }
  get isLawyer() { return this.userType.includes('lawyer'); }
}

// Usage from list item position 4
const flagsJson = JSON.parse(item[4]);
const flags = new UserFlags(flagsJson);
```

### Best Practices


1. **Always check for empty arrays** - Flags categories may be empty
2. **Use contains/includes** - Flags are arrays, not single values
3. **Cache user flags** - Store after login, refresh on session changes
4. **Handle missing flags gracefully** - Older API responses may not include flags


---

## Migration from Legacy System

### Legacy to New Mapping

| Legacy Field | New Equivalent |
|----|----|
| `status1 = 1` | `user_type.contains("lawyer")` |
| `verified = 1` | `user_state.contains("verified")` |
| `enabled = 1` | `!user_state.contains("deleted")` |

### Breaking Changes


1. **Position 4 in list items** - Now contains full JSON flags, not "0"/"1" status
2. **API_CheckLogin P3** - Now array with \[flagsJSON, verified\], not status array
3. **Admin checks** - Client apps should not infer elevated permissions from flags


---

## Admin APIs

### APPADMIN_USERFLAGS

**Requires:** Root privileges (Cato system role: `root`)

Allows root users to view and modify flags for any user. Changes are replicated via Sacred Timeline.

**Request:**

```json
{
  "p1": "root_user@email.com",
  "p2": "access_token",
  "p3": ["action", "target_uuid", "parameter"]
}
```

**Actions:**

| Action | P3\[2\] Parameter | Description |
|----|----|----|
| `get` | (none) | Returns current flags for target user |
| `set` | flags JSON | Replaces all flags with provided JSON |
| `add_state` | state name | Adds a user_state value |
| `remove_state` | state name | Removes a user_state value |
| `add_role` | role name | Grants a Cato system role |
| `remove_role` | role name | Revokes a Cato system role |
| `set_type` | type name | Sets user_type (replaces existing) |
| `set_level` | level name | Sets user_level (replaces existing) |

**Example - Get user flags:**

```json
{
  "p1": "admin@lexcv.co",
  "p2": "token123",
  "p3": ["get", "user-uuid-here"]
}
```

**Example - Make user a lawyer:**

```json
{
  "p1": "admin@lexcv.co",
  "p2": "token123",
  "p3": ["set_type", "user-uuid-here", "lawyer"]
}
```

**Example - Ban user:**

```json
{
  "p1": "admin@lexcv.co",
  "p2": "token123",
  "p3": ["add_state", "user-uuid-here", "banned"]
}
```

**Example - Grant admin role:**

```json
{
  "p1": "admin@lexcv.co",
  "p2": "token123",
  "p3": ["add_role", "user-uuid-here", "admin"]
}
```

**Success Response:**

```json
{
  "p1": "Success",
  "p2": "Flags updated for user uuid-here",
  "p3": ["{\"user_state\":[\"verified\",\"banned\"],\"user_type\":[\"lawyer\"],\"user_level\":[\"level1\"]}"]
}
```

**Fail Response:**

```json
{
  "p1": "Fail",
  "p2": "Insufficient permissions - root access required"
}
```


---

## Troubleshooting

### Common Issues

**Q: Flags JSON is empty** `{}`
A: User has default flags (public, level1). Empty categories mean no flags of that type.

**Q: Old API format with "0"/"1" in position 4**
A: Client may be hitting an older server version. Update server or use legacy fallback.

**Q: User not appearing in search results**
A: Check if user has `deleted`, `deactivated`, `banned`, or `hidden` in `user_state`.


---

## Version History

* **v2.0** (December 2025) - Unified flags system, human-readable JSON format
* **v1.0** (Legacy) - Numeric status1/status2/status3 columns


