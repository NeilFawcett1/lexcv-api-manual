# Search APIs

APIs for searching and discovering users.


---

## API_SearchMain

**Purpose:** Advanced search for lawyers with multiple filter criteria.

**Authentication:** Requires valid login (p1=user uuid, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_SearchMain"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Offset for pagination (default: "0") |
| `p5` | array | Yes | Search criteria (see structure below) |

### Search Criteria Structure (p5)

The `p5` field is a 3D array with 5 category groups. **Send IDs only, not names**:

| Index | Category | Format |
|----|----|----|
| `p5[0]` | Countries | `[[id], ...]` |
| `p5[1]` | Geographic Areas | `[[id], ...]` |
| `p5[2]` | Lawyer Types | `[[id], ...]` |
| `p5[3]` | Work Types | `[[id], ...]` |
| `p5[4]` | Courts | `[[id], ...]` |

### Request Example

```json
{
  "req": "API_SearchMain",
  "p1": "user@email.com",
  "p2": "token",
  "p3": ["0"],
  "p5": [
    [["1"], ["2"]],
    [["10"], ["20"]],
    [["100"]],
    [["200"]],
    []
  ]
}
```

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |
| `p3[0]` | Total matching results count |
| `p4` | Array of user result arrays (2D array) |

### User Result Array Structure (each item in p4)

| Index | Description |
|----|----| 
| `[0]` | User UUID |
| `[1]` | Full name |
| `[2]` | Headline |
| `[3]` | Small profile picture URL (empty if none) |
| `[4]` | Flags JSON (user_state, user_type, user_level) |
| `[5]` | Position (offset + index) |
| `[6]` | Viewer follows this user: `"1"` or `"0"` |
| `[7]` | This user follows viewer: `"1"` or `"0"` |
| `[8]` | Viewer has shortlisted: `"1"` or `"0"` |
| `[9]` | Shortlist rank (if shortlisted) |
| `[10]` | Small pic CDN token |
| `[11]` | Message keys initialized: `"1"` or `"0"` |
| `[12]` | Allow messages: `"1"` or `"0"` |
| `[13]` | Blog count: Count of user's public blog articles |

### Response Example

```json
{
  "p1": "Success",
  "p3": ["156"],
  "p4": [
    ["uuid1", "John Smith", "Commercial Barrister", "https://cdn...", "{\"user_type\":[\"lawyer\"]}", "0", "0", "0", "0", "0", "token...", "1", "1", "3"]
  ]
}
```

### Search Logic

* All criteria within a category are OR'd (match any)
* Categories are AND'd (must match at least one in each non-empty category)
* Empty categories are ignored
* Results sorted by uuid, paginated
* Blocked users are excluded from results


---

## API_SearchAuto

**Purpose:** Autocomplete/typeahead search for quick user lookup.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_SearchAuto"` |
| `p1` | string | Yes | User UUID |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Search query text |
| `p3[1]` | string | No | Optional user-type filter (comma-separated) |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |
| `p3[0]` | Result count |
| `p4` | Array of matching user arrays (2D array) |

### User Result Array Structure (each item in p4)

Same format as SearchMain results - see SearchMain documentation for field details.

### Request Example

```json
{
  "req": "API_SearchAuto",
  "p1": "user_uuid",
  "p2": "token",
  "p3": ["john sm", "lawyer,company"]
}
```

### Response Example

```json
{
  "p1": "Success",
  "p3": ["2"],
  "p4": [
    ["uuid1", "John Smith", "Senior Partner", "https://cdn...", "{...}", "0", "0", "0", "0", "0", "token...", "1", "1", "3"],
    ["uuid2", "John Smithson", "Associate", "", "{...}", "1", "0", "0", "0", "0", "", "0", "1", "0"]
  ]
}
```

### Search Behavior

* Searches across user names
* Case-insensitive
* Results limited by config (default: 10)
* Optional filter values: `lawyer`, `otherpro` (alias `proi`), `company`, `student`, `public`, `newuser`
* Filter values are OR'd together (e.g. `lawyer,company`)


---

## API_CountriesGroups

**Purpose:** Retrieves the list of available countries/jurisdictions grouped by region.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_CountriesGroups"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p5` | Grouped countries: `[[[groupName], [country1, code1], [country2, code2], ...], ...]` |

### Response Example

```json
{
  "p1": "Success",
  "p5": [
    [["Europe"], ["United Kingdom", "GB"], ["France", "FR"], ["Germany", "DE"]],
    [["Asia Pacific"], ["Australia", "AU"], ["Singapore", "SG"], ["Hong Kong", "HK"]],
    [["Americas"], ["United States", "US"], ["Canada", "CA"]]
  ]
}
```


---

## API_CountriesGroupsByLawyer

**Purpose:** Retrieves countries/jurisdictions filtered to those with at least one registered lawyer.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_CountriesGroupsByLawyer"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

Same structure as `API_CountriesGroups`, but only includes countries with active lawyers.


---

## Search Filter Options

All filter options are stored in MySQL index tables with the structure: `id`, `countryid`, `name`. They are referenced by **database ID** in search criteria.

### Countries (countriesindex)

- Have unique short codes (e.g., "GB", "US")
- Are the only entities with codes
- Format returned: `[[id, name, code, rank], ...]`

### Geographic Areas (geoareasindex)

- Linked to countries via countryid
- No short codes
- Examples: London, Paris, New York

### Lawyer Types (lawyertypeindex)

- Linked to countries via countryid
- No short codes
- Examples: Barrister, Solicitor, Advocate, Attorney, Mediator

### Work Types (worktypeindex)

- No country association
- No short codes
- Examples: Contentious, Advisory, Transactional

### Courts (courtsindex)

- Optional `countryid` and `international` flag
- No short codes
- Examples: Supreme Court, Court of Appeal, International Court of Justice (international=1)

**Important:** When building search criteria for p5, use numeric database IDs, not names or codes. The frontend should fetch available options via the Countries/Areas APIs and use the IDs when constructing search requests.
