# AI Context Bridge for HubSpot

A [Claude Code plugin](https://code.claude.com/docs/en/plugins) for **[DataLabs.store](https://datalabs.store)**'s "AI Context Bridge for HubSpot" — the MCP connector that mirrors your HubSpot portal into a real relational SQL Server database, so Claude can run the joins, CTEs, `HAVING` clauses, and window functions that HubSpot's own report builder and native MCP structurally refuse.

This plugin bundles two things:

1. **The live MCP connector** (`https://mcp.datalabs.store/mcp`) — wired up automatically on install via `.mcp.json`. Claude Code will prompt you to authenticate via OAuth 2.0 the first time a tool is used.
2. **A skill** (`skills/hubspot-analytics/`) that teaches Claude the schema, SQL Server dialect gotchas specific to the synced data, and a set of proven analytics recipes (weighted pipeline, forecast calibration, stuck deals, engagement-vs-win-rate, revenue concentration, email attribution) — so Claude reaches for the right analysis instead of flailing on schema discovery or generating subtly-wrong SQL.

## Prerequisites

A DataLabs.store account with HubSpot sync enabled. If you don't have one yet, authenticating for the first time (see below) takes you to a login screen with a **"Register as a new user"** option — you can start there.

## Install

```
/plugin marketplace add vvinogradoff/AIContextBridgeForHubspot
/plugin install ai-context-bridge-for-hubspot@datalabs-hubspot
/reload-plugins
```

On first use of any tool from this connector, Claude Code will prompt you to sign in through `auth.datalabs.store`.

Once this plugin is approved on Anthropic's community marketplace, it will also be installable as `ai-context-bridge-for-hubspot@claude-community` after `/plugin marketplace add anthropics/claude-plugins-community`.

## What you get

Once connected, ask Claude things like:

- "What's my weighted pipeline by rep this quarter?"
- "Which deals have been stuck in the same stage longest?"
- "Is there a relationship between engagement count and win rate?"
- "Show revenue concentration across my top accounts."

Claude explores the schema, checks data volume before running expensive queries, and explains findings honestly (including small-sample and null-result caveats) rather than dumping a table.

All queries are **read-only, enforced by the database itself** — INSERT/UPDATE/DELETE/DDL are rejected server-side, not just filtered client-side.

## Learn more

- [DataLabs.store](https://datalabs.store) — product site and account signup

## License

MIT — see [LICENSE](LICENSE).
