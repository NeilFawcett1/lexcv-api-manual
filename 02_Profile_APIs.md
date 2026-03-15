# Profile APIs

APIs for viewing and editing user profile sections.


---

## API_ProfileBasic

**Purpose:** Retrieves basic profile information for the current user.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileBasic"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |
| `p3[0]` | Username (uname) |
| `p3[1]` | First name |
| `p3[2]` | Surname |
| `p3[3]` | Headline |
| `p3[4]` | Email (contact email) |
| `p3[5]` | Telephone |
| `p3[6]` | Website URL |
| `p3[7]` | Following count (string) |
| `p3[8]` | Followed-by count (string) |
| `p3[9]` | Firm/company name (max 80 chars, empty if not set) |

### Response Example

```json
{
  "p1": "Success",
  "p3": ["johnsmith", "John", "Smith", "Senior Partner", "john@firm.com", "+44 123 456", "www.firm.com", "42", "156", "Smith & Partners LLP"]
}
```


---

## API_ProfileBasicEdit

**Purpose:** Updates basic profile information.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileBasicEdit"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | First name |
| `p3[1]` | string | Yes | Surname |
| `p3[2]` | string | Yes | Headline |
| `p3[3]` | string | Yes | Contact email |
| `p3[4]` | string | Yes | Telephone |
| `p3[5]` | string | Yes | Website URL |
| `p3[6]` | string | No | Firm/company name (optional, max 80 chars) |
| `p3[7]` | string | No | Basic user flag: `"1"` to set user type to `public`, `"0"` or omit for no change |

**Notes:**

* Server computes `fullname` automatically from `name + " " + surname`.
* Backward compatible: omitting `p3[6]` will set `firmname` to empty string.
* `firmname` is sanitized: HTML stripped, emojis removed, truncated to 80 chars.
* When `p3[7]` is `"1"`, the user's `user_type` flag is set to `"public"` in both MySQL (`user_flags` JSON) and Dgraph (`userType` predicate). This uses the composite transactional outbox pattern for replication.
* When `p3[7]` is absent or `"0"`, behaviour is unchanged (MySQL-only event, no type modification).

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |


---

## API_ProfileBio

**Purpose:** Retrieves the user's bio/about section.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileBio"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Bio text (may contain newlines) |


---

## API_ProfileBioEdit

**Purpose:** Updates the user's bio/about section.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileBioEdit"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | New bio text |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message |


---

## API_ProfilePic

**Purpose:** Retrieves the user's profile picture CDN URL with signed access token.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfilePic"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | CDN URL for profile picture (or error message on fail) |
| `p3[0]` | Signed CDN access token |
| `p3[1]` | Header name: `"x-lexcv-token"` |

### Response Example (with picture)

```json
{
  "p1": "Success",
  "p2": "https://cdn.lexcv.co/media/profile/pic/550e8400.jpg",
  "p3": ["eyJhbGciOiJFZDI1NTE5...", "x-lexcv-token"]
}
```

### CDN Token Usage

To fetch the image, include the token in the request header:

```
GET https://cdn.lexcv.co/media/profile/pic/550e8400.jpg
x-lexcv-token: eyJhbGciOiJFZDI1NTE5...
```


---

## API_ProfilePicEdit

**Purpose:** Initiates profile picture upload by generating a signed upload URL.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfilePicEdit"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |
| `p3[0]` | Signed upload URL (PUT to this URL with image data) |
| `p3[1]` | Media token UUID (for tracking the upload) |

### Upload Process


1. Call `API_ProfilePicEdit` to get upload URL
2. Receive signed upload URL in `p3[0]` and media token in `p3[1]`
3. PUT image binary data to the upload URL
4. The Michelangelo server processes and moves the image to final location

### Response Example

```json
{
  "p1": "Success",
  "p3": ["https://storage.googleapis.com/lexcvmedia/temp?signature=...", "a1b2c3d4-uuid"]
}
```


---

## API_ProfilePicDelete

**Purpose:** Deletes the user's profile picture.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfilePicDelete"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message |


---

## API_ProfileSmallPic

