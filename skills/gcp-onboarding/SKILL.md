---
name: gcp-onboarding
description: >
  Guide users through onboarding their GCP cloud accounts to the Eon platform.
  Covers viewing existing connections, preparing the GCP environment,
  connecting organizations/folders/projects, and triggering resource discovery.
---

# GCP Account Onboarding

You help Eon customers onboard their GCP cloud accounts through a guided
conversational flow. Follow these four steps in order, adapting to what the
user needs.

## Step 1 — View Current State

When the user asks about their GCP accounts, query and present a unified view:

1. **Connected organizations** — call `list_gcp_organizations`.
2. **Connected folders** — call `list_gcp_folders`.
3. **Connected source accounts** — call `list_source_accounts` (filter to GCP).

Summarize what is already connected: how many orgs, folders, and individual
projects, their status (active/inactive), and when they were last synced.

## Step 2 — Prepare GCP Environment

When the user wants to connect a new account, ask which scope they need:

| Scope | When to use |
|-------|-------------|
| **Organization** | Protect all projects across the entire GCP org |
| **Folder** | Protect all projects under a specific folder |
| **Individual project** | Protect a single GCP project |

### Verify permissions

Before running the setup script the user must have the right GCP IAM
permissions on the target project. Present the required roles based on the
chosen scope and ask the user to confirm they have them.

**Project-level (individual project):**

| Role | Purpose |
|------|---------|
| `roles/iam.serviceAccountAdmin` | Create and configure the Eon service account |
| `roles/iam.securityAdmin` | Manage IAM policy bindings on the project |
| `roles/iam.roleAdmin` | Create custom IAM roles in the project |
| `roles/serviceusage.serviceUsageAdmin` | Enable required GCP APIs |

**Organization / Folder-level** — all of the above, plus:

| Role | Purpose |
|------|---------|
| `roles/iam.organizationRoleAdmin` | Create custom roles at the organization level |
| `roles/resourcemanager.organizationViewer` | List projects and folders in the org |

Provide the following `gcloud` command so the user can verify their
permissions before proceeding:

```bash
gcloud projects test-iam-permissions <PROJECT_ID> \
  --permissions=iam.serviceAccounts.create,iam.roles.create,iam.roles.update,resourcemanager.projects.setIamPolicy,serviceusage.services.enable
```

Replace `<PROJECT_ID>` with the user's GCP project ID. All five permissions
should appear in the output. If any are missing, advise the user to request
the corresponding role from their GCP admin before continuing.

### Resolve script parameters

Before sharing the setup script, gather the required values so you can
present a ready-to-run command. Only ask the user for their GCP project ID.

| Parameter | How to get it |
|-----------|---------------|
| `--project-id` | **Ask the user** for their GCP project ID. |
| `--eon-account-id` | Call `get_viewer` and use `eonAccount.id` from the response. |
| `--scanning-project-id` | Call `list_project_scanning_accounts`, find the account where `cloudProvider` is `GCP`, and use its `providerAccountId`. |
| `--control-plane-service-account` | Use the GCP control-plane service account email from your platform config context. |

### Share the script

Load the script template from the `references/setup_script.md` resource,
substitute the resolved values, and present the command to the user.
Explain what the script does before the user runs it:

- Downloads and runs the Eon GCP setup script
- Creates the required GCP service account
- Grants IAM roles for discovery, backup, and restore
- Sets the service account description with the version payload

## Step 3 — Connect

Once the user confirms they have run the setup script, collect the required
parameters and connect.

### Organization scope
- Required: `organizationId`, `managementServiceAccountId`
- Optional: `excludeProjectPatterns`
- Call `connect_gcp_organization`.

### Folder scope
- Required: `organizationId`, `folderId`, `managementServiceAccountId`
- Optional: `excludeProjectPatterns`
- Call `connect_gcp_folder`.

### Individual project scope
- Required: service account email (`eon-source-<id>@<projectId>.iam.gserviceaccount.com`)
- Call `connect_source_account`.

**Always confirm with the user before executing the connect operation.** After
success, verify by listing the newly connected entity.

## Step 4 — Run Discovery

After a successful connection:

- **Organization / Folder** — discovery runs automatically per-project as they
  sync. Inform the user and offer to check progress.
- **Individual project** — call `invoke_discover` to trigger discovery
  explicitly.

Use `discovery_status` to check progress and report back until discovery
completes.

## Guidelines

- Use `get_onboarding_available_providers` to verify GCP is available before
  starting the flow.
- Use `activate_gcp_organization` / `deactivate_gcp_organization` and
  `activate_gcp_folder` / `deactivate_gcp_folder` when the user asks to
  enable or disable existing connections.
- Never fabricate account IDs or service account emails — always ask the user
  or retrieve them from API responses.
- If a step fails, explain the error clearly and suggest corrective actions.
