# Weekly Search Health Report — Routine Run Prompt

This is a **scheduled weekly routine** that runs every Monday morning. It produces the Global Search 7-day health report covering Mon-Sun of the prior week, publishes it to Confluence, and outputs a Slack share message.

This file is the **self-contained prompt** for the scheduled task. The task runs fresh with no prior context, so everything needed is below.

---

## 0. What this routine does

1. Computes the analysis window: yesterday-back-7-days through yesterday (Mon-Sun).
2. Pulls search, click, and trade data from Redshift for that window.
3. Pulls per-canonical-query click→trade data.
4. Pulls long-tail (5-49 searches per query) with verbatim queries + top clicked entities.
5. Spawns a sub-agent to cluster the long-tail.
6. Generates a `.docx` report + a long-tail `.md` companion file.
7. Publishes the report contents as a **Confluence page** in the Product space.
8. Writes a **Slack share message** to a file the PM can paste manually (Slack MCP not yet wired — see §Publishing).

Total expected runtime: ~15-25 minutes (heaviest steps are the Redshift CTE rebuilds and the long-tail sub-agent clustering).

---

## 1. Run-time inputs (compute at start)

```python
from datetime import date, timedelta
TODAY = date.today()
WINDOW_END   = TODAY - timedelta(days=1)   # yesterday (Sunday if run on Monday)
WINDOW_START = TODAY - timedelta(days=7)   # last Monday
WINDOW_LABEL = f"{WINDOW_START} to {WINDOW_END}"
DATE_TAG     = TODAY.isoformat()            # used in filenames

OUTPUT_DIR              = "/Users/shruti.kumari/analytics_project"
DOCX_PATH               = f"{OUTPUT_DIR}/search_health_{DATE_TAG}.docx"
LONG_TAIL_MD_PATH       = f"{OUTPUT_DIR}/long_tail_enriched_{WINDOW_START}_{WINDOW_END}.md"
SLACK_MESSAGE_FILE_PATH = f"{OUTPUT_DIR}/slack_share_{DATE_TAG}.txt"
```

---

## 2. Data sources

**Cluster + database:**
- Redshift cluster: `redshift-cluster-2`
- Database: `indwealth`
- Connector tool: `mcp__445f9cb2-03c8-42cd-bce9-7131b7e14a9f__execute_query`
- Role is read-only (`IAMR:mcp-gateway-role`). No temp tables, no staging tables — every analysis SQL must rebuild the dedup pipeline as CTE from scratch.

**Tables / event names:**
| Source | What | Event-name filter |
|---|---|---|
| `di.mixpanel_all_events` | All Mixpanel events. Filter by `event_name`. | — |
| ↳ Search events | Raw search-typed events | `global_search_keyword_searched_backend_event` |
| ↳ Click events | Search-result clicks | `gs_all` AND `properties.event_type = 'card_clicked'` AND `COALESCE(properties.event_sub_type,'') NOT IN ('watchlist','watchlist_clicked')` AND query non-empty |
| ↳ FnO trades | F&O orders | `fno_order_executed_backend_event` |
| ↳ Indian-stocks trades | CNC/MTF/intraday combined | `indstocks_order_executed_backend_event` |
| ↳ US-stocks trades | US trade placements | `usstocks_trade_placed_backend_event` |
| ↳ MF trades | MF order statuses | `mutualfunds_order_status_backend_event` |
| `di.gloabl_seach_search_query_merging` | Small index/derivative alias overlay (78 rows, 7 entities only). **NOT the master alias system** — pod-supplied catalogs are the real alias source. | LEFT JOIN `LOWER(TRIM(query)) = m.query` on search/click side |

**Event that does NOT fire (do NOT depend on it):**
- `global_search_opened_backend_event` — referenced in the original prompt for a 4-stage funnel; collapse to 3-stage (typed → clicked → traded).

---

## 3. Architecture context the report must respect

### 3.1 Pod-supplied catalogs (CRITICAL — affects ownership routing in §3 actionables)
The master alias system is **per-pod**, NOT centralized in the merging table:
- Indian Stocks pod owns Indian listed entities + Indian ETFs + Indian commodities (crude oil, gold ETFs, etc.)
- US Stocks pod owns US listed entities (incl. private-co disambiguation cards like Anthropic / SpaceX / Tenstorrent)
- MF pod owns mutual funds + AMC names + fund-house catalogs
- F&O / Commodities owns derivative contracts + strike resolution

When proposing an alias-gap or catalog-gap fix, route owner to the specific pod that supplies that catalog. Do NOT route to a generic "Catalog" owner; do NOT recommend backfilling the merging table.

