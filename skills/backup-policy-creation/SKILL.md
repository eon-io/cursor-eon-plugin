---
name: backup-policy-creation
description: >
  Guide users through creating a backup policy in Eon: choosing the policy
  type, defining the resource selector, configuring schedules with vaults and
  retention, and submitting the create request. Also covers viewing existing
  policies before creating a new one.
---

# Backup Policy Creation

You help Eon customers create backup policies through a guided conversational
flow that mirrors the in-product wizard. Follow these steps in order, but
skip any step where the user already provided the answer.

## Step 0 — Show Existing Policies (Optional)

When the user asks about their backup policies, or before creating a new one,
call `list_backup_policies` and summarize: name, type, enabled, number of
schedules, and which resource selector mode each uses.

Offer to either:
- View a specific policy (`get_backup_policy` by ID), or
- Create a new one (continue to Step 1).

## Step 1 — Name and Policy Type

Ask the user:

1. **Name** — a human-readable label for the policy (e.g. `Production Daily +
   Weekly`).
2. **Policy type** — present the common options and let the user pick one:

| Type | When to use |
|------|-------------|
| `STANDARD` | Most workloads. Periodic snapshots, copied to an Eon vault. |
| `HIGH_FREQUENCY` | Sub-daily protection (interval-based, e.g. every 4 hours). |
| `PITR` | Continuous point-in-time recovery for supported databases. |
| `AWS_NATIVE_PITR` | AWS-native PITR for RDS/DynamoDB without copying to Eon. |

The canonical, full list of policy types lives in the OpenAPI schema
(`BackupPolicyType`) that the agent already has loaded — consult it if the
user asks about a type not shown above (e.g. native-standard or
Azure-Files-specific variants), and surface the server response verbatim
rather than guessing.

Default to `STANDARD` if the user is unsure.

## Step 2 — Resource Selector

Ask which resources this policy should apply to. There are three modes:

| Mode | Behavior |
|------|----------|
| `ALL` | Every resource in the project matches. |
| `NONE` | No resource matches (useful for staging a policy before enabling it). |
| `CONDITIONAL` | Match by an expression over resource attributes (provider, tags, region, resource type, etc.). |

For `CONDITIONAL`, walk the user through the filter expression:

- **Provider**: `AWS`, `AZURE`, `GCP`.
- **Resource types**: e.g. `EBS_VOLUME`, `RDS_INSTANCE`, `S3_BUCKET`.
- **Tags / labels**: key-value pairs.
- **Regions**: limit by region(s).

The user may also supply explicit overrides:

- `resourceInclusionOverride[]` — extra resource IDs always included.
- `resourceExclusionOverride[]` — resource IDs always excluded.

**Preview the match.** Before continuing, call `list_resources` with the same
filter and show the user the count and a few examples so they can confirm the
selector is correct.

## Step 3 — Schedules (Standard / High-Frequency)

For each schedule the user wants to add, gather:

1. **Vault** — **only for policy types that store copies in an Eon vault**
   (currently `STANDARD` and `HIGH_FREQUENCY`). Call `list_vaults` and
   present available vaults (id, name, provider, region); ask the user which
   one to write backups to. **Skip this step entirely** for the native /
   PITR families (`PITR`, `AWS_NATIVE_PITR`, and any other `*_NATIVE_*`
   variants) — those keep snapshots in the provider's own storage and do
   not accept a `vaultId`.
2. **Frequency** — pick one:
   - `INTERVAL` — every N hours (HIGH_FREQUENCY only).
   - `DAILY` — at a fixed hour:minute (UTC by default).
   - `WEEKLY` — day-of-week + time.
   - `MONTHLY` — day-of-month + time.
   - `ANNUALLY` — month, day, time.
3. **Retention** — `backupRetentionDays` (integer). Common defaults: 7, 30,
   90, 365.

Repeat until the user has all the schedules they want. Multiple schedules are
common (e.g. daily for 30 days + weekly for 1 year).

For native PITR plans (and any other non-`STANDARD` / non-`HIGH_FREQUENCY`
plan), the schedule shape is different — consult the matching plan schema
(`awsNativePitrPlan`, `pitrPlan`, etc. — see `BackupPolicyPlan` in the
OpenAPI spec) and gather only the fields the corresponding plan accepts
before submitting.

## Step 4 — Confirm and Create

Before calling the API:

1. Show the user a complete summary: name, type, enabled (default `true`),
   resource selector mode, number of matched resources from the preview, and
   the list of schedules with their vault, cadence, and retention.
2. **Wait for explicit user confirmation** ("create it", "yes", etc.).

Then call `create_backup_policy` with the assembled request:

```json
{
  "name": "<name>",
  "enabled": true,
  "scheduleMode": "SCHEDULE_MODE_BASIC",
  "resourceSelector": {
    "resourceSelectionMode": "CONDITIONAL",
    "expression": { ... },
    "resourceInclusionOverride": [],
    "resourceExclusionOverride": []
  },
  "backupPlan": {
    "backupPolicyType": "STANDARD",
    "standardPlan": {
      "backupSchedules": [
        {
          "vaultId": "<uuid>",
          "scheduleConfig": {
            "frequency": "DAILY",
            "dailyConfig": { "hour": 2, "minute": 0 }
          },
          "backupRetentionDays": 30
        }
      ]
    }
  }
}
```

On success the server returns the created policy with its generated `id`.
Tell the user the policy is live and that the backend will immediately:

- Evaluate the resource selector against current inventory.
- Register Temporal schedules for each matching resource.
- Start producing backup jobs at the configured cadence — no further action
  needed.

Offer to verify with `get_backup_policy <id>` or to view matched resources.

## Step 5 — Common Follow-Ups

- **Disable temporarily** — call `update_backup_policy` with `enabled: false`.
- **Delete** — call `delete_backup_policy` (warn that this stops future
  backups and ask for confirmation).
- **Iterate the selector** — call `update_backup_policy` with a revised
  `resourceSelector` and rerun a `list_resources` preview first.

## Guidelines

- Never invent vault IDs, resource IDs, or tag values — always read them from
  API responses or ask the user.
- Always preview the resource selector match before creating the policy.
- Always confirm the full assembled request with the user before calling
  `create_backup_policy`.
- If the API returns a feature-flag error (e.g. BigQuery native PITR), tell
  the user the feature isn't enabled on their account and offer to fall back
  to a supported policy type.
- If `create_backup_policy` fails, surface the server error verbatim and
  suggest the most likely fix (invalid vault ID, missing required schedule
  fields, malformed expression).
