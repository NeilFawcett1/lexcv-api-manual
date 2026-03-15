# Social APIs

APIs for viewing other users' profiles, following, and shortlisting.


---

## API_PeerProfile

**Purpose:** Retrieves another user's public profile by UUID or vanity URL.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PeerProfile"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Target user UUID OR vanity URL slug |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message on failure |
| `p3` | Basic profile fields (20-element string array) |
| `p5` | Professional profile and extended data (see below) |

### Profile Fields (p3)

| Index | Field |
|----|----|
| `p3[0]` | User UUID |
| `p3[1]` | First name |
| `p3[2]` | Surname |
| `p3[3]` | Headline |
| `p3[4]` | Contact email |
| `p3[5]` | Telephone |
| `p3[6]` | Website |
| `p3[7]` | User flags JSON |
| `p3[8]` | Bio text |
| `p3[9]` | Vanity URL slug |
| `p3[10]` | Profile picture CDN URL |
| `p3[11]` | Profile picture CDN token |
| `p3[12]` | Banner CDN URL |
| `p3[13]` | Banner CDN token |
| `p3[14]` | Banner X position |
| `p3[15]` | Banner Y position |
| `p3[16]` | Message keys initialized: `"1"` or `"0"` |
| `p3[17]` | Allow messages setting |
| `p3[18]` | QR code CDN URL |
| `p3[19]` | QR code CDN token |
| `p3[20]` | Viewer follows this user: `"1"` or `"0"` |
| `p3[21]` | This user follows viewer: `"1"` or `"0"` |
| `p3[22]` | Viewer has shortlisted: `"1"` or `"0"` |
| `p3[23]` | Shortlist rank (if shortlisted) |
| `p3[24]` | Blog count: Count of user's public blog articles |

### Professional Profile and Extended Data (p5)

`p5` depends on the target user's `user_flags.user_type`:

- If `user_type` is `lawyer`, `p5` uses the lawyer profile layout below.
- If `user_type` is `company`, `p5[0..4]` contains the company profile groups (with `p5[4]` reserved empty for stable indexing). Company profiles share the same overall layout, but `p5[6]` (Education) and `p5[9]` (Experience) are empty arrays.

| Index | Content | Field Format |
|----|----|----|
| `p5[0]` | Countries | `[[id, name, short, rank], ...]` |
| `p5[1]` | Practice areas | `[[id, name, countryid, rank], ...]` |
| `p5[2]` | Lawyer types | `[[id, name, countryid, qualYear, rank], ...]` |
| `p5[3]` | Courts | `[[id, name, countryid, international, rank], ...]` |
| `p5[4]` | Work types | `[[id, name, rank], ...]` |
| `p5[5]` | Cases | `[[uuid, casename, casedesc, iorder], ...]` |
| `p5[6]` | Education | `[[uuid, school, subject, iorder, grade, startdate, enddate, ongoing], ...]` |
| `p5[7]` | Highlights | `[[uuid, highlight, iorder], ...]` |
| `p5[8]` | Recommendations | `[[uuid, recommender, recommend, iorder], ...]` |
| `p5[9]` | Experience | `[[uuid, jobtitle, organisation, organisationuuid, locationid, description, startdate, enddate, ongoing, iorder], ...]` |
| `p5[10]` | Publications | `[[uuid, title, authors, publisher, publicationtype, publicationdate, doi, url, abstract, iorder], ...]` |

#### Company profile layout (when `user_type` is `company`)

| Index | Content | Field Format |
|----|----|----|
| `p5[0]` | Countries | `[[id, name, short, rank], ...]` |
| `p5[1]` | Organisation type | `[[id, name], ...]` (typically exactly 1 row) |
| `p5[2]` | Locations | `[[uuid, name, addr1, addr2, city, region, postcode, country, lat, lng, locType, rank], ...]` |
| `p5[3]` | Work types | `[[id, name, rank], ...]` |
| `p5[4]` | Reserved | `[]` |
| `p5[5]` | Cases | `[[uuid, casename, casedesc, iorder], ...]` |
| `p5[6]` | Education | `[]` |
| `p5[7]` | Highlights | `[[uuid, highlight, iorder], ...]` |
| `p5[8]` | Recommendations | `[[uuid, recommender, recommend, iorder], ...]` |
| `p5[9]` | Experience | `[]` |
| `p5[10]` | Publications | `[[uuid, title, authors, publisher, publicationtype, publicationdate, doi, url, abstract, iorder], ...]` |

**Note:** All date fields (startdate, enddate, publicationdate) are Unix timestamps in milliseconds.

### Response Example