### 3.2 Diagnose-first rule (CRITICAL — affects §3 voice)
Any P0/P1 finding that does NOT have a verified diagnostic basis ships as **"investigate, then prescribe"** — NOT "do X." Specifically:
- A "high rank depth" finding does NOT automatically mean "pin the entity at rank 1." First verify what's at rank 1-5 today (Quick Link? Collection? Disambiguation? Already-pinned card?).
- A "low CTR / 0 clicks" finding does NOT automatically mean "search is broken." First check whether the click filter `event_type='card_clicked'` is excluding Quick Link / Collection clicks (see §5.4).
- A "high click→trade" finding is real signal but the **trade lift estimate** must be a range, not a point — multiplying three optimistic assumptions yields a fictional number.

Each P0/P1 actionable in §3 ends with a "Diagnose" subsection and a "Hypothesis (subject to confirmation)" — only direct fixes with clear single-edit answers (e.g., `pharma bees` ranker bug) get prescribed without a diagnose step.

### 3.3 Long-tail is the PRIMARY focus
Per the original prompt's Task 3: long-tail queries (<50 searches each) collectively represent ~37% of platform volume and carry the highest signal for emerging entities, catalog gaps, and unmet intent. The head-only view (top 200 by volume) misses this entirely. The routine MUST include §4 + §5 long-tail clustering with verbatim queries.

---

## 4. Methodology — what's applied vs what's a known caveat

### 4.1 Applied (no editorial framing on these — they're settled)

| # | Method | Implementation |
|---|---|---|
| 1 | **Zero-result split** | `hard_zero` = `result=false AND semantic_result=false` (vanishingly rare, ~0.009%). `semantic_fallback` = `result=false AND semantic_result=true` (~3-4%). Do NOT conflate these as a single "zero-result" rate. |
| 2 | **Blended CTR removed from §2 headline** | "Blended CTR = clicks / searches × 100" is a click-density metric, not a true CTR (no session pairing, multi-click sessions inflate numerator). Do NOT report it. Replace with the explanatory note: "Session-paired CTR pending — see Appendix B." |
| 3 | **Cross-filter mismatch denominator** | Numerator = clicks where `filter ≠ 'ALL' AND filter NOT NULL AND vertical NOT NULL AND filter ≠ vertical`. Denominator = clicks where `filter ≠ 'ALL' AND filter NOT NULL` (NOT total clicks). Reason: ALL-tab clicks can't ever be cross-filter mismatches by definition. |
| 4 | **Pod-ownership re-routing** | Every alias-gap / catalog-gap finding routes to a specific pod (Indian Stocks / US Stocks / MF / F&O). NO finding routes to "Catalog" or "Search Infra (alias)" generically. |
| 5 | **Strike attribution** | Nifty strikes = 22000-26999; BankNifty strikes = 50000-55999; Sensex strikes = 70000-80000. Do not confuse Sensex strikes (75xxx) with BankNifty as v1/v2 did. |
| 6 | **Diagnose-first** | See §3.2. |
| 7 | **Dedup ratio** | Report the observed number with no "below expected ~56% baseline" editorial framing — that baseline was never empirically validated. |
| 8 | **Trade-conversion = user-day proxy** | A search is "converted" if the same user_id traded on the same calendar day OR the next calendar day. NOT the spec's strict "within 24h of search timestamp" — the strict version times out under read-only API. Note this looseness as a §7 caveat. Expect ~5-10% absolute inflation. |
| 9 | **/day metrics** | Are 7-day daily averages (cluster total ÷ 7). NOT the latest day's number. NOT the 7-day sum. |
| 9a | **Hard rule on time-unit consistency in cluster-summary tables** | The §4 cluster summary table MUST use the SAME time unit for both `Searches` and `Clicks` columns. Mixing `Searches/day` (daily avg) with `Total clicks` (7-day sum) is a rendering bug that makes weekly clicks visually look larger than daily searches — even though weekly clicks are typically half of weekly searches. **Default: both as `/day` (daily averages, total ÷ 7).** Per-query tables in §5 use 7-day totals because individual query volumes are too small for daily averages to read cleanly (66 searches/wk vs 9.4 sw/day) — but those columns MUST be explicitly labelled as 7-day totals. |
| 9b | **Hard rule on CTR computation in cluster-summary tables** | The CTR column in §4 MUST be the cluster-level honest rate: `total cluster clicks / total cluster searches × 100`. NOT the simple average of per-query CTRs (which weights all queries equally regardless of volume and produces misleading numbers when query volumes vary widely within the cluster). The same rule applies to click→trade%. If both methodologies are useful for context, label them separately as `Cluster CTR %` and `Avg per-query CTR %`. |
| 9c | **Hard rule on search-side filter context (MANDATORY per cluster)** | Every cluster narrative in §5 MUST report the **filter the user was on when typing the query** (search-side filter), not just the dominant click-vertical. This is a separate dimension from "where the click landed" — and it explains a lot of "low CTR" mysteries. **Field:** the search event's `properties.source` is the filter tab on the search side (verified — values are `IN_STOCKS`, `IN_STOCKS_FNO`, `US_STOCKS`, `MF`, `ALL`, `SIF`). Note this is DIFFERENT from the click event's `properties.filter` (same conceptual thing, different field name) and from the search event's other "source" semantics (acquisition source = different sub-field). Each cluster MUST include a "Search-side filter mix" line showing top-3 filters with %. Flag clusters where a substantial fraction of searches happen on the WRONG filter (e.g., LT-11 Tata-AMC has 72% IN_STOCKS / 4% MF — the MF intent is being expressed from the IN_STOCKS tab, explaining the wrong-entity routing). |
| 9d | **Hard rule on "SILENT FAIL" / "0 clicks" framing** | Do NOT label any cluster row as "SILENT FAIL" or "no card surfaced" without independently verifying the search UI does not render a usable result. The current pipeline filters clicks to `event_type='card_clicked'` — Quick Link / Collection / Trending / suggestion-tap / option-chain-open clicks may fire under other event_types and be invisible. PM has confirmed via app-side check that strike queries (e.g., `23650 ce`) DO render results. Therefore the right framing for any "0 captured clicks" row is **"0 captured clicks under `event_type='card_clicked'` — likely a click-filter artefact (see deferred fix #9 in Appendix B). Click likely fires under a different event_type for FnO contract cards / Quick Links / Collections."** Only after deferred fix #9 (widen click filter) lands can we identify TRUE silent-fail rows with confidence. |
| 9e | **Hard rule on intent assertion** | NEVER assert what intent a user "wanted" without evidence. Phrases like "user wants the MF fund" or "user is looking for the stock" are off-limits unless backed by data (e.g., they later clicked on that exact entity, or the typed query is unambiguous like `nifty pharma etf`). For AMC-prefix queries like `tata inv` / `nippon l` / `axis glob` the user could reasonably want (a) the matching listed stock (Tata Investment Corp, Nippon Life India AMC), (b) a fund from that house, (c) browse mode. The CLICK data shows WHAT they landed on but not what they originally intended. When discussing wrong-routing or low-conversion patterns, frame as "consistent with multiple hypotheses; needs cohort-split / intent-survey diagnostic to disambiguate" rather than asserting a single intent. The narrative should say "the data shows X" not "the user wanted Y." |
| 9f | **Hard rule on populating every column from the source file** | Before publishing, every per-query table in §5 MUST have the `click→trade %` column populated from the long-tail enriched markdown file for every query that has clicks ≥ 3. The placeholder `—` is allowed ONLY when clicks = 0 (no trade-conversion can be computed for unclicked queries). Recurring rendering bug across v3.0-v3.3: I shipped pages with "—" everywhere in click→trade columns despite the source file having the values — never repeat this. Sanity check before publishing: visually scan each cluster's table, confirm click→trade is populated wherever clicks > 0. |
| 10 | **`{{REPORT_DATE}} = run date`** | Window = REPORT_DATE-7 to REPORT_DATE-1 inclusive. |

