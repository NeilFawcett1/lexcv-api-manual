# Blog APIs

APIs for managing blog posts and folders.


---

## Overview

The blog system is organized into folders, each containing blog posts. Folders have access permissions that control who can view their contents.

### Access Permissions

| Permission | Description |
|----|----|
| `public` | Anyone can view blog contents |
| `private` | Only users with explicit access can view |
| `paidonly` | Only paid/premium users can view |

### Access Control Methods

Private folders support two types of access control:


1. **Individual User Access** (`API_BlogAddUser` / `API_BlogDelUser`)
   * Creates `userCanAccessFolder` edge in DGraph
   * Grants specific users access to folder contents
2. **Group-Based Access** (`API_BlogAddGroup` / `API_BlogDelGroup`)
   * Sets `folderAllow*` predicates on the folder
   * Grants access to dynamic groups: `followers`, `following`, `shortlist1`, `shortlist2`, `shortlist3`
   * Access is evaluated in real-time based on current relationships


---

## API_BlogFolderNew

**Purpose:** Creates a new blog folder for the current user.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_BlogFolderNew"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Folder name |
| `p3[1]` | string | No | Access type: `"public"`, `"private"`, `"paidonly"` (default: `"public"`) |
| `p3[2]` | string | No | Allow comments: `"true"` or `"false"` (default: `"true"`) |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Folder UUID on success, error message on failure |

### Request Example

```json
{
  "req": "API_BlogFolderNew",
  "p1": "user@email.com",
  "p2": "token",
  "p3": ["Legal Analysis", "public", "true"]
}
```


---

## API_BlogFolderEdit

**Purpose:** Updates a blog folder's name and/or permissions.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_BlogFolderEdit"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Folder UUID |
| `p3[1]` | string | Yes | New folder name |
| `p3[2]` | string | No | Access type: `"public"`, `"private"`, `"paidonly"` |
| `p3[3]` | string | No | Allow comments: `"true"` or `"false"` |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Folder UUID on success, error message on failure |


---

## API_BlogFolderDelete

**Purpose:** Soft-deletes a blog folder and all its contents.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_BlogFolderDelete"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Folder UUID to delete |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Folder UUID on success, error message on failure |

### Notes

* All blog posts in the folder are also soft-deleted
* User can only delete their own folders


---

## API_BlogAddNew

**Purpose:** Creates a new blog entry in a folder.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_BlogAddNew"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Folder UUID |
| `p3[1]` | string | Yes | Blog title |
| `p3[2]` | string | No | Intro/description |
| `p3[3+]` | string | No | Keywords (variable length array) |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Blog UUID on success, error message on failure |
| `p4` | Upload URLs: `[[type, signed_upload_url], ...]` for PDF/image |

### Request Example

```json
{
  "req": "API_BlogAddNew",
  "p1": "user@email.com",
  "p2": "token",
  "p3": ["folder-uuid", "Contract Law Analysis 2024", "An overview of recent developments...", "contracts", "litigation", "commercial"]
}
```

### Response Example

```json
{
  "p1": "Success",
  "p2": "blog-uuid-here",
  "p4": [
    ["application/pdf", "https://storage.googleapis.com/upload?sig=..."]
  ]
}
```


---

## API_BlogDelete

**Purpose:** Soft-deletes a blog entry.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_BlogDelete"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Blog UUID to delete |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Blog UUID on success, error message on failure |

### Notes

* User can only delete their own blog posts
* Idempotent: returns success if already deleted
* Uses transactional outbox pattern for replication


---

## API_BlogListPosts

**Purpose:** Returns a paginated list of blog posts for a target user. Supports two modes:


1. **Hierarchical mode** (no folder UUID): Returns all folders with nested blog entries
2. **Single folder mode** (with folder UUID): Returns only blogs from that folder with proper pagination

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_BlogListPosts"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Target user UUID (whose blog to view) |
| `p3[1]` | string | No | Offset for pagination (default: "0") |
| `p3[2]` | string | No | Folder UUID (if provided, single folder mode) |

### Response RObj (Hierarchical Mode - no folder UUID)

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message on failure |
| `p3[0]` | Total count of blog posts |
| `p3[1]` | Has more pages: `"1"` or `"0"` |
| `p6` | Array of folder RObjs with nested blog entries |

### Response RObj (Single Folder Mode - with folder UUID)

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message on failure |
| `p3[0]` | Total count of blog posts in folder |
| `p3[1]` | Has more pages: `"1"` or `"0"` |
| `p3[2]` | Offset used |
| `p3[3]` | Limit used |
| `p3[4]` | Viewer has access: `"1"` or `"0"` |
| `p6[0]` | Folder RObj (without nested blogs) |
| `p6[1+]` | Blog entry RObjs (flat, not nested) |

