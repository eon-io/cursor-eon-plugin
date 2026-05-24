# Eon Cursor Plugin

Connect Cursor to Eon via this plugin. Bundles the Eon MCP server and guided cloud onboarding skills into a single installable package.

## Capabilities

- **Eon MCP server** — 27 tools for multi-cloud backup, recovery, inventory, source account management, backup policies, snapshot browsing, restore jobs, GCP organization/folder onboarding, and resource discovery.
- **AWS onboarding skill** — step-by-step guided workflow to connect AWS source accounts via the Eon CloudFormation template.
- **GCP onboarding skill** — step-by-step guided workflow to connect GCP organizations, folders, and individual projects to Eon.
- **Backup policy creation skill** — guided wizard to create backup policies (STANDARD, HIGH_FREQUENCY, PITR, AWS_NATIVE_PITR) with resource selectors, vault schedules, and retention.

## Installation

Copy the plugin into Cursor's local plugin directory and reload Cursor:

```bash
mkdir -p ~/.cursor/plugins/local
cp -R . ~/.cursor/plugins/local/eon
```

## Skills

| Skill | Description |
|-------|-------------|
| AWS Onboarding | Guide users through connecting AWS source accounts: deploy the Eon CloudFormation stack, register the IAM role ARN, and run discovery. |
| GCP Onboarding | Guide users through connecting GCP cloud accounts (orgs, folders, or individual projects) to the Eon platform. |
| Backup Policy Creation | Guide users through creating backup policies: pick a type, define the resource selector, configure schedules with vaults and retention. |

## MCP Server

The plugin connects to `https://mcp.eon.io/mcp` via HTTP. Authentication is handled by the Eon platform.

## Local Development

Copy the plugin to Cursor's local plugin directory, then run `/reload-plugins` inside Cursor after making local edits.

## Links

- [Eon Platform](https://eon.io)
- [Documentation](https://docs.eon.io)

## License

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE) for details. Use of the Eon service and API is governed separately by the Eon Terms of Service.
