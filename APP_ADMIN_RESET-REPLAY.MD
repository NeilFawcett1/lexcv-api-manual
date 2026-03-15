# APPADMIN_DBRESETREPLAY API

Comprehensive database reset and timeline replay with full before/after verification.
Used to verify Sacred Timeline idempotency - if you reset everything and replay, you should get identical state.

## Endpoint

```
POST /api/APPADMIN_DBRESETREPLAY
```

**Authentication:** Requires root admin login (RoleRoot)

**Logging:** All operations logged to nflog with prefix `appadmin_dbresetreplay`


---

## Request Format (RObj)

```json
{
  "P1": "<username>",
  "P2": "<access_token>",
  "P3": ["<action>", "<confirmation_code>"]
}
```

| Field | Type | Required | Description |
|----|----|----|----|
| `P1` | string | Yes | Root admin username |
| `P2` | string | Yes | Access token from APPADMIN_CHECKLOGIN |
| `P3[0]` | string | Yes | Action: `"analyze"`, `"execute"`, or `"generate_confirmation"` |
| `P3[1]` | string | Only for execute | Time-based confirmation code (valid 10 minutes) |


---

## Actions

### 1. `analyze` - Read-Only State Snapshot

Returns current database state without making any changes. Use this to see what would be compared.

**Request:**

```json
{
  "P1": "admin",
  "P2": "eyJhbGciOiJIUzI1NiIs...",
  "P3": ["analyze"]
}
```

**Response:**

```json
{
  "P1": "Success",
  "P2": "Current database state analysis",
  "P3": ["<DatabaseSnapshot JSON>"]
}
```


---

### 2. `generate_confirmation` - Get Confirmation Code

Generates a time-based confirmation code required for the `execute` action.

**Request:**

```json
{
  "P1": "admin",
  "P2": "eyJhbGciOiJIUzI1NiIs...",
  "P3": ["generate_confirmation"]
}
```

**Response:**

```json
{
  "P1": "Success",
  "P2": "Confirmation code generated. Valid for 10 minutes.",
  "P3": ["{\"action\":\"db_reset_replay\",\"code\":\"A1B2C3D4\",\"expires_at\":\"2025-12-23T10:10:00Z\",\"warning\":\"This is a DESTRUCTIVE operation...\"}"]
}
```


---

### 3. `execute` - Full Reset and Replay ⚠️ DESTRUCTIVE

Performs complete database reset and timeline replay with before/after verification.

**Request:**

```json
{
  "P1": "admin",
  "P2": "eyJhbGciOiJIUzI1NiIs...",
  "P3": ["execute", "A1B2C3D4"]
}
```

**Response:**

```json
{
  "P1": "Success",
  "P2": "Reset and replay completed. Verdict: PASS - All data matches",
  "P3": ["<ResetReplayResult JSON>"]
}
```


---

## Response Structures

### DatabaseSnapshot

Returned by `analyze` action and included in `execute` result as `before_state` and `after_state`.

```json
{
  "collected_at": "2025-12-23T10:00:00.123Z",
  
  "mysql_tables": [
    {
      "table_name": "users",
      "row_count": 1523,
      "data_size_kb": 256,
      "index_size_kb": 64
    },
    {
      "table_name": "postsindex",
      "row_count": 8432,
      "data_size_kb": 1024,
      "index_size_kb": 512
    }
  ],
  "mysql_total_rows": 45678,
  "mysql_total_data_kb": 12345,
  
  "dgraph_health": "healthy",
  "dgraph_types": [
    { "type_name": "users", "node_count": 1523 },
    { "type_name": "posts", "node_count": 8432 },
    { "type_name": "nations", "node_count": 195 },
    { "type_name": "countries", "node_count": 250 },
    { "type_name": "courts", "node_count": 1234 }
  ],
  "dgraph_edges": [
    { "edge_name": "userFollows", "edge_count": 5678 },
    { "edge_name": "postAuthor", "edge_count": 8432 },
    { "edge_name": "countryInNation", "edge_count": 250 },
    { "edge_name": "lawyerInCountry", "edge_count": 1200 }
  ],
  "dgraph_total_nodes": 15234,
  "dgraph_total_edges": 45678,
  
  "timeline_events": 123456,
  "timeline_watermarks": {
    "0": 41234,
    "1": 41235,
    "2": 41230
  }
}
```