### Folder RObj Structure

| Field | Description |
|----|----|
| `p3[0]` | Type: `"folder"` |
| `p3[1]` | Folder UUID |
| `p3[2]` | Folder name |
| `p3[3]` | Access permission: `"public"`, `"private"`, `"paidonly"` |
| `p3[4]` | Viewer has access: `"1"` or `"0"` |
| `p6` | Array of blog entry RObjs (only in hierarchical mode) |

### Blog Entry RObj Structure

| Field | Description |
|----|----|
| `p3[0]` | Type: `"blog"` |
| `p3[1]` | Blog UUID |
| `p3[2]` | Title |
| `p3[3]` | Intro/description |
| `p3[4]` | Creation date (Unix milliseconds) |
| `p3[5]` | Last edited date (Unix milliseconds) |
| `p3[6]` | Author name |
| `p3[7]` | Author UUID |
| `p3[8]` | Author flags JSON |
| `p4[0]` | `[promo_image_url, promo_token, pdf_url, pdf_token]` |
| `p4[1]` | `[keyword1, keyword2, ...]` (never null) |
| `p4[2]` | `[impressions, reads, likes, supports, laughs, dislikes, comments, reposts]` |

### Request Example (Hierarchical Mode)

```json
{
  "req": "API_BlogListPosts",
  "p1": "user@email.com",
  "p2": "token",
  "p3": ["target-user-uuid", "0"]
}
```

### Request Example (Single Folder Mode)

```json
{
  "req": "API_BlogListPosts",
  "p1": "user@email.com",
  "p2": "token",
  "p3": ["target-user-uuid", "0", "folder-uuid"]
}
```

### Response Example (Hierarchical Mode)

```json
{
  "p1": "Success",
  "p3": ["25", "1"],
  "p6": [
    {
      "p3": ["folder", "folder-uuid-1", "Legal Analysis", "public", "1"],
      "p6": [
        {
          "p3": ["blog", "blog-uuid-1", "Contract Law 2024", "An overview...", "1705312200000", "1705312200000", "John Smith", "author-uuid", "{\"user_type\":[\"lawyer\"]}"],
          "p4": [
            ["https://cdn.lexcv.co/promo.jpg", "token1", "https://cdn.lexcv.co/doc.pdf", "token2"],
            ["contracts", "litigation"],
            ["150", "45", "12", "3", "0", "0", "5", "2"]
          ]
        }
      ]
    },
    {
      "p3": ["folder", "folder-uuid-2", "Private Notes", "private", "0"],
      "p6": []
    }
  ]
}
```

### Response Example (Single Folder Mode)

```json
{
  "p1": "Success",
  "p3": ["15", "1", "0", "10", "1"],
  "p6": [
    {"p3": ["folder", "folder-uuid", "Legal Analysis", "public", "1"], "p6": []},
    {"p3": ["blog", "blog-uuid-1", "Contract Law", "Overview...", "1705312200000", "1705312200000", "John Smith", "author-uuid", "{}"], "p4": [["", "", "", ""], [], ["0", "0", "0", "0", "0", "0", "0", "0"]]},
    {"p3": ["blog", "blog-uuid-2", "Tort Law", "Analysis...", "1705312200000", "1705312200000", "John Smith", "author-uuid", "{}"], "p4": [["", "", "", ""], [], ["0", "0", "0", "0", "0", "0", "0", "0"]]}
  ]
}
```

### Access Control

* Private folders are listed but contents are hidden if viewer lacks access
* `p3[4]` in folder indicates whether viewer can see blog contents
* Single folder mode returns empty blogs array if viewer lacks access


---

## API_BlogAddUser

**Purpose:** Grants a user access to a blog folder. Only the folder owner can grant access.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_BlogAddUser"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Folder UUID |
| `p3[1]` | string | Yes | Target user UUID (user to grant access to) |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Folder UUID on success, error message on failure |

### Notes

* Creates `userCanAccessFolder` edge in the graph
* Only the folder owner can grant access
* Idempotent: granting access to a user who already has it returns success


---

## API_BlogDelUser

**Purpose:** Revokes a user's access to a blog folder. Only the folder owner can revoke access.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_BlogDelUser"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Folder UUID |
| `p3[1]` | string | Yes | Target user UUID (user to revoke access from) |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Folder UUID on success, error message on failure |

### Notes

* Removes `userCanAccessFolder` edge from the graph
* Only the folder owner can revoke access


---

## API_BlogView