**Purpose:** Retrieves the user's small/thumbnail profile picture.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileSmallPic"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p3[0]` | CDN URL for small profile picture |
| `p4[0][0]` | Signed CDN access token |
| `p4[0][1]` | Header name: `"x-lexcv-token"` |


---

## API_ProfileBanner

**Purpose:** Retrieves the user's banner/cover image with position data.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileBanner"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | CDN URL for banner image (or error message on fail) |
| `p3[0]` | Banner X position (for cropping/panning) |
| `p3[1]` | Banner Y position |
| `p4[0][0]` | Signed CDN access token |
| `p4[0][1]` | Header name: `"x-lexcv-token"` |

### Response Example

```json
{
  "p1": "Success",
  "p2": "https://cdn.lexcv.co/media/profile/banner/550e8400.jpg",
  "p3": ["50", "30"],
  "p4": [["eyJhbGciOiJFZDI1NTE5...", "x-lexcv-token"]]
}
```


---

## API_ProfileBannerEdit

**Purpose:** Initiates banner image upload.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileBannerEdit"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Media type: `"image/jpeg"`, `"image/png"`, `"image/webp"` |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message |
| `p3[0]` | Signed upload URL |


---

## API_ProfileBannerEditXY

**Purpose:** Updates the banner X/Y position (for cropping/panning without re-uploading).

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileBannerEditXY"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | X position (integer as string) |
| `p3[1]` | string | Yes | Y position (integer as string) |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message |


---

## API_ProfileBannerDelete

**Purpose:** Deletes the user's banner image.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileBannerDelete"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message |


---

## API_ProfileLink

**Purpose:** Retrieves the user's vanity URL/profile link and optional QR code.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileLink"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Vanity URL slug (e.g., `"john-smith"`) |
| `p3[0]` | QR code CDN URL (if available) |
| `p3[1]` | QR code auth token for cache header |

### Response Example

```json
{
  "p1": "Success",
  "p2": "john-smith",
  "p3": ["https://cdn.lexcv.co/media/qr/550e8400.png", "eyJhbGciOiJFZDI1NTE5..."]
}
```


---

## API_ProfileLinkEdit

**Purpose:** Updates the user's vanity URL/profile link.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileLinkEdit"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | New vanity URL slug |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message (e.g., `"link updated"` or `"link already taken"`) |

### Validation Rules

* Lowercase letters, numbers, and hyphens only
* 3-50 characters
* Must be unique across all users


---

## API_ProfileLawyer

**Purpose:** Retrieves the user's professional lawyer profile (jurisdictions, practice areas, courts, etc.).

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileLawyer"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |
| `p5[0]` | Countries: `[[id, name, short, rank], ...]` |
| `p5[1]` | Geographic Areas: `[[id, name, countryid, rank], ...]` |
| `p5[2]` | Lawyer Types: `[[id, name, countryid, qualYear, rank], ...]` |
| `p5[3]` | Courts: `[[id, name, countryid, international, rank], ...]` |
| `p5[4]` | Work Types: `[[id, name, rank], ...]` |

### Field Details

* **Countries:** Countries only have short codes (e.g., "GB", "US")
* **Geographic Areas:** Linked to a country via countryid
* **Lawyer Types:** Include qualification year facet
* **Courts:** `international` field is "1" for international courts, "0" otherwise; countryid may be "0" for international courts
* **Work Types:** No country association
* **rank:** Position in user's preferred ordering (1-indexed)

### Response Example

```json
{
  "p1": "Success",
  "p5": [
    [["1", "United Kingdom", "GB", "1"], ["2", "France", "FR", "2"]],
    [["10", "London", "1", "1"], ["20", "Paris", "2", "2"]],
    [["100", "Barrister", "1", "2015", "1"], ["101", "Mediator", "1", "0", "2"]],
    [["500", "Supreme Court", "1", "0", "1"], ["501", "International Court of Justice", "0", "1", "2"]],
    [["200", "Contentious", "1"], ["201", "Advisory", "2"]]
  ]
}
```


---

## API_ProfileLawyerEdit

