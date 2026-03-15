# Cato Authorization Manual

Cato is LexCV’s in-house, object-based authorization system.

It is inspired by LDAP/ActiveDirectory group membership + ACLs, but implemented as a **tuple-based relationship system** (Zanzibar-style ReBAC) to support:

- Extremely fast permission checks
- Fine-grained per-object roles (company profile permissions, blog folder permissions, etc.)
- Complex relationships (usersets / groups-of-groups)
- Deterministic replication via Sacred Timeline

Cato is implemented as a self-contained subsystem in Go under `lib/cato/` and stores its authoritative state in MySQL (`cato_tuples`). Tuples are also mirrored to Dgraph as diagnostic/indexable nodes (`type(cato_tuple)`).

---

## Contents

1. [Concepts](#concepts)
2. [Data Model](#data-model)
3. [Tuple ID (Deterministic)](#tuple-id-deterministic)
4. [Schemas](#schemas)
5. [Replication (Sacred Timeline)](#replication-sacred-timeline)
6. [Dgraph Mirroring](#dgraph-mirroring)
7. [Go API](#go-api)
8. [Usage Patterns & Examples](#usage-patterns--examples)
9. [How Cato Plugs Into LexCV](#how-cato-plugs-into-lexcv)
10. [Operational Notes](#operational-notes)
11. [Current Limitations / Roadmap](#current-limitations--roadmap)

---

## Concepts

### Objects
Cato permissions are always granted **on an object**.

An object is identified by:

- `type` (namespace): `system`, `company`, `profile`, `blogfolder`, `post`, ...
- `id` (opaque string): typically UUIDs, but can be a stable literal like `main`

Examples:

- `system:main`
- `company:<uuid>`
- `blogfolder:<uuid>`

### Relations
A relation is the name of the **role / capability** on a resource.

Examples:

- System: `root`, `admin`, `moderator`
- Company: `jobs_editor`, `post_publisher`, `post_editor`
- Blog folder: `viewer`, `contributor`, `editor`

In the current implementation, **relations are permissions** (i.e. “relation-as-permission”).

### Subjects
A tuple grants a resource relation to a subject.

Subjects can be:

1) A direct user: `user:<uuid>`
2) A userset: `company:<uuid>#member` meaning “any user who has relation `member` on object `company:<uuid>`”

Usersets allow group membership and group nesting.

---

## Data Model

The core record is a tuple:

```
(resource_type, resource_id, relation, subject_type, subject_id, subject_relation)
```

Interpretation:

- `resource_type:resource_id#relation @ subject_type:subject_id#subject_relation`

Where:

- If `subject_type == "user"`, then `subject_relation` must be empty (`""`).
- If `subject_relation != ""`, then it denotes a userset.

---

## Tuple ID (Deterministic)

Cato uses deterministic tuple IDs so that the same tuple created on different nodes produces the same primary key.

Implementation: `lib/cato/types.go` → `TupleID(t Tuple)`.

- Canonical tuple string is built and hashed using SHA-256.
- The hex SHA-256 is stored as `tuple_id` in MySQL and as `uuid` in Dgraph.

Benefits:

- Timeline replication is naturally idempotent.
- No coordination required across nodes.
- Replays/rebuilds are stable.

---

## Schemas

### MySQL
Schema file: `schemas/cato/cato.sql`

Primary table: `cato_tuples`

Key properties:

- `tuple_id` is the primary key.
- A unique key enforces tuple uniqueness:
  - `(resource_type, resource_id, relation, subject_type, subject_id, subject_relation)`
- `deleted` is soft-delete for idempotent revoke.
- Indexed for both resource-side and subject-side lookups.

Note on dynamic object types (including groups):

- Cato does **not** require schema changes to introduce new `resource_type` values (like `group`) or new relation names.
- Tuples store `resource_type`, `resource_id`, `relation`, `subject_type`, `subject_id`, `subject_relation` as strings.
- The only hard limits are the column sizes (currently `VARCHAR(64)` for the type/id/relation fields).
### Dgraph
Dgraph schema is applied via `AdminAPI_ResetGraph` using:

- `schemas/predicates/edges.graph`
- `schemas/predicates/types.graph`

Cato adds:

- Predicates: `catoResourceType`, `catoResourceID`, `catoRelation`, `catoSubjectType`, `catoSubjectID`, `catoSubjectRelation`
- Type: `cato_tuple`

Cato tuple nodes are standalone nodes (no edges to domain objects).

---

## Replication (Sacred Timeline)

Cato tuple writes are replicated through Sacred Timeline like other subsystems.

API codes (constants) are defined in `lib/sacredtimeline/sacredtimeline.go`:

- `APICatoTupleGrant` (`"CatoTupleGrant"`)
- `APICatoTupleRevoke` (`"CatoTupleRevoke"`)

### How writes replicate

- `cato.Grant()` performs:
  1) MySQL upsert into `cato_tuples`
  2) Dgraph conditional upsert to create/update `type(cato_tuple)`
  3) Outbox insert in the same MySQL transaction

- `cato.Revoke()` performs:
  1) MySQL update `deleted=1`
  2) Dgraph upsert to mark the tuple node `deleted=1`
  3) Outbox insert in the same MySQL transaction

This matches the existing “atomic MySQL + Dgraph + outbox” pattern used elsewhere (e.g., blog folder access edges).

### Reset + replay coverage

`APPADMIN_DBRESETREPLAY` includes `cato_tuples` in the reset list, and tracks `cato_tuple` in the Dgraph type snapshot lists.

---

## Dgraph Mirroring

Cato’s authoritative state is MySQL.

Dgraph stores a mirrored copy of tuples as nodes (mainly for diagnostics and uniformity with other graph-modeled relationships).

Migration helper:

- `lib/graph/cato.go` → `graph.AddCatoTuplesToGraph()`

Graph reset integration:

- `api/admin/adminapi_resetgraph.go` includes a step to call `graph.AddCatoTuplesToGraph()`.

Important: `cato.Check()` currently reads from MySQL only.

Summary:

- **Source of truth:** MySQL (`cato_tuples`)
- **Mirror/diagnostics:** Dgraph (`type(cato_tuple)` nodes)
- **Evaluation:** MySQL-backed (today)

---

## Go API

Package: `lib/cato`

### Types

- `ObjectRef{Type, ID}`
- `SubjectRef{Type, ID, Relation}`
- `Tuple{Resource, Relation, Subject}`

### Grant

```
err := cato.Grant(cato.Tuple{ ... })
```

- Validates tuple
- Upserts into MySQL
- Mirrors to Dgraph
- Replicates via Sacred Timeline outbox

### Revoke

```
err := cato.Revoke(cato.Tuple{ ... })
```

- Soft-deletes tuple in MySQL (`deleted=1`)
- Marks Dgraph node deleted
- Replicates via Sacred Timeline outbox

### Check

```
d, err := cato.Check(userID, cato.ObjectRef{Type: "blogfolder", ID: folderUUID}, "viewer")
if err != nil {
  // handle
}

switch d.Kind {
case cato.DecisionAllow:
  // proven allow by Cato tuples alone
case cato.DecisionDeny:
  // proven deny by explicit deny tuples (or lack of any allow proof)
case cato.DecisionExternalCheck:
  // caller must evaluate d.DenyConditions (first), then d.AllowConditions
}
```

### CheckBool (strict boolean)

For call sites that require a definitive boolean (and want to treat external predicates as an error), use `cato.CheckBool()`:

```go
ok, err := cato.CheckBool(userID, cato.ObjectRef{Type: "blogfolder", ID: folderUUID}, "viewer")
```

---

### Tri-state semantics (Allow / Deny / ExternalCheck)

For policies that reference *external sets* (followers, shortlists, user_flags attributes like verified/level/type), `cato.Check()` may return `external_check`.

`Check()` returns one of three results:

1) `allow` — proven true by Cato tuples alone
2) `deny` — proven false by explicit deny tuples alone
3) `external_check` — caller must evaluate one or more external predicates

