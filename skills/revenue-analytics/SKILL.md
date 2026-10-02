---
name: revenue-analytics
description: Use when the user has connected their DataLabs "Revenue Analytics for HubSpot" MCP connector (tools like execute_query, get_tables_list, get_full_database_schema, get_semantic_metadata, save_query, list_saved_queries, search, fetch pointed at mcp.datalabs.store) and asks analytical questions about their HubSpot data - deals, pipeline, contacts, companies, engagements, email campaigns, or workflows. Also use for schema exploration ("what tables do I have"), SQL generation/debugging against this connector, or reproducing/adapting known revenue-ops analyses (weighted pipeline, forecast calibration, stuck deals, engagement-vs-win-rate, revenue concentration, email attribution).
---

# Revenue Analytics

This connector ("Revenue Analytics") mirrors the user's HubSpot portal into a real relational SQL Server database and exposes it as MCP tools. The entire value proposition is running **joins, CTEs, HAVING clauses, and window functions** that HubSpot's standard reporting system has no path for - single-object reports, a 2-object cap on native cross-object dot notation, true multi-object custom reporting gated to Enterprise-tier Data Hub (see `reference/analytics-playbook.md` for the specific limit each recipe falls outside of). Lean into that - don't limit yourself to what a single-object HubSpot report could already do. Never cite a live Breeze (or other native-AI) chat response as proof of a limitation - HubSpot has tuned Breeze to approximate rather than refuse, so it will attempt an answer instead of erroring out; the reporting system's own documented, tier-gated structure is the stable claim, not a chat transcript.

## Tool map

| Group | Tools | Use for |
|---|---|---|
| Semantic discovery | `search`, `fetch` | Vectorized search over everything on file for this customer - every table, business-terminology entry, and saved query - in one call; the fastest starting point when you don't already know exact names. `fetch(id)` pulls a hit's full detail. |
| Schema discovery | `get_tables_list`, `get_database_schema_subtree`, `get_full_database_schema`, `get_object_relationships`, `get_data_statistics` | Going deeper once `search` has pointed you at a table, or confirming exact join cardinality/full DDL |
| Business semantics | `get_semantic_metadata`, `set_semantic_metadata` | Established terminology, categorical-value meanings; contribute back what you learn |
| Query | `execute_query` | Read-only T-SQL, with `parameters`, `maxRows`, `timeoutSec`, `explainPlan` |
| Reuse | `save_query`, `list_saved_queries`, `execute_saved_query` | Persisting and re-running queries the user will likely ask for again |

## Golden workflow

1. **Start with `search`.** `search(query)` runs a semantic match against every table, business-terminology entry, and saved query on file for this customer in a single call - it's the most effective way to locate the right starting point, especially before you know exact table/column names or whether this question has already been answered. Use `fetch(id)` on a promising hit to pull its full detail (full schema for a table hit, full definition for a saved-query hit, full text for a metadata hit).
2. **Prefer an exact saved query when one exists.** If `search` surfaces a saved query matching the user's question (or `list_saved_queries` does), run it with `execute_saved_query` instead of regenerating SQL.
3. **Go deeper on structure from whatever `search` surfaced.** `get_database_schema_subtree(table)` for full column detail, `get_object_relationships` to confirm join cardinality. Read `reference/schema-reference.md` for the commonly-present object map, but always verify against the live schema - portal schemas vary. Fall back to `get_tables_list`/`get_full_database_schema` only if `search` didn't return a usable lead.
4. **Check semantics.** `get_semantic_metadata` for any categorical column or business term you're not certain about before filtering on it - `search` may already have surfaced the relevant metadata entry.
5. **Check scale before writing an expensive query.** `get_data_statistics(table)` for row counts, especially for anything touching `EmailCampaignEvent` or other high-cardinality tables.
6. **Draft SQL following `reference/sql-dialect.md`** - this file has the specific gotchas (string-typed booleans, decimal overflow on `AVG`, null-safe ratios) that make the difference between a query that runs and one that silently returns the wrong thing.
7. **For an expensive-looking query, run `explainPlan: true` first**, then run for real with an explicit `maxRows`/`timeoutSec`.
8. **Explain the finding, not just the numbers** - see `reference/analytics-playbook.md` for the "aha" framing and honesty caveats (association vs. causation, small-sample warnings, honest null results) that make these analyses credible rather than just a table dump.
9. **Offer to `save_query`** anything the user is likely to want again.

## Hard constraints

- **Read-only, enforced by the database itself** - INSERT/UPDATE/DELETE/DDL are rejected server-side. Don't try to work around this; there's no path around it by design.
- **Never invent a table/column name.** Confirm it via a schema tool this session before referencing it in SQL.
- **Don't present an empty or near-empty result as a real finding** - commerce objects (`Subscription`, `Payment`, `Discount`, `LineItem`) are sparse on many portals; check row counts first.

## Reference files (read before non-trivial queries)

- `reference/schema-reference.md` - the commonly-present object map (Deal/pipeline/stage-history, engagements, email, workflows)
- `reference/sql-dialect.md` - SQL Server T-SQL gotchas specific to this synced schema
- `reference/analytics-playbook.md` - proven recipes for the questions this product is built to answer

## Connecting (if not already connected)

This plugin bundles the live MCP connector (`https://mcp.datalabs.store/mcp`), so installing it is enough to register the connection - Claude Code prompts for OAuth 2.0 authentication through `auth.datalabs.store` the first time a tool from this connector is used. Existing DataLabs customers sign in with their portal account; the same login screen has a "Register as a new user" option for anyone trying this for the first time. This skill assumes the connection exists once authenticated - it's about using the tools well, not about setting up the connector itself.