### ResetReplayResult

Returned by `execute` action.

```json
{
  "status": "success",
  "message": "Reset and replay completed",
  "started_at": "2025-12-23T10:00:00.000Z",
  "completed_at": "2025-12-23T10:05:23.456Z",
  "duration": "5m23.456s",
  
  "before_state": { /* DatabaseSnapshot */ },
  "after_state": { /* DatabaseSnapshot */ },
  
  "mysql_analysis": [
    {
      "table_name": "users",
      "before_count": 1523,
      "after_count": 1523,
      "difference": 0,
      "status": "match"
    },
    {
      "table_name": "postsindex",
      "before_count": 8432,
      "after_count": 8430,
      "difference": -2,
      "status": "mismatch"
    }
  ],
  
  "dgraph_analysis": [
    {
      "table_name": "users",
      "before_count": 1523,
      "after_count": 1523,
      "difference": 0,
      "status": "match"
    },
    {
      "table_name": "userFollows",
      "before_count": 5678,
      "after_count": 5678,
      "difference": 0,
      "status": "match"
    }
  ],
  
  "replay_stats": {
    "partitions_processed": 3,
    "total_events_replayed": 123456,
    "events_skipped": 0,
    "errors": [],
    "duration": "4m58.123s"
  },
  
  "verdict": "PASS - All 45 MySQL tables and 30 DGraph types/edges match exactly"
}
```

### ComparisonResult

Used in `mysql_analysis` and `dgraph_analysis` arrays.

```json
{
  "table_name": "string",
  "before_count": 1523,
  "after_count": 1523,
  "difference": 0,
  "status": "match | mismatch | new | missing"
}
```

| Status | Meaning |
|----|----|
| `match` | Before and after counts are identical ✅ |
| `mismatch` | Counts differ (difference shows delta) ❌ |
| `new` | Table/type exists after but not before |
| `missing` | Table/type existed before but not after |


---

## Verdict Interpretation

| Verdict | Meaning |
|----|----|
| `PASS - All X MySQL tables and Y DGraph types/edges match exactly` | Timeline replay is idempotent ✅ |
| `FAIL - MySQL mismatches: users(-1), posts(+3)` | Data loss or duplication ❌ |
| `FAIL - DGraph mismatches: userFollows(-5)` | Graph relationships lost ❌ |


---

## Execution Steps (logged to nflog)

The `execute` action performs these steps in order:

**Pre-checks (CRITICAL SAFETY):**


1. **Verify outbox is empty** - If there are pending events in outbox, they haven't been published to Redpanda yet. Resetting would lose them permanently!
2. **Stop consumer** - The normal consumer (started at server boot) must be stopped to prevent race conditions during reset

**Reset steps:**


1. **Collect BEFORE state** - Snapshot all MySQL row counts, DGraph nodes/edges, watermarks
2. **Reset MySQL tables** - TRUNCATE user data tables (with FK checks disabled). **Static lookup tables are NOT reset!**
3. **Reset watermarks** - TRUNCATE watermarks table so replay starts from offset 0
4. **Reset DGraph** - DROP_ALL, recreate schema, insert base data (nations/countries/courts/etc)
5. **Re-initialize Sacred Timeline** - Reconnect producer after Close()
6. **Replay Sacred Timeline (FORCE mode)** - Full replay from offset 0 using `ReconcileAllPartitionsForce()` which executes ALL events including those originally from this node
7. **Collect AFTER state** - Same snapshot as step 1
8. **Restart consumer and outbox worker** - Resume normal operation
9. **Analyze differences** - Compare before/after, generate verdict

