# HubSpot-Synced Schema Reference

Your database is a relational mirror of your HubSpot portal, synced by DataLabs. **Portal schemas vary** — which tables exist, and the exact names of custom properties, depend on what's enabled/customized on your specific HubSpot account. Treat everything below as a map of *commonly-present* objects to orient yourself faster, never as a substitute for checking the live schema. Always confirm exact table/column names with `get_tables_list`, `get_database_schema_subtree`, or `get_object_relationships` before writing a query that references them — an invented column name fails loudly (the reader role and SQL parser reject unknown identifiers), so there's no silent-wrong-answer risk, but it wastes a round trip.

## Discovery tool order (cheapest first)

1. `get_tables_list` — every table with a one-line business-meaning description. Start here for "what's even in this database."
2. `get_semantic_metadata` — business terminology and categorical-field allowed values contributed by you or DataLabs. Check this before assuming what a code/enum column means.
3. `get_database_schema_subtree(table_name)` — the table you care about plus everything it's related to (columns, types, FKs, sample values). Use this for "I'm about to query Deal and friends" — it's scoped, so it's cheap.
4. `get_object_relationships(table_name?)` — FK cardinality without full column detail. Useful to sanity-check a join before writing it.
5. `get_data_statistics(table_name)` — row counts, null %, cardinality, common value distributions. Use before filtering on a column you haven't inspected, especially before assuming a boolean-like or categorical column's shape.
6. `get_full_database_schema` — **last resort only**. The tool's own description warns it returns large content; it dumps every table's full detail at once. Reach for the scoped alternatives above first.

## Core CRM objects

| Table | What it holds | Notes |
|---|---|---|
| `Deal` | The sales-pipeline object | `dealstage` + `pipeline` together identify the current stage (see `PipelineDeal` below). `amount` is decimal/money — see SQL dialect notes. `hs_is_closed`, `hs_is_closed_won` are HubSpot's boolean-like flags **stored as the strings `'true'`/`'false'`**, not SQL `BIT`. `OwnerID` FKs to `Owner`. `createdate` is the deal's creation timestamp. |
| `Contact` | People | Standard + custom contact properties. |
| `Company` | Organizations | `numberofemployees` is commonly populated. Custom properties (e.g. an ARR-style field) exist under whatever name the portal defined — names are **not** standardized across portals; check the schema before assuming one. |
| `Owner` | HubSpot users assigned to records | `firstName`/`lastName`; a Deal/Contact/Company with no owner has `OwnerID = NULL` — use `LEFT JOIN` and `ISNULL(... , '(unassigned)')`-style handling, not an inner join. |
| `Ticket`, `Lead` | Support/lead objects | Same sync pattern as Deal/Contact/Company; check `get_tables_list` for the exact set enabled on this portal. |

## Pipeline & stage-history objects

| Table | What it holds | Notes |
|---|---|---|
| `PipelineDeal` | Stage **definitions** (not deal instances, despite the name) | Columns include `ID`, `Pipeline`, `Label`, `PipelineLabel`, `Probability` (HubSpot's configured win probability for the stage), `IsClosed`, `DisplayOrder`. Join to `Deal` via `p.ID = d.dealstage AND p.Pipeline = d.pipeline` — **both** columns are needed, since stage IDs are only unique within a pipeline. |
| `Deal_Stage` | Stage-transition **history** — when each deal entered/exited each stage it ever passed through | Columns include `DealID`, `PipelineStageID`, `hs_v2_date_entered`, `hs_v2_date_exited`. A deal's *current* stage row has `hs_v2_date_exited IS NULL`. This is the only place time-in-stage for **open** deals lives — HubSpot's own reporting only computes time-in-stage after a deal moves or closes, so this table is the wedge for pipeline-hygiene analysis. |

## Engagement & association objects

| Table | What it holds | Notes |
|---|---|---|
| `Engagement` | Logged activity (calls, emails, meetings, notes, tasks, communications, postal mail) | A separate object from `Deal`/`Contact` — reach it only through a bridge table. |
| `DealEngagements` | Bridge: which engagements are logged against which deal | One row per (Deal, Engagement) pair. `COUNT(*) GROUP BY DealID` gives a deal's engagement count without needing to join `Engagement` itself if you only need the count. |
| `ContactDeals` | Bridge: which contacts are associated with which deals | Carries a `Name` column labeling the association type. In at least one portal, both `'CONTACT_TO_DEAL'` and a `'..._UNLABELED'` variant exist for what's conceptually the same association — filter to the labeled one (`WHERE cd.Name = 'CONTACT_TO_DEAL'`) to avoid double-counting; verify the exact label set on your portal with `get_data_statistics` or a quick `SELECT DISTINCT Name`. |

## Marketing / email objects

| Table | What it holds | Notes |
|---|---|---|
| `EmailCampaign` | One row per email **send** | `Name`, `Subject`. Some portals also have a separate `MarketingEmail` (the authored asset, linked via `MarketingEmail.PrimaryEmailCampaignId`) — others don't; if `MarketingEmail` isn't in `get_tables_list`, read `Name`/`Subject` directly off `EmailCampaign`. |
| `EmailCampaignEvent` | One row per recipient-level interaction (`OPEN`, `CLICK`, `DELIVERED`, `SENT`, `BOUNCE`, `DROPPED`, `STATUSCHANGE`, `SPAMREPORT`, …) | **The highest-cardinality table in a synced portal** — every send × every recipient × every interaction. Can run into the millions of rows. `ContactID` is nullable (recipient didn't match a known Contact) — filter `WHERE ContactID IS NOT NULL` before joining to Contact-scoped analysis. `Created` is a real `DateTime`. Never pull this table client-side to join by hand; push every join into the SQL query itself (see the SQL dialect notes on row limits). |

## Workflow (automation) objects

| Table | What it holds | Notes |
|---|---|---|
| `Workflow` | A workflow **blueprint** (trigger + action graph definition) | Not a per-run instance — HubSpot has no queryable "workflow run" object. A record *enrolling* in a workflow is state on the record, not a new row here. |
| `WorkflowAction` | The action graph, exploded into rows | One row per action step (`actionId`, `actionTypeId`, connection to next action). |
| `WorkflowPerformance` | Aggregate enrollment/completion counts | Time-bucketed by `Granularity` (DAY/WEEK/MONTH) + `StartDate`. This is where "how well is this workflow performing" lives — not per-record enrollment detail. |
| `WorkflowEmailCampaign` | Bridge: which `EmailCampaign`/`MarketingEmail` rows a workflow references | |

## Commerce objects (sparse on many portals)

`Subscription`, `Payment`, `Discount`, `LineItem` exist in the schema but are frequently near-empty on portals that don't run HubSpot Commerce — always check `get_data_statistics` (row count) before building an analysis that depends on them, and say so plainly if the data isn't there rather than presenting a technically-correct-but-empty result as if it were a finding.

## Custom properties

Any object can carry customer-defined custom properties in addition to the standard fields listed above. Their names follow whatever convention the portal's HubSpot admin chose — there is no universal naming rule to rely on (in particular, don't assume a Salesforce-style `__c` suffix or any other fixed pattern). Discover them the same way as everything else: `get_database_schema_subtree`/`get_full_database_schema` for the object, and `get_semantic_metadata` for what a cryptically-named one actually means.