### 4.2 Caveats to surface in §7 every run

The routine MUST surface these in §7 (Data Caveats) every run until the deferred fix is applied:

| # | Caveat |
|---|---|
| A | CTR is click-density (clicks ÷ searches), NOT session-paired CTR. Multi-click sessions inflate it; CTR > 100% is common. |
| B | Trade attribution uses user-day proxy, not search-timestamp + 24h join (see §4.1 #8). |
| C | Cross-filter only catches click-side scenarios. Search-side mismatches (user searches Tesla on IN_STOCKS, sees nothing, doesn't click) are invisible. |
| D | Click filter `event_type='card_clicked'` may exclude Quick Link / Collection / Trending / Recent-search clicks if those use different event_types. Any "high searches / 0 clicks" finding may be artefact, not real silent-fail. |
| E | Avg click rank is unpaired-average. Multi-click sessions over-weight; cross-filter rank averaging is conflated. |
| F | Long-tail coverage is the top ~2,000 of ~28K long-tail queries (~7% by count, ~50-60% by volume). Super-tail (1-4 searches/query, 17.7% of platform volume) is not analyzed. |
| G | Cohort segmentation uses tab-filter proxy, not user × trade-history join. |
| H | First scheduled run on a given window has no prior-week baseline for "new-this-week" / LT-9 classification. |
| I | LT-17 (catalog gaps zero-click) count may be inflated by the `clicks BETWEEN 3 AND 50` threshold — queries with 1-2 clicks aren't in the click file and are inferred as 0-click. Treat LT-17's count as an upper bound. |
| J | `global_search_opened_backend_event` does not fire in warehouse. Funnel is 3-stage not 4-stage. |
| K | Filter tab `SIF` is excluded per scope decision. |

### 4.3 Deferred fixes (still pending — track and surface in §Appendix B)

| # | Fix | When applied | Where it changes the report |
|---|---|---|---|
| F1 | Recompute CTR session-wise via `ind_session_id` | When session-join works under the read-only API | §2 headline CTR will be the real rate (likely 50-80% range, not the ~100% click-density we currently hide) |
| F2 | Cross-filter Mixpanel funnels for X→Y mismatch pairs | When Mixpanel MCP funnel-create is available | §3 P0 strike finding + tesla-on-IN_STOCKS narrative |
| F3 | Rebuild avg click rank session-wise (paired to search, first-click-per-session, filter-stratified) | F1's prerequisite is session-id on the search event too | §2 avg click rank + §3 P2 thematic rank-depth |
| F4 | Widen click filter to include Quick Link / Collection / Trending / Recent-search clicks | Need event_type inventory + confirmation from Analytics | Many §3 "0 clicks" findings (Anthropic, kotak, etc.) re-evaluate |
| F5 | Re-validate thematic recommendations against existing Quick Link & Collection coverage | Need UI inspection + cross-reference with Collections team | §3 P2 (build vs improve vs delete) |
| F6 | Tighten LT-17 catalog-gap count via clicks≥1 re-pull | Easy SQL change, defer to keep run lean | §3 P0-D upper-bound becomes a verified number |
| F7 | Extend long-tail coverage (top 2,000 → 10,000 + super-tail aggregation pass) | When run-time budget allows | §4 / §5 coverage |

---

## 5. SQL queries the routine must run

### 5.1 Headline metrics (split zero-result)
```sql
WITH base AS (
  SELECT mp_user_id,
         LOWER(TRIM(properties.query::varchar)) AS query,
         TIMESTAMP 'epoch' + properties.time::bigint * INTERVAL '1 second' AS event_time,
         mp_date::date AS d,
         properties.result AS has_result,
         properties.semantic_result AS semantic_result
  FROM di.mixpanel_all_events
  WHERE event_name = 'global_search_keyword_searched_backend_event'
    AND mp_date::date BETWEEN '{WINDOW_START}' AND '{WINDOW_END}'
    AND LEN(properties.query::varchar) > 3
),
lagged AS (SELECT *, EXTRACT(EPOCH FROM (event_time - LAG(event_time) OVER (PARTITION BY mp_user_id, d ORDER BY event_time))) AS td FROM base),
grouped AS (SELECT *, SUM(CASE WHEN td > 1 OR td IS NULL THEN 1 ELSE 0 END) OVER (PARTITION BY mp_user_id, d ORDER BY event_time ROWS UNBOUNDED PRECEDING) AS sg FROM lagged),
dedup AS (SELECT *, ROW_NUMBER() OVER (PARTITION BY mp_user_id, d, sg ORDER BY event_time DESC) AS rn FROM grouped)
SELECT
  COUNT(*) AS deduped_searches,
  COUNT(DISTINCT mp_user_id) AS unique_users,
  SUM(CASE WHEN has_result = false AND semantic_result = false THEN 1 ELSE 0 END) AS hard_zero_result,
  SUM(CASE WHEN has_result = false AND semantic_result = true  THEN 1 ELSE 0 END) AS semantic_fallback,
  SUM(CASE WHEN has_result = false THEN 1 ELSE 0 END) AS any_primary_miss
FROM dedup WHERE rn = 1;
```

### 5.2 Click metrics (with corrected cross-filter denominator)
```sql
SELECT
  COUNT(*) AS clicks_excl_watchlist,
  SUM(CASE WHEN filter_tab <> 'ALL' AND filter_tab IS NOT NULL AND vertical IS NOT NULL AND filter_tab <> vertical THEN 1 ELSE 0 END) AS cross_filter_mismatches,
  SUM(CASE WHEN filter_tab <> 'ALL' AND filter_tab IS NOT NULL THEN 1 ELSE 0 END) AS non_all_clicks_denom,
  AVG(over_all_rank::float) AS avg_click_rank,
  SUM(CASE WHEN over_all_rank > 5 THEN 1 ELSE 0 END) AS clicks_rank_gt5,
  COUNT(DISTINCT mp_user_id) AS uniq_clickers
FROM (
  SELECT properties.mp_user_id::varchar AS mp_user_id,
         properties.filter::varchar AS filter_tab,
         properties.vertical::varchar AS vertical,
         NULLIF(properties.over_all_rank::varchar,'')::int AS over_all_rank
  FROM di.mixpanel_all_events
  WHERE event_name = 'gs_all'
    AND mp_date::date BETWEEN '{WINDOW_START}' AND '{WINDOW_END}'
    AND properties.event_type::varchar = 'card_clicked'
    AND COALESCE(properties.event_sub_type::varchar,'') NOT IN ('watchlist','watchlist_clicked')
    AND properties.query::varchar IS NOT NULL AND properties.query::varchar <> ''
) c;
```

### 5.3 Trade attribution (search-attributed trades by segment)
```sql
SELECT t.event_name,
       COUNT(*) AS total_trades,
       SUM(CASE WHEN s_same.mp_user_id IS NOT NULL OR s_prev.mp_user_id IS NOT NULL THEN 1 ELSE 0 END) AS search_attributed_trades
FROM (
  SELECT properties.mp_user_id::varchar AS mp_user_id, mp_date::date AS td, event_name
  FROM di.mixpanel_all_events
  WHERE event_name IN ('fno_order_executed_backend_event','indstocks_order_executed_backend_event','usstocks_trade_placed_backend_event','mutualfunds_order_status_backend_event')
    AND mp_date::date BETWEEN '{WINDOW_START}' AND '{WINDOW_END}'
) t
LEFT JOIN (SELECT DISTINCT properties.mp_user_id::varchar AS mp_user_id, mp_date::date AS sd
           FROM di.mixpanel_all_events
           WHERE event_name = 'global_search_keyword_searched_backend_event'
             AND mp_date::date BETWEEN '{WINDOW_START - 1d}' AND '{WINDOW_END}' AND LEN(properties.query::varchar) > 3) s_same
  ON s_same.mp_user_id = t.mp_user_id AND s_same.sd = t.td
LEFT JOIN (SELECT DISTINCT properties.mp_user_id::varchar AS mp_user_id, mp_date::date AS sd
           FROM di.mixpanel_all_events
           WHERE event_name = 'global_search_keyword_searched_backend_event'
             AND mp_date::date BETWEEN '{WINDOW_START - 1d}' AND '{WINDOW_END}' AND LEN(properties.query::varchar) > 3) s_prev
  ON s_prev.mp_user_id = t.mp_user_id AND s_prev.sd = (t.td - 1)
GROUP BY 1 ORDER BY 2 DESC;
```

### 5.4 Per-canonical-query rollup for the head (top-500 by search)
```sql
WITH base AS (...same dedup pipeline as 5.1...),
canon AS (SELECT COALESCE(m.merge_string, d.query) AS canonical_query, d.mp_user_id, d.has_result, d.semantic_result
          FROM dedup d LEFT JOIN di.gloabl_seach_search_query_merging m ON d.query = m.query WHERE d.rn = 1)
SELECT canonical_query, COUNT(*) AS searches, COUNT(DISTINCT mp_user_id) AS uniq_users,
       SUM(CASE WHEN has_result = false AND semantic_result = false THEN 1 ELSE 0 END) AS hard_zero,
       SUM(CASE WHEN has_result = false AND semantic_result = true  THEN 1 ELSE 0 END) AS semantic_fallback
FROM canon GROUP BY 1 ORDER BY 2 DESC LIMIT 500;
```

### 5.5 Long-tail canonical rollup (5-49 searches each) — uses raw events, not dedup
The full LAG dedup CTE times out at this query scale. Use raw events; ~26% inflation acceptable for pattern detection.
```sql
SELECT COALESCE(m.merge_string, LOWER(TRIM(e.properties.query::varchar))) AS canonical_query,
       COUNT(*) AS raw_searches, COUNT(DISTINCT e.properties.mp_user_id::varchar) AS uniq_users,
       SUM(CASE WHEN e.properties.result = false AND e.properties.semantic_result = false THEN 1 ELSE 0 END) AS hard_zero,
       SUM(CASE WHEN e.properties.result = false AND e.properties.semantic_result = true  THEN 1 ELSE 0 END) AS semantic_fallback
FROM di.mixpanel_all_events e
LEFT JOIN di.gloabl_seach_search_query_merging m ON LOWER(TRIM(e.properties.query::varchar)) = m.query
WHERE event_name = 'global_search_keyword_searched_backend_event'
  AND mp_date::date BETWEEN '{WINDOW_START}' AND '{WINDOW_END}' AND LEN(e.properties.query::varchar) > 3
GROUP BY 1 HAVING COUNT(*) BETWEEN 7 AND 66 ORDER BY 2 DESC LIMIT 2000;
```
*(7-66 raw covers the 5-49 deduped range with the ~26% inflation factor.)*

### 5.6 Long-tail click rollup with top entity per query
```sql
WITH clicks AS (
  SELECT COALESCE(m.merge_string, LOWER(TRIM(e.properties.query::varchar))) AS canonical_query,
         e.properties.name::varchar AS entity_name, e.properties.vertical::varchar AS vertical,
         NULLIF(e.properties.over_all_rank::varchar,'')::int AS over_all_rank
  FROM di.mixpanel_all_events e
  LEFT JOIN di.gloabl_seach_search_query_merging m ON LOWER(TRIM(e.properties.query::varchar)) = m.query
  WHERE event_name = 'gs_all' AND mp_date::date BETWEEN '{WINDOW_START}' AND '{WINDOW_END}'
    AND e.properties.event_type::varchar = 'card_clicked'
    AND COALESCE(e.properties.event_sub_type::varchar,'') NOT IN ('watchlist','watchlist_clicked')
    AND e.properties.query::varchar IS NOT NULL AND e.properties.query::varchar <> ''
),
totals AS (SELECT canonical_query, COUNT(*) AS total_clicks, AVG(over_all_rank::float) AS avg_rank FROM clicks GROUP BY 1),
top_entity AS (SELECT canonical_query, entity_name, vertical, entity_clicks
               FROM (SELECT canonical_query, entity_name, vertical, COUNT(*) AS entity_clicks,
                            ROW_NUMBER() OVER (PARTITION BY canonical_query ORDER BY COUNT(*) DESC) AS rn
                     FROM clicks GROUP BY 1,2,3) x WHERE rn = 1)
SELECT t.canonical_query, t.total_clicks, ROUND(t.avg_rank::numeric, 2) AS avg_rank,
       te.entity_name, te.vertical, te.entity_clicks
FROM totals t LEFT JOIN top_entity te ON te.canonical_query = t.canonical_query
WHERE t.total_clicks BETWEEN 3 AND 50 ORDER BY t.total_clicks DESC LIMIT 3000;
```

### 5.7 Per-canonical click→trade (for long-tail enrichment)
```sql
WITH clicks AS (
  SELECT COALESCE(m.merge_string, LOWER(TRIM(e.properties.query::varchar))) AS canonical_query,
         e.properties.mp_user_id::varchar AS mp_user_id, e.mp_date::date AS d
  FROM di.mixpanel_all_events e
  LEFT JOIN di.gloabl_seach_search_query_merging m ON LOWER(TRIM(e.properties.query::varchar)) = m.query
  WHERE event_name = 'gs_all' AND mp_date::date BETWEEN '{WINDOW_START}' AND '{WINDOW_END}'
    AND e.properties.event_type::varchar = 'card_clicked'
    AND COALESCE(e.properties.event_sub_type::varchar,'') NOT IN ('watchlist','watchlist_clicked')
    AND e.properties.query::varchar IS NOT NULL AND e.properties.query::varchar <> ''
),
trade_user_days AS (
  SELECT DISTINCT properties.mp_user_id::varchar AS mp_user_id, mp_date::date AS td
  FROM di.mixpanel_all_events
  WHERE event_name IN ('fno_order_executed_backend_event','indstocks_order_executed_backend_event','usstocks_trade_placed_backend_event','mutualfunds_order_status_backend_event')
    AND mp_date::date BETWEEN '{WINDOW_START}' AND '{WINDOW_END + 1d}'
)
SELECT canonical_query, COUNT(*) AS clicks,
       SUM(CASE WHEN ts.mp_user_id IS NOT NULL OR tn.mp_user_id IS NOT NULL THEN 1 ELSE 0 END) AS converted_clicks,
       ROUND(100.0 * SUM(CASE WHEN ts.mp_user_id IS NOT NULL OR tn.mp_user_id IS NOT NULL THEN 1 ELSE 0 END) / NULLIF(COUNT(*),0), 1) AS click_to_trade_pct
FROM clicks c
LEFT JOIN trade_user_days ts ON ts.mp_user_id = c.mp_user_id AND ts.td = c.d
LEFT JOIN trade_user_days tn ON tn.mp_user_id = c.mp_user_id AND tn.td = c.d + 1
GROUP BY 1 HAVING COUNT(*) BETWEEN 3 AND 50 ORDER BY clicks DESC;
```

### 5.8 Merging-table verification (Appendix A)
```sql
SELECT merge_string, COUNT(*) AS variant_count FROM di.gloabl_seach_search_query_merging GROUP BY 1 ORDER BY 2 DESC;
```

---

## 6. Long-tail clustering (sub-agent step)

After SQL 5.5 + 5.6 + 5.7 return their results (write each large result to a file path provided by the MCP tool when output exceeds token cap), spawn a sub-agent (`general-purpose` subagent_type) with this brief:

> Cluster the long-tail canonical queries into ~20 root-intent buckets and produce a markdown report with verbatim queries + per-query click→trade. Read the 3 input files (search-side, click-side, click→trade), merge on `canonical_query`, then apply clustering rules in priority order: (1) numeric strike ranges (22-26k = Nifty, 50-55k = BankNifty, 70-80k = Sensex), (2) same top_entity, (3) shared substring stem ≥4 chars, (4) themed/sectoral keyword, (5) AMC name, (6) navigational keyword, (7) misspelling heuristic (edit-distance 1-2 from head entities), (8) private-co heuristic. Render every cluster with: header (volume + averages + interpretation), full verbatim query table (every query — no "+N more" truncation), fix recommendation (diagnose-first), confidence. Write the full output to `{LONG_TAIL_MD_PATH}` (use the Write tool). Reply to the parent agent with: file confirmation, top-5 click→trade clusters, bottom-5, 3 actionable findings the click→trade column unlocked, total queries rendered.

The output markdown file becomes the §4/§5 companion to the docx.

---

## 7. Docx generation

Use `python-docx` (verified available on the runtime). The structure follows v3 of `search_health_2026-05-25.docx`:

```
Title — "Global Search — 7-Day Health Report"
       Run date: {TODAY}   Window: {WINDOW_LABEL}   Owner squad: Global Search

§1 Executive Summary  (one paragraph; lead with headline trade attribution %, hard-zero rate, long-tail bucket sizes, count of distinct actionables).
  **HARD RULE — every run MUST include a `Top 5-10 Action Items` sub-section directly under §1**, BEFORE §2. **Format: one-liners only.** Each item is a single line with: P-tier badge · cluster/finding name · key metric (volume + one conversion number) · action verb ("Fix X" or "Diagnose Y") · Owner · Confidence. Aim for 5 items minimum, 10 maximum, ordered by P-tier then by trade-impact within tier. **Do NOT include verbatim query lists, multi-step diagnostic plans, or multi-paragraph write-ups in this section** — those belong in §3. The point of this section is rapid triage: PM should be able to read all 8 items in under 30 seconds.

§2 Headline Metrics (table with the rows specified in §4.1)
   - Raw search events, deduped, dedup ratio (observed, no editorial)
   - Unique users
   - Hard zero (both engines fail), semantic-fallback fires (split, NOT conflated)
   - Card clicks (excl watchlist)
   - Cross-filter mismatch (corrected denominator, with old vs new for transparency)
   - Avg click rank (with §7 caveat reference)
   - Trade attribution by segment (FnO, IndStocks, US, MF) + total
   - Searcher → trader %, Clicker → trader %
   - Blended CTR row: REMOVED (with note that it was meaningless density metric; session-paired CTR in deferred queue)
   - Search Open event: "Not in warehouse" (note)

§3 Key Actionables (P0 → P3)
   - Every finding adopts diagnose-first framing (see §3.2)
   - Owners route to specific pods (see §3.1)
   - Trade-lift estimates are RANGES not points
   - Confidence: HIGH / MEDIUM / LOW

§4 Long-Tail Findings — cluster summary table (20 rows)
   - Columns: LT-tag, cluster name, # queries, searches/day, total clicks, avg CTR %, avg click→trade %, zero-result %, dominant vertical, confidence
   - **HARD RULE:** the companion `.md` file is referenced for the FULL verbatim detail (every query), but the Confluence/docx body itself MUST also surface verbatim examples — see §5.

§5 Per-cluster narrative — concise health-tagged narrative + verbatim example queries
   - Color tags: 🟢 (healthy) / 🟡 (medium concern) / 🔴 (priority fix)
   - **HARD RULE — VERBATIM USER QUERY EXAMPLES ARE MANDATORY FOR EVERY CLUSTER.** Each of the 20 clusters MUST render a small inline table (5-10 example queries) directly in the report body — Confluence page AND docx. Do NOT defer all verbatim queries to the companion `.md` file alone. The PM reads this section to develop intuition for what users are typing; an abstract cluster name + averages is not enough.
   - Each cluster's example table MUST include columns: `query` (verbatim, exact user typing), `searches`, `CTR %`, `click→trade %`, `top clicked entity` (the actual product the user landed on). Add `zero-result %` for clusters where any query has non-zero zero-result.
   - For clusters with mixed health (e.g., LT-1 strikes where some queries convert at 94% and others silent-fail), the examples MUST show BOTH modes side-by-side: pick top 3-4 working queries AND top 3-4 silent-fail queries so the bimodal pattern is visible.
   - For wrong-entity routing clusters (LT-11 AMC, LT-6 pharma bees, LT-8 PsiQuantum), the example MUST highlight the wrong-entity match (e.g., "tata inv → Tata Investment STOCK card, not the intended MF").
   - The companion `.md` file remains the source for the FULL verbatim list (all queries per cluster); the inline examples are a curated 5-10-row subset for scan-reading.

§6 Consolidated Action Summary (single table sorted by P-tier)
   - Columns: P, Action, Cluster reference, Searches/day, Click→trade %, Owner, Note

§7 Data Caveats (every caveat from §4.2 of this routine spec)

§8 Methodology (table of data sources, dedup approach, CTR formula, cross-filter denominator, trade conversion approach, long-tail clustering)

Appendix A — Merging-table contents (small, fits inline)
Appendix B — Deferred fixes queue (from §4.3 of this routine spec)
```

Reuse the existing `_build_report_v3.py` script structure — it's a working reference. Update window dates and data values, keep the section structure.

---

## 8. Publishing

### 8.1 Confluence
- cloudId: `b155c4fa-66d0-4847-b4f6-363fbf53f646`
- Site: `https://finzoom.atlassian.net`
- Space: `GS` (id: `2545778696`, "Global Search")
- Parent page: `3779100773` ("Weekly Reports" — `/wiki/spaces/GS/pages/3779100773/Weekly+Reports`)
- Page title pattern: **`Weekly Report — {DATE_TAG}`** (em-dash `—` U+2014, date in ISO `YYYY-MM-DD`). Example existing pages under this parent: `Weekly Report — 2026-05-23`, `Weekly Report — 2026-05-25`. Match this exactly.
- Page format: `html` (Confluence-flavoured)
- Tool: `mcp__f3560b83-7d59-492a-a7ec-ab3b55990df5__createConfluencePage`
- Required arguments: `cloudId`, `spaceId="2545778696"`, `parentId="3779100773"`, `title="Weekly Report — {DATE_TAG}"`, `contentFormat="html"`, `body=<html>`

The Confluence page should contain a condensed version of the docx — primarily §1, §2, §3, §4 (cluster summary table only), and §6 Consolidated Action Summary. Long-tail verbatim detail and methodology appendix stay in the `.md` and `.docx` files (link to them in the page).

Use HTML body with Confluence-specific elements:
- `<div data-type="panel-warning">` for the diagnose-first banner
- `<div data-type="panel-info">` for the trade-attribution headline
- `<table>` for the headline metrics + cluster summary
- `<details><summary>` to collapse long sections

Capture the returned page URL — write it to a runtime variable `CONFLUENCE_URL`.

### 8.2 Slack (current state — manual paste)
**Slack MCP is not currently wired into the routine sandbox.** Per workspace memory (`project_ccr_sandbox_egress.md`), the routine sandbox blocks arbitrary HTTP egress — raw `curl` to a Slack webhook returns 403. Until a Slack MCP connector is added, the routine writes a ready-to-paste message to a file:

```python
slack_message = f"""📊 *Global Search Health — Week of {WINDOW_START}*

{search_attrib_pct}% of platform trades attributed to search this week ({trades_per_day:,}/day). {long_tail_clusters} long-tail clusters with verbatim queries + per-query click→trade.

Headline:
• Hard zero-result: {hard_zero_pct}% (vanishingly rare)
• Semantic fallback fires: {sem_fallback_pct}%
• Cross-filter mismatches: {cross_filter_pct}% of non-ALL clicks
• Largest long-tail bucket: {largest_long_tail_cluster} ({largest_long_tail_searches}/day)

Top 3 P0 actionables:
{p0_one_liners}

📄 Full report: {CONFLUENCE_URL}
📎 Long-tail detail: {LONG_TAIL_MD_PATH}
📎 Docx: {DOCX_PATH}
"""
with open(SLACK_MESSAGE_FILE_PATH, "w") as f: f.write(slack_message)
```

Then the routine reports to the parent: "Slack message written to {SLACK_MESSAGE_FILE_PATH}. Paste it into #global-search (or whichever channel) — Slack MCP not wired yet."

**To enable automatic Slack posting later:** add a Slack MCP server (e.g., the official Slack connector via Anthropic MCP marketplace) and replace the file-write step with a `slack_post_message` call to the configured channel.

---

## 9. Completion checklist (the routine reports these to the parent on success)

- [ ] `DOCX_PATH` written, file size > 30 KB
- [ ] `LONG_TAIL_MD_PATH` written, contains 20 clusters with verbatim queries
- [ ] Confluence page created, `CONFLUENCE_URL` captured
- [ ] `SLACK_MESSAGE_FILE_PATH` written
- [ ] Headline metrics rendered with hard-zero / semantic split
- [ ] §3 findings adopt diagnose-first framing
- [ ] Pod-ownership routing applied (no generic "Catalog" owner)
- [ ] §7 caveats all present
- [ ] Appendix B lists deferred fixes with current pending state

---

## 10. Error handling

- **Redshift session unavailable / 502s**: wait 60s and retry once. If still failing, halt the run and notify the parent — do not produce a partial report.
- **Query timeout on dedup CTE**: fall back to raw-events aggregation (as in SQL 5.5). Note in §7 caveats.
- **Click result set exceeds output cap**: results are auto-saved to a file by the MCP tool. Read in chunks via Read tool, OR delegate to a sub-agent (see §6).
- **Confluence publish fails**: still write the docx + long-tail .md locally. Surface the failure and the Confluence error in the completion message.

---

## 11. References

- Original v3 docx: `/Users/shruti.kumari/analytics_project/search_health_2026-05-25.docx`
- v3 build script (template): `/Users/shruti.kumari/analytics_project/_build_report_v3.py`
- v3 long-tail companion: `/Users/shruti.kumari/analytics_project/long_tail_enriched_2026-05-18_2026-05-24.md`
- Original prompt: `/Users/shruti.kumari/analytics_project/daily_search_analysis_prompt.md`
- This routine spec: `/Users/shruti.kumari/analytics_project/weekly_search_health_routine.md`

---

*End of routine spec. Adapted from v3 of the manual report after PM review surfaced 15 methodology fixes (10 applied, 7 still deferred). The diagnose-first voice is the most important architectural choice — keep findings investigative until the diagnostic basis is verified.*
