# Analytics Playbook

These are proven analysis patterns — validated live against real synced HubSpot data (see `Docs/marketing-showcases.md` in the DataLabs repo for the full write-ups, including verbatim errors captured from HubSpot's own native MCP attempting the same questions). Each is triggered by a recognizable class of question, needs a specific join/CTE/window-function shape HubSpot's own query surface refuses, and comes with the honesty caveats to carry into your answer. Treat the SQL below as a **template to adapt** to this portal's actual column/table names (confirm via the schema-discovery tools first, per `schema-reference.md`) — not as literal copy-paste.

For every recipe: after producing a good answer, offer to `save_query` it if the customer is likely to ask again (weekly leaderboard, monthly forecast check, etc.).

---

## 1. Weighted pipeline leaderboard ("who's really worth the most")

**Triggered by:** "rank my reps by pipeline", "who's my best rep", "pipeline by owner"

**Why raw pipeline misleads:** ranking by raw open deal value ignores how likely those deals actually are to close. A rep with a small number of high-probability deals can carry more real expected revenue than a rep with a large pile of long-shot deals.

**Shape:** CTE joining `Deal → PipelineDeal` (for stage `Probability`) `→ Owner` (for name), computing both raw open value and probability-weighted value plus historical win rate, then ranking by each independently.

```sql
WITH base AS (
  SELECT
    ISNULL(o.firstName + ' ' + o.lastName, '(unassigned)') AS owner,
    CAST(d.amount AS FLOAT)      AS amount,
    CAST(p.Probability AS FLOAT) AS prob,
    CASE WHEN p.IsClosed = 0 THEN 1 ELSE 0 END AS is_open,
    CASE WHEN d.hs_is_closed_won = 'true' THEN 1 ELSE 0 END AS won,
    CASE WHEN d.hs_is_closed = 'true'     THEN 1 ELSE 0 END AS closed
  FROM Deal d
  JOIN PipelineDeal p ON p.ID = d.dealstage AND p.Pipeline = d.pipeline
  LEFT JOIN Owner o   ON o.ID = d.OwnerID
)
SELECT owner,
  SUM(is_open) AS open_deals,
  CAST(ROUND(SUM(CASE WHEN is_open=1 THEN amount ELSE 0 END), 0) AS BIGINT)        AS open_value,
  CAST(ROUND(SUM(CASE WHEN is_open=1 THEN amount * prob ELSE 0 END), 0) AS BIGINT) AS weighted_value,
  ROUND(100e0 * SUM(won) / NULLIF(SUM(closed), 0), 1) AS win_rate_pct
FROM base GROUP BY owner ORDER BY open_value DESC;
```

**Present it as:** two rankings side by side (raw vs. weighted) — the rank-swap between them is the finding, not either number alone.

**Why HubSpot can't:** requires joining `Deal` to its stage-probability definition and (separately) to owner records — both JOINs, which HubSpot's report builder and native MCP reject outright.

---

## 2. Forecast calibration ("are our stage probabilities actually true")

**Triggered by:** "is our forecast accurate", "check our stage probabilities", "forecast vs. reality"

**Why it matters:** every deal in a stage is weighted by the stage's *configured* probability in forecast rollups — but if the configured number doesn't match the stage's real historical win rate, the whole forecast is systematically biased, in a direction that's invisible unless someone checks.

**Shape:** aggregate the stage-transition history (`Deal_Stage`) by stage, compute actual win rate, compare to `PipelineDeal.Probability`. Requires a CTE (to aggregate entries before joining to definitions) and a minimum-sample floor (`HAVING`) to avoid noisy small-N stages.

```sql
WITH entered AS (
  SELECT ds.DealID, ds.PipelineStageID, d.pipeline, d.hs_is_closed_won
  FROM Deal_Stage ds JOIN Deal d ON d.ID = ds.DealID
),
agg AS (
  SELECT pipeline, PipelineStageID, COUNT(*) AS deals_entered,
    SUM(CASE WHEN hs_is_closed_won = 'true' THEN 1 ELSE 0 END) AS won
  FROM entered GROUP BY pipeline, PipelineStageID
)
SELECT p.PipelineLabel, p.Label AS stage,
  ROUND(CAST(p.Probability AS FLOAT), 2) AS configured_prob,
  a.deals_entered,
  ROUND(100e0 * a.won / NULLIF(a.deals_entered, 0), 1) AS actual_win_rate_pct
FROM agg a JOIN PipelineDeal p ON p.ID = a.PipelineStageID AND p.Pipeline = a.pipeline
WHERE p.IsClosed = 0 AND a.deals_entered >= 25
ORDER BY p.PipelineLabel, p.DisplayOrder;
```

**Present it as:** a table (or grey-vs-actual bar chart) of configured vs. actual per stage. Flag any stage with a large gap, and especially any stage whose actual win rate is near zero despite a nonzero configured probability — that's dead pipeline inflating the forecast.

**Why HubSpot can't:** `Deal_Stage` (the stage-history table) isn't a first-class object HubSpot's MCP exposes; the aggregation needs a CTE; the sample-size floor needs `HAVING`. All three are refused.

---

## 3. Pipeline hygiene / stuck deals ("what's rotting")

**Triggered by:** "what deals are stuck", "pipeline hygiene", "aging deals", "what's been sitting too long"

**Why it matters:** HubSpot only computes time-in-stage after a deal moves or closes — for *open* deals sitting untouched, there's no native alert or report. This query is the only way to surface a silently-aging backlog.

**Shape:** join open deals to their current (still-open) stage-history row (`hs_v2_date_exited IS NULL`), compute days-in-stage, flag value stuck past a threshold (90 days is a reasonable default; ask the customer if they have a different SLA in mind).

```sql
SELECT p.PipelineLabel, p.Label AS stage,
  COUNT(*) AS open_deals,
  CAST(ROUND(SUM(CAST(d.amount AS FLOAT)), 0) AS BIGINT) AS open_value,
  AVG(DATEDIFF(day, ds.hs_v2_date_entered, GETUTCDATE())) AS avg_days_in_stage,
  CAST(ROUND(SUM(CASE WHEN DATEDIFF(day, ds.hs_v2_date_entered, GETUTCDATE()) > 90
    THEN CAST(d.amount AS FLOAT) ELSE 0 END), 0) AS BIGINT) AS value_stuck_90d_plus
FROM Deal d
JOIN PipelineDeal p ON p.ID = d.dealstage AND p.Pipeline = d.pipeline
JOIN Deal_Stage ds  ON ds.DealID = d.ID AND ds.PipelineStageID = d.dealstage AND ds.hs_v2_date_exited IS NULL
WHERE p.IsClosed = 0
GROUP BY p.PipelineLabel, p.Label
HAVING COUNT(*) >= 10
ORDER BY value_stuck_90d_plus DESC;
```

**Why HubSpot can't:** stage-transition timestamps for still-open deals live only in the history table, joined against current deal state — a JOIN HubSpot's engine refuses.

---

## 4. Engagement vs. outcome ("does more activity actually help")

**Triggered by:** "does more activity help close deals", "engagement vs win rate", "are reps over-engaging"

**Why it matters:** the intuitive assumption ("more touches = more likely to close") is often wrong, and the relationship is frequently **non-monotonic** — a middle band of heavily-worked deals can convert *worse* than both lightly-touched and very-heavily-touched deals. That's a coaching-relevant finding, not just a data point.

**Shape:** count engagements per deal via the bridge table, bucket deals by engagement count, compare win rate and average deal size per bucket. Exclude the zero-engagement bucket explicitly if it's dominated by bulk-imported/legacy records with no logged activity — check with `get_data_statistics` first and say so if you exclude it.

```sql
WITH de AS (
  SELECT DealID, COUNT(*) AS eng_count FROM DealEngagements GROUP BY DealID
),
deal_eng AS (
  SELECT d.ID,
    CASE WHEN d.hs_is_closed_won = 'true' THEN 1 ELSE 0 END AS won,
    CASE WHEN d.hs_is_closed = 'true'     THEN 1 ELSE 0 END AS closed,
    CAST(d.amount AS FLOAT) AS amount,
    ISNULL(de.eng_count, 0) AS eng_count
  FROM Deal d LEFT JOIN de ON de.DealID = d.ID
)
SELECT
  CASE WHEN eng_count BETWEEN 1 AND 5   THEN '1-5 touches'
       WHEN eng_count BETWEEN 6 AND 15  THEN '6-15 touches'
       WHEN eng_count BETWEEN 16 AND 40 THEN '16-40 touches'
       ELSE '41+ touches' END AS engagement_bucket,
  COUNT(*) AS deals, SUM(closed) AS closed_deals, SUM(won) AS won_deals,
  ROUND(100e0 * SUM(won) / NULLIF(SUM(closed), 0), 1) AS win_rate_pct,
  CAST(ROUND(AVG(amount), 0) AS BIGINT) AS avg_deal_size
FROM deal_eng
WHERE eng_count > 0
GROUP BY CASE WHEN eng_count BETWEEN 1 AND 5   THEN '1-5 touches'
              WHEN eng_count BETWEEN 6 AND 15  THEN '6-15 touches'
              WHEN eng_count BETWEEN 16 AND 40 THEN '16-40 touches'
              ELSE '41+ touches' END
ORDER BY MIN(eng_count);
```

**Why HubSpot can't:** `Engagement` is a separate object reached only through a bridge table — two JOINs, and HubSpot's dot-notation cross-object syntax caps at 2 associated object types with no aggregation across that dimension anyway.

---

## 5. Revenue concentration by segment ("where the money actually is")

**Triggered by:** "logo count vs revenue", "customer segments", "where's our ARR concentrated"

**Why it matters:** a logo-count view and a revenue view of the same customer base often tell opposite stories — investment decisions made off logo counts can be pointed at the wrong segment entirely.

**Shape:** bucket companies by size (employee count or another firmographic proxy), compute each bucket's share of both logo count and total ARR using window functions (`SUM(...) OVER()`), so percentages compute in the same pass as the aggregation.

```sql
WITH seg AS (
  SELECT
    CASE WHEN numberofemployees IS NULL OR numberofemployees = 0 THEN '5. Unknown'
         WHEN numberofemployees < 50   THEN '1. SMB (<50)'
         WHEN numberofemployees < 250  THEN '2. Mid (50-249)'
         WHEN numberofemployees < 1000 THEN '3. Upper-Mid (250-999)'
         ELSE '4. Enterprise (1000+)' END AS size_band,
    CAST(<arr_column> AS FLOAT) AS arr   -- confirm the actual ARR/revenue column name for this portal first
  FROM Company WHERE <arr_column> IS NOT NULL AND <arr_column> > 0
)
SELECT size_band, COUNT(*) AS companies,
  CAST(ROUND(SUM(arr), 0) AS BIGINT) AS total_arr,
  ROUND(100e0 * SUM(arr)  / SUM(SUM(arr))  OVER(), 1) AS pct_of_arr,
  ROUND(100e0 * COUNT(*)  / SUM(COUNT(*))  OVER(), 1) AS pct_of_logos
FROM seg GROUP BY size_band ORDER BY size_band;
```

**Why HubSpot can't:** window functions aren't part of HubSpot's SQL dialect; percent-of-total needs a separate query plus manual math without them.

---

## 6. Email-to-first-deal attribution ("which emails actually turn prospects into customers")

**Triggered by:** "which campaigns drive deals", "email attribution", "does opening an email more predict faster conversion"

**Why it matters:** raw reach (opens/clicks) and true downstream conversion routinely rank in **opposite order** — the highest-reach mass sends can convert far worse than small, targeted ones. This requires the full engaged population as the denominator and each contact's entire deal history to establish "first-ever deal," so it can't be narrowed to a sample or split into separately-joined pieces (see caveats below).

**Shape:** first engagement + open-count per (send, contact) from `EmailCampaignEvent`, each contact's first-ever deal date via the association bridge, then a 1:1 match within a conversion window (90 days is a reasonable default).

```sql
WITH eng AS (
  SELECT evt.EmailCampaignID, evt.ContactID,
    MIN(evt.Created) AS first_touch,
    SUM(CASE WHEN evt.Type = 'OPEN' THEN 1 ELSE 0 END) AS opens
  FROM EmailCampaignEvent evt
  WHERE evt.ContactID IS NOT NULL AND evt.Type IN ('OPEN','CLICK')
  GROUP BY evt.EmailCampaignID, evt.ContactID
),
first_deal AS (
  SELECT cd.ContactID, MIN(d.createdate) AS first_deal_date
  FROM ContactDeals cd JOIN Deal d ON d.ID = cd.DealID
  WHERE cd.Name = 'CONTACT_TO_DEAL'   -- verify this association label on your portal first
  GROUP BY cd.ContactID
),
prospects AS (
  SELECT eng.EmailCampaignID, eng.ContactID, eng.opens,
    CASE WHEN fd.first_deal_date > eng.first_touch AND fd.first_deal_date <= DATEADD(day, 90, eng.first_touch)
      THEN DATEDIFF(day, eng.first_touch, fd.first_deal_date) END AS days_to_deal
  FROM eng LEFT JOIN first_deal fd ON fd.ContactID = eng.ContactID
  WHERE fd.first_deal_date IS NULL OR fd.first_deal_date > eng.first_touch
)
SELECT ec.Name AS marketing_email,
  COUNT(*) AS prospects_engaged,
  COUNT(days_to_deal) AS became_opportunities,
  ROUND(100.0 * COUNT(days_to_deal) / NULLIF(COUNT(*), 0), 1) AS conversion_pct,
  ROUND(AVG(CAST(days_to_deal AS FLOAT)), 1) AS avg_days_to_first_deal
FROM prospects p JOIN EmailCampaign ec ON ec.ID = p.EmailCampaignID
GROUP BY ec.Name HAVING COUNT(*) >= 25 ORDER BY conversion_pct DESC;
```

**Honesty notes to carry into the answer:**
- This is association, not causation — a prospect who engaged then created a deal may have already been warming up for other reasons.
- Low absolute conversion counts are normal (a handful of "became opportunities" out of dozens-to-thousands engaged) — don't over-read small differences between adjacent rows in the ranking; the real signal is a large gap (5-20x) between clusters, not precise ordering.
- Don't assume repeat opens predict faster conversion without checking — in at least one validated run this hypothesis did **not** hold (conversion was flat across open-count buckets). Report what the query actually shows, not the intuitive hypothesis.

**Why HubSpot can't:** needs `EmailCampaignEvent → Contact → ContactDeals → Deal` (three joins), a CTE to establish each contact's first-ever deal, a date-difference, and a `HAVING` floor — plus `EmailCampaignEvent` is too large to pull client-side and hand-join.

---

## Not yet buildable on every portal

Discount-vs-outcome (line-item discount % vs. deal win rate and subsequent tenure) and cohorted NRR (subscription cohorts by acquisition date/segment) are architecturally supported by the schema but need populated `LineItem`/`Subscription`/`Payment`/`Discount` data. Check `get_data_statistics` on those tables before attempting either — if they're empty or near-empty, say so rather than presenting a technically-valid-but-meaningless result.