External predicates are returned as structured conditions, split into deny-first and allow-second lists.

Implementation: `lib/cato/check.go`, `lib/cato/decision.go` (and `lib/cato/evaluate.go` is a thin wrapper).

---

Evaluation rules:

- Direct tuple grants:
  - `resource#relation @ user:userID`
- Userset expansion:
  - If `resource#relation @ obj:ID#rel` then `Check(user, obj:ID, rel)` must be true.

Constraints:

- Recursion depth limit: `maxDepth = 32` (see `lib/cato/check.go`)
- Cycles are prevented with a `visited` map.

---

## Usage Patterns & Examples

### 1) System roles (Cato tuples)

Grant root on the system object:

```go
_ = cato.Grant(cato.Tuple{
  Resource: cato.ObjectRef{Type: "system", ID: "main"},
  Relation: "root",
  Subject:  cato.SubjectRef{Type: "user", ID: userUUID},
})
```

Check:

```go
isRoot, _ := cato.CheckBool(userUUID, cato.ObjectRef{Type:"system", ID:"main"}, "root")
```

### 2) Company “jobs_editor” for a specific user

```go
_ = cato.Grant(cato.Tuple{
  Resource: cato.ObjectRef{Type: "company", ID: companyUUID},
  Relation: "jobs_editor",
  Subject:  cato.SubjectRef{Type: "user", ID: userUUID},
})
```

