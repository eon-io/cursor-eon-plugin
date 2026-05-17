# GCP Setup Script

The following command downloads and runs the Eon GCP onboarding script. It
creates the required service account, grants IAM roles, and configures the
service account description with the version payload.

## Command

```bash
curl -fsSL https://eon-public-b2b628cc-1d96-4fda-8dae-c3b1ad3ea03b.s3.amazonaws.com/gcp-eon-setup.sh \
  -o gcp-source-setup.sh && \
chmod +x gcp-source-setup.sh && \
./gcp-source-setup.sh \
  --account-type Source \
  --project-id <PROJECT_ID> \
  --eon-account-id <EON_ACCOUNT_ID> \
  --scanning-project-id <SCANNING_PROJECT_ID> \
  --control-plane-service-account <CONTROL_PLANE_SERVICE_ACCOUNT> \
  --enable-gcs-notification-management
```

## Parameters

| Flag | Required | Auto-filled | How to resolve |
|------|----------|-------------|----------------|
| `--account-type` | Yes | Yes | Always `Source`. |
| `--project-id` | Yes | No | **Ask the user** for their GCP project ID. |
| `--eon-account-id` | Yes | Yes | Call `get_viewer` → use `eonAccount.id`. |
| `--scanning-project-id` | Yes | Yes | Call `list_project_scanning_accounts` → filter `cloudProvider == "GCP"` → use `providerAccountId`. |
| `--control-plane-service-account` | Yes | Yes | Use the GCP control-plane service account email from your platform config context. |
| `--enable-gcs-notification-management` | No | No | Include when the user wants Eon to manage GCS notifications automatically. |

## What the script does

1. Creates a GCP service account named `eon-source-<eon-account-id>` in the
   target project.
2. Grants IAM roles required for resource discovery, snapshot management, and
   backup operations (Compute, Cloud Storage, CloudSQL, BigQuery).
3. Grants the Eon control-plane service account permission to impersonate the
   newly created service account.
4. Sets the service account description with a version payload so Eon can
   detect the onboarding version.

## Scope variations

- **Organization-level**: The script is run once. Eon discovers and manages all
  projects in the organization automatically via `connect_gcp_organization`.
- **Folder-level**: Same as organization, scoped to a folder via
  `connect_gcp_folder`.
- **Individual project**: The script is run per project. The resulting service
  account email is used with `connect_source_account`.
