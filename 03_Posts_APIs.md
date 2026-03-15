# Posts & Comments APIs

APIs for creating, viewing, and interacting with posts and comments.


---

## API_PostAddNew

**Purpose:** Creates a new post with optional media attachments.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PostAddNew"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Post text content (required if no media) |
| `p4` | array | No | Media requests: `[[type], ...]` |

### Media Types

| Type | Description |
|----|----|
| `"1"` | Image (jpg/png) |
| `"2"` | PDF document |
| `"3"` | Video (mp4) |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Post UUID on success, error message on failure |
| `p4` | Upload URLs: `[[type, signed_upload_url], ...]` |

### Request Example (with media)

```json
{
  "req": "API_PostAddNew",
  "p1": "user@email.com",
  "p2": "token",
  "p3": ["Check out this case analysis!"],
  "p4": [["1"], ["1"], ["2"]]
}
```

### Response Example

```json
{
  "p1": "Success",
  "p2": "550e8400-e29b-41d4-a716-446655440000",
  "p4": [
    ["1", "https://storage.googleapis.com/upload1?sig=..."],
    ["2", "https://storage.googleapis.com/upload2?sig=..."]
  ]
}
```

### Upload Process


1. Call `API_PostAddNew` with post text and media types (numeric)
2. Receive post UUID and signed upload URLs
3. PUT media binary data to each upload URL
4. Post becomes visible after media processing completes


---

## API_PostDelete

**Purpose:** Soft-deletes a post (marks as deleted, preserves for moderation/recovery).

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PostDelete"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Post UUID to delete |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message |

### Notes

* User can only delete their own posts
* Comments on deleted posts remain but show "\[deleted\]" author
* Media files are cleaned up asynchronously


---

## API_PostsListAll

