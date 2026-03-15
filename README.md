# API Reference Documentation

Complete API reference for the LexCV Server.


---

## Quick Navigation

| Document | Description |
|----|----|
| [00_RObj_Structure.md](00_RObj_Structure.md) | Core RObj data structure and conventions |
| [01_Authentication_APIs.md](01_Authentication_APIs.md) | Login, logout, verification, password reset |
| [02_Profile_APIs.md](02_Profile_APIs.md) | Profile viewing and editing |
| [03_Posts_APIs.md](03_Posts_APIs.md) | Posts, comments, reactions |
| [04_Messaging_APIs.md](04_Messaging_APIs.md) | Mercury E2E encrypted messaging |
| [05_Search_APIs.md](05_Search_APIs.md) | User search and discovery |
| [06_Blog_APIs.md](06_Blog_APIs.md) | Blog posts and folders |
| [07_Social_APIs.md](07_Social_APIs.md) | Follow, shortlist, peer profiles |
| [08_System_APIs.md](08_System_APIs.md) | Config, WebSocket, utilities |
| [09_Admin_APIs.md](09_Admin_APIs.md) | Administrative functions |


---

## API Overview

### Protocol

All APIs use JSON over HTTP POST (except WebSocket which uses GET for upgrade).

### Base URL

```
https://api.lexcv.co/
```

### Request Format

```json
{
  "req": "API_Name",
  "p1": "username",
  "p2": "token",
  "p3": ["param1", "param2"],
  "p4": [["row1col1", "row1col2"], ["row2col1", "row2col2"]],
  "p5": [[["nested", "data"]]],
  "p6": [{"req": "nested", "p1": "robj"}]
}
```

### Response Format

```json
{
  "req": "API_Name",
  "p1": "Success",
  "p2": "status message",
  "p3": ["result", "data"],
  "p4": [],
  "p5": [],
  "p6": []
}
```


---

## Common Patterns

### Authentication

Most APIs require authentication via `p1` (email) and `p2` (server token):

```json
{
  "req": "API_ProfileBasic",
  "p1": "user@email.com",
  "p2": "server_token_from_login"
}
```

### Success Response

```json
{
  "p1": "Success",
  "p2": "optional message or ID"
}
```

### Error Response

```json
{
  "p1": "Fail",
  "p2": "error description"
}
```

### Pagination

```json
{
  "p3": ["offset", "limit"]
}
```

Response includes total count in `p3[0]` and items in `p6`.

### CDN Tokens

Protected media requires signed tokens:

```json
{
  "p3": ["https://cdn.lexcv.co/media/file.jpg"],
  "p4": [["signed_token", "x-lexcv-token"]]
}
```


---

## API Categories

### Public APIs (No Auth)

* `API_ProfileResetPassword` - Password reset

### Standard APIs (Auth Required)

* `API_ProfileBasic`, `API_ProfileBio`, etc.
* `API_PostAddNew`, `API_PostsListAll`, etc.
* `API_SearchMain`, `API_SearchAuto`
* `API_MessageSend`, `API_MessageConversation`, etc.

### Admin APIs (Admin Role)

* `API_AdminRateLimits` - Rate limit management

### Root APIs (Root Role)

* `APPADMIN_USERFLAGS` - User flag management

### Deployment APIs (No Auth, Network Protected)

* `AdminAPI_SetCFG` - Configuration setup
* `AdminAPI_ResetGraph` - Graph reset
* `AdminAPI_SampleData` - Test data


---

## Version History

| Date | Version | Changes |
|----|----|----|
| 2024-01 | 1.0 | Initial API documentation |
| 2024-01 | 1.1 | Added Mercury messaging APIs |
| 2024-01 | 1.2 | Added Blog APIs |
| 2024-01 | 1.3 | Sacred Timeline audit completed |


---

## Related Documentation

* [../MERCURY_FLUTTER_IMPLEMENTATION.md](../MERCURY_FLUTTER_IMPLEMENTATION.md) - E2E messaging implementation guide
* [../Media Tokens.md](../Media%20Tokens.md) - CDN token signing details
* [../SacredTimeline_Audit.md](../SacredTimeline_Audit.md) - Replication audit results