### 3) Company role granted to a group (userset)

Make every company member a post editor:

```go
_ = cato.Grant(cato.Tuple{
  Resource: cato.ObjectRef{Type: "company", ID: companyUUID},
  Relation: "post_editor",
  Subject:  cato.SubjectRef{Type: "company", ID: companyUUID, Relation: "member"},
})
```

This means:

- `company:ID#post_editor @ company:ID#member`
- So any user who satisfies `company:ID#member @ user:<uuid>` is also a `post_editor`.

### 4) Nested groups (“admins are editors”)

```go
_ = cato.Grant(cato.Tuple{
  Resource: cato.ObjectRef{Type: "company", ID: companyUUID},
  Relation: "post_editor",
  Subject:  cato.SubjectRef{Type: "company", ID: companyUUID, Relation: "admin"},
})
```

Now `company#admin` implies `company#post_editor`.

### 4.1) Dynamic groups (create/delete at runtime)

Cato supports dynamic creation and deletion of groups without any Dgraph schema changes.

Groups are modeled as ordinary objects plus a membership relation, then referenced as a userset.

Recommended modeling:

- Group object: `group:<groupID>` where `groupID` is a stable identifier you store in MySQL (often a UUID).
- Membership relation on the group: `member` (optionally also `manager`, `owner`, etc.).

Create a group (conceptually):

- There is no separate “create group” operation in Cato. A group exists as soon as you write tuples that reference it.

Add users to the group (membership tuple):

```go
// group:<groupUUID>#member @ user:<aliceUUID>
_ = cato.Grant(cato.Tuple{
  Resource: cato.ObjectRef{Type: "group", ID: groupUUID},
  Relation: "member",
  Subject:  cato.SubjectRef{Type: "user", ID: aliceUUID},
})
```

Grant a company permission to the group (userset subject):

```go
// company:<companyUUID>#post_editor @ group:<groupUUID>#member
_ = cato.Grant(cato.Tuple{
  Resource: cato.ObjectRef{Type: "company", ID: companyUUID},
  Relation: "post_editor",
  Subject:  cato.SubjectRef{Type: "group", ID: groupUUID, Relation: "member"},
})
```

Check:

```go
ok, err := cato.CheckBool(aliceUUID, cato.ObjectRef{Type: "company", ID: companyUUID}, "post_editor")
```

Delete a group:

- Cato uses soft-delete at the tuple level (`deleted=1`).
- Deleting a group is done by revoking the tuples that define it:
  - Revoke all `group:<groupID>#member @ user:*` membership tuples.
  - Revoke all tuples that grant permissions to `group:<groupID>#member` on any resource.

Operationally, you typically do this in your domain layer (a “delete group” API) with SQL that finds the affected tuples and calls `cato.Revoke(...)` for each (so it replicates via Sacred Timeline).

### 5) Blog folder access

Grant a user viewing access:

```go
_ = cato.Grant(cato.Tuple{
  Resource: cato.ObjectRef{Type: "blogfolder", ID: folderUUID},
  Relation: "viewer",
  Subject:  cato.SubjectRef{Type: "user", ID: viewerUUID},
})
```

### 6) Blog folder access granted to “owner’s followers”

If you want “anyone who follows Alice can view Alice’s private blog folder”, Cato can represent this as a **userset-based grant**.

Important constraint in the current Cato model:

- Direct subjects are `user:<uuid>`.
- Usersets must be expressed as `subject_type:subject_id#subject_relation` where `subject_type != "user"`.

So we model a follower set as its own object type (recommended: `followerset`).

**Follower membership tuples** (maintained by follow/unfollow events):

- `followerset:<ownerUUID>#member @ user:<followerUUID>`

**Blog folder grant to the followerset**:

- `blogfolder:<folderUUID>#viewer @ followerset:<ownerUUID>#member`

Concrete code:

```go
// When Bob follows Alice:
_ = cato.Grant(cato.Tuple{
  Resource: cato.ObjectRef{Type: "followerset", ID: aliceUUID},
  Relation: "member",
  Subject:  cato.SubjectRef{Type: "user", ID: bobUUID},
})

// Make Alice's followers viewers of a folder:
_ = cato.Grant(cato.Tuple{
  Resource: cato.ObjectRef{Type: "blogfolder", ID: folderUUID},
  Relation: "viewer",
  Subject:  cato.SubjectRef{Type: "followerset", ID: aliceUUID, Relation: "member"},
})

// Check:
ok, _ := cato.Check(bobUUID, cato.ObjectRef{Type: "blogfolder", ID: folderUUID}, "viewer")
```