**Purpose:** Updates the user's professional lawyer profile.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileLawyerEdit"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p5[0]` | array | Yes | Countries: `[[id], ...]` (at least 1 required) |
| `p5[1]` | array | Yes | Geographic Areas: `[[id], ...]` (at least 1 required) |
| `p5[2]` | array | Yes | Lawyer Types: `[[id, qualYear], ...]` (at least 1 required; qualYear optional) |
| `p5[3]` | array | No | Courts: `[[id], ...]` |
| `p5[4]` | array | Yes | Work Types: `[[id], ...]` (at least 1 required) |

### Notes

* **Input format differs from output:** Only IDs are sent, names come from index tables
* Rank is determined by array position (first item = rank 1)
* This API updates DGraph search edges for lawyer discovery
* Maximum items per category controlled by server config

### Request Example

```json
{
  "req": "API_ProfileLawyerEdit",
  "p1": "user@email.com",
  "p2": "token",
  "p5": [
    [["1"], ["2"]],
    [["10"], ["20"]],
    [["100", "2015"], ["101"]],
    [["500"], ["501"]],
    [["200"], ["201"]]
  ]
}
```

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |


---

## API_ProfileCompany

**Purpose:** Retrieves company/organisation profile data for the current user. Returns the company's countries, organisation type, work types, and locations.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileCompany"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |
| `p5[0]` | Countries: `[[id, name, shortName], ...]` |
| `p5[1]` | Organisation Type: `[[id, name]]` (single entry) |
| `p5[2]` | Work Types: `[[id, name], ...]` |
| `p5[3]` | Locations: `[[uuid, name, addr1, addr2, city, region, postcode, country, lat, lng, locationType], ...]` |

### P5 Array Structure

The response `p5` is a 3D array containing 4 groups:

**p5\[0\] - Countries** (where the company operates):
| Index | Field | Description |
|----|----|----|----|
| 0 | id | Country ID from countries table |
| 1 | name | Country name |
| 2 | shortName | Country short code (e.g., "GB", "US") |

**p5\[1\] - Organisation Type** (single entry):
| Index | Field | Description |
|----|----|----|----|
| 0 | id | Organisation type ID from organisationtypes table |
| 1 | name | Organisation type name (e.g., "Law Firm", "Chambers", "In-House") |

**p5\[2\] - Work Types** (areas of practice):
| Index | Field | Description |
|----|----|----|----|
| 0 | id | Work type ID from worktype table |
| 1 | name | Work type name (e.g., "Commercial Litigation", "Corporate M&A") |

**p5\[3\] - Locations** (office locations):
| Index | Field | Description |
|----|----|----|----|
| 0 | uuid | Location UUID |
| 1 | name | Location/office name |
| 2 | addr1 | Address line 1 |
| 3 | addr2 | Address line 2 |
| 4 | city | City |
| 5 | region | Region/state/province |
| 6 | postcode | Postal/ZIP code |
| 7 | country | Country name (freeform text, max 100 chars) |
| 8 | lat | Latitude (string) |
| 9 | lng | Longitude (string) |
| 10 | locationType | Location type ID |

### Response Example

```json
{
  "p1": "Success",
  "p5": [
    [["1", "United Kingdom", "GB"], ["2", "Ireland", "IE"]],
    [["5", "Law Firm"]],
    [["10", "Commercial Litigation"], ["15", "Corporate M&A"]],
    [["uuid-1", "London Office", "123 Fleet St", "", "London", "Greater London", "EC4A 2BB", "United Kingdom", "51.5074", "-0.1278", "1"]]
  ]
}
```

### Notes

* Order of items within each group reflects display rank (first = rank 1).
* Empty groups return as empty arrays `[]`.
* Used to pre-populate the company profile edit form.


---

## API_ProfileCompanyEdit

**Purpose:** Updates company/organisation profile data. Sets the user's `user_type` to `company` and creates DGraph edges for searchability.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileCompanyEdit"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p5[0]` | array | Yes | Countries: `[[id], [id], ...]` (min 1 required) |
| `p5[1]` | array | Yes | Organisation Type: `[[id]]` (exactly 1 required) |
| `p5[2]` | array | Yes | Work Types: `[[id], [id], ...]` (min 1 required) |
| `p5[3]` | array | Yes | Locations (see below, min 1 required) |

