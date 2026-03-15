# Jobs APIs

All endpoints are `POST` to `/api/API_*` and use the standard `RObj` envelope.

## Common

### Data encoding conventions
This API uses the `RObj` envelope where many numeric values are sent as **strings**.

- **Epoch timestamps**: all `*_at_ms` and `*_deadline_ms` fields are **milliseconds since Unix epoch**.
	- Encoded as a base-10 string (e.g. `"1736236800000"`).
	- `"0"` means "not set" in responses.
- **Integers**: encoded as base-10 strings unless otherwise noted.
- **UUIDs**: canonical UUID strings.
- **JSON fields**: `benefits` and `meta` are JSON-encoded strings.

### Login
- `p1`: user uuid
- `p2`: login token

### Pagination
- `offset`: 0-based
- `limit`: server default varies by endpoint

### Permissions (Cato tuples)
Resource: `company:<company_uuid>`
- `jobs_admin`: full job lifecycle + delegation
- `jobs_recruiter`: can manage candidates (accept/reject) and view applicants
- `jobs_viewer`: can view applicants

Note: `jobs_admin` implies recruiter+viewer; `jobs_recruiter` implies viewer.

### Recommended Flutter UI flows

#### Public company jobs page (viewer is any logged-in user)
- List jobs: call `API_JobsListByCompany` (returns OPEN jobs only).
- View job details: call `API_JobsGet`.
	- For `open|closed` jobs: any logged-in user can view.
	- For `draft` jobs: only company self UUID or delegated jobs roles can view.

#### Company job management (company self UUID OR delegated roles)
Use this for the “Manage jobs” screen (drafts + open + closed).

- List all jobs (draft/open/closed): call `API_JobsListByCompanyAdmin`.
- Create draft job: call `API_JobsCreate` (requires `jobs_admin`).
- Edit draft/open job fields: call `API_JobsUpdate` (requires `jobs_admin`).
- Publish: call `API_JobsPublish` (requires `jobs_admin`).
- Close: call `API_JobsClose` (requires `jobs_admin`).
- Extend deadline: call `API_JobsExtendDeadline` (requires `jobs_admin`).
- Delete: call `API_JobsDelete` (requires `jobs_admin`).

#### Applicant management (company self UUID OR delegated roles)
- View applicants for a job: call `API_JobsListApplicants` (requires `jobs_viewer`).
- View immutable applicant submission snapshot (JSON + frozen media): call `API_JobsGetApplicationPackage` (requires `jobs_viewer`).
- Accept/reject applicants: call `API_JobsSetApplicantStatus` (requires `jobs_recruiter`).
- Add a private note: call `API_JobsApplicationNoteAdd` (requires `jobs_admin` or `jobs_recruiter`).
- Edit/delete a note: call `API_JobsApplicationNoteEdit` / `API_JobsApplicationNoteDelete` (see role rules below).

#### Delegation management (company self UUID OR delegated admins)
Use this for “Team access” UI.

- Grant a role: call `API_JobsDelegateGrant` (requires `jobs_admin`).
- Revoke a role: call `API_JobsDelegateRevoke` (requires `jobs_admin`).
- List delegates: call `API_JobsDelegateList` or `API_JobsDelegateListDetailed` (requires `jobs_admin`).
- Search delegates (existing only): call `API_JobsDelegateSearch` (requires `jobs_admin`).
- Search all users to add a new delegate: call `API_SearchAuto`, then pass the selected `user_uuid` to `API_JobsDelegateGrant`.

#### Applicant self-service
- Apply: call `API_JobsApply` (returns `application_uuid`).
- Withdraw: call `API_JobsWithdrawApplication` (only the applicant can withdraw their own application).

### Job record mapping in responses
Jobs are returned in `p6[]` as `RObj` items with:
- `p3[0]`: job_uuid
- `p3[1]`: company_uuid
- `p3[2]`: title
- `p3[3]`: status (`draft|open|closed|deleted`)
- `p3[4]`: published_at_ms (epoch ms string)
- `p3[5]`: apply_deadline_ms (epoch ms string)
- `p3[6]`: closed_at_ms (epoch ms string)
- `p3[7]`: created_at_ms (epoch ms string)
- `p3[8]`: updated_at_ms (epoch ms string)

Job `p4` is structured so the client can render the owning company without extra calls:
- `p4[0][0]`: description (empty string for list endpoints; populated for `API_JobsGet`)
- `p4[1]`: company UI: `[display_name, headline, avatar_url, avatar_token, banner_url, banner_token]`
- `p4[2][0]`: company flags JSON (human-readable, e.g. `{"user_state":["verified"],"user_type":["company"],...}`)
- `p4[3]`: company messaging: `[message_keys_initialized (0/1), allow_messages (all|followers|none)]`

Job `p5` contains extended job fields as key/value pairs:
- `p5[0]`: list of rows `[key, value]` (each is a string)
- Keys and types:
	- `reference_code` (string, max 64)
	- `employment_type` (int string)
	- `workplace_type` (int string)
	- `location_uuid` (uuid string; empty means NULL)
	- `location_text` (string, max 300)
	- `salary_min` (int64 string; `"0"` means NULL)
	- `salary_max` (int64 string; `"0"` means NULL)
	- `salary_currency` (ISO 4217 code, string; empty if invalid)
	- `salary_period` (int string)
	- `hours_per_week` (int string; `"0"` means NULL)
	- `benefits` (JSON string; defaults to `[]` when empty)
	- `meta` (JSON string; defaults to `{}` when empty)

Enums:
- `employment_type`: `0=unspecified, 1=full_time, 2=part_time, 3=contract, 4=internship, 5=temporary`
- `workplace_type`: `0=unspecified, 1=onsite, 2=hybrid, 3=remote`
- `salary_period`: `0=unspecified, 1=hour, 2=day, 3=week, 4=month, 5=year`

### Application record mapping in responses
Applications are returned in `p6[]` as `RObj` items with:
- `p3[0]`: application_uuid
- `p3[1]`: job_uuid
- `p3[2]`: company_uuid
- `p3[3]`: applicant_uuid
- `p3[4]`: status (`applied|withdrawn|rejected|accepted`)

Applicant-provided contact fields:
- `p3[5]`: contact_email
- `p3[6]`: contact_phone

Timestamps:
- `p3[7]`: created_at_ms
- `p3[8]`: updated_at_ms

Workflow / messaging fields:
- `p3[9]`: provisional_status (`""|accepted|rejected`) (staged decision; company-only)
- `p3[10]`: status_message (per-applicant custom message accompanying final status; may be empty)
- `p3[11]`: bulk_status_message (bulk/general message set on commit; may be empty)
- `p3[12]`: provisional_status_message (company-only while sifting; applicant-visible only after commit)

Note: provisional fields (`p3[9]` and `p3[12]`) are only returned to company/delegate endpoints
(e.g. `API_JobsListApplicants`). Applicant endpoints do not expose them.

Cover letter (when included):
- `p4[0][0]`: cover_letter

Immutable snapshot artifacts:
- Application list/get endpoints do NOT embed the immutable CV/profile snapshot bytes.
- Company/delegates should call `API_JobsGetApplicationPackage` to retrieve signed URL+token pairs for `profile.json`, `avatar`, `banner`, and `bundle.zip`.

---

## Flutter Integration Guide (Agent Explainer)

This section is a complete implementation reference for building a Flutter client against the Jobs API. It covers Dart models, parsing helpers, every API call pattern, CDN media access, and full UI workflow sequences. An AI coding agent can use this section to generate a fully working integration without guessing.

### How API calls work

Every call is a JSON `POST` to `https://<host>/api/<API_NAME>`. The request and response both use the `RObj` envelope. Auth credentials (`p1` = user UUID, `p2` = server token) are injected automatically by the `ApiClient` singleton.