**IMPORTANT: Static Lookup Tables Are PRESERVED**

The following tables are **NOT reset** because they are populated by `countriestablescript.php`, not by Sacred Timeline events:

* `nationsindex`, `countriesindex`, `geoareasindex`
* `courtsindex`, `courtservescountry`
* `lawyertypeindex`, `worktypeindex`
* `organisationtypes`, `professionaltypes`

Resetting these would cause **permanent data loss** that cannot be recovered from timeline replay!

**Thread Safety:** `forceReplayAll` flag uses `sync/atomic.Bool` to prevent race conditions.


---

## Tables Reset

### MySQL Tables RESET (user data - recreated from timeline)

**Messaging:**

* `message_attachments`, `message_receipts`, `messages`
* `conversation_participants`, `conversations`
* `user_keys`, `user_blocks`

**Posts:**

* `bookmarks`, `usertimelines`, `nationtimelines`
* `commentsindex`, `postsindex`

**Profiles:**

* `casesindex`, `educindex`, `publicationsindex`
* `experienceindex`, `highlightsindex`, `recommendsindex`
* `shortlistindex`, `blogkeywords`, `blogindex`, `blogfolders`
* `mediatickets`, `media_tokens`, `aliasindex`, `locations`
* `userprofessionaltypes`, `userorganisationtypes`
* `users`

**Timeline:**

* `sacred_timeline`, `sacred_timeline_outbox`, `sacred_timeline_watermarks`

### MySQL Tables PRESERVED (static lookup data - NOT from timeline)

* `nationsindex`, `countriesindex`, `geoareasindex`
* `courtsindex`, `courtservescountry`
* `lawyertypeindex`, `worktypeindex`
* `organisationtypes`, `professionaltypes`

### DGraph Types Tracked

* `users`, `nations`, `countries`, `geoareas`, `courts`
* `lawyertype`, `worktype`, `organisationtype`, `professionaltype`, `location`
* `posts`, `comment`, `blog`, `blogfolders`, `blogs`
* `conversation`, `message`

### DGraph Edges Tracked

* **Geographic:** `countryInNation`, `areaInCountry`, `courtInCountry`, `courtServesCountry`, `lawyerTypeInCountry`
* **Lawyer profile:** `lawyerIsType`, `lawyerInCountry`, `lawyerInArea`, `lawyerDoesWorkType`, `lawyerUsesCourt`
* **Company/Professional:** `companyIsType`, `companyInCountry`, `companyInArea`, `companyDoesWorkType`, `companyAtLocation`, `professionalIsType`, `professionalInCountry`, `professionalInArea`
* **Social:** `userFollows`, `userShortlists`, `userBlocks`
* **Posts:** `postAuthor`, `commentAuthor`, `commentOnPost`, `commentParent`, `reactedBy`, `repostOf`, `bestAnswer`
* **Blogs:** `blogAuthor`, `folderOwner`, `folderContainsBlog`, `userCanAccessFolder`
* **Messaging:** `userInConversation`, `conversationHasUser`, `messageInConversation`, `messageAuthor`


---

## Error Responses

```json
{
  "P1": "Fail",
  "P2": "<error message>",
  "P3": []
}
```

| P2 Message | Cause |
|----|----|
| `Root access required` | User doesn't have RoleRoot |
| `Reset+replay requires confirmation...` | Missing or invalid confirmation code |
| `ABORTED: Outbox has N pending events...` | Outbox not empty - events would be lost. Wait for drain. |
| `Unknown action: xyz` | Invalid action parameter |
| `Database not initialized` | MySQL connection not available |
| `DGraph reset failed: ...` | Failed to reset DGraph schema |
| `Failed to re-initialize Sacred Timeline...` | Could not reconnect to Redpanda after reset |


---

## Rust Client Example

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

#[derive(Serialize)]
struct RObj {
    #[serde(rename = "P1")]
    p1: String,
    #[serde(rename = "P2")]
    p2: String,
    #[serde(rename = "P3")]
    p3: Vec<String>,
}