Why this works:

- The folder says “viewer is the followerset members”.
- The check expands the userset and validates membership via the `followerset:<ownerUUID>#member` tuples.

Performance note:

- Follower sets can be very large.
- `cato.Check()` has an indexed SQL fast-path for direct user membership (`SELECT 1 ... LIMIT 1`) so it does not need to load all follower tuples to answer “is Bob a member?”.

---

## Deny semantics (explicit deny)

Some policies require an explicit per-resource deny (blocklist) that overrides allow.

Cato encodes deny as a separate relation name:

- Allow: `resource#relation @ subject`
- Deny:  `resource#deny_<relation> @ subject`

Example: “Bob is denied viewer access to this folder even if followers are allowed”

```go
_ = cato.Grant(cato.Tuple{
  Resource: cato.ObjectRef{Type: "blogfolder", ID: folderUUID},
  Relation: "deny_viewer",
  Subject:  cato.SubjectRef{Type: "user", ID: bobUUID},
})
```

Evaluation precedence:

1) Deny (including external deny conditions) must be evaluated first.
2) Then allow.

---

## “Who has access?” (policy description)

Cato can answer “who has access” in **policy form** (sets), without enumerating all users.

Use:

```go
policy, err := cato.Policy(cato.ObjectRef{Type: "blogfolder", ID: folderUUID}, "viewer")
```

This returns:

- `allows`: subjects/usersets granted `blogfolder#viewer`
- `denies`: subjects/usersets granted `blogfolder#deny_viewer`

This is the right primitive for APIs that need to display access rules like:

- “explicit users: …”
- “followers of <uuid>”

without leaking or computing large membership expansions.

---

## How Cato Plugs Into LexCV

## Architecture Decision: Attributes vs Authorization

LexCV uses two complementary mechanisms:

- **Attributes (identity/state/tier)** stay in `users.user_flags` (and related profile fields).
  - Examples: `user_type` (lawyer/company/student/public/newuser), `user_level` (level1–level4), `user_state` (verified/banned/deactivated/etc.).
- **Authorization (who can do what to which object)** is implemented by Cato tuples.
  - Examples: system roles on `system:main`, company permissions, blog folder ACLs.

This split keeps common attribute queries fast/simple (SQL indexed lookups) and keeps Cato focused on relationship-based permissions.

### Where it lives

- Cato implementation: `lib/cato/`
- Cato MySQL schema: `schemas/cato/cato.sql`
- Cato Dgraph schema additions: `schemas/predicates/types.graph`

### How other systems should use it

Cato is intended to be called from API handlers and/or domain libraries (e.g. `lib/blog`, `lib/users`, future `lib/company`).

Pattern:

1) Existing API handler performs authentication with `CheckLoginOrFail`.
2) Apply **attribute gates** (from flags/state) when needed (e.g., banned/deactivated/verified/level gating).
3) Apply **object permission checks** using `cato.Check(...)`.
4) For permission changes, handler validates the caller is allowed to grant/revoke, then calls `cato.Grant()` / `cato.Revoke()`.

Typical enforcement structure:

- Global/state gates first (cheap and unambiguous)
- Then Cato authorization checks (object-scoped permissions)

### How it replicates

- Cato writes include an outbox insert.
- The outbox worker publishes to the Sacred Timeline topic.
- Other nodes consume and execute the replicated event, reapplying the same MySQL + Dgraph mutations.

### Reset + replay

- `APPADMIN_DBRESETREPLAY` resets `cato_tuples` and replays events, ensuring tuples rebuild deterministically.

### Graph reset

- `AdminAPI_ResetGraph` drops Dgraph, reapplies schema, repopulates nodes from MySQL, including `cato_tuple` nodes.

---

## Operational Notes

### Performance

- `Check()` is MySQL-backed and uses indexed lookups.
- Checks involving usersets recurse by reading “subjects for (resource, relation)”.
- Keep relation graphs reasonably shallow.

### Naming conventions

- Use lowercase relation names (Cato normalizes to lowercase internally).
- Keep `type` values stable; treat them as namespaces.

### Soft delete

- Revokes do not remove rows; they set `deleted=1`.
- Re-grants flip `deleted` back to `0`.

---

## Current Limitations / Roadmap

Implemented now:

- Tuple storage, deterministic IDs
- Grant/Revoke with full replication
- Check() with userset recursion

Not implemented yet (expected future work):

- “Derived permissions” (e.g., `can_edit = owner OR editor OR contributor`) as a programmable model
- List queries (e.g., “list all folders user can view”) as a first-class API
- Cache layer for ultra-hot checks (likely per-request or short-lived in-memory)
- Migration of legacy flags/roles into `system:main` tuples