```dart
// Pattern for every API call:
final res = await api.call('API_JobsGet', p3: [jobUuid]);
if (res.p1 != 'Success') throw Exception(res.p2);
// Parse res.p3, res.p4, res.p5, res.p6 as documented per-endpoint below.
```

All fields are **strings**. Ints are `"123"`, bools are `"0"`/`"1"`, timestamps are epoch-ms strings like `"1736236800000"`, and `"0"` means "not set" for optional timestamps.

### Dart models

Below are the recommended Dart model classes. They map directly to the wire format documented in "Job record mapping" and "Application record mapping" above.

#### Job model (parsed from `p6[]` items)

```dart
class Job {
  final String uuid;
  final String companyUuid;
  final String title;
  final String status; // draft|open|closed|deleted
  final int publishedAt; // epoch ms, 0 = not set
  final int applyDeadline; // epoch ms, 0 = not set
  final int closedAt; // epoch ms, 0 = not set
  final int createdAt; // epoch ms
  final int updatedAt; // epoch ms

  // Description (populated by API_JobsGet; empty in list endpoints)
  final String description;

  // Owning company (embedded in p4 so no extra API call needed)
  final CompanyUI company;

  // Extended fields from p5[0] key/value pairs
  final JobExtras extras;

  // Only present in API_JobsGet — the requesting user's own application status
  final MyApplicationStatus? myApplicationStatus;

  Job({
    required this.uuid,
    required this.companyUuid,
    required this.title,
    required this.status,
    required this.publishedAt,
    required this.applyDeadline,
    required this.closedAt,
    required this.createdAt,
    required this.updatedAt,
    required this.description,
    required this.company,
    required this.extras,
    this.myApplicationStatus,
  });

  bool get isOpen => status == 'open';
  bool get isDraft => status == 'draft';
  bool get isClosed => status == 'closed';
  bool get isDeleted => status == 'deleted';
  bool get hasDeadline => applyDeadline > 0;
  bool get deadlinePassed => hasDeadline && applyDeadline < DateTime.now().millisecondsSinceEpoch;
}
```

#### CompanyUI model (parsed from job `p4[1..3]`)

```dart
class CompanyUI {
  final String displayName;
  final String headline;
  final String avatarUrl; // CDN signed URL (may be empty)
  final String avatarToken; // x-lexcv-token header value for CDN
  final String bannerUrl;
  final String bannerToken;
  final Map<String, dynamic> flags; // decoded from flags JSON
  final bool keysInitialized; // message encryption keys ready
  final String allowMessages; // "all"|"following"|"none"

  CompanyUI({
    required this.displayName,
    required this.headline,
    required this.avatarUrl,
    required this.avatarToken,
    required this.bannerUrl,
    required this.bannerToken,
    required this.flags,
    required this.keysInitialized,
    required this.allowMessages,
  });
}
```

#### JobExtras model (parsed from `p5[0]` key/value pairs)

```dart
class JobExtras {
  final String referenceCode;
  final int employmentType; // 0=unspecified,1=full_time,2=part_time,3=contract,4=internship,5=temporary
  final int workplaceType; // 0=unspecified,1=onsite,2=hybrid,3=remote
  final String locationUuid; // empty = not set
  final String locationText;
  final int salaryMin; // 0 = not set
  final int salaryMax; // 0 = not set
  final String salaryCurrency; // ISO 4217, e.g. "EUR", "GBP", "USD"; empty = not set
  final int salaryPeriod; // 0=unspecified,1=hour,2=day,3=week,4=month,5=year
  final int hoursPerWeek; // 0 = not set
  final List<dynamic> benefits; // decoded from JSON string, defaults to []
  final Map<String, dynamic> meta; // decoded from JSON string, defaults to {}

  JobExtras({
    required this.referenceCode,
    required this.employmentType,
    required this.workplaceType,
    required this.locationUuid,
    required this.locationText,
    required this.salaryMin,
    required this.salaryMax,
    required this.salaryCurrency,
    required this.salaryPeriod,
    required this.hoursPerWeek,
    required this.benefits,
    required this.meta,
  });

  String get employmentTypeLabel => const {
    0: 'Unspecified', 1: 'Full-time', 2: 'Part-time',
    3: 'Contract', 4: 'Internship', 5: 'Temporary',
  }[employmentType] ?? 'Unspecified';

  String get workplaceTypeLabel => const {
    0: 'Unspecified', 1: 'On-site', 2: 'Hybrid', 3: 'Remote',
  }[workplaceType] ?? 'Unspecified';

  String get salaryPeriodLabel => const {
    0: '', 1: '/hr', 2: '/day', 3: '/wk', 4: '/mo', 5: '/yr',
  }[salaryPeriod] ?? '';

  bool get hasSalary => salaryMin > 0 || salaryMax > 0;
}
```

#### MyApplicationStatus (only in API_JobsGet response, parsed from `p5[1]`)

```dart
class MyApplicationStatus {
  final bool hasApplied;
  final String applicationUuid; // empty if not applied
  final String status; // applied|withdrawn|rejected|accepted; empty if not applied

  MyApplicationStatus({
    required this.hasApplied,
    required this.applicationUuid,
    required this.status,
  });
}
```

#### JobApplication model (parsed from `p6[]` in API_JobsListApplicants)

```dart
class JobApplication {
  final String uuid;
  final String jobUuid;
  final String companyUuid;
  final String applicantUuid;
  final String status; // applied|withdrawn|rejected|accepted
  final String contactEmail;
  final String contactPhone;
  final int createdAt;
  final int updatedAt;
  final String provisionalStatus; // ""|accepted|rejected
  final String statusMessage;
  final String bulkStatusMessage;
  final String provisionalStatusMessage;

  // Applicant profile (embedded in p4[0])
  final ApplicantUI applicant;

  // Cover letter (from p4[1][0])
  final String coverLetter;

  JobApplication({
    required this.uuid,
    required this.jobUuid,
    required this.companyUuid,
    required this.applicantUuid,
    required this.status,
    required this.contactEmail,
    required this.contactPhone,
    required this.createdAt,
    required this.updatedAt,
    required this.provisionalStatus,
    required this.statusMessage,
    required this.bulkStatusMessage,
    required this.provisionalStatusMessage,
    required this.applicant,
    required this.coverLetter,
  });

  bool get isApplied => status == 'applied';
  bool get isWithdrawn => status == 'withdrawn';
  bool get isRejected => status == 'rejected';
  bool get isAccepted => status == 'accepted';
  bool get hasProvisionalDecision => provisionalStatus.isNotEmpty;
}
```

#### ApplicantUI model (parsed from application `p4[0]`)

```dart
class ApplicantUI {
  final String uuid;
  final String displayName;
  final String headline;
  final String avatarUrl;
  final String avatarToken;
  final Map<String, dynamic> flags;
  final bool keysInitialized;
  final String allowMessages;

  ApplicantUI({
    required this.uuid,
    required this.displayName,
    required this.headline,
    required this.avatarUrl,
    required this.avatarToken,
    required this.flags,
    required this.keysInitialized,
    required this.allowMessages,
  });
}
```

#### ApplicationPackage model (parsed from API_JobsGetApplicationPackage response)

```dart
class ApplicationPackage {
  final String applicationUuid;
  final String jobUuid;
  final String companyUuid;
  final String applicantUuid;
  final String contactEmail;
  final String contactPhone;
  final String coverLetter;
  final int snapshotVersion;
  final String snapshotState; // ready|pending|failed
  final int createdAt;
  final int updatedAt;

  // Signed CDN URLs for immutable snapshot media
  final MediaPair profileJson; // the full CV as JSON
  final MediaPair avatar; // frozen avatar at apply-time
  final MediaPair banner; // frozen banner at apply-time
  final MediaPair bundleZip; // profile.json + avatar + banner in one zip

  // SHA-256 hashes for integrity verification
  final String profileJsonSha256;
  final String avatarSha256;
  final String bannerSha256;
  final String bundleZipSha256;

  ApplicationPackage({
    required this.applicationUuid,
    required this.jobUuid,
    required this.companyUuid,
    required this.applicantUuid,
    required this.contactEmail,
    required this.contactPhone,
    required this.coverLetter,
    required this.snapshotVersion,
    required this.snapshotState,
    required this.createdAt,
    required this.updatedAt,
    required this.profileJson,
    required this.avatar,
    required this.banner,
    required this.bundleZip,
    required this.profileJsonSha256,
    required this.avatarSha256,
    required this.bannerSha256,
    required this.bundleZipSha256,
  });
}

class MediaPair {
  final String url;
  final String token;
  bool get isEmpty => url.isEmpty;
  const MediaPair({required this.url, required this.token});
}
```