**Purpose:** Returns full details of a single blog entry with permission check.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_BlogView"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Blog UUID to view |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message on failure |
| `p3[0]` | Blog UUID |
| `p3[1]` | Title |
| `p3[2]` | Intro/description |
| `p3[3]` | Creation date (Unix milliseconds) |
| `p3[4]` | Last edited date (Unix milliseconds) |
| `p3[5]` | Author name |
| `p3[6]` | Author UUID |
| `p3[7]` | Author flags JSON |
| `p3[8]` | Folder UUID |
| `p3[9]` | Folder name |
| `p3[10]` | Folder access permission |
| `p4[0]` | `[promo_image_url, promo_token, pdf_url, pdf_token]` |
| `p4[1]` | `[keyword1, keyword2, ...]` |
| `p4[2]` | `[impressions, reads, likes, supports, laughs, dislikes, comments, reposts]` |
| `p5[0]` | `[[uuid, displayName, headline, smallPicUrl, flagsJSON, position, followByViewer, followsViewer, shortlistFlag, shortlistRank, smallPicToken, messageKeysInitialized, allowMessages, blogCount]]` (full author summary package) |

### Permission Rules

Permission is granted if:


1. The viewer is the blog owner (always has access)
2. The blog's folder has `"public"` access, OR
3. The viewer has a `userCanAccessFolder` edge to the folder in the graph, OR
4. The folder has group access enabled and the viewer is a member of that group:
   * `folderAllowFollowers`: viewer follows the folder owner
   * `folderAllowFollowing`: folder owner follows the viewer
   * `folderAllowShortlist1/2/3`: viewer is in owner's shortlist with that rank

### Response Example

```json
{
  "p1": "Success",
  "p3": [
    "blog-uuid",
    "Understanding Arbitration Clauses",
    "A comprehensive guide to drafting effective arbitration clauses...",
    "1705312200000",
    "1705398600000",
    "John Smith QC",
    "author-uuid",
    "{\"user_state\":[\"verified\"],\"user_type\":[\"lawyer\"]}",
    "folder-uuid",
    "Legal Analysis",
    "public"
  ],
  "p4": [
    ["https://cdn.lexcv.co/promo/abc.jpg", "promo-token", "https://cdn.lexcv.co/pdf/abc.pdf", "pdf-token"],
    ["arbitration", "contracts", "commercial"],
    ["523", "187", "45", "12", "2", "0", "23", "8"]
  ]
}
```


---

## CDN Token Usage

Blog media (promo images and PDFs) are served from CDN with signed tokens.

### Fetching Protected Media

```http
GET https://cdn.lexcv.co/media/blogs/uuid/promo.jpg
x-lexcv-token: eyJhbGciOiJFZDI1NTE5...
```

### Token Lifetime

* CDN tokens are short-lived (typically 1 hour)
* Re-fetch via API to get fresh tokens if expired


---

## Statistics Tracking

Blog entries track the following engagement metrics (returned in `p4[2]`):

| Index | Metric | Description |
|----|----|----|
| 0 | Impressions | Times shown in lists |
| 1 | Reads | Times the full blog was viewed |
| 2 | Likes | Like reaction count |
| 3 | Supports | Support reaction count |
| 4 | Laughs | Laugh reaction count |
| 5 | Dislikes | Dislike reaction count |
| 6 | Comments | Comment count |
| 7 | Reposts | Times reposted |


---

## API_BlogAddGroup

**Purpose:** Grants a group access to a blog folder. Only the folder owner can grant access.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_BlogAddGroup"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Folder UUID |
| `p3[1]` | string | Yes | Group name |

### Valid Group Names

| Group | Description |
|----|----|
| `followers` | Users who follow the folder owner |
| `following` | Users the folder owner follows |
| `shortlist1` | Users in folder owner's shortlist with rank 1 |
| `shortlist2` | Users in folder owner's shortlist with rank 2 |
| `shortlist3` | Users in folder owner's shortlist with rank 3 |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Folder UUID on success, error message on failure |

### Request Example

```json
{
  "req": "API_BlogAddGroup",
  "p1": "user@email.com",
  "p2": "token",
  "p3": ["folder-uuid", "followers"]
}
```

### Notes

* Sets the `folderAllow*` predicate in DGraph (e.g., `folderAllowFollowers: "true"`)
* Only the folder owner can grant group access
* Idempotent: granting access to a group that already has it returns success
* Uses transactional outbox pattern for replication


---

## API_BlogDelGroup

**Purpose:** Revokes a group's access to a blog folder. Only the folder owner can revoke access.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_BlogDelGroup"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Folder UUID |
| `p3[1]` | string | Yes | Group name |

### Valid Group Names

