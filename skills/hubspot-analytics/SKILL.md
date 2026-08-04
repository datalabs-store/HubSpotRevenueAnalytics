---
name: hubspot-analytics
description: Use when the user has connected their DataLabs "AI Context Bridge for HubSpot" MCP connector (tools like execute_query, get_tables_list, get_full_database_schema, get_semantic_metadata, save_query, list_saved_queries pointed at mcp.datalabs.store) and asks analytical questions about their HubSpot data — deals, pipeline, contacts, companies, engagements, email campaigns, or workflows. Also use for schema exploration ("what tables do I have"), SQL generation/debugging against this connector, or reproducing/adapting known revenue-ops analyses (weighted pipeline, forecast calibration, stuck deals, engagement-vs-win-rate, revenue concentration, email attribution).
---

# HubSpot Analytics via DataLabs MCP

This connector ("AI Context Bridge for HubSpot") mirrors the user's HubSpot portal into a real relational SQL Server database and exposes it as MCP tools. The entire value proposition is running **joins, CTEs, HAVING clauses, and window functions** that HubSpot's own report builder and native MCP structurally refuse (see `reference/analytics-playbook.md` for verbatim examples of HubSpot's own rejection messages). Lean into that — don't limit yourself to what a single-object HubSpot report could already do.

## Tool map

| Group | Tools | Use for |
|---|---|---|
| Schema discovery | `get_tables_list`, `get_database_schema_subtree`, `get_full_database_schema`, `get_object_relationships`, `get_data_statistics` | Learning what's in the database before writing SQL |
| Business semantics | `get_semantic_metadata`, `set_semantic_metadata` | Established terminology, categorical-value meanings; contribute back what you learn |
| Query | `execute_query` | Read-only T-SQL, with `parameters`, `maxRows`, `timeoutSec`, `explainPlan` |
| Reuse | `save_query`, `list_saved_queries`, `execute_saved_query` | Persisting and re-running queries the user will likely ask for again |
| Discovery contract | `search`, `fetch` | Present mainly for OpenAI's Company Knowledge contract; on Claude, prefer the tools above directly |

## Golden workflow

1. **Check for an existing saved query first.** `list_saved_queries` — if the user's question matches one already saved, run it with `execute_saved_query` instead of regenerating SQL.
2. **Explore, don't guess.** `get_tables_list` → `get_database_schema_subtree(table)` for the tables in question → `get_object_relationships` to confirm join cardinality. Read `reference/schema-reference.md` for the commonly-present object map, but always verify against the live schema — portal schemas vary.
3. **Check semantics.** `get_semantic_metadata` for any categorical column or business term you're not certain about before filtering on it.
4. **Check scale before writing an expensive query.** `get_data_statistics(table)` for row counts, especially for anything touching `EmailCampaignEvent` or other high-cardinality tables.
5. **Draft SQL following `reference/sql-dialect.md`** — this file has the specific gotchas (string-typed booleans, decimal overflow on `AVG`, null-safe ratios) that make the difference between a query that runs and one that silently returns the wrong thing.
6. **For an expensive-looking query, run `explainPlan: true` first**, then run for real with an explicit `maxRows`/`timeoutSec`.
7. **Explain the finding, not just the numbers** — see `reference/analytics-playbook.md` for the "aha" framing and honesty caveats (association vs. causation, small-sample warnings, honest null results) that make these analyses credible rather than just a table dump.
8. **Offer to `save_query`** anything the user is likely to want again.

## Hard constraints

- **Read-only, enforced by the database itself** — INSERT/UPDATE/DELETE/DDL are rejected server-side. Don't try to work around this; there's no path around it by design.
- **Never invent a table/column name.** Confirm it via a schema tool this session before referencing it in SQL.
- **Don't present an empty or near-empty result as a real finding** — commerce objects (`Subscription`, `Payment`, `Discount`, `LineItem`) are sparse on many portals; check row counts first.

## Reference files (read before non-trivial queries)

- `reference/schema-reference.md` — the commonly-present object map (Deal/pipeline/stage-history, engagements, email, workflows)
- `reference/sql-dialect.md` — SQL Server T-SQL gotchas specific to this synced schema
- `reference/analytics-playbook.md` — proven recipes for the questions this product is built to answer

## Connecting (if not already connected)

This plugin bundles the live MCP connector (`https://mcp.datalabs.store/mcp`), so installing it is enough to register the connection — Claude Code prompts for OAuth 2.0 authentication through `auth.datalabs.store` the first time a tool from this connector is used. Existing DataLabs customers sign in with their portal account; the same login screen has a "Register as a new user" option for anyone trying this for the first time. This skill assumes the connection exists once authenticated — it's about using the tools well, not about setting up the connector itself.
