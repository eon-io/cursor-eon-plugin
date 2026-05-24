# AWS Source Account CloudFormation Template

Eon publishes the source-account template at a fixed S3 URL. The customer
deploys it once per AWS account they want to onboard.

## Template URL

```
https://eon-public-b2b628cc-1d96-4fda-8dae-c3b1ad3ea03b.s3.amazonaws.com/source-account.yml
```

## One-click deploy link

Build a direct AWS console link by substituting the resolved values into the
template below. Default region is `us-east-1`; let the user pick a different
region if they ask.

```
https://console.aws.amazon.com/cloudformation/home?region=<REGION>#/stacks/create/review?templateURL=https%3A%2F%2Feon-public-b2b628cc-1d96-4fda-8dae-c3b1ad3ea03b.s3.amazonaws.com%2Fsource-account.yml&stackName=EonSourceAccount&param_EonAccountId=<EON_ACCOUNT_ID>&param_RoleName=<ROLE_NAME>
```

## Parameters

### Core (always resolved by the agent)

| Param | Required | Auto-filled | How to resolve |
|-------|----------|-------------|----------------|
| `stackName` | Yes | Yes | Always `EonSourceAccount`. |
| `param_RoleName` | Yes | Yes | Default `EonSourceAccountRole`; let the user override if asked. |
| `param_EonAccountId` | Yes | Yes | Call `get_viewer` → use `eonAccount.id`. |

The user supplies the AWS account ID separately (it is implicit: whichever
account they are signed into when they open the link).

### Environment-specific (ask the user when applicable)

| Param | When to set | Guidance |
|-------|-------------|----------|
| `param_PermissionsBoundaryName` | The user's AWS org enforces a permissions boundary via SCP or a mandatory boundary policy. | **Always ask the user** whether their org requires a permissions boundary. If yes, set this to the boundary policy name (not ARN). If the boundary is mandatory and this value is missing, the stack will fail `CREATE_STACK` with no useful console error. |

### Feature toggles (optional, opt-out scoping)

The CloudFormation console exposes `Enable*` toggle parameters that let the
customer scope down the role's permissions to only the AWS services they want
Eon to protect. Defaults match Eon's recommended coverage; the user only needs
to touch these if they want to *exclude* a service.

Common toggles surfaced by the template (the CF console will show the full
list with descriptions when the user reviews the stack):

- `param_EnableEKS`
- `param_EnableS3CdcBackup`
- `param_EnableDynamoDBStreams`
- `param_EnableAwsNativePitr`
- `param_EnableAuroraClone`

If the user wants to disable any of these, tell them to flip the toggle on
the **Specify stack details** page before clicking **Create stack** — the
agent does not need to pre-populate them in the deploy URL.

## What the stack does

1. Creates an IAM role named `<RoleName>` in the target AWS account.
2. Configures the role's trust policy so Eon's control plane (identified by
   `param_EonAccountId`) is allowed to assume it.
3. Attaches the managed/inline policies Eon needs for resource discovery,
   snapshot management, and backup/restore operations across the supported
   AWS services (EC2/EBS, RDS, S3, etc.).

## After the stack reaches CREATE_COMPLETE

Build the role ARN and call `connect_source_account`:

```
arn:aws:iam::<AwsAccountId>:role/<RoleName>
```