### Parsing helpers

#### Parse a Job from an `RObj` item in `p6[]`

```dart
Job parseJob(RObj item) {
  final p3 = item.p3;
  final p4 = item.p4;
  final p5 = item.p5;

  // --- p3: core job fields ---
  final uuid = p3[0];
  final companyUuid = p3[1];
  final title = p3[2];
  final status = p3[3];
  final publishedAt = int.tryParse(p3[4]) ?? 0;
  final applyDeadline = int.tryParse(p3[5]) ?? 0;
  final closedAt = int.tryParse(p3[6]) ?? 0;
  final createdAt = int.tryParse(p3[7]) ?? 0;
  final updatedAt = int.tryParse(p3[8]) ?? 0;

  // --- p4[0][0]: description (empty in list endpoints) ---
  final description = (p4.isNotEmpty && p4[0].isNotEmpty) ? p4[0][0] : '';

  // --- p4[1..3]: owning company UI ---
  final companyRow = (p4.length > 1) ? p4[1] : <String>[];
  final flagsJson = (p4.length > 2 && p4[2].isNotEmpty) ? p4[2][0] : '{}';
  final msgRow = (p4.length > 3) ? p4[3] : <String>[];

  final company = CompanyUI(
    displayName: _safeIdx(companyRow, 0),
    headline: _safeIdx(companyRow, 1),
    avatarUrl: _safeIdx(companyRow, 2),
    avatarToken: _safeIdx(companyRow, 3),
    bannerUrl: _safeIdx(companyRow, 4),
    bannerToken: _safeIdx(companyRow, 5),
    flags: _tryJsonDecode(flagsJson),
    keysInitialized: _safeIdx(msgRow, 0) == '1',
    allowMessages: _safeIdx(msgRow, 1, fallback: 'all'),
  );

  // --- p5[0]: extended job fields as key/value pairs ---
  final extras = _parseJobExtras(p5.isNotEmpty ? p5[0] : []);

  // --- p5[1]: my application status (only from API_JobsGet) ---
  MyApplicationStatus? myStatus;
  if (p5.length > 1) {
    final kv = _kvMap(p5[1]);
    myStatus = MyApplicationStatus(
      hasApplied: kv['has_applied'] == '1',
      applicationUuid: kv['application_uuid'] ?? '',
      status: kv['status'] ?? '',
    );
  }

  return Job(
    uuid: uuid,
    companyUuid: companyUuid,
    title: title,
    status: status,
    publishedAt: publishedAt,
    applyDeadline: applyDeadline,
    closedAt: closedAt,
    createdAt: createdAt,
    updatedAt: updatedAt,
    description: description,
    company: company,
    extras: extras,
    myApplicationStatus: myStatus,
  );
}
```

#### Parse a JobApplication from an `RObj` item in `p6[]`

```dart
JobApplication parseApplication(RObj item) {
  final p3 = item.p3;
  final p4 = item.p4;

  // --- p3: application fields ---
  final uuid = p3[0];
  final jobUuid = p3[1];
  final companyUuid = p3[2];
  final applicantUuid = p3[3];
  final status = p3[4];
  final contactEmail = p3[5];
  final contactPhone = p3[6];
  final createdAt = int.tryParse(p3[7]) ?? 0;
  final updatedAt = int.tryParse(p3[8]) ?? 0;
  final provisionalStatus = _safeIdx(p3, 9);
  final statusMessage = _safeIdx(p3, 10);
  final bulkStatusMessage = _safeIdx(p3, 11);
  final provisionalStatusMessage = _safeIdx(p3, 12);

  // --- p4[0]: applicant profile (canonical search format) ---
  final aRow = (p4.isNotEmpty) ? p4[0] : <String>[];
  final applicant = ApplicantUI(
    uuid: _safeIdx(aRow, 0),
    displayName: _safeIdx(aRow, 1),
    headline: _safeIdx(aRow, 2),
    avatarUrl: _safeIdx(aRow, 3),
    avatarToken: _safeIdx(aRow, 10),
    flags: _tryJsonDecode(_safeIdx(aRow, 4, fallback: '{}')),
    keysInitialized: _safeIdx(aRow, 11) == '1',
    allowMessages: _safeIdx(aRow, 12, fallback: 'all'),
  );

  // --- p4[1][0]: cover letter ---
  final coverLetter = (p4.length > 1 && p4[1].isNotEmpty) ? p4[1][0] : '';

  return JobApplication(
    uuid: uuid,
    jobUuid: jobUuid,
    companyUuid: companyUuid,
    applicantUuid: applicantUuid,
    status: status,
    contactEmail: contactEmail,
    contactPhone: contactPhone,
    createdAt: createdAt,
    updatedAt: updatedAt,
    provisionalStatus: provisionalStatus,
    statusMessage: statusMessage,
    bulkStatusMessage: bulkStatusMessage,
    provisionalStatusMessage: provisionalStatusMessage,
    applicant: applicant,
    coverLetter: coverLetter,
  );
}
```

#### Parse an ApplicationPackage from API_JobsGetApplicationPackage response

```dart
ApplicationPackage parseApplicationPackage(RObj item) {
  final p3 = item.p3;
  final p4 = item.p4;
  final p5 = item.p5;

  final hashes = (p5.isNotEmpty) ? _kvMap(p5[0]) : <String, String>{};

  return ApplicationPackage(
    applicationUuid: p3[0],
    jobUuid: p3[1],
    companyUuid: p3[2],
    applicantUuid: p3[3],
    contactEmail: p3[4],
    contactPhone: p3[5],
    coverLetter: p3[6],
    snapshotVersion: int.tryParse(p3[7]) ?? 1,
    snapshotState: p3[8],
    createdAt: int.tryParse(p3[9]) ?? 0,
    updatedAt: int.tryParse(p3[10]) ?? 0,
    profileJson: MediaPair(url: _safeIdx2d(p4, 0, 0), token: _safeIdx2d(p4, 0, 1)),
    avatar: MediaPair(url: _safeIdx2d(p4, 1, 0), token: _safeIdx2d(p4, 1, 1)),
    banner: MediaPair(url: _safeIdx2d(p4, 2, 0), token: _safeIdx2d(p4, 2, 1)),
    bundleZip: MediaPair(url: _safeIdx2d(p4, 3, 0), token: _safeIdx2d(p4, 3, 1)),
    profileJsonSha256: hashes['profile_json_sha256'] ?? '',
    avatarSha256: hashes['avatar_sha256'] ?? '',
    bannerSha256: hashes['banner_sha256'] ?? '',
    bundleZipSha256: hashes['bundle_zip_sha256'] ?? '',
  );
}
```

#### Parse API_JobsListMyApplications — each item is a Job + application extras

```dart
/// Items returned by API_JobsListMyApplications are full job RObj items with
/// application metadata appended in p5[1] and cover letter appended in p4[last].
class MyApplicationItem {
  final Job job;
  final Map<String, String> applicationExtras; // application_uuid, status, etc.
  final String coverLetter;

  MyApplicationItem({
    required this.job,
    required this.applicationExtras,
    required this.coverLetter,
  });
}

MyApplicationItem parseMyApplicationItem(RObj item) {
  final job = parseJob(item);

  // p5[1] = application metadata key/value pairs
  final appKV = (item.p5.length > 1) ? _kvMap(item.p5[1]) : <String, String>{};

  // p4[last] = [cover_letter] (appended after company UI rows)
  final coverLetter = (item.p4.length > 4 && item.p4.last.isNotEmpty)
      ? item.p4.last[0]
      : '';

  return MyApplicationItem(job: job, applicationExtras: appKV, coverLetter: coverLetter);
}
```