| Group | Description |
|----|----|
| `followers` | Users who follow the folder owner |
| `following` | Users the folder owner follows |
| `shortlist1` | Users in folder owner's shortlist with rank 1 |
| `shortlist2` | Users in folder owner's shortlist with rank 2 |
| `shortlist3` | Users in folder owner's shortlist with rank 3 |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Folder UUID on success, error message on failure |

### Request Example

```json
{
  "req": "API_BlogDelGroup",
  "p1": "user@email.com",
  "p2": "token",
  "p3": ["folder-uuid", "followers"]
}
```

### Notes

* Deletes the `folderAllow*` predicate from DGraph
* Only the folder owner can revoke group access
* Idempotent: revoking access from a group that doesn't have it returns success
* Uses transactional outbox pattern for replication


---

## API_BlogListAccess

**Purpose:** Returns a paginated list of users with full details and groups that have access to a folder. Only the folder owner can call this API.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_BlogListAccess"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Folder UUID |
| `p3[1]` | string | No | Offset (default 0) |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message on failure |
| `p3[0]` | Total user count |
| `p3[1]` | Has more (`"1"` or `"0"`) |
| `p3[2]` | Offset used |
| `p3[3]` | Limit used |
| `p5[0][0]` | Array of group names with access |
| `p5[1+][0]` | User detail row (14 fields, see below) |

**User detail row format (14 fields):**

| Index | Field |
|----|----|
| 0 | uuid |
| 1 | displayName |
| 2 | headline |
| 3 | smallPicUrl (CDN signed URL) |
| 4 | flagsJSON |
| 5 | position |
| 6 | followByViewer (`"1"`/`"0"`) |
| 7 | followsViewer (`"1"`/`"0"`) |
| 8 | shortlistFlag (`"1"`/`"0"`) |
| 9 | shortlistRank |
| 10 | smallPicToken |
| 11 | messageKeysInitialized (`"1"`/`"0"`) |
| 12 | allowMessages (`"1"`/`"0"`) |
| 13 | blogCount |

### Request Example

```json
{
  "req": "API_BlogListAccess",
  "p1": "user@email.com",
  "p2": "token",
  "p3": ["folder-uuid", "0"]
}
```

### Response Example

```json
{
  "p1": "Success",
  "p3": ["2", "0", "0", "20"],
  "p5": [
    [["followers", "shortlist1"]],
    [["uuid-1", "John Smith", "Corporate Lawyer", "https://cdn.lexcv.co/...", "{}", "0", "1", "0", "0", "0", "abc123", "1", "1", "5"]],
    [["uuid-2", "Jane Doe", "Tax Specialist", "https://cdn.lexcv.co/...", "{}", "1", "0", "1", "1", "1", "def456", "1", "1", "3"]]
  ]
}
```

### Notes

* Returns users with direct `userCanAccessFolder` edge
* Returns groups with `folderAllow*` predicate set to `"true"`
* Only the folder owner can view the access list
* User details use standard 14-field summary format
* Pagination uses `profile.followers.list.per.page` config for limit


---

## API_BlogCheckAccess

**Purpose:** Checks if a specified user has access to a specific blog.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_BlogCheckAccess"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Blog UUID to check access for |
| `p3[1]` | string | No | Target user UUID (defaults to logged-in user) |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message on failure |
| `p3[0]` | `"1"` if target user has access, `"0"` otherwise |
| `p3[1]` | Blog owner UUID |
| `p3[2]` | Folder UUID |

### Request Example

```json
{
  "req": "API_BlogCheckAccess",
  "p1": "user@email.com",
  "p2": "token",
  "p3": ["blog-uuid"]
}
```

### Response Example

```json
{
  "p1": "Success",
  "p3": ["1", "owner-uuid", "folder-uuid"]
}
```

### Access Check Logic

Access is granted if any of the following conditions are met:


1. Target user is the blog owner
2. The blog's folder has `"public"` access
3. Target user has a direct `userCanAccessFolder` edge to the folder
4. The folder has group access enabled and the target user is a member:
   * `folderAllowFollowers`: target follows the folder owner
   * `folderAllowFollowing`: folder owner follows the target
   * `folderAllowShortlist1/2/3`: target is in owner's shortlist with that rank


---

## API_BlogGetFolders

**Purpose:** Returns a paginated list of folders for a target user, plus the first N blog posts per folder (preview) for folders the viewer can access.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_BlogGetFolders"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Target user UUID |
| `p3[1]` | string | No | Folder offset (default `0`) |
| `p3[2]` | string | No | Preview blog limit override. Use `"0"` for folders-only (no previews). |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message on failure |
| `p3[0]` | Total folder count |
| `p3[1]` | Has more folders (`"1"` or `"0"`) |
| `p3[2]` | Offset used |
| `p3[3]` | Limit used (folders per page) |
| `p3[4]` | Preview blog limit (blogs per folder) |
| `p6` | Array of folder RObj |