### Strict payload shape (read carefully)

This API is strict about `p5`.

- `p5` must contain **exactly 4 arrays**: indices `0..3` only.
- Do **not** include an extra trailing group (e.g. a reserved `[]` at `p5[4]`).
- Do **not** swap the order of Work Types and Locations.

If `p5` is not exactly 4 groups, the server returns:

```json
{"p1":"Fail","p2":"invalid profile payload: expected 4 group arrays"}
```

### P5 Input Array Structure

**p5\[0\] - Countries** (IDs only):

```json
[["1"], ["2"], ["3"]]
```

**p5\[1\] - Organisation Type** (single ID):

```json
[["5"]]
```

**p5\[2\] - Work Types** (IDs only):

```json
[["10"], ["15"], ["20"]]
```

**p5\[3\] - Locations** (full location data):
| Index | Field | Description |
|----|----|----|----|
| 0 | uuid | Location UUID (empty string or `"new"` for new locations) |
| 1 | name | Location/office name |
| 2 | addr1 | Address line 1 |
| 3 | addr2 | Address line 2 |
| 4 | city | City |
| 5 | region | Region/state/province |
| 6 | postcode | Postal/ZIP code |
| 7 | country | Country name (freeform text, max 100 chars) |
| 8 | lat | Latitude (string) |
| 9 | lng | Longitude (string) |
| 10 | locationType | Location type ID |

### Common client mistakes (will fail)

#### Mistake A: sending 5 groups (extra trailing `[]`)

```json
{
  "req": "API_ProfileCompanyEdit",
  "p5": [
    [["19"]],
    [["1"]],
    [["28"],["39"]],
    [["uuid","Office","Addr1","Addr2","City","Region","Postcode","Country","-26.1","28.0","0"]],
    []
  ]
}
```

#### Mistake B: swapping Work Types and Locations

```json
{
  "req": "API_ProfileCompanyEdit",
  "p5": [
    [["19"]],
    [["1"]],
    [["uuid","Office","Addr1","Addr2","City","Region","Postcode","Country","-26.1","28.0","0"]],
    [["28"],["39"]]
  ]
}
```

### Request Example

```json
{
  "req": "API_ProfileCompanyEdit",
  "p1": "admin@lawfirm.com",
  "p2": "token",
  "p5": [
    [["1"], ["2"]],
    [["5"]],
    [["10"], ["15"]],
    [
      ["", "London Office", "123 Fleet St", "", "London", "Greater London", "EC4A 2BB", "United Kingdom", "51.5074", "-0.1278", "1"],
      ["uuid-existing", "Dublin Office", "45 St Stephen's Green", "", "Dublin", "Dublin", "D02 NY63", "Ireland", "53.3393", "-6.2576", "1"]
    ]
  ]
}
```

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |

### Validation Rules

| Group | Constraint |
|----|----|
| Countries (`p5[0]`) | Min 1 required. Max from config `ConfigKeyProfileCountriesMax`. |
| Org Type (`p5[1]`) | Exactly 1 required. |
| Work Types (`p5[2]`) | Min 1 required. Max from config `ConfigKeyProfileWorkMax`. |
| Locations (`p5[3]`) | Min 1 required. Max from config `ConfigKeyProfileLocationsMax`. Each entry must have 11 fields. |

### Error Responses

```json
{"p1": "Fail", "p2": "invalid profile payload: expected 4 group arrays"}
{"p1": "Fail", "p2": "at least one country is required"}
{"p1": "Fail", "p2": "countries exceed max of 10"}
{"p1": "Fail", "p2": "exactly one organisation type is required"}
{"p1": "Fail", "p2": "at least one work type is required"}
{"p1": "Fail", "p2": "at least one location is required"}
```

### Side Effects

* Sets `user_flags.user_type = ["company"]` in MySQL.
* Creates DGraph edges: `companyInCountry`, `companyIsType`, `companyDoesWorkType`, `companyAtLocation`.
* Replaces all existing company edges (DELETE then SET).
* Uses Sacred Timeline transactional outbox for replication.