#### Shared utility functions

```dart
String _safeIdx(List<String> list, int i, {String fallback = ''}) =>
    (i < list.length) ? list[i] : fallback;

String _safeIdx2d(List<List<String>> list, int row, int col) =>
    (row < list.length && col < list[row].length) ? list[row][col] : '';

Map<String, dynamic> _tryJsonDecode(String s) {
  try { return jsonDecode(s) as Map<String, dynamic>; } catch (_) { return {}; }
}

/// Converts a list of [key, value] rows into a flat map.
Map<String, String> _kvMap(List<List<String>> rows) {
  final m = <String, String>{};
  for (final row in rows) {
    if (row.length >= 2) m[row[0]] = row[1];
  }
  return m;
}

JobExtras _parseJobExtras(List<List<String>> rows) {
  final kv = _kvMap(rows);
  return JobExtras(
    referenceCode: kv['reference_code'] ?? '',
    employmentType: int.tryParse(kv['employment_type'] ?? '') ?? 0,
    workplaceType: int.tryParse(kv['workplace_type'] ?? '') ?? 0,
    locationUuid: kv['location_uuid'] ?? '',
    locationText: kv['location_text'] ?? '',
    salaryMin: int.tryParse(kv['salary_min'] ?? '') ?? 0,
    salaryMax: int.tryParse(kv['salary_max'] ?? '') ?? 0,
    salaryCurrency: kv['salary_currency'] ?? '',
    salaryPeriod: int.tryParse(kv['salary_period'] ?? '') ?? 0,
    hoursPerWeek: int.tryParse(kv['hours_per_week'] ?? '') ?? 0,
    benefits: _tryJsonDecode(kv['benefits'] ?? '[]') is List
        ? _tryJsonDecode(kv['benefits'] ?? '[]') as List
        : [],
    meta: _tryJsonDecode(kv['meta'] ?? '{}'),
  );
}
```

### API call recipes — every endpoint

Below is the exact Dart code for calling every Jobs endpoint, showing what to send and how to parse the response. The `api.call()` method auto-injects `p1`/`p2` login credentials.

#### Job lifecycle (company admin)

```dart
// Create a job (returns job UUID)
Future<String> createJob(String companyUuid, String title, String description,
    {int applyDeadlineMs = 0, Map<String, String>? extras}) async {
  final res = await api.call('API_JobsCreate',
    p3: [companyUuid, title, applyDeadlineMs > 0 ? '$applyDeadlineMs' : ''],
    p4: [[description]],
    p5: extras != null ? [extras.entries.map((e) => [e.key, e.value]).toList()] : [],
  );
  if (res.p1 != 'Success') throw Exception(res.p2);
  return res.p2; // job UUID
}

// Update a job (partial — only non-empty fields are updated)
Future<void> updateJob(String companyUuid, String jobUuid,
    {String? title, String? description, int? applyDeadlineMs,
     Map<String, String>? extras}) async {
  final res = await api.call('API_JobsUpdate',
    p3: [companyUuid, jobUuid, title ?? '', applyDeadlineMs != null ? '$applyDeadlineMs' : ''],
    p4: description != null ? [[description]] : [],
    p5: extras != null ? [extras.entries.map((e) => [e.key, e.value]).toList()] : [],
  );
  if (res.p1 != 'Success') throw Exception(res.p2);
}

// Publish, close, delete, extend deadline
Future<void> publishJob(String companyUuid, String jobUuid) async {
  final res = await api.call('API_JobsPublish', p3: [companyUuid, jobUuid]);
  if (res.p1 != 'Success') throw Exception(res.p2);
}

Future<void> closeJob(String companyUuid, String jobUuid) async {
  final res = await api.call('API_JobsClose', p3: [companyUuid, jobUuid]);
  if (res.p1 != 'Success') throw Exception(res.p2);
}

Future<void> deleteJob(String companyUuid, String jobUuid) async {
  final res = await api.call('API_JobsDelete', p3: [companyUuid, jobUuid]);
  if (res.p1 != 'Success') throw Exception(res.p2);
}

Future<void> extendDeadline(String companyUuid, String jobUuid, int newDeadlineMs) async {
  final res = await api.call('API_JobsExtendDeadline',
    p3: [companyUuid, jobUuid, '$newDeadlineMs']);
  if (res.p1 != 'Success') throw Exception(res.p2);
}
```

#### Job viewing / listing

```dart
// Get a single job (includes description + my application status)
Future<Job> getJob(String jobUuid) async {
  final res = await api.call('API_JobsGet', p3: [jobUuid]);
  if (res.p1 != 'Success') throw Exception(res.p2);
  return parseJob(res.p6[0]);
}

// List OPEN jobs for a company (public view)
Future<List<Job>> listJobsByCompany(String companyUuid, {int offset = 0, int limit = 20}) async {
  final res = await api.call('API_JobsListByCompany',
    p3: [companyUuid, '$offset', '$limit']);
  if (res.p1 != 'Success') throw Exception(res.p2);
  return res.p6.map(parseJob).toList();
}

// List ALL jobs for company admin (draft+open+closed)
Future<List<Job>> listJobsByCompanyAdmin(String companyUuid, {int offset = 0, int limit = 20}) async {
  final res = await api.call('API_JobsListByCompanyAdmin',
    p3: [companyUuid, '$offset', '$limit']);
  if (res.p1 != 'Success') throw Exception(res.p2);
  return res.p6.map(parseJob).toList();
}

// List OPEN jobs across all nations the user belongs to (feed)
Future<List<Job>> listAllJobs({int offset = 0, int limit = 20}) async {
  final res = await api.call('API_JobsListAll', p3: ['$offset', '$limit']);
  if (res.p1 != 'Success') throw Exception(res.p2);
  return res.p6.map(parseJob).toList();
}

// List OPEN jobs by nation
Future<List<Job>> listJobsByNation(int nationId, {int offset = 0, int limit = 20}) async {
  final res = await api.call('API_JobsListByNation', p3: ['$nationId', '$offset', '$limit']);
  if (res.p1 != 'Success') throw Exception(res.p2);
  return res.p6.map(parseJob).toList();
}

// Search jobs by company
Future<List<Job>> searchJobsByCompany(String companyUuid, String query,
    {int offset = 0, int limit = 20}) async {
  final res = await api.call('API_JobsSearchByCompany',
    p3: [companyUuid, query, '$offset', '$limit']);
  if (res.p1 != 'Success') throw Exception(res.p2);
  return res.p6.map(parseJob).toList();
}
```

#### Applicant self-service

```dart
// Apply to a job (returns application UUID)
Future<String> applyToJob(String jobUuid, String contactEmail,
    {String contactPhone = '', String coverLetter = ''}) async {
  final res = await api.call('API_JobsApply',
    p3: [jobUuid, contactEmail, contactPhone, coverLetter]);
  if (res.p1 != 'Success') throw Exception(res.p2);
  return res.p2; // application UUID
}

// Withdraw an application
Future<void> withdrawApplication(String applicationUuid) async {
  final res = await api.call('API_JobsWithdrawApplication', p3: [applicationUuid]);
  if (res.p1 != 'Success') throw Exception(res.p2);
}

// List my applications (applicant's "My Applications" screen)
Future<List<MyApplicationItem>> listMyApplications({int offset = 0, int limit = 20}) async {
  final res = await api.call('API_JobsListMyApplications', p3: ['$offset', '$limit']);
  if (res.p1 != 'Success') throw Exception(res.p2);
  return res.p6.map(parseMyApplicationItem).toList();
}
```

