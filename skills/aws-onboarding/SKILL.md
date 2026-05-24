---
name: aws-onboarding
description: >
  Guide users through connecting their AWS source accounts to the Eon platform.
  Covers viewing existing connections, deploying the Eon CloudFormation stack
  to provision the IAM role, registering the role ARN with Eon, and triggering
  resource discovery.
---

# AWS Source Account Onboarding

You help Eon customers connect their AWS source accounts through a guided
conversational flow. Follow these four steps in order, adapting to what the
user needs.

## Step 1 — View Current State

When the user asks about their AWS accounts, present a unified view:

1. Call `list_source_accounts` and filter the result to `cloudProvider == "AWS"`.
2. Summarize: how many AWS accounts are connected, their `providerAccountId`,
   `status`, `connectedTime`, and the role ARN currently in use.

If the user has no AWS accounts yet, offer to walk them through connecting one.

## Step 2 — Prepare the AWS Environment

Connecting an AWS account requires an IAM role in the customer's AWS account
that Eon can assume. The customer creates that role by deploying Eon's public
CloudFormation template.

### Verify permissions

Before deploying, the user must have the right IAM permissions in the target
AWS account. Present this table and ask the user to confirm:

| Permission | Purpose |
|------------|---------|
| `cloudformation:CreateStack` | Deploy the Eon source-account stack |
| `iam:CreateRole`, `iam:AttachRolePolicy`, `iam:PutRolePolicy` | Create the role Eon will assume |
| `iam:CreatePolicy` | Create the inline/managed policies the role uses |

If the user is unsure, advise them to ask their AWS admin for
`AdministratorAccess` on the target account for the duration of the deploy,
or to delegate the stack deploy to someone who has it.

### Resolve script parameters

Before sharing the stack-deploy link, gather the values you can resolve
yourself. Only ask the user for their AWS account ID.

| Parameter | How to get it |
|-----------|---------------|
| `AwsAccountId` | **Ask the user** for their 12-digit AWS account ID. |
| `RoleName` | Default to `EonSourceAccountRole` unless the user requests a different name. |
| `EonAccountId` | Call `get_viewer` and use `eonAccount.id` from the response. |

### Share the CloudFormation deploy link

Load the template details from the `references/cloudformation_template.md`
resource, substitute the resolved values, and present the **direct AWS console
link** to the user. Explain what the stack does before they run it:

- Creates an IAM role named `<RoleName>` (default `EonSourceAccountRole`)
- Configures the role's trust policy so Eon's control plane can assume it
- Attaches the managed/inline policies Eon needs for inventory, backup, and
  restore operations on AWS resources

Tell the user to:
1. Open the link in a new tab while signed into the **target AWS account**.
2. Review the template, acknowledge the IAM resources notice, and click
   **Create stack**.
3. Wait for the stack status to reach `CREATE_COMPLETE`.
4. Come back here and confirm when it's done.

### Troubleshooting stack-deploy failures

If the user reports the stack reached `CREATE_FAILED` or `ROLLBACK_COMPLETE`,
always direct them to the **Events** tab of the stack in the CloudFormation
console — the root-cause line is the first event with status `CREATE_FAILED`.
Then map the error to the most likely fix:

| Symptom in CF Events | Likely cause | Fix |
|---|---|---|
| `AlreadyExistsException` on the role, or stack name already in use | A previous attempt left a stack or role behind. | Delete the old `EonSourceAccount` stack (and the leftover role if it survived), or deploy with a different `RoleName` + `stackName`. |
| `AccessDenied` mentioning `permissions boundary` or `SCP` | The org enforces a permissions boundary the role must use, or an SCP is blocking IAM role creation. | Re-run the deploy and set `PermissionsBoundaryName` (see `references/cloudformation_template.md`). If an SCP is blocking outright, the user needs to ask their AWS admin to allow `iam:CreateRole`/`iam:AttachRolePolicy` for this account. |
| `User: arn:aws:iam::... is not authorized to perform: iam:...` | The signed-in user doesn't have enough IAM perms to deploy the stack. | Re-check the permissions table at the top of Step 2 — the user (or someone with `AdministratorAccess`) needs to deploy the stack. |
| Any other `CREATE_FAILED` | Region-specific quota, conflicting resource, etc. | Read the exact Events-tab message back to the user and suggest the corresponding remediation; if unsure, advise opening an Eon support ticket with the Events screenshot. |

After the user resolves the issue, ask them to delete the failed stack (a
failed stack must be deleted before re-deploying) and re-open the deploy
link.

## Step 3 — Connect

Once the user confirms the stack reached `CREATE_COMPLETE`, register the role
with Eon.

Required to call `connect_source_account`:

- `cloudProvider`: `AWS`
- `sourceAccountAttributes.aws.roleArn`:
  `arn:aws:iam::<AwsAccountId>:role/<RoleName>`
- `name`: a friendly display name (ask the user, or default to the account ID)

**Always confirm the resolved role ARN with the user before executing the
connect call.** After success, call `list_source_accounts` to verify the new
account appears with `status` set to a connected/active state.

If the call fails with a role-assumption error, the most likely causes are:

- The stack is still deploying — ask the user to wait until `CREATE_COMPLETE`.
- The role name doesn't match what the stack created — re-check `RoleName`.
- The wrong AWS account ID was used in the ARN — re-check `AwsAccountId`.

If the user wants to verify stack state directly, you can call
`get_aws_stack_details` with the account ID to retrieve the CloudFormation
stack ID.

## Step 4 — Run Discovery

After the account is connected, call `invoke_discover` for the new account ID
to populate the inventory. Use `discovery_status` if available, or
`list_resources` filtered to the newly connected account, to confirm
discovery has produced results.

## Guidelines

- Use `get_onboarding_available_providers` to verify AWS is available before
  starting the flow.
- Never fabricate AWS account IDs or role ARNs — always ask the user or read
  them back from API responses.
- Always show the resolved ARN before calling `connect_source_account`.
- If the user already has the same AWS account connected, surface that
  immediately and ask whether they want to reconnect or onboard a different
  account.
- If a step fails, explain the error and suggest corrective actions before
  re-trying.