```json
{
  "p1": "Success",
  "p3": [
    "user-uuid",
    "John",
    "Smith",
    "Commercial Barrister",
    "john@chambers.com",
    "+44 20 1234 5678",
    "www.chambers.com/john-smith",
    "{\"user_state\":[\"verified\"],\"user_type\":[\"lawyer\"]}",
    "Over 20 years experience in commercial litigation...",
    "john-smith",
    "https://cdn.lexcv.co/profilepics/uuid.jpg",
    "pic-token",
    "https://cdn.lexcv.co/banners/uuid.jpg",
    "banner-token",
    "0",
    "50",
    "1",
    "all",
    "https://cdn.lexcv.co/qr/uuid.png",
    "qr-token"
  ],
  "p5": [
    [["123", "United Kingdom", "GB", "1"], ["456", "France", "FR", "2"]],
    [["789", "Commercial Law", "123", "1"], ["790", "Arbitration", "123", "2"]],
    [["12", "Barrister", "123", "2015", "1"]],
    [["34", "Supreme Court", "123", "0", "1"]],
    [["56", "Contentious", "1"]],
    [["case-uuid", "Smith v Jones", "High profile commercial dispute", "1"]],
    [["educ-uuid", "Oxford University", "Law", "1", "First Class", "1283299200000", "1378080000000", "0"]],
    [["high-uuid", "Awarded Barrister of the Year 2022", "1"]],
    [["rec-uuid", "Jane Doe, Senior Partner", "Excellent advocate and thorough preparation", "1"]],
    [["exp-uuid", "Senior Barrister", "Temple Chambers", "", "", "Leading complex commercial cases", "1420070400000", "0", "1", "1"]],
    [["pub-uuid", "Contract Law in the Digital Age", "John Smith, Jane Doe", "Legal Publishing Ltd", "0", "1609459200000", "10.1234/legal.2021", "https://legal.example.com/article", "Analysis of modern contract law challenges", "1"]]
  ]
}
```


---

## API_ProfileAll

**Purpose:** Retrieves the complete profile for the currently logged-in user in a single request. Consolidates multiple profile API calls (ProfileBasic, ProfilePic, ProfileBanner, ProfileLawyer, ProfileLink, ProfileBio, ProfileCases, ProfileHighs, ProfileRecs, ProfileEduc, ProfileExperience, ProfilePublications) into one request.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileAll"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message on failure |
| `p3` | Basic profile fields (same as API_ProfileBasic) |
| `p4` | Extended profile data (Pic, Banner, Link, Bio) |
| `p5` | Professional profile arrays (same as API_ProfileLawyer, etc.) |

### Basic Profile Fields (p3)

Same format as `API_ProfileBasic`:

| Index | Field |
|----|----|
| `p3[0]` | Username (uname) |
| `p3[1]` | First name |
| `p3[2]` | Surname |
| `p3[3]` | Headline |
| `p3[4]` | Contact email |
| `p3[5]` | Telephone |
| `p3[6]` | Website |
| `p3[7]` | Following count |
| `p3[8]` | Followed by count |

### Extended Profile Data (p4)

| Index | Content | Field Format |
|----|----|----|
| `p4[0]` | ProfilePic | `[cdnURL, token, headerName]` |
| `p4[1]` | ProfileBanner | `[cdnURL, token, headerName, bannerX, bannerY]` |
| `p4[2]` | ProfileLink | `[alias, qrURL, qrToken]` |
| `p4[3]` | ProfileBio | `[bioText]` |

### Professional Profile Arrays (p5)

`p5` depends on the logged-in user's `user_flags.user_type`.

#### Lawyer profile layout (when `user_type` is `lawyer`)

| Index | Content | Field Format |
|----|----|----|
| `p5[0]` | Countries | `[[id, name, short, rank], ...]` |
| `p5[1]` | Practice areas | `[[id, name, countryid, rank], ...]` |
| `p5[2]` | Lawyer types | `[[id, name, countryid, qualYear, rank], ...]` |
| `p5[3]` | Courts | `[[id, name, countryid, international, rank], ...]` |
| `p5[4]` | Work types | `[[id, name, rank], ...]` |
| `p5[5]` | Cases | `[[uuid, casename, casedesc, iorder], ...]` |
| `p5[6]` | Education | `[[uuid, school, subject, iorder, grade, startdate, enddate, ongoing], ...]` |
| `p5[7]` | Highlights | `[[uuid, highlight, iorder], ...]` |
| `p5[8]` | Recommendations | `[[uuid, recommender, recommend, iorder], ...]` |
| `p5[9]` | Experience | `[[uuid, jobtitle, organisation, organisationuuid, locationid, description, startdate, enddate, ongoing, iorder], ...]` |
| `p5[10]` | Publications | `[[uuid, title, authors, publisher, publicationtype, publicationdate, doi, url, abstract, iorder], ...]` |

#### Company profile layout (when `user_type` is `company`)