#### Applicant management (company/delegates)

```dart
// List applicants for a job
Future<List<JobApplication>> listApplicants(String companyUuid, String jobUuid,
    {int offset = 0, int limit = 50}) async {
  final res = await api.call('API_JobsListApplicants',
    p3: [companyUuid, jobUuid, '$offset', '$limit']);
  if (res.p1 != 'Success') throw Exception(res.p2);
  return res.p6.map(parseApplication).toList();
}

// Accept or reject an applicant (immediate)
Future<void> setApplicantStatus(String companyUuid, String applicationUuid,
    String newStatus, {String note = '', String statusMessage = ''}) async {
  final res = await api.call('API_JobsSetApplicantStatus',
    p3: [companyUuid, applicationUuid, newStatus, note, statusMessage]);
  if (res.p1 != 'Success') throw Exception(res.p2);
}

// Stage a provisional (sift) decision
Future<void> setProvisionalStatus(String companyUuid, String applicationUuid,
    String provisionalStatus, {String note = '', String provisionalMessage = ''}) async {
  final res = await api.call('API_JobsSetApplicantProvisionalStatus',
    p3: [companyUuid, applicationUuid, provisionalStatus, note, provisionalMessage]);
  if (res.p1 != 'Success') throw Exception(res.p2);
}

// Commit all provisional decisions for a job (bulk "notify all")
Future<int> commitProvisionalStatuses(String companyUuid, String jobUuid,
    {String acceptedMessage = '', String rejectedMessage = '', String note = ''}) async {
  final res = await api.call('API_JobsCommitProvisionalStatuses',
    p3: [companyUuid, jobUuid, acceptedMessage, rejectedMessage, note]);
  if (res.p1 != 'Success') throw Exception(res.p2);
  return int.tryParse(res.p2) ?? 0; // updated count
}
```

#### Application snapshot (immutable submission package)

```dart
// Get the immutable application package (profile JSON + frozen media)
Future<ApplicationPackage> getApplicationPackage(
    String companyUuid, String applicationUuid) async {
  final res = await api.call('API_JobsGetApplicationPackage',
    p3: [companyUuid, applicationUuid]);
  if (res.p1 != 'Success') throw Exception(res.p2);
  return parseApplicationPackage(res.p6[0]);
}
```

**How to display snapshot media in Flutter:**

```dart
// The URLs are CDN-signed and require the x-lexcv-token header.
// Use the same pattern as avatar/banner display elsewhere in the app:

Widget buildSnapshotAvatar(ApplicationPackage pkg) {
  if (pkg.avatar.isEmpty) return const Icon(Icons.person);
  return Image.network(
    pkg.avatar.url,
    headers: {'x-lexcv-token': pkg.avatar.token},
  );
}

// To download and parse the profile JSON snapshot:
Future<Map<String, dynamic>> fetchProfileSnapshot(ApplicationPackage pkg) async {
  final response = await http.get(
    Uri.parse(pkg.profileJson.url),
    headers: {'x-lexcv-token': pkg.profileJson.token},
  );
  return jsonDecode(response.body) as Map<String, dynamic>;
  // Returns: { "applicant_uuid", "taken_at", "display_name", "headline",
  //            "flags_json", "profile_groups": [[[...],...],...]  }
}

// The profile_groups field contains the full lawyer CV in the same
// format used by API_ProfileLawyer (groups → sections → fields).
```

**Snapshot profile JSON structure (returned by `profile.json`):**

`profile.json` includes the full PeerProfile payload so the client can
render the complete profile without any live API calls. The structure is:

```json
{
  "applicant_uuid": "<uuid>",
  "taken_at": 1770389968621,
  "display_name": "Mary Johnson",
  "headline": "International Trade and Commercial Litigation Expert",
  "flags_json": "{\"user_type\":[\"lawyer\"],\"user_level\":[\"level1\"],\"user_state\":[\"verified\"]}",
  "profile_groups": [
    // Same structure as API_ProfileLawyer groups
  ],
  "peerprofile": {
    "sL": [
      // PeerProfile scalar list (same as API_PeerProfile p3)
    ],
    "oL": [
      // PeerProfile grouped lists (same as API_PeerProfile p5)
    ]
  }
}
```

Notes:
- `peerprofile.sL` matches `API_PeerProfile` `p3` (scalar fields).
- `peerprofile.oL` matches `API_PeerProfile` `p5` (grouped lists).
- `profile_groups` remains the raw lawyer profile groups used by the legacy CV UI.

#### Private notes (company-only)

Role rules:
- Company self UUID and `jobs_admin` can add/edit/delete any note.
- `jobs_recruiter` can add notes, but can only edit/delete notes created by recruiters.
- `jobs_viewer` cannot add/edit/delete notes.

**API: Add note** (`API_JobsApplicationNoteAdd`)

Request:
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: application_uuid
- `p3[2]`: note

Response:
- `p1`: `Success|Fail`
- `p2`: note_uuid (success) or error

**API: List notes** (`API_JobsApplicationNotesList`)

Request:
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: application_uuid
- `p3[2]`: offset (optional)
- `p3[3]`: limit (optional)

Response:
- `p1`: `Success|Fail`
- `p3[0]`: count
- `p4`: rows in this order:
  1) `uuid`
  2) `application_uuid`
  3) `company_uuid`
  4) `actor_uuid`
  5) `note`
  6) `created_at`
  7) `updated_at`
  8) `deleted`
  9) `actor_display_name`
  10) `actor_headline`
  11) `actor_avatar_url`
  12) `actor_avatar_token`
  13) `actor_flags_json`
  14) `actor_keys_initialized`
  15) `actor_allow_messages`

**API: Edit note** (`API_JobsApplicationNoteEdit`)

Request:
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: note_uuid
- `p3[2]`: note

Response:
- `p1`: `Success|Fail`
- `p2`: error (if Fail)

**API: Delete note** (`API_JobsApplicationNoteDelete`)

Request:
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: note_uuid

Response:
- `p1`: `Success|Fail`
- `p2`: error (if Fail)

```dart
// Add a private note to an application
Future<String> addApplicationNote(
    String companyUuid, String applicationUuid, String note) async {
  final res = await api.call('API_JobsApplicationNoteAdd',
    p3: [companyUuid, applicationUuid, note]);
  if (res.p1 != 'Success') throw Exception(res.p2);
  return res.p2; // note UUID
}

// List private notes for an application
Future<List<List<String>>> listApplicationNotes(
    String companyUuid, String applicationUuid,
    {int offset = 0, int limit = 50}) async {
  final res = await api.call('API_JobsApplicationNotesList',
    p3: [companyUuid, applicationUuid, '$offset', '$limit']);
  if (res.p1 != 'Success') throw Exception(res.p2);
  return res.p4; // rows include actor display info appended after deleted
}

// Edit a private note
Future<void> editApplicationNote(
    String companyUuid, String noteUuid, String note) async {
  final res = await api.call('API_JobsApplicationNoteEdit',
    p3: [companyUuid, noteUuid, note]);
  if (res.p1 != 'Success') throw Exception(res.p2);
}

// Delete a private note
Future<void> deleteApplicationNote(
    String companyUuid, String noteUuid) async {
  final res = await api.call('API_JobsApplicationNoteDelete',
    p3: [companyUuid, noteUuid]);
  if (res.p1 != 'Success') throw Exception(res.p2);
}
```

#### Delegation management

