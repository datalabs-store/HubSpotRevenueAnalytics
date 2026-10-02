# SQL Dialect & Execution Notes

`execute_query` runs real T-SQL against a SQL Server (Microsoft SQL Server / Azure SQL) database, through a **read-only** database role. This is the entire point of the product: full joins, CTEs, `HAVING`, window functions, subqueries - everything HubSpot's standard reporting system has no path for (single-object reports, a 2-object cap on native cross-object dot notation, no subqueries/UNION/DISTINCT). Use them freely; don't hand-roll workarounds HubSpot forces you into elsewhere. Note this against the reporting system's own documented limits, not against a live Breeze chat response - Breeze is tuned to approximate rather than refuse, so it will attempt an answer instead of erroring out.

## Read-only is enforced by the database, not just checked client-side

INSERT/UPDATE/DELETE/DDL and any other write statement are rejected by SQL Server itself (the connection uses a read-only login), not merely blocked by client-side validation. If a write is rejected, don't retry with a different phrasing or a system stored procedure - there is no path around it, by design (see product decision: MCP never allows write operations against customer data).

## Known gotchas (learned the hard way - apply proactively)

- **Boolean-like flags are strings, not `BIT`.** Fields like `Deal.hs_is_closed`, `Deal.hs_is_closed_won` store the literal strings `'true'`/`'false'`. Compare with `= 'true'` / `= 'false'`, never `= 1`/`= 0`. This pattern is common across HubSpot-sourced boolean properties generally - if a `WHERE`/`CASE` on a flag column returns nothing when you expect rows, suspect this first.
- **Decimal/money columns can overflow on aggregation.** `AVG()` (and other aggregates) on columns like `Deal.amount` can throw an arithmetic-overflow error. Wrap with `CAST(amount AS FLOAT)` before aggregating: `AVG(CAST(amount AS FLOAT))`, `SUM(CAST(amount AS FLOAT) * prob)`, etc. Apply this defensively any time you're aggregating a money/decimal-typed column, not only after hitting the error once.
- **Null-safe ratios.** Use `NULLIF(denominator, 0)` inside percentage/rate calculations (`100.0 * won / NULLIF(closed, 0)`) so a zero-count bucket returns `NULL` instead of erroring or divide-by-zero-ing.
- **Unassigned/missing FKs are common, not exceptional.** `Deal.OwnerID`, `Contact`/`Company` associations, etc. can be `NULL`. Default to `LEFT JOIN` when the presence of the related row isn't guaranteed, and handle the null case explicitly (`ISNULL(o.firstName + ' ' + o.lastName, '(unassigned)')`) rather than silently dropping unassigned rows via an inner join.
- **`TOP N`, not `LIMIT`.** This is SQL Server T-SQL.
- **Date arithmetic**: `DATEDIFF(day, start_col, end_col)`, current time via `GETUTCDATE()`.
- **CTEs (`WITH ... AS (...)`) are fully supported** and are the idiomatic way to stage an intermediate aggregation before a final `SELECT`/`HAVING` - use them instead of nested subqueries when a step needs its own `GROUP BY`.
- **Window functions are fully supported** - `SUM(x) OVER()`, `SUM(x) OVER(PARTITION BY ...)` etc. - the standard way to compute a row's share of a total (percent-of-total, running totals) in one pass instead of a second query plus manual division.

## Row limits, timeouts, and cost control

- `execute_query` accepts optional `maxRows` and `timeoutSec` - set these deliberately (rather than relying on defaults) when exploring an unfamiliar or high-cardinality table, especially `EmailCampaignEvent` or any table `get_data_statistics` reports as large.
- `explainPlan: true` returns the estimated query plan **instead of running the query** - use it to sanity-check an expensive-looking query (multiple joins over large tables) before committing to a full run.
- Never guess at scale. Check `get_data_statistics` for a table's row count before writing a query you expect to be cheap - "this should be fast" is a guess, a stats check is a fact.

## Identifier discipline

Never invent a table or column name from a guess about HubSpot's public API property names - the synced schema's actual column names are what matters, and they don't always match HubSpot's API field names one-for-one. Confirm via the schema-discovery tools (see `schema-reference.md`) before referencing an identifier you haven't seen in a schema response this session.

## Saving useful queries

Once a query proves useful for this customer's specific analysis, offer to `save_query` it (with a clear `tool_name`/`description` and any parameters extracted as `@param` placeholders) so it's available next time via `list_saved_queries` / `execute_saved_query` without regenerating the SQL from scratch. This is especially worth doing for anything from `analytics-playbook.md` that the customer asks for more than once.