---

## API_ProfileCases

**Purpose:** Retrieves the user's notable cases.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileCases"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |
| `p4` | Cases array: `[[uuid, casename, casedesc, iorder], ...]` |

### Field Details

* **uuid:** Unique identifier for the case entry
* **casename:** Case title/name
* **casedesc:** Case description
* **iorder:** Display order (string, 1-indexed)

### Response Example

```json
{
  "p1": "Success",
  "p4": [
    ["uuid1", "Smith v Jones [2023]", "Landmark contract dispute...", "1"],
    ["uuid2", "Re: Major Corp", "Complex restructuring matter...", "2"]
  ]
}
```


---

## API_ProfileCasesEdit

**Purpose:** Updates the user's notable cases (full replacement).

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileCasesEdit"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p4` | array | Yes | Cases: `[[uuid, casename, casedesc, iorder], ...]` |

### Notes

* Use empty string `""` for uuid when adding new cases
* Server generates UUID for new cases
* Send complete list - this replaces all existing cases


---

## API_ProfileEduc

**Purpose:** Retrieves the user's education history.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileEduc"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |
| `p4` | Education array: `[[uuid, school, subject, iorder, grade, startyear, endyear, ongoing], ...]` |

### Field Details

* **uuid:** Unique identifier for the education entry
* **school:** Institution name
* **subject:** Field of study/degree
* **iorder:** Display order (string)
* **grade:** Degree classification/grade
* **startyear:** Start year (string, "0" if not set)
* **endyear:** End year (string, "0" if not set)
* **ongoing:** "1" if currently studying, "0" otherwise


---

## API_ProfileEducEdit

**Purpose:** Updates the user's education history (full replacement).

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileEducEdit"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p4` | array | Yes | Education: `[[school, subject, iorder, grade, ongoing, startyear, endyear], ...]` |

**Note:** Client sends entries without UUID. Server generates new UUIDs during save. The `iorder` field from client is ignored; server assigns order based on array position.


---

## API_ProfileExperience

**Purpose:** Retrieves the user's work experience history.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileExperience"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |
| `p4` | Experience array (see below) |

### P4 Array Structure

Each entry in `p4`:

```
[uuid, jobtitle, organisation, organisationuuid, locationid, description, startdate, enddate, ongoing, iorder]
```

| Index | Field | Description |
|----|----|----|
| 0 | uuid | Unique identifier for the experience entry |
| 1 | jobtitle | Job title/position |
| 2 | organisation | Organisation name (free text) |
| 3 | organisationuuid | UUID of linked company profile (empty if unlinked) |
| 4 | locationid | Location ID from locations table (empty if not set) |
| 5 | description | Job description text |
| 6 | startdate | Unix timestamp in milliseconds ("0" if not set) |
| 7 | enddate | Unix timestamp in milliseconds ("0" if not set) |
| 8 | ongoing | "1" if current role, "0" otherwise |
| 9 | iorder | Display order (string) |


---

## API_ProfileExperienceEdit

**Purpose:** Updates the user's work experience history (full replacement).

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileExperienceEdit"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p4` | array | Yes | Experience entries (see below) |

### P4 Input Array Structure

Each entry in `p4`:

```
[jobtitle, organisation, organisationuuid, locationid, description, startdate, enddate, ongoing, iorder]
```

| Index | Field | Description |
|----|----|----|
| 0 | jobtitle | Job title/position |
| 1 | organisation | Organisation name (free text) |
| 2 | organisationuuid | UUID of linked company profile (empty string if unlinked) |
| 3 | locationid | Location ID (empty string if not set) |
| 4 | description | Job description text |
| 5 | startdate | Unix timestamp in milliseconds (empty string for NULL) |
| 6 | enddate | Unix timestamp in milliseconds (empty string for NULL) |
| 7 | ongoing | "1" if current role, "0" otherwise |
| 8 | iorder | Display order (ignored; server assigns based on array position) |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |

**Replication:** Uses Sacred Timeline transactional outbox. DELETE + INSERT is atomic and idempotent.