```dart
// Grant a role to a user
Future<void> delegateGrant(String companyUuid, String userUuid, String role) async {
  // role must be: "jobs_admin", "jobs_recruiter", or "jobs_viewer"
  final res = await api.call('API_JobsDelegateGrant', p3: [companyUuid, userUuid, role]);
  if (res.p1 != 'Success') throw Exception(res.p2);
}

// Revoke a role
Future<void> delegateRevoke(String companyUuid, String userUuid, String role) async {
  final res = await api.call('API_JobsDelegateRevoke', p3: [companyUuid, userUuid, role]);
  if (res.p1 != 'Success') throw Exception(res.p2);
}

// List delegates (simple — returns CSV UUIDs)
Future<({List<String> admins, List<String> recruiters, List<String> viewers})>
    listDelegates(String companyUuid) async {
  final res = await api.call('API_JobsDelegateList', p3: [companyUuid]);
  if (res.p1 != 'Success') throw Exception(res.p2);
  return (
    admins: res.p3[0].split(',').where((s) => s.isNotEmpty).toList(),
    recruiters: res.p3[1].split(',').where((s) => s.isNotEmpty).toList(),
    viewers: res.p3[2].split(',').where((s) => s.isNotEmpty).toList(),
  );
}

// List delegates with details (for management UI)
Future<({int total, List<List<String>> rows})> listDelegatesDetailed(
    String companyUuid, {String role = '', int offset = 0, int limit = 50, String query = ''}) async {
  final res = await api.call('API_JobsDelegateListDetailed',
    p3: [companyUuid, role, '$offset', '$limit', query]);
  if (res.p1 != 'Success') throw Exception(res.p2);
  return (total: int.tryParse(res.p3[0]) ?? 0, rows: res.p4);
  // Each row: [user_uuid, role, uname, name, surname, firmname]
}

// Search delegates
Future<List<List<String>>> searchDelegates(
    String companyUuid, String query, {String role = '', int maxResults = 10}) async {
  final res = await api.call('API_JobsDelegateSearch',
    p3: [companyUuid, role, query, '$maxResults']);
  if (res.p1 != 'Success') throw Exception(res.p2);
  return res.p4; // SearchAuto-style rows; first column is user uuid
}
```

### Sending extended job fields (P5) when creating or updating a job

When calling `API_JobsCreate` or `API_JobsUpdate`, extended fields are sent as `p5[0]` — a list of `[key, value]` pairs. Only include the keys you want to set; omitted keys are left unchanged on update.

```dart
// Example: create a full-time remote job with salary range
await createJob(
  companyUuid,
  'Senior Flutter Developer',
  'We are looking for an experienced Flutter developer...',
  applyDeadlineMs: DateTime(2026, 3, 1).millisecondsSinceEpoch,
  extras: {
    'reference_code': 'FLT-2026-001',
    'employment_type': '1',  // full_time
    'workplace_type': '3',   // remote
    'location_text': 'London, UK',
    'salary_min': '80000',
    'salary_max': '120000',
    'salary_currency': 'GBP',
    'salary_period': '5',    // year
    'hours_per_week': '40',
    'benefits': '["health_insurance","pension","remote_work"]',
    'meta': '{"department":"engineering","team":"mobile"}',
  },
);
```

### Complete UI workflow sequences

#### Workflow 1: Company posts a job and manages applicants

```
1. createJob(companyUuid, title, desc, extras: {...})    → jobUuid (draft)
2. updateJob(companyUuid, jobUuid, extras: {...})        → refine fields
3. publishJob(companyUuid, jobUuid)                       → status=open, visible to users
4. ... applicants apply ...
5. listApplicants(companyUuid, jobUuid)                   → List<JobApplication>
6. getApplicationPackage(companyUuid, appUuid)             → immutable snapshot + media
7. setProvisionalStatus(companyUuid, appUuid, 'accepted') → stage decision
8. setProvisionalStatus(companyUuid, appUuid2, 'rejected')→ stage decision
9. commitProvisionalStatuses(companyUuid, jobUuid,
     acceptedMessage: 'Congrats!',
     rejectedMessage: 'Thank you for applying.')          → bulk notify all
10. closeJob(companyUuid, jobUuid)                         → status=closed
```

#### Workflow 2: User browses and applies for jobs

```
1. listAllJobs()                                     → jobs feed (open jobs in user's nations)
   OR listJobsByCompany(companyUuid)                  → company's open jobs
   OR listJobsByNation(nationId)                      → nation feed
2. getJob(jobUuid)                                    → full details + myApplicationStatus
3. Check job.myApplicationStatus?.hasApplied
   - If false: show "Apply" button
   - If true: show status badge (applied/accepted/rejected/withdrawn)
4. applyToJob(jobUuid, email, phone: phone,
     coverLetter: letter)                             → applicationUuid
5. listMyApplications()                               → "My Applications" screen
6. withdrawApplication(applicationUuid)               → if user changes their mind
```

#### Workflow 3: Delegation management

```
1. listDelegatesDetailed(companyUuid)                 → current team with details
2. delegateGrant(companyUuid, userUuid, 'jobs_recruiter') → add a recruiter
3. delegateRevoke(companyUuid, userUuid, 'jobs_viewer')   → remove access
4. searchDelegates(companyUuid, 'john')               → find existing delegates by name
```

### Understanding the immutable submission snapshot

When a user calls `API_JobsApply`, the server automatically:

1. Captures the applicant's **full lawyer profile** (all CV groups/sections/fields) as `profile.json`
2. Downloads the applicant's **current avatar** and **banner** images from CDN storage as raw bytes
3. Bundles everything into a `bundle.zip` containing `profile.json` + `avatar.{ext}` + `banner.{ext}`
4. Computes **SHA-256 hashes** of each artifact for tamper detection
5. Stores all artifacts in GCS under the **company's** media namespace (not the applicant's)
6. Creates `media_tokens` entries owned by the company UUID

**Key properties:**
- The snapshot is **immutable** — it cannot be changed after submission
- It is **company-owned** — the company retains access even if the applicant deletes their account
- It is **independent** of the live profile — later profile edits do not affect the snapshot
- Each artifact has a **SHA-256 hash** for integrity verification

**The `profile.json` structure** (when fetched via the signed URL):
```json
{
  "applicant_uuid": "abc123...",
  "taken_at": 1736236800000,
  "display_name": "Jane Smith",
  "headline": "Senior Lawyer | Corporate & M&A",
  "flags_json": "{\"user_state\":[\"verified\"],\"user_type\":[\"lawyer\"]}",
  "profile_groups": [
    [
      ["group_name", "value"],
      ["field1", "value1"],
      ...
    ],
    ...
  ]
}
```

The `profile_groups` array uses the same `[][][]string` structure as `API_ProfileLawyer` — it contains the applicant's complete CV organized by groups and sections.

### Error handling reference

All endpoints return `p1 = "Fail"` with `p2` containing the error message. Common error strings:

| Error (`p2`) | Cause | Action |
|---|---|---|
| `missing parameters` | Required P3 fields not provided | Check call signature |
| `permission denied` | Caller lacks the required Cato role | Verify delegation setup |
| `jobs admin required` | Caller is not `jobs_admin` | Need admin delegation |
| `jobs recruiter required` | Caller is not `jobs_recruiter` or higher | Need recruiter delegation |
| `jobs viewer required` | Caller is not `jobs_viewer` or higher | Need viewer delegation |
| `already applied` | Applicant already has a non-withdrawn application for this job | Show "already applied" UI; user must withdraw first to re-apply |
| `companies cannot apply for jobs` | Company account tried to apply | Only individual accounts can apply |
| `contact email is required` | Empty email on apply | Validate before calling |
| `invalid email format` | Email didn't pass regex validation | Validate before calling |
| `title required` | Empty title on create/update | Validate before calling |
| `description required` | Empty description on create | Validate before calling |
| `invalid status` | SetApplicantStatus called with something other than `accepted` or `rejected` | Only these two are valid |
| `invalid provisional status` | SetProvisionalStatus called with invalid value | Only `accepted`, `rejected`, or `""` (clear) |
| `job lookup failed` | Job UUID doesn't exist or is deleted | Refresh job list |
| `application lookup failed` | Application UUID doesn't exist for this company | Refresh applicant list |
| `invalid benefits json` | `benefits` field is not valid JSON | Send valid JSON array |
| `invalid meta json` | `meta` field is not valid JSON | Send valid JSON object |

