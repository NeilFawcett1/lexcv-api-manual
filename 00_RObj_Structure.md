# RObj Structure Reference

The RObj (Request Object) is the standard JSON structure used for all client-server communication in the LexCV API.

---

## Structure Definition

```go
type RObj struct {
    Req string       `json:"req"`  // API name being called
    P1  string       `json:"p1"`   // Primary string field
    P2  string       `json:"p2"`   // Secondary string field
    P3  []string     `json:"p3"`   // 1D string array
    P4  [][]string   `json:"p4"`   // 2D string array
    P5  [][][]string `json:"p5"`   // 3D string array
    P6  []RObj       `json:"p6"`   // Nested RObj array
}
```

---

## Field Conventions

### Request Fields (Client → Server)

| Field | Typical Usage |
|-------|---------------|
| `req` | API endpoint name (e.g., `"API_FirebaseLogin"`) |
| `p1` | Username/email for authentication |
| `p2` | Token (access_token or server_token) for authentication |
| `p3` | Primary parameters (single values or lists) |
| `p4` | Table/matrix data (2D arrays) |
| `p5` | Complex nested data (3D arrays, grouped lists) |
| `p6` | Nested objects (recursive structures) |

### Response Fields (Server → Client)

| Field | Typical Usage |
|-------|---------------|
| `p1` | `"Success"` or `"Fail"` - operation result |
| `p2` | Message (error description or status info) |
| `p3` | Primary response data (UUIDs, counts, JSON strings) |
| `p4` | List/table data (search results, profile items) |
| `p5` | Grouped data (lawyer profile categories) |
| `p6` | Nested objects (posts with comments, folders with blogs) |

---

## Authentication Pattern

Most authenticated APIs expect:

```json
{
  "req": "API_Name",
  "p1": "user@email.com",
  "p2": "server_token_here",
  "p3": ["param1", "param2"],
  "p4": [],
  "p5": [],
  "p6": []
}
```

---

## Common Response Patterns

### Success with Simple Data

```json
{
  "p1": "Success",
  "p2": "optional message",
  "p3": ["data1", "data2"],
  "p4": [],
  "p5": [],
  "p6": []
}
```

### Success with Nested Objects

```json
{
  "p1": "Success",
  "p2": "",
  "p3": ["100"],  // total count
  "p4": [],
  "p5": [],
  "p6": [
    {
      "p3": ["uuid1", "data1"],
      "p4": [["nested", "data"]]
    }
  ]
}
```

### Failure Response

```json
{
  "p1": "Fail",
  "p2": "Error description here",
  "p3": [],
  "p4": [],
  "p5": [],
  "p6": []
}
```

---

## Type Encoding Conventions

Since all data is encoded as strings, the following conventions apply:

| Type | Encoding |
|------|----------|
| Integer | `"123"` |
| Boolean | `"1"` (true) or `"0"` (false) |
| Timestamp | Unix milliseconds as string: `"1702819200000"` |
| JSON | Stringified JSON: `"{\"key\":\"value\"}"` |
| UUID | Standard format: `"550e8400-e29b-41d4-a716-446655440000"` |
| Null/Empty | Empty string: `""` |

---

## Media/CDN Token Pattern

When media URLs are returned, they typically include access tokens:

```json
{
  "p3": ["cdn_signed_url_here"],
  "p4": [["token_value", "x-lexcv-token"]]
}
```

The client must send the token as an HTTP header when fetching the CDN URL:
```
GET <cdn_url>
x-lexcv-token: <token_value>
```
