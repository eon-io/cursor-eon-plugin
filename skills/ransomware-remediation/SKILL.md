---
name: ransomware-remediation
description: >
  Investigate ransomware, malware, and data-anomaly findings on Eon-protected
  resources, and build a remediation playbook to restore affected resources to
  their last verified-clean recovery point. Covers counting and listing
  infected resources, reading security-scan evidence and anomaly graphs,
  resolving a clean snapshot and restore template per resource, and executing
  a human-confirmed bulk restore including the multi-party-approval handoff.
  Use whenever the user asks about ransomware/malware/data-anomaly detections,
  infected resources, or wants to clean up, remediate, or recover from a
  security incident.
---

# Ransomware Investigation & Remediation

Eon's security scanner examines every backup snapshot with three detectors
(ransomware behavior, malware, data anomaly) and assigns each resource a
scan conclusion. There is no named-strain concept; you work from
conclusions and their evidence.

**Part A — Investigate** produces a per-resource incident report.
**Part B — Remediate** restores affected resources to their latest clean
snapshot. The user may enter at either part, but always show which
resources will be touched before restoring anything.

Wire shapes, filter JSON, enum values, status codes, and the
enum-to-plain-language translation table live in
[references/api-details.md](references/api-details.md). Load it before
constructing any call or presenting results.

## Part A — Investigate

### A1 — Scope the incident

1. Resolve the project via `list_projects` (never ask the user for IDs you
   can look up).
2. `count_infected_resources` for the headline count (excludes muted).
3. `list_resources` filtered on scan conclusion (exact JSON in the
   reference doc): `HIGH_CHANCE_FOR_RANSOMWARE` for ransomware; add
   `MALWARE_DETECTED` / `DATA_ANOMALY_DETECTED` when relevant. Page with
   `pageSize` up to 500.

Present the list grouped by conclusion (name, type, account, region).
Each resource carries `classifications.dataClassesDetails.dataClasses`
(e.g. PII, PHI) — lead with resources holding classified data; investigate
and recover those first. Ask whether to investigate specific resources,
all, or jump to remediation.

### A2 — Per-resource investigation report

Assemble from:

| What | Tool |
|------|------|
| Resource info (type, account, region, tags, policies, restore template) | `get_resource` |
| Timeline per detector (last clean / first infected / last scan) | `get_security_scans_summary` |
| Infected recovery points | `list_infected_snapshots` |
| Full snapshot history (clean vs infected over time) | `list_resource_snapshots` |
| Evidence (paths/tables, scores, justifications, anomaly graph) | `list_security_findings` by `resourceId` |
| Aggregate evidence counts | `get_resource_finding_counts` |
| Backup / restore history around the window | `list_backup_jobs`, `list_restore_jobs` |
| Data classifications (PII, PHI, …) | `get_resource` → `classifications.dataClassesDetails.dataClasses` |

The report has a fixed structure — one-line header, then exactly three
sections:

**Header** — resource, what was detected (plain language), confidence,
and the window: the gap between the last clean and first infected
snapshot ("data was likely compromised between X and Y").

**1. Attack flow** — the event sequence reconstructed from findings (see
below), told as a short narrative with timestamps, plus a compact
snapshot timeline when it helps.

**2. What was infected** (ransomware/malware) or **Affected resources**
(data anomaly — a deviation, not an infection; never use "infected" or
"infection window" language for it, say "affected" / "anomaly window").
The concrete files/tables/databases hit, blast radius (affected snapshot
count, backup jobs, prior restores, account/region/tags/policy/template),
and data classifications — "this database contains PII" changes severity and
who the user may need to notify.

**3. Recommended recovery** — latest clean snapshot, how much data
(time-wise) restoring rolls back, any blockers (no template, no clean
snapshot), and the offer to build the remediation plan.

### Reconstructing the attack

Each finding lists the snapshots it was observed in. Sort findings by
their **earliest** snapshot to recover event order, then narrate. Known
patterns:

| Observed together | Meaning |
|---|---|
| Mass data change + deleted tables/rows + ransom-note table/file | Database ransomware: data deleted/encrypted, ransom note left |
| Files renamed to ransomware extensions + note file | File-encryption ransomware |
| Large fraction of data changed, no note | Suspicious mass modification / possible in-place encryption; note not (yet) observed |
| Metric far outside forecast, nothing else | Data anomaly — may be unintended deletion, bad migration, or malicious; don't presume attack |

Name the likely mechanism when data supports it (dropped tables →
DROP-TABLE-style destruction; renamed extensions → file encryption). If
findings don't match a known pattern, say what was observed and that the
mechanism is unclear — never invent a story.

### Presentation — plain language only

Never show raw enum values, evidence-type identifiers, or wire field
names; translate per the table in the reference doc (e.g.
"high chance of ransomware", "97% confidence", use
`evidenceDisplayName`). Weave `justification` into the narrative; report
tables should read like a timeline, not an API response.

## Part B — Remediate

Mirrors Eon's Remediation Center: restore each affected resource from its
latest clean snapshot. Restores create new infrastructure — every
mutating step requires explicit user confirmation.

### B1 — Confirm the resource set

`count_infected_resources` (0 → nothing to do); `list_resources` with the
ransomware filter (or the set confirmed in Part A); collect each `id` and
`restoreTemplateId`. Batches cap at **500 items** — say so and repeat if
larger.

### B2 — Restore template per resource

Use the resource's `restoreTemplateId` if attached; otherwise
`list_restore_templates_v2` filtered by resource type and let the user
pick. No compatible template → that resource waits until one is created
in the console. Never invent a template ID.

### B3 — Clean snapshot per resource

Ask for the lookback window (default **7 days**, offer up to 30), then
`bulk_select_snapshots_for_resources` with `isClean: true` and
`pick: "SNAPSHOT_RANGE_PICK_LATEST"`. Keep only OK items. Resources with
no clean snapshot in the window: report separately; widening the window
needs explicit consent (older snapshot = more data loss). Errors:
surface and exclude.

### B4 — Present and confirm

Table: resource, evidence one-liner, clean snapshot point-in-time,
template. Require an explicit "yes" — never proceed on silence.

### B5 — Validate

`bulk_restore_validate` with `items: [{resourceId, templateId,
snapshotId}]`. Keep items that are OK **and**
`resolvedSnapshotHasThreatFindings: false`; if that flag is true, drop
the item — never restore from it. Other statuses (no template, invalid
template, unsupported type, drifted snapshot): surface reason, fix or
drop, and re-confirm the updated playbook.

### B6 — Execute

`bulk_restore` with validated items. By status code:

- **200** — jobs started; report `bulkRestoreGroup` + job IDs, offer
  tracking via `list_restore_jobs` / `get_restore_job`.
- **201** — **MPA-intercepted; the restore did NOT run.** Ask the user,
  then `create_action_approval_request` with `action: "CONFIRM"` on the
  returned request ID. State clearly it's now only PENDING_APPROVAL — an
  approver must approve before anything runs. Never imply the restore
  happened. (`DISCARD` cancels.)
- **409** — MPA conflict (pending/actioned/expired request); surface and
  offer to rebuild.
- **422** — per-item validation failures; present and return to B5.

### B7 — Report honestly and verify

- "Remediation started" only when a job ID exists; "pending approval"
  when an MPA request was confirmed; "remediated" only when the job
  completes.
- Close the loop: track jobs to completion; after the restored resource
  is backed up and scanned again, its scan should conclude clean
  (`get_security_scans_summary`) — only then call the incident
  remediated.
- Restores create **new** resources; the infected originals are not
  deleted or disinfected. Recommend isolating/decommissioning them via
  the user's incident-response process.

## Guidelines

- **Never fabricate IDs** — every resource/snapshot/template/MPA-request
  ID comes from a prior tool response.
- **Never restore from a snapshot not verified clean** (`isClean: true` +
  validate re-check).
- **Confirm before every mutating call** (`bulk_restore`,
  `create_action_approval_request`); Part A reads are safe without
  confirmation.
- **Plain language everywhere** — no raw enums or wire field names in
  user-facing output.
- **Narrate, don't speculate** — attack narrative only from observed
  findings.
- Unsupported type or missing template: say so and ask, don't guess.