**Purpose:** Retrieves the combined timeline feed (posts from user's linked nations and followed users).

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PostsListAll"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | No | Offset (default: "0") |
| `p3[1]` | string | No | Limit (default from config, max: 100) |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message on failure |
| `p3[0]` | Total count of posts in timeline |
| `p6` | Array of post RObjs |

### Post RObj Structure (each item in p6)

| Field | Description |
|----|----|
| `p3[0]` | Type: `"0"`=post, `"1"`=comment |
| `p3[1]` | Post/comment UUID |
| `p3[2]` | Author UUID |
| `p3[3]` | Post text content |
| `p3[4]` | Creation date (unix ms, string) |
| `p3[5]` | Last edited (unix ms, string) |
| `p4[0]` | Reaction counts: `[like_count, support_count, laugh_count, dislike_count]` (policy: `dislike_count` is hidden and always returned as `"0"`) |
| `p4[1]` | Engagement: `[comment_count, repost_count, impressions]` (impressions = unique impressions estimate from Clio) |
| `p4[2]` | Related: `[repost_of_uuid, best_answer_uuid]` |
| `p4[3]` | Author info: `[display_name, headline, avatar_url, avatar_token]` |
| `p4[4]` | Viewer reaction: `[reaction_type]` (`-1`=none, `0`=like, `1`=support, `2`=laugh, `3`=dislike) |
| `p4[5]` | Author flags JSON (string) |
| `p4[6]` | Author messaging info: `[messageKeysInitialized, allowMessages]` where `allowMessages` is one of `all`, `followers`, `none` |
| `p5` | Media: `[[type, cdnURL], ...]` where `type` is `"1"`=image, `"2"`=pdf, `"3"`=video |
| `p6` | Optional nested comment RObjs. If `best_answer_uuid` is set, `p6[0]` contains the best-answer comment only (no nested replies). |

### Response Example

```json
{
  "p1": "Success",
  "p3": ["156"],
  "p6": [
    {
      "p3": ["0", "post-uuid-1", "author-uuid", "Post content here...", "1699876543210", "0"],
      "p4": [
        ["5", "2", "1", "0"],
        ["3", "0", "42"],
        ["", ""],
        ["John Smith", "Senior Partner", "https://cdn.lexcv.co/pic.jpg"]
      ]
    }
  ]
}
```

**Impressions semantics**

- `impressions` is the post's **unique impressions estimate** (approximate), sourced from Clio all-time counter `post.impression` (`clio_counter_alltime.unique_estimate`).
- This value should not increase when the same logged-in user refreshes.


---

## API_PostsListFollowing

**Purpose:** Retrieves timeline feed from followed users only.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PostsListFollowing"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | No | Offset (default: "0") |
| `p3[1]` | string | No | Limit (default from config, max: 100) |
---

## API_PostsGetPostsForUser

**Purpose:** Retrieves posts authored by a specific user (profile feed).

**Authentication:** Requires valid login (p1=email, p2=token)

**Notes:** Response post RObj format is identical to `API_PostsListAll`.

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PostsGetPostsForUser"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Target user UUID |
| `p3[1]` | string | No | Offset (default: "0") |
| `p3[2]` | string | No | Limit (default from config, max: 100) |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message on failure |
| `p3[0]` | Total count of posts authored by target user |
| `p6` | Array of post RObjs (same structure as `API_PostsListAll`) |


### Response RObj

Same structure as `API_PostsListAll`


---

## API_PostGetPost

**Purpose:** Retrieves a single post by UUID with full details.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PostGetPost"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Post UUID |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p6[0]` | Post RObj (same structure as in list) |
| `p6[0].p6[0]` | If `best_answer_uuid` is set, contains the best-answer comment only (flat, no nested replies). |


---

## API_PostGetComments

**Purpose:** Retrieves comments for a post or comment with recursive nesting.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PostGetComments"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Parent UUID (post or comment) |
| `p3[1]` | string | No | Parent type: `"0"`=post (default), `"1"`=comment |
| `p3[2]` | string | No | Page offset (0-based, default: 0) |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message on failure |
| `p3[0]` | Total count of direct comments on parent |
| `p3[1]` | Has more pages: `"1"`=yes, `"0"`=no |
| `p6` | Array of comment RObjs (recursive structure) |

### Comment RObj Structure (each item in p6)

| Field | Description |
|----|----|
| `p3[0]` | Type: always `"1"` for comment |
| `p3[1]` | Comment UUID |
| `p3[2]` | Author UUID |
| `p3[3]` | Comment text |
| `p3[4]` | Creation date (unix ms, string) |
| `p3[5]` | Last edited date (unix ms, string) |
| `p4[0]` | Reactions: `[like_count, support_count, laugh_count, dislike_count]` (currently only `like_count` is used for comment counters; others are returned as `"0"`. Policy: `dislike_count` is hidden and always returned as `"0"`) |
| `p4[1]` | Stats: `[has_replies, repost_count, impressions]` |
| `p4[2]` | References: `[root_post_uuid, parent_uuid]` |
| `p4[3]` | Author: `[display_name, headline, avatar_url, avatar_token]` |
| `p4[4]` | Viewer reaction: `[reaction_type]` (`-1`=none, `0`=like, `1`=support, `2`=laugh, `3`=dislike) |
| `p4[5]` | Author status: `[author_reaction, is_favourite]` |
| `p4[6]` | Author flags JSON (string) |
| `p6` | Nested replies (same structure, recursive) |
---

## API_PostsGetCommentsForUser

**Purpose:** Retrieves comments authored by a specific user (profile activity feed).

**Authentication:** Requires valid login (p1=email, p2=token)

**Notes:** Response comment RObj format matches `API_PostGetComments`, but the list is flat (no nested replies in `p6`).

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PostsGetCommentsForUser"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Target user UUID |
| `p3[1]` | string | No | Offset (default: "0") |
| `p3[2]` | string | No | Limit (default from config, max: 100) |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message on failure |
| `p3[0]` | Total count of comments authored by target user |
| `p3[1]` | Has more pages: `"1"`=yes, `"0"`=no |
| `p6` | Array of comment RObjs (same structure as `API_PostGetComments`, flat) |


### Nesting Behavior

* Top-level comments are returned in `p6`
* Each comment may contain nested replies in its own `p6`
* Maximum nesting depth controlled by server config
* Replies can be fetched separately using parent_uuid with type `"1"`


---

## API_PostGetPostReactsForUser

**Purpose:** Retrieves posts that a specific user has reacted to.

**Authentication:** Requires valid login (p1=email, p2=token)

**Notes:** Response post RObj format is identical to `API_PostsListAll`.

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PostGetPostReactsForUser"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Target user UUID |
| `p3[1]` | string | No | Reaction type filter: `likes`, `supports`, `laughs`, `dislikes` (or `0`-`3`). Empty/absent returns all reactions (policy: dislikes excluded). |
| `p3[2]` | string | No | Offset (default: `"0"`) |
| `p3[3]` | string | No | Limit (default from config, max: 100) |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message on failure |
| `p3[0]` | Total count of reacted posts by target user |
| `p3[1]` | Has more pages: `"1"`=yes, `"0"`=no |
| `p6` | Array of post RObjs (same structure as `API_PostsListAll`) |

## API_PostsCommentAddNew

**Purpose:** Adds a comment to a post or a reply to an existing comment.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PostsCommentAddNew"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Post UUID |
| `p3[1]` | string | Yes | Comment text |
| `p3[2]` | string | No | Parent comment UUID (for replies) |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | New comment UUID |


---

## API_PostsCommentDelete

**Purpose:** Deletes a comment (soft-delete).

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PostsCommentDelete"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Comment UUID |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message |

### Notes

* User can only delete their own comments
* Post author can delete any comment on their post
* Deleted comments show "\[deleted\]" text, preserving thread structure


---

## API_PostCommentSetFavourite

**Purpose:** Marks a comment as the "best answer" or favourite (post author only).

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PostCommentSetFavourite"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Comment UUID |
| `p3[1]` | string | Yes | Action: `"SET"` or `"CLEAR"` |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message |

### Notes

* Only the post author can set/clear favourite
* Only one comment per post can be marked as favourite
* Setting a new favourite automatically clears any previous favourite


---

## API_PostAddReact

**Purpose:** Adds or updates a reaction on a post or comment.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PostAddReact"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Target UUID (post or comment) |
| `p3[1]` | string | Yes | Target type: `"0"`=post, `"1"`=comment |
| `p3[2]` | string | Yes | Reaction type (see below) |

### Reaction Types

| Type | Description |
|----|----|
| `"0"` | Like |
| `"1"` | Support |
| `"2"` | Laugh |
| `"3"` | Dislike |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Previous reaction type (`"-1"` if none), or error message on failure |

### Notes

* User can only have one reaction per post/comment
* Calling again with different type changes the reaction
* Uses Sacred Timeline replication


---

## API_PostDelReact

**Purpose:** Removes a reaction from a post or comment.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PostDelReact"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Target UUID (post or comment) |
| `p3[1]` | string | Yes | Target type: `"0"`=post, `"1"`=comment |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Status message |


---

## API_PostGetReacts

**Purpose:** Returns a list of users who have reacted to a post, with full user profile summaries.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|----|----|----|----|
| `req` | string | Yes | `"API_PostGetReacts"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Post UUID |
| `p3[1]` | string | No | Reaction type filter (see below) |
| `p3[2]` | string | No | Offset for pagination (default `"0"`) |

### Reaction Type Filter (p3[1])

| Value | Description |
|----|----|
| (empty) | Return all reactions, grouped by type (likes first) |
| `"likes"` or `"0"` | Only users who liked |
| `"supports"` or `"1"` | Only users who supported |
| `"laughs"` or `"2"` | Only users who laughed |
| `"dislikes"` or `"3"` | Policy: dislikers are hidden; always returns 0 results |

### Response RObj

| Field | Description |
|----|----|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message on failure |
| `p3[0]` | Total count of matching reactors |
| `p3[1]` | Post UUID |
| `p3[2]` | Offset used |
| `p3[3]` | Limit used |
| `p3[4]` | Reaction type filter applied (empty if all) |
| `p4` | Array of user rows (see below) |

### User Row Format (p4)

Each row in p4 is an array with the following fields:

| Index | Field | Description |
|----|----|----|
| 0 | uuid | User UUID |
| 1 | fullName | User's full name |
| 2 | headline | User's headline |
| 3 | smallPicUrl | Signed CDN URL for avatar |
| 4 | flagsJSON | User flags JSON (verified, lawyer, level, etc.) |
| 5 | position | Position in result set |
| 6 | followByViewer | `"1"` if logged-in user follows this user |
| 7 | followsViewer | `"1"` if this user follows logged-in user |
| 8 | shortlistFlag | `"1"` if user is in viewer's shortlist |
| 9 | shortlistRank | Shortlist rank if applicable |
| 10 | smallPicToken | Token for avatar URL |
| 11 | messageKeysInitialized | `"1"` if user has messaging keys |
| 12 | allowMessages | Messaging discovery policy: `"all"`, `"followers"`, or `"none"` |
| 13 | blogCount | Count of user's public blog articles |
| 14 | reactType | Reaction type: `"0"`=like, `"1"`=support, `"2"`=laugh (policy: `"3"`=dislike is not returned in lists) |

### Example Request

```json
{
  "req": "API_PostGetReacts",
  "p1": "user@example.com",
  "p2": "auth-token",
  "p3": ["post-uuid-123", "likes", "0", "20"]
}
```

### Example Response

```json
{
  "p1": "Success",
  "p3": ["42", "post-uuid-123", "0", "20", "likes"],
  "p4": [
    ["user-uuid-1", "John Doe", "Attorney at Law", "https://cdn.example.com/pic1.jpg", "{\"user_state\":[\"verified\"]}", "0", "1", "0", "0", "0", "token1", "1", "1", "3", "0"],
    ["user-uuid-2", "Jane Smith", "Partner", "https://cdn.example.com/pic2.jpg", "{}", "1", "0", "1", "0", "0", "token2", "1", "1", "5", "0"]
  ]
}
```

### Notes

* User info format matches the followers list API (API_PeerProfileFollowersList)
* The `blogCount` field (index 13) shows the count of user's public blog articles
* The `reactType` field (index 14) is appended as the last element of each user row
* When no filter is specified, reactions are returned grouped by type (likes first, then supports, laughs). Dislikes are not included.
* Pagination is supported via offset; limit is controlled by server config `posts.reactors.list.per.page` (default 1000)