**Folder RObj Structure (in p6 array):**

| Field | Description |
|----|----|
| `p3[0]` | Type - `"folder"` |
| `p3[1]` | Folder UUID |
| `p3[2]` | Folder name |
| `p3[3]` | Access permission (`"public"`, `"private"`, `"paidonly"`) |
| `p3[4]` | `"1"` if viewer has access, `"0"` otherwise |
| `p3[5]` | Blog count in folder (only accurate if viewer has access) |
| `p3[6]` | Folder has more blogs beyond preview (`"1"` or `"0"`) (only meaningful if viewer has access) |
| `p6` | Blog preview entries (first N blogs) if viewer has access |

### Request Example

```json
{
  "req": "API_BlogGetFolders",
  "p1": "user@email.com",
  "p2": "token",
  "p3": ["target-user-uuid", "0", "0"]
}
```

### Response Example

```json
{
  "p1": "Success",
  "p3": ["2", "0", "0", "5", "5"],
  "p6": [
    {
      "p3": ["folder", "folder-uuid-1", "General", "public", "1", "12", "1"],
      "p6": [
        {"p3": ["blog", "blog-uuid-1", "...", "...", "1705312200000", "1705312200000", "John Smith", "author-uuid", "{}"], "p4": [["", "", "", ""], [], ["0","0","0","0","0","0","0","0"]]},
        {"p3": ["blog", "blog-uuid-2", "...", "...", "1705312200000", "1705312200000", "John Smith", "author-uuid", "{}"], "p4": [["", "", "", ""], [], ["0","0","0","0","0","0","0","0"]]}
      ]
    },
    {"p3": ["folder", "folder-uuid-2", "Private", "private", "0", "0", "0"], "p4": [], "p5": [], "p6": []}
  ]
}
```

### Notes

* Returns folders-only when `p3[2] = "0"`
* Returns folders with blog previews for accessible folders by default (use API_BlogListPosts for full per-folder paging)
* Blog count and per-folder `hasMoreBlogs` are only accurate for folders the viewer has access to


---

## API_BlogSetFolder

**Purpose:** Moves a blog entry from one folder to another. Only the blog owner can move their own blogs.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_BlogSetFolder"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Blog UUID |
| `p3[1]` | string | Yes | New folder UUID |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message on failure |
| `p3[0]` | Blog UUID |
| `p3[1]` | Old folder UUID |
| `p3[2]` | New folder UUID |

### Request Example

```json
{
  "req": "API_BlogSetFolder",
  "p1": "user@email.com",
  "p2": "token",
  "p3": ["blog-uuid", "new-folder-uuid"]
}
```

### Response Example

```json
{
  "p1": "Success",
  "p3": ["blog-uuid", "old-folder-uuid", "new-folder-uuid"]
}
```

### Notes

* Only the blog owner can move their own blogs
* Both old and new folders must belong to the blog owner
* Idempotent: moving to the same folder returns success
* Uses transactional outbox pattern for atomic MySQL + DGraph + replication
* Cannot move deleted blogs or move to deleted folders


---

## Group-Based Access Control

Blog folders support group-based access control in addition to individual user access.

### Group Types

| Group | DGraph Predicate | Access Condition |
|----|----|----|
| `followers` | `folderAllowFollowers` | Viewer follows the folder owner |
| `following` | `folderAllowFollowing` | Folder owner follows the viewer |
| `shortlist1` | `folderAllowShortlist1` | Viewer is in owner's shortlist rank 1 |
| `shortlist2` | `folderAllowShortlist2` | Viewer is in owner's shortlist rank 2 |
| `shortlist3` | `folderAllowShortlist3` | Viewer is in owner's shortlist rank 3 |

### How It Works


1. Folder owner uses `API_BlogAddGroup` to enable group access
2. When a viewer requests `API_BlogListPosts` or `API_BlogView`:
   * System checks if viewer has direct `userCanAccessFolder` edge
   * If not, system checks if any enabled group includes the viewer
   * Access is granted if any check passes

### Example Flow


1. Alice creates a private folder "Legal Updates"
2. Alice calls `API_BlogAddGroup` with group `"followers"`
3. Bob follows Alice
4. Bob requests `API_BlogListPosts` for Alice's profile
5. System checks: Bob has no direct edge, but folder has `folderAllowFollowers: "true"` and Bob follows Alice → Access granted


