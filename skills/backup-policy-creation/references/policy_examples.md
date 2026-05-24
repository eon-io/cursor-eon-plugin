# Backup Policy Request Examples

Reference payloads for `create_backup_policy`. Substitute UUIDs and field
values with what the user provides; never reuse the example values verbatim.

## Standard — daily + weekly, all production EC2 volumes

```json
{
  "name": "Production EBS — daily 30d + weekly 1y",
  "enabled": true,
  "scheduleMode": "SCHEDULE_MODE_BASIC",
  "resourceSelector": {
    "resourceSelectionMode": "CONDITIONAL",
    "expression": {
      "and": [
        { "field": "cloudProvider", "op": "EQ", "value": "AWS" },
        { "field": "resourceType",  "op": "EQ", "value": "EBS_VOLUME" },
        { "field": "tags.env",      "op": "EQ", "value": "production" }
      ]
    }
  },
  "backupPlan": {
    "backupPolicyType": "STANDARD",
    "standardPlan": {
      "backupSchedules": [
        {
          "vaultId": "<vault-uuid>",
          "scheduleConfig": {
            "frequency": "DAILY",
            "dailyConfig": { "hour": 2, "minute": 0 }
          },
          "backupRetentionDays": 30
        },
        {
          "vaultId": "<vault-uuid>",
          "scheduleConfig": {
            "frequency": "WEEKLY",
            "weeklyConfig": { "dayOfWeek": "SUNDAY", "hour": 3, "minute": 0 }
          },
          "backupRetentionDays": 365
        }
      ]
    }
  }
}
```

## High-frequency — every 4 hours for critical RDS

```json
{
  "name": "Critical RDS — 4h interval",
  "enabled": true,
  "backupPlan": {
    "backupPolicyType": "HIGH_FREQUENCY",
    "highFrequencyPlan": {
      "backupSchedules": [
        {
          "vaultId": "<vault-uuid>",
          "scheduleConfig": {
            "frequency": "INTERVAL",
            "intervalConfig": { "hours": 4 }
          },
          "backupRetentionDays": 14
        }
      ]
    }
  },
  "resourceSelector": {
    "resourceSelectionMode": "CONDITIONAL",
    "expression": {
      "and": [
        { "field": "resourceType", "op": "EQ", "value": "RDS_INSTANCE" },
        { "field": "tags.tier",    "op": "EQ", "value": "critical" }
      ]
    }
  }
}
```

## Stage with NONE first

When the user wants to set up a policy before turning it on, use selector
mode `NONE`. Later, update the policy with the real expression.

```json
{
  "name": "(staged) GCP CloudSQL nightly",
  "enabled": true,
  "resourceSelector": { "resourceSelectionMode": "NONE" },
  "backupPlan": { ... }
}
```

## Notes on schedule shapes

- `DAILY` → `dailyConfig { hour, minute }`
- `WEEKLY` → `weeklyConfig { dayOfWeek, hour, minute }`
- `MONTHLY` → `monthlyConfig { dayOfMonth, hour, minute }`
- `ANNUALLY` → `annuallyConfig { month, dayOfMonth, hour, minute }`
- `INTERVAL` → `intervalConfig { hours }` (HIGH_FREQUENCY only)

All times are UTC unless the user explicitly specifies `useUtc: false`.

## Frequently used `backupPolicyType` → plan-field mapping

| Type | Plan field | Needs `vaultId` per schedule? |
|------|------------|-------------------------------|
| `STANDARD` | `standardPlan.backupSchedules[]` | Yes |
| `HIGH_FREQUENCY` | `highFrequencyPlan.backupSchedules[]` | Yes |
| `PITR` | `pitrPlan` | No |
| `AWS_NATIVE_PITR` | `awsNativePitrPlan` | No |

Pick exactly one plan field per request, matching `backupPolicyType`. For
other types the API may accept (see the canonical `BackupPolicyType` enum
and `BackupPolicyPlan` schema in the OpenAPI spec), consult the matching
plan-field schema before constructing the payload — do not guess.