---

## API_ProfilePublications

**Purpose:** Retrieves the user's publications (articles, books, papers).

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfilePublications"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |
| `p4` | Publications array (see below) |

### P4 Array Structure

Each entry in `p4`:

```
[uuid, title, authors, publisher, publicationtype, publicationdate, doi, url, abstract, iorder]
```

| Index | Field | Description |
|----|----|----|
| 0 | uuid | Unique identifier for the publication entry |
| 1 | title | Publication title |
| 2 | authors | Authors (comma-separated or formatted string) |
| 3 | publisher | Publisher name |
| 4 | publicationtype | Type: "0"=article, "1"=book, "2"=paper, "3"=chapter, "4"=report, "5"=thesis |
| 5 | publicationdate | Unix timestamp in milliseconds ("0" if not set) |
| 6 | doi | Digital Object Identifier (empty if not set) |
| 7 | url | URL to publication (empty if not set) |
| 8 | abstract | Abstract/summary text |
| 9 | iorder | Display order (string) |


---

## API_ProfilePublicationsEdit

**Purpose:** Updates the user's publications (full replacement).

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfilePublicationsEdit"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p4` | array | Yes | Publication entries (see below) |

### P4 Input Array Structure

Each entry in `p4`:

```
[title, authors, publisher, publicationtype, publicationdate, doi, url, abstract, iorder]
```

| Index | Field | Description |
|----|----|----|
| 0 | title | Publication title |
| 1 | authors | Authors (comma-separated or formatted string) |
| 2 | publisher | Publisher name |
| 3 | publicationtype | Type: "0"=article, "1"=book, "2"=paper, "3"=chapter, "4"=report, "5"=thesis |
| 4 | publicationdate | Unix timestamp in milliseconds (empty string for NULL) |
| 5 | doi | Digital Object Identifier (empty string if not set) |
| 6 | url | URL to publication (empty string if not set) |
| 7 | abstract | Abstract/summary text |
| 8 | iorder | Display order (ignored; server assigns based on array position) |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |

**Replication:** Uses Sacred Timeline transactional outbox. DELETE + INSERT is atomic and idempotent.


---

## API_ProfileHighs

**Purpose:** Retrieves the user's career highlights.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileHighs"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |
| `p4` | Highlights array: `[[uuid, highlight, iorder], ...]` |

### Field Details

* **uuid:** Unique identifier for the highlight entry
* **highlight:** Highlight text
* **iorder:** Display order (string)


---

## API_ProfileHighsEdit

**Purpose:** Updates the user's career highlights (full replacement).

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileHighsEdit"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p4` | array | Yes | Highlights: `[[uuid, highlight, iorder], ...]` |


---

## API_ProfileRecs

**Purpose:** Retrieves recommendations received by the user.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileRecs"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |
| `p4` | Recommendations: `[[uuid, recommender, recommend, iorder], ...]` |

### Field Details

* **uuid:** Unique identifier for the recommendation
* **recommender:** Name of the person who wrote the recommendation
* **recommend:** Recommendation text
* **iorder:** Display order (string)


---

## API_ProfileRecsEdit

**Purpose:** Manages recommendations (request, approve, decline).

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileRecsEdit"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Action: `"request"`, `"write"`, `"approve"`, `"decline"`, `"delete"` |
| `p3[1]` | string | Varies | Target user UUID (for request) or rec UUID (for others) |
| `p3[2]` | string | For write | Recommendation text |


---

## API_ProfileShortlist

**Purpose:** Retrieves the user's shortlisted (bookmarked) profiles.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileShortlist"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p6` | Array of shortlisted user arrays (2D array) |

### User Entry Format (each in p6)

Uses the standard 14-field user array format - see [05_Search_APIs.md](05_Search_APIs.md) for field details.


---

## API_ProfileShortlistList

**Purpose:** Alternative shortlist retrieval with pagination.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileShortlistList"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | No | Offset (default: "0") |
| `p3[1]` | string | No | Limit (default: "20", max: "50") |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p3[0]` | Total count |
| `p6` | Array of shortlisted user arrays (same standard 14-field format as above) |