### CDN media access pattern

All CDN-signed URLs returned by the API (avatar, banner, snapshot media) require the `x-lexcv-token` header. The token is a short-lived signature.

```dart
// Generic CDN image widget used throughout the app:
Widget cdnImage(String url, String token, {double? width, double? height}) {
  if (url.isEmpty) return const SizedBox.shrink();
  return Image.network(
    url,
    width: width,
    height: height,
    headers: {'x-lexcv-token': token},
    errorBuilder: (_, __, ___) => const Icon(Icons.broken_image),
  );
}
```

---

## Endpoints

### API_JobsCreate
Creates a draft job for a company.

**Auth:** `jobs_admin` on company

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: title
- `p3[2]`: apply_deadline_ms (epoch ms string; optional)
- `p4[0][0]`: description
- `p5[0]`: optional job fields as key/value pairs (see "Job record mapping")

**Response**
- `p1`: `Success|Fail`
- `p2`: job_uuid (success) or error message

### API_JobsUpdate
Edits job fields.

**Auth:** `jobs_admin` on company

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: job_uuid
- `p3[2]`: title (optional)
- `p3[3]`: apply_deadline_ms (epoch ms string; optional)
- `p4[0][0]`: description (optional)
- `p5[0]`: optional job fields as key/value pairs (see "Job record mapping")

**Response**
- `p1`: `Success|Fail`
- `p2`: `updated` or error

### API_JobsPublish
Sets status to `open` and publishes.

**Auth:** `jobs_admin` on company

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: job_uuid

**Response**
- `p1`: `Success|Fail`
- `p2`: `published`

### API_JobsClose
Sets status to `closed`.

**Auth:** `jobs_admin` on company

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: job_uuid

**Response**
- `p1`: `Success|Fail`
- `p2`: `closed`

### API_JobsExtendDeadline
Updates the application deadline.

**Auth:** `jobs_admin` on company

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: job_uuid
- `p3[2]`: new_deadline_ms

**Response**
- `p1`: `Success|Fail`
- `p2`: `extended`

### API_JobsDelete
Soft deletes a job.

**Auth:** `jobs_admin` on company

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: job_uuid

**Response**
- `p1`: `Success|Fail`
- `p2`: `deleted`

### API_JobsGet
Gets a single job by uuid.

**Auth:** login required.
- For `open|closed`: any logged-in user can view.
- For `draft`: company self UUID OR (`jobs_viewer` / `jobs_recruiter` / `jobs_admin`) on company.

**Request**
- `p1/p2`: login
- `p3[0]`: job_uuid

**Response**
- `p1`: `Success|Fail`
- `p6[0]`: job RObj (includes description)

Note: job `p5[0]` extras are included in the job RObj.

Additionally, `API_JobsGet` ALWAYS includes the requesting user's application status for this job in `p5[1]`:
- `has_applied`: `0|1`
- `application_uuid`: empty string if not applied
- `status`: `applied|withdrawn|rejected|accepted` (empty string if not applied)

### API_JobsListAll
Lists OPEN jobs for companies in any nation linked to the requesting user.

This is analogous to a “nation timeline” feed but is computed purely at runtime via a Dgraph traversal:
user → (lawyerInCountry, companyInCountry) → country → nation → countries → companies → jobs.

**Auth:** login required

**Request**
- `p1/p2`: login
- `p3[0]`: offset (optional)
- `p3[1]`: limit (optional)

**Response**
- `p1`: `Success|Fail`
- `p6[]`: job list (no description; includes `p5` extras)

### API_JobsListByCompany
Lists OPEN jobs for a company.

**Auth:** login required

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: offset (optional)
- `p3[2]`: limit (optional)

**Response**
- `p1`: `Success|Fail`
- `p6[]`: job list (no description; includes `p5` extras)

### API_JobsListByCompanyAdmin
Lists ALL jobs for a company (draft/open/closed). Intended for company job management screens.

**Auth:** company self UUID OR (`jobs_viewer` / `jobs_recruiter` / `jobs_admin`) on company

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: offset (optional)
- `p3[2]`: limit (optional)

**Response**
- `p1`: `Success|Fail`
- `p6[]`: job list (includes `p5` extras)

### API_JobsListByNation
Lists jobs for a nation via graph traversal (nation → countries → companies → jobs).

Note: this returns OPEN jobs only.

**Auth:** login required

**Request**
- `p1/p2`: login
- `p3[0]`: nation_id
- `p3[1]`: offset (optional)
- `p3[2]`: limit (optional)

**Response**
- `p1`: `Success|Fail`
- `p6[]`: job list (no description; includes `p5` extras)

### API_JobsSearchByCompany
Searches jobs for a company.

Note: this searches OPEN jobs only.

**Auth:** login required

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: query
- `p3[2]`: offset (optional)
- `p3[3]`: limit (optional)

**Response**
- `p1`: `Success|Fail`
- `p6[]`: job list (includes `p5` extras)

### API_JobsApply
Applies to a job.

**Auth:** login required

**Request**
- `p1/p2`: login
- `p3[0]`: job_uuid
- `p3[1]`: contact_email (required)
- `p3[2]`: contact_phone (optional)
- `p3[3]`: cover_letter (optional)

**Response**
- `p1`: `Success|Fail`
- `p2`: application_uuid

**Notes**
- Company accounts cannot apply for jobs.
- If the applicant has already applied for the same `(job_uuid, applicant)` and the application is not withdrawn, the API returns `Fail` with message `already applied`.
- To change contact details or cover letter after applying, the applicant must call `API_JobsWithdrawApplication` and then apply again.

### API_JobsWithdrawApplication
Withdraws an application.

**Auth:** login required (must be applicant)

**Request**
- `p1/p2`: login
- `p3[0]`: application_uuid

**Response**
- `p1`: `Success|Fail`
- `p2`: `withdrawn`

### API_JobsListMyApplications
Lists job applications created by the logged-in user (paged, for infinite scroll).

**Auth:** login required

**Request**
- `p1/p2`: login
- `p3[0]`: offset (optional)
- `p3[1]`: limit (optional)

**Response**
- `p1`: `Success|Fail`
- `p6[]`: list of job objects (canonical job structure; includes job description)
	- Paging/ordering:
		- ordered by application `created_at` descending
		- `offset` default 0
		- `limit` default 20 (max 200)
	- Deleted jobs:
		- applications are still returned even if the job was deleted
		- the returned job object will have status `deleted`
	- Each job item is an `RObj` containing:
		- `p3[]`: job fields
			- `p3[0]`: job_uuid
			- `p3[1]`: company_uuid
			- `p3[2]`: title
			- `p3[3]`: status (`draft|open|closed|deleted`)
			- `p3[4]`: published_at_ms
			- `p3[5]`: apply_deadline_ms
			- `p3[6]`: closed_at_ms
			- `p3[7]`: created_at_ms
			- `p3[8]`: updated_at_ms
		- `p4[]`: embedded company + job description
			- `p4[0][0]`: description
			- `p4[1][]`: company row
				- `p4[1][0]`: display_name
				- `p4[1][1]`: headline
				- `p4[1][2]`: avatar_url (CDN signed URL)
				- `p4[1][3]`: avatar_token
				- `p4[1][4]`: banner_url (CDN signed URL)
				- `p4[1][5]`: banner_token
			- `p4[2][0]`: flags_json (human-readable public JSON)
			- `p4[3][]`: messaging prefs
				- `p4[3][0]`: message_keys_initialized (`0|1`)
				- `p4[3][1]`: allow_messages
			- `p4[last][0]`: cover_letter
		- `p5[]`: extras arrays
			- `p5[0][]`: job extras key/value pairs (same as other jobs endpoints)
			- `p5[1][]`: application metadata key/value pairs
				- `application_uuid`
				- `job_uuid`
				- `company_uuid`
				- `applicant_uuid`
				- `status`
				- `status_message`
				- `bulk_status_message`
				- `contact_email`
				- `contact_phone`
				- `created_at`
				- `updated_at`