#[derive(Deserialize)]
struct RResponse {
    #[serde(rename = "P1")]
    p1: String,
    #[serde(rename = "P2")]
    p2: String,
    #[serde(rename = "P3")]
    p3: Vec<String>,
}

#[derive(Deserialize)]
struct DatabaseSnapshot {
    collected_at: String,
    mysql_tables: Vec<MySQLTableSnapshot>,
    mysql_total_rows: i64,
    mysql_total_data_kb: i64,
    dgraph_health: String,
    dgraph_types: Vec<DGraphTypeSnapshot>,
    dgraph_edges: Vec<DGraphEdgeSnapshot>,
    dgraph_total_nodes: i64,
    dgraph_total_edges: i64,
    timeline_events: i64,
    timeline_watermarks: HashMap<String, i64>,
}

#[derive(Deserialize)]
struct MySQLTableSnapshot {
    table_name: String,
    row_count: i64,
    data_size_kb: i64,
    index_size_kb: i64,
}

#[derive(Deserialize)]
struct DGraphTypeSnapshot {
    type_name: String,
    node_count: i64,
}

#[derive(Deserialize)]
struct DGraphEdgeSnapshot {
    edge_name: String,
    edge_count: i64,
}

#[derive(Deserialize)]
struct ComparisonResult {
    table_name: String,
    before_count: i64,
    after_count: i64,
    difference: i64,
    status: String,
}

#[derive(Deserialize)]
struct ResetReplayResult {
    status: String,
    message: String,
    started_at: String,
    completed_at: String,
    duration: String,
    before_state: DatabaseSnapshot,
    after_state: DatabaseSnapshot,
    mysql_analysis: Vec<ComparisonResult>,
    dgraph_analysis: Vec<ComparisonResult>,
    verdict: String,
}

#[derive(Deserialize)]
struct ConfirmationInfo {
    action: String,
    code: String,
    expires_at: String,
    warning: String,
}

