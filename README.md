# Eon Cursor Plugin

Connect Cursor to Eon via this plugin. Bundles the Eon MCP server and guided cloud onboarding skills into a single installable package.

## Capabilities

- **Eon MCP server** — 27 tools for multi-cloud backup, recovery, inventory, source account management, backup policies, snapshot browsing, restore jobs, GCP organization/folder onboarding, and resource discovery.
- **GCP onboarding skill** — step-by-step guided workflow to connect GCP organizations, folders, and individual projects to Eon.

## Installation

Copy the plugin into Cursor's local plugin directory and reload Cursor:

```bash
mkdir -p ~/.cursor/plugins/local
cp -R . ~/.cursor/plugins/local/eon
```

## Skills

| Skill | Description |
|-------|-------------|
| GCP Onboarding | Guide users through connecting GCP cloud accounts (orgs, folders, or individual projects) to the Eon platform. |

## MCP Server

The plugin connects to `https://mcp.eon.io/mcp` via HTTP. Authentication is handled by the Eon platform.

## Local Development

Copy the plugin to Cursor's local plugin directory, then run `/reload-plugins` inside Cursor after making local edits.

## Links

- [Eon Platform](https://eon.io)
- [Documentation](https://docs.eon.io)