### API_JobsListApplicants
Lists applications for a job.

**Auth:** `jobs_viewer` on company

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: job_uuid
- `p3[2]`: offset (optional)
- `p3[3]`: limit (optional)

**Response**
- `p1`: `Success|Fail`
- `p6[]`: application list
	- Each list item is an `RObj`:
		- `p3[]`: application metadata
			- `p3[0]`: application_uuid
			- `p3[1]`: job_uuid
			- `p3[2]`: company_uuid
			- `p3[3]`: applicant_uuid
			- `p3[4]`: status
			- `p3[5]`: contact_email
			- `p3[6]`: contact_phone
			- `p3[7]`: created_at_ms
			- `p3[8]`: updated_at_ms
			- `p3[9]`: provisional_status (`""|accepted|rejected`)
			- `p3[10]`: status_message (applicant-visible; may be empty)
			- `p3[11]`: bulk_status_message (applicant-visible bulk message; may be empty)
			- `p3[12]`: provisional_status_message (company-only; applicant-visible only after commit)
		- `p4[0][]`: applicant profile row (canonical search format)
			- `p4[0][0]`: applicant_uuid
			- `p4[0][1]`: display_name
			- `p4[0][2]`: headline
			- `p4[0][3]`: avatar_url
			- `p4[0][4]`: flags_json (human-readable public JSON)
			- `p4[0][5]`: position (currently `0`)
			- `p4[0][6]`: follow_by_viewer (currently `0`)
			- `p4[0][7]`: follows_viewer (currently `0`)
			- `p4[0][8]`: shortlist_flag (currently `0`)
			- `p4[0][9]`: shortlist_rank (currently `0`)
			- `p4[0][10]`: avatar_token
			- `p4[0][11]`: message_keys_initialized (`0|1`)
			- `p4[0][12]`: allow_messages
		- `p4[1][0]`: cover_letter

### API_JobsGetApplicationPackage
Returns the immutable, company-owned application package snapshot (JSON + frozen media access).

This is the compliance/audit-safe "what they submitted" payload; it is not affected by later profile edits or media deletes.

**Auth:** `jobs_viewer` on company

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: application_uuid

**Response**
- `p1`: `Success|Fail`
- `p6[0]`: package object
	- `p3[]`: package fields
		- `p3[0]`: application_uuid
		- `p3[1]`: job_uuid
		- `p3[2]`: company_uuid
		- `p3[3]`: applicant_uuid (informational)
		- `p3[4]`: contact_email
		- `p3[5]`: contact_phone
		- `p3[6]`: cover_letter
		- `p3[7]`: snapshot_version (int string)
		- `p3[8]`: snapshot_state
		- `p3[9]`: created_at_ms
		- `p3[10]`: updated_at_ms
	- `p4[]`: signed URL + token pairs
		- `p4[0]`: `[profile_json_url, profile_json_token]`
		- `p4[1]`: `[avatar_url, avatar_token]` (may be empty)
		- `p4[2]`: `[banner_url, banner_token]` (may be empty)
		- `p4[3]`: `[bundle_zip_url, bundle_zip_token]`
	- `p5[0]`: hashes key/value pairs
		- `profile_json_sha256`
		- `avatar_sha256`
		- `banner_sha256`
		- `bundle_zip_sha256`

### API_JobsSetApplicantProvisionalStatus
Sets a provisional (staged) decision for an applicant without changing the actual application status.

Use this while sifting applicants. Then call `API_JobsCommitProvisionalStatuses` once to commit and notify.

**Auth:** `jobs_recruiter` on company

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: application_uuid
- `p3[2]`: provisional_status (`accepted|rejected|""`)
- `p3[3]`: note (optional, internal)
- `p3[4]`: provisional_status_message (optional, applicant-visible after commit)

**Response**
- `p1`: `Success|Fail`
- `p2`: `updated`

### API_JobsCommitProvisionalStatuses
Commits all staged decisions for a job by setting `status = provisional_status` and clearing `provisional_status`.

This is the intended "notify all" step after a recruiter has sifted applicants.

**Auth:** `jobs_recruiter` on company

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: job_uuid

Backwards-compatible modes:
- **Legacy:**
	- `p3[2]`: note (optional, internal)
- **Recommended:**
	- `p3[2]`: accepted_message (optional, applicant-visible)
	- `p3[3]`: rejected_message (optional, applicant-visible)
	- `p3[4]`: note (optional, internal)

**Response**
- `p1`: `Success|Fail`
- `p2`: updated_count

### API_JobsSetApplicantStatus
Accepts or rejects an applicant.

**Auth:** `jobs_recruiter` on company

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: application_uuid
- `p3[2]`: new_status (`accepted|rejected`)
- `p3[3]`: note (optional, internal)
- `p3[4]`: status_message (optional, applicant-visible)

**Response**
- `p1`: `Success|Fail`
- `p2`: `updated`

### API_JobsApplicationNoteAdd
Adds a PRIVATE note to an application (visible only to the company/delegates).

**Auth:** `jobs_recruiter` on company

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: application_uuid
- `p3[2]`: note

**Response**
- `p1`: `Success|Fail`
- `p2`: note_uuid

### API_JobsApplicationNotesList
Lists PRIVATE notes for an application (visible only to the company/delegates).

**Auth:** `jobs_recruiter` on company

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: application_uuid
- `p3[2]`: offset (optional)
- `p3[3]`: limit (optional)

**Response**
- `p1`: `Success|Fail`
- `p2`: JSON array of notes

### API_JobsDelegateGrant
Grants a jobs role to a user.

**Auth:** `jobs_admin` on company

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: user_uuid (opaque UUID only; emails/usernames are not accepted)
- `p3[2]`: role (`jobs_admin|jobs_recruiter|jobs_viewer`)

**Response**
- `p1`: `Success|Fail`
- `p2`: `granted`

### API_JobsDelegateRevoke
Revokes a jobs role from a user.

**Auth:** `jobs_admin` on company

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: user_uuid (opaque UUID only; emails/usernames are not accepted)
- `p3[2]`: role (`jobs_admin|jobs_recruiter|jobs_viewer`)

**Response**
- `p1`: `Success|Fail`
- `p2`: `revoked`

### API_JobsDelegateList
Lists current direct delegates for the three jobs roles.

**Auth:** `jobs_admin` on company

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid

**Response**
- `p1`: `Success|Fail`
- `p3[0]`: admins_csv
- `p3[1]`: recruiters_csv
- `p3[2]`: viewers_csv

### API_JobsDelegateListDetailed
Lists delegates with basic user details (recommended for the company-owner management UI).

**Auth:** `jobs_admin` on company

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: role (optional: `jobs_admin|jobs_recruiter|jobs_viewer`; empty means any)
- `p3[2]`: offset (optional)
- `p3[3]`: limit (optional; default 50, max 200)
- `p3[4]`: query (optional; filters by uname/name/surname/firmname)

**Response**
- `p1`: `Success|Fail`
- `p3[0]`: total_count
- `p3[1]`: returned_count
- `p4`: rows: `[user_uuid, role, uname, name, surname, firmname]`

### API_JobsDelegateSearch
Search delegates by partial name.

This uses the same matching logic as `API_SearchAuto`, then filters results down to the users who are currently delegated on the company.

**Auth:** `jobs_admin` on company

**Request**
- `p1/p2`: login
- `p3[0]`: company_uuid
- `p3[1]`: role (optional: `jobs_admin|jobs_recruiter|jobs_viewer`; empty means any)
- `p3[2]`: query text
- `p3[3]`: max_results (optional; clamped to 50)

**Response**
- `p1`: `Success|Fail`
- `p3[0]`: result count
- `p4`: SearchAuto-style rows (first column is user uuid)