async fn reset_replay_workflow(
    client: &reqwest::Client,
    base_url: &str,
    username: &str,
    token: &str,
) -> Result<ResetReplayResult, Box<dyn std::error::Error>> {
    
    // Step 1: Analyze current state (optional, for preview)
    let analyze_req = RObj {
        p1: username.to_string(),
        p2: token.to_string(),
        p3: vec!["analyze".to_string()],
    };
    
    let analyze_resp: RResponse = client
        .post(format!("{}/api/APPADMIN_DBRESETREPLAY", base_url))
        .json(&analyze_req)
        .send()
        .await?
        .json()
        .await?;
    
    if analyze_resp.p1 != "Success" {
        return Err(format!("Analyze failed: {}", analyze_resp.p2).into());
    }
    
    let snapshot: DatabaseSnapshot = serde_json::from_str(&analyze_resp.p3[0])?;
    println!("Current state: {} MySQL rows, {} DGraph nodes", 
             snapshot.mysql_total_rows, snapshot.dgraph_total_nodes);
    
    // Step 2: Get confirmation code
    let confirm_req = RObj {
        p1: username.to_string(),
        p2: token.to_string(),
        p3: vec!["generate_confirmation".to_string()],
    };
    
    let confirm_resp: RResponse = client
        .post(format!("{}/api/APPADMIN_DBRESETREPLAY", base_url))
        .json(&confirm_req)
        .send()
        .await?
        .json()
        .await?;
    
    if confirm_resp.p1 != "Success" {
        return Err(format!("Confirmation failed: {}", confirm_resp.p2).into());
    }
    
    let confirm_info: ConfirmationInfo = serde_json::from_str(&confirm_resp.p3[0])?;
    println!("Got confirmation code: {} (expires: {})", confirm_info.code, confirm_info.expires_at);
    
    // Step 3: Execute reset+replay
    let execute_req = RObj {
        p1: username.to_string(),
        p2: token.to_string(),
        p3: vec!["execute".to_string(), confirm_info.code],
    };
    
    let execute_resp: RResponse = client
        .post(format!("{}/api/APPADMIN_DBRESETREPLAY", base_url))
        .json(&execute_req)
        .send()
        .await?
        .json()
        .await?;
    
    if execute_resp.p1 != "Success" {
        return Err(format!("Execute failed: {}", execute_resp.p2).into());
    }
    
    let result: ResetReplayResult = serde_json::from_str(&execute_resp.p3[0])?;
    println!("Verdict: {}", result.verdict);
    
    Ok(result)
}
```


---

## nflog Output Example

```
[appadmin_dbresetreplay] ========================================
[appadmin_dbresetreplay] STARTING DATABASE RESET AND REPLAY
[appadmin_dbresetreplay] ========================================
[appadmin_dbresetreplay] Pre-check: Verifying outbox is empty...
[appadmin_dbresetreplay] Outbox is clear
[appadmin_dbresetreplay] Pre-check: Stopping Sacred Timeline consumer...
[sacredtimeline] Sacred Timeline closed
[appadmin_dbresetreplay] Consumer stopped
[appadmin_dbresetreplay] Step 1: Collecting BEFORE state...
[appadmin_dbresetreplay] Before: MySQL 45678 rows, DGraph 15234 nodes/45678 edges
[appadmin_dbresetreplay] Step 2: Resetting MySQL tables...
[appadmin_dbresetreplay] MySQL tables reset complete
[appadmin_dbresetreplay] NOTE: Static lookup tables preserved: [nationsindex countriesindex geoareasindex courtsindex courtservescountry lawyertypeindex worktypeindex organisationtypes professionaltypes]
[appadmin_dbresetreplay] Step 3: Resetting watermarks...
[appadmin_dbresetreplay] Watermarks reset
[appadmin_dbresetreplay] Step 4: Resetting DGraph...
[appadmin_dbresetreplay] DGraph reset complete
[appadmin_dbresetreplay] Step 5: Re-initializing Sacred Timeline...
[sacredtimeline] Initialized with brokers: [redpanda:9092]
[appadmin_dbresetreplay] Sacred Timeline re-initialized
[appadmin_dbresetreplay] Step 6: Replaying Sacred Timeline from beginning (FORCE mode - all events including own)...
[sacredtimeline::ReconcileAllPartitionsForce] ========================================
[sacredtimeline::ReconcileAllPartitionsForce] FORCE RECONCILIATION - REPLAYING ALL EVENTS INCLUDING OWN
[sacredtimeline::ReconcileAllPartitionsForce] ========================================
[sacredtimeline::CatchupFromWatermarks] Partition 0 is 41234 events behind...
[sacredtimeline::CatchupFromWatermarks] Partition 1 is 41235 events behind...
[sacredtimeline::CatchupFromWatermarks] Partition 2 is 41230 events behind...
[sacredtimeline::ReconcileAllPartitionsForce] Force reconciliation complete in 4m58s - replayed 123699 events
[sacredtimeline::ReconcileAllPartitionsForce] ========================================
[appadmin_dbresetreplay] Replay complete: 123699 replayed, 0 skipped, duration: 4m58s
[appadmin_dbresetreplay] Step 7: Collecting AFTER state...
[appadmin_dbresetreplay] After: MySQL 45678 rows, DGraph 15234 nodes/45678 edges
[appadmin_dbresetreplay] Step 8: Restarting consumer and outbox worker...
[sacredtimeline] Starting consumer group: lexcv-belgium-consumer
[sacredtimeline::outbox] Starting outbox worker
[appadmin_dbresetreplay] Consumer and outbox worker restarted
[appadmin_dbresetreplay] Step 9: Analyzing differences...
[appadmin_dbresetreplay] ========================================
[appadmin_dbresetreplay] RESET AND REPLAY COMPLETE
[appadmin_dbresetreplay] Verdict: PASS - All data restored correctly from timeline replay
[appadmin_dbresetreplay] Duration: 5m23.456s
[appadmin_dbresetreplay] ========================================
```


