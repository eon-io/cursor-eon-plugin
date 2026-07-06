# Ransomware Investigation & Remediation — API wire formats

Exact request/response shapes for the tools this skill calls. Field names
and enum values below are the JSON wire format — use them verbatim.

## Enum → plain language (for user-facing output)

Wire values are internal; translate before showing the user:

| Wire value | Say instead |
|---|---|
| `HIGH_CHANCE_FOR_RANSOMWARE` | "high chance of ransomware" |
| `MALWARE_DETECTED` | "malware detected" |
| `DATA_ANOMALY_DETECTED` | "data anomaly detected" |
| `RANSOMWARE_BEHAVIOR` (detector) | "ransomware behavior scan" |
| `DATA_ANOMALY` (detector) | "data anomaly scan" |
| evidence types (e.g. `significant_row_decrease`) | use `evidenceDisplayName`, or describe: "a large drop in row count" |
| `score: 0.97` | "97% confidence" |
| status enums (e.g. `BULK_SELECT_SNAPSHOT_STATUS_NO_CLEAN_SNAPSHOT`) | describe: "no clean snapshot found in the selected window" |

## Enums

**SecurityScanConclusion** (a resource/snapshot's scan verdict):
`CLEAN`, `HIGH_CHANCE_FOR_RANSOMWARE`, `DATA_ANOMALY_DETECTED`,
`MALWARE_DETECTED`

**DetectorType**: `MALWARE`, `RANSOMWARE_BEHAVIOR`, `DATA_ANOMALY`

**SnapshotRangePick**: `SNAPSHOT_RANGE_PICK_LATEST`,
`SNAPSHOT_RANGE_PICK_EARLIEST`

**BulkSelectSnapshotStatus**: `BULK_SELECT_SNAPSHOT_STATUS_OK`,
`..._NO_SNAPSHOT`, `..._NO_CLEAN_SNAPSHOT`, `..._ERROR`

**BulkRestoreItemStatus**: `BULK_RESTORE_ITEM_STATUS_UNSPECIFIED`, `..._OK`,
`..._NO_SNAPSHOT`, `..._OUT_OF_RANGE`, `..._NO_CLEAN_SNAPSHOT`,
`..._INELIGIBLE`, `..._UNSUPPORTED_TYPE`, `..._NO_TEMPLATE`,
`..._TEMPLATE_INVALID`, `..._SNAPSHOT_DRIFTED`

**MPASubmitAction**: `CONFIRM`, `DISCARD`

## Investigation tools

### count_infected_resources

GET — no body. Response:

```json
{ "count": 12 }
```

Counts infected resources that are **not muted** (excluded resources don't
count).

### list_resources — filter for flagged resources

POST with `pageSize` (≤ 500) / `pageToken` query params. Body filter on the
resource's **latest** scan conclusion:

```json
{
  "filters": {
    "securityScanConclusion": {
      "in": ["HIGH_CHANCE_FOR_RANSOMWARE"]
    }
  }
}
```

Add `"MALWARE_DETECTED"` / `"DATA_ANOMALY_DETECTED"` to the `in` list to
cover the other detectors. Each returned resource includes
`restoreTemplateId` (may be absent) and `securityScans` (latest scan per
detector).

### get_security_scans_summary

GET per resource, query param `detectorType` (default
`RANSOMWARE_BEHAVIOR` — call once per detector you care about). Response:

```json
{
  "firstInfected": { "snapshotId": "...", "conclusion": "HIGH_CHANCE_FOR_RANSOMWARE", "createdTime": "...", "confidenceScore": 0.95, "justification": "...", "detectorType": "RANSOMWARE_BEHAVIOR", "totalEvidenceCount": 8, "results": [ ... ] },
  "lastClean":  { ...SecurityScan... },
  "lastScan":   { ...SecurityScan... },
  "infectedSnapshotsCount": 4
}
```

`results` is capped to the first 3 evidence entries; use
`list_security_findings` for the full evidence list.

### list_security_findings

POST with `pageSize` (50–200) / `pageToken`. Body:

```json
{
  "filters": {
    "resourceId":   { "in": ["<resource-id>"] },
    "detectorType": { "in": ["RANSOMWARE_BEHAVIOR", "DATA_ANOMALY"] },
    "path":         { "contains": ["optional substring"] },
    "snapshotId":   { "in": ["optional snapshot id"] }
  }
}
```

All filters optional. Each `FindingEntry` in the response:

- `detectorType`, `evidenceType`, `evidenceDisplayName`
- `path` — affected file, directory, table, or database
- `objectType` — path / table / database
- `conclusion`, `score` (0.0–1.0), `justification` — from the latest
  snapshot the finding was observed in
- `snapshots[]` — `{id, pointInTime, vaultId}` newest-first, every snapshot
  this finding was observed in
- `anomalyGraph.dataPoints[]` (DATA_ANOMALY only):
  `{timestamp, actualValue, forecastValue, upperBound, lowerBound, isAnomaly, snapshotId}` —
  observed vs forecast; `isAnomaly: true` points are the anomalous
  snapshots
- `excludedSince` — set if the finding was excluded (muted) by the user

### get_resource_finding_counts

GET per resource. Returns aggregate finding counts per detector and
evidence type plus the affected snapshot count — use for the report's
evidence summary before pulling full findings.

### list_infected_snapshots

GET per resource. Response:

```json
{ "snapshots": [ { "id": "...", "pointInTime": "...", "vaultId": "..." } ] }
```

Newest-first, capped at 100.

## Remediation tools

### bulk_select_snapshots_for_resources

POST. Body:

```json
{
  "resourceIds": ["<uuid>", "..."],
  "from": "2026-06-25T00:00:00Z",
  "to":   "2026-07-02T23:59:59Z",
  "pick": "SNAPSHOT_RANGE_PICK_LATEST",
  "isClean": true
}
```

- `resourceIds`: 1–500 items. `from`/`to`: RFC 3339 (default window: last
  7 days UTC).
- `isClean: true` makes only snapshots with a clean scan verdict eligible —
  this is the core of the playbook; never omit it.

Response per item:

```json
{ "items": [ { "resourceId": "...", "status": "BULK_SELECT_SNAPSHOT_STATUS_OK", "snapshotId": "...", "pointInTime": "...", "vaultId": "..." } ] }
```

Only `..._OK` items carry a usable `snapshotId`.

### list_restore_templates_v2

POST with `pageSize`/`pageToken`. Body (all optional):

```json
{ "filters": { "resourceType": { "in": ["<RestoreTemplateResourceType>"] }, "name": { "contains": ["prod"] } } }
```

Response: `restoreTemplates[]` of
`{id, name, description, resourceType, attachedResourceCount, ...}` plus
`pagination.nextPageToken`. Match `resourceType` to the resource being
restored.

### bulk_restore_validate

POST. Body:

```json
{ "items": [ { "resourceId": "<id>", "templateId": "<uuid>", "snapshotId": "<uuid>" } ] }
```

All three fields required per item; 1–500 items. Response per item:

```json
{
  "items": [ {
    "resourceId": "...",
    "status": "BULK_RESTORE_ITEM_STATUS_OK",
    "reason": "...", "message": "...",
    "resolvedPointInTime": "...",
    "templateId": "...",
    "resolvedTargetAccountId": "...",
    "resolvedResourceType": "...",
    "resolvedSnapshotHasThreatFindings": false,
    "resolvedVaultId": "..."
  } ]
}
```

Gate on `status == BULK_RESTORE_ITEM_STATUS_OK` **and**
`resolvedSnapshotHasThreatFindings == false`.

### bulk_restore

POST. Same body shape as validate (`items[]` of
`{resourceId, templateId, snapshotId}`, 1–500). Requires the
`create:restore_resource` permission; protected by MPA
(`MPA_OPERATION_RESTORE_RESOURCE`).

Responses:

- **200** — jobs started:

  ```json
  {
    "bulkRestoreGroup": "<group-id>",
    "jobs":  [ { "resourceId": "...", "jobId": "..." } ],
    "items": [ { "resourceId": "...", "status": "...", "resolvedSnapshotId": "...", "resolvedPointInTime": "...", ... } ]
  }
  ```

  Track jobs with `list_restore_jobs` / `get_restore_job`.

- **201** — MPA intercepted. **No restore ran.** Body:

  ```json
  { "actionApprovalRequest": { "id": "<request-id>", "operation": "...", "status": "...", "resourceIds": [...], ... }, "mpaPolicies": { "policies": [ ... ] } }
  ```

  The request is in state CREATED; it must be submitted (below) and then
  approved by an approver before anything runs.

- **409** — MPA conflict (pending request already exists / already
  actioned / expired).

- **422** — batch failed validation; body is the
  `bulk_restore_validate` response shape (per-item
  `status`/`reason`/`message`).

### create_action_approval_request

POST to submit a CREATED MPA request, path param `requestId` from the 201
body's `actionApprovalRequest.id`. Body:

```json
{ "action": "CONFIRM", "comment": "Ransomware remediation for 12 flagged resources" }
```

- `CONFIRM` → request moves to PENDING_APPROVAL and the approval timer
  starts. The restore still has NOT run — an approver must approve it.
- `DISCARD` → request is cancelled.
- Only the requester can submit their own request (403 otherwise).

Response: `{ "actionApprovalRequest": { ...updated request... } }`.