| Index | Content | Field Format |
|----|----|----|
| `p5[0]` | Countries | `[[id, name, short, rank], ...]` |
| `p5[1]` | Organisation type | `[[id, name], ...]` (typically exactly 1 row) |
| `p5[2]` | Locations | `[[uuid, name, addr1, addr2, city, region, postcode, country, lat, lng, locType, rank], ...]` |
| `p5[3]` | Work types | `[[id, name, rank], ...]` |
| `p5[4]` | Reserved | `[]` |
| `p5[5]` | Cases | `[[uuid, casename, casedesc, iorder], ...]` |
| `p5[6]` | Education | `[]` |
| `p5[7]` | Highlights | `[[uuid, highlight, iorder], ...]` |
| `p5[8]` | Recommendations | `[[uuid, recommender, recommend, iorder], ...]` |
| `p5[9]` | Experience | `[]` |
| `p5[10]` | Publications | `[[uuid, title, authors, publisher, publicationtype, publicationdate, doi, url, abstract, iorder], ...]` |

**Note:** All date fields (startdate, enddate, publicationdate) are Unix timestamps in milliseconds.

### Notes

* This API is designed for the logged-in user to load their own profile
* Use `API_PeerProfile` to view other users' profiles
* `p3` matches `API_ProfileBasic` exactly for client-side compatibility
* Fetches all data concurrently for optimal performance


---

## API_PeerProfileAddFollow

**Purpose:** Follows another user.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PeerProfileAddFollow"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Target user UUID to follow |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message |

### Notes

* Idempotent: following an already-followed user returns success
* Cannot follow yourself
* Creates graph edge from follower to followed


---

## API_PeerProfileDelFollow

**Purpose:** Unfollows a user.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PeerProfileDelFollow"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Target user UUID to unfollow |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message |

### Notes

* Idempotent: unfollowing a non-followed user returns success


---

## API_PeerProfileAddShortlist

**Purpose:** Adds a user to the shortlist (bookmarks).

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PeerProfileAddShortlist"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Target user UUID to shortlist |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message |

### Notes

* Shortlist is private - the other user is not notified
* Maximum shortlist size: 500 users (configurable)


---

## API_PeerProfileDelShortlist

**Purpose:** Removes a user from the shortlist.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PeerProfileDelShortlist"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Target user UUID to remove |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message |


---

## API_PeerProfileFollowsCounts

**Purpose:** Gets following and followed-by counts for a user.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PeerProfileFollowsCounts"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Target user UUID |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p3[0]` | Following count (users this person follows) |
| `p3[1]` | Followed-by count (users who follow this person) |


---

## API_PeerProfileFollowersList

**Purpose:** Lists users who follow the target user.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PeerProfileFollowersList"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | No | Target user UUID (defaults to logged-in user) |
| `p3[1]` | string | No | Offset (default: "0") |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p3[0]` | Total follower count |
| `p3[1]` | Target user UUID |
| `p3[2]` | Offset used |
| `p3[3]` | Limit used |
| `p4` | Array of follower user entries (2D array) |

### User Entry Format (in p4)

Each entry in `p4` is an array with user details using the standard 14-field format:

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


---

## API_PeerProfileFollowingList

**Purpose:** Lists users that the target user follows.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PeerProfileFollowingList"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | No | Target user UUID (defaults to logged-in user) |
| `p3[1]` | string | No | Offset (default: "0") |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p3[0]` | Total following count |
| `p3[1]` | Target user UUID |
| `p3[2]` | Offset used |
| `p3[3]` | Limit used |
| `p4` | Array of followed user entries (2D array) |

### User Entry Format (in p4)

Same 14-field format as API_PeerProfileFollowersList - see that API for field details.


---

## API_ProfileFollowersList

**Purpose:** Lists users who follow the current user (own followers).

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileFollowersList"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | No | Offset (default: "0") |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p3[0]` | Total follower count |
| `p3[1]` | User UUID |
| `p3[2]` | Offset used |
| `p3[3]` | Limit used |
| `p4` | Array of follower user entries (2D array) |

### User Entry Format (in p4)

Same 14-field format as API_PeerProfileFollowersList - see that API for field details.


---

## API_ProfileFollowingList

**Purpose:** Lists users that the current user follows (own following).

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_ProfileFollowingList"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | No | Offset (default: "0") |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p3[0]` | Total following count |
| `p3[1]` | User UUID |
| `p3[2]` | Offset used |
| `p3[3]` | Limit used |
| `p4` | Array of followed user entries (2D array) |

### User Entry Format (in p4)

Same 14-field format as API_PeerProfileFollowersList - see that API for field details.


---

## Follow vs Shortlist

| Feature | Follow | Shortlist |
|----|----|----|
| Visibility | Public (appears in follower lists) | Private (only you see) |
| Notification | Yes (followed user notified) | No notification |
| Use case | Professional networking | Personal bookmarks |
| Limit | Unlimited | 500 maximum |
| Affects feed | Yes (see their posts) | No (just bookmark) |


