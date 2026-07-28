---
name: run
description: Analyze Michaels SMS performance — by fiscal week (all campaigns in a week) or by individual campaign (with click-attributed revenue and baseline comparison). Always asks which mode first and shows recent campaigns.
---

# Michaels SMS Analysis

---

## Step 0: Ask what they want to analyze (ALWAYS run first)

First, run this query to show the 6 most recent campaigns — always display this list before asking the question:

```sql
SELECT
    message_name,
    MIN(substring(timestamp,1,10)) AS send_date,
    COUNT(DISTINCT CASE WHEN type = 'MESSAGE_RECEIPT' THEN phone END) AS sends,
    COUNT(DISTINCT CASE WHEN type = 'MESSAGE_LINK_CLICK' THEN phone END) AS clicks,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN type = 'MESSAGE_LINK_CLICK' THEN phone END)
        / NULLIF(COUNT(DISTINCT CASE WHEN type = 'MESSAGE_RECEIPT' THEN phone END), 0), 2) AS click_rate_pct
FROM mk_src.attentive_general_histunion
WHERE type IN ('MESSAGE_RECEIPT','MESSAGE_LINK_CLICK')
  AND (upper(message_name) LIKE 'MASS%' OR upper(message_name) LIKE 'PROMO%')
  AND lower(message_name) NOT LIKE '%welcome%'
  AND lower(message_name) NOT LIKE '%makerplace%'
  AND upper(message_name) NOT LIKE '%_QUE%'
  AND upper(message_name) NOT LIKE '%_CAN%'
GROUP BY 1
ORDER BY send_date DESC
LIMIT 6;
```

Then ask:

> "Here are the most recent SMS campaigns:
>
> | Campaign | Send Date | Sends | Click Rate |
> |---|---|---|---|
> | ... | ... | ... | ...% |
>
> How would you like to analyze SMS performance?
> - **By fiscal week** — all campaigns sent in a given week, aggregate view with Mass vs Journey/Trigger breakdown
> - **By individual campaign** — deep dive on one campaign with click-attributed revenue, AOV, and baseline comparison vs similar campaigns"

---

## PATH A: Fiscal Week Analysis

### Step A1: Resolve fiscal week ID

Ask: *"Which week? Give me a date, a description like 'last week', or a fiscal week ID like `202620`."*

If the user provides a date:
```sql
SELECT DISTINCT wk_idnt,
    MIN(day_dt) AS week_start,
    MAX(day_dt) AS week_end
FROM cdp_unification_mk.bq_date_dim
WHERE CAST(day_dt AS DATE) = DATE '{input_date}'
GROUP BY 1;
```

If the user provides a description, resolve it to a date first, then run the above. Also fetch the week range for the dashboard header:
```sql
SELECT MIN(day_dt) AS week_start, MAX(day_dt) AS week_end
FROM cdp_unification_mk.bq_date_dim
WHERE wk_idnt = '{input_week}';
```

### Step A2: Run queries in parallel

#### Q1 — Overall Metrics + Mass vs Journey/Trigger Breakdown
```sql
WITH TY_week_messages AS (
    SELECT DISTINCT message_name
    FROM mk_src.attentive_general_histunion a
    JOIN mk_src.bq_date_dim b ON substring(a.timestamp,1,10) = b.day_dt
    WHERE b.wk_idnt = '{input_week}'
      AND a.type = 'MESSAGE_RECEIPT'
      AND lower(message_name) NOT LIKE '%welcome%'
      AND lower(message_name) NOT LIKE '%% off%'
      AND lower(message_name) NOT LIKE '%makerplacesellerevent%'
      AND lower(message_name) NOT LIKE '%makerplace%'
      AND upper(message_name) NOT LIKE '%_QUE%'
      AND upper(message_name) NOT LIKE '%_CAN%'
),
SMS_Raw AS (
    SELECT
        a.phone,
        a.message_name,
        CASE WHEN (upper(a.message_name) LIKE '%WALLET%'
                   OR lower(a.message_name) LIKE '%journey%'
                   OR lower(a.message_name) LIKE '%abandon%')
             THEN 'Journey/Trigger'
             ELSE 'Mass'
        END AS message_segment,
        COUNT(DISTINCT CASE WHEN a.type = 'MESSAGE_RECEIPT' THEN a.timestamp END) AS sends,
        COUNT(DISTINCT CASE WHEN a.type = 'MESSAGE_LINK_CLICK' THEN a.timestamp END) AS clicks
    FROM mk_src.attentive_general_histunion a
    JOIN mk_src.bq_date_dim b ON substring(a.timestamp,1,10) = b.day_dt
    JOIN TY_week_messages wm ON a.message_name = wm.message_name
    WHERE b.wk_idnt = '{input_week}'
    GROUP BY 1, 2, 3
),
segment_summary AS (
    SELECT
        message_segment,
        SUM(sends) AS total_sends,
        SUM(clicks) AS total_clicks,
        ROUND(100.0 * SUM(clicks) / NULLIF(SUM(sends), 0), 2) AS click_rate_pct
    FROM SMS_Raw
    GROUP BY 1
),
total_summary AS (
    SELECT
        SUM(sends) AS total_sends,
        SUM(clicks) AS total_clicks,
        ROUND(100.0 * SUM(clicks) / NULLIF(SUM(sends), 0), 2) AS click_rate_pct,
        COUNT(DISTINCT phone) AS unique_recipients
    FROM (SELECT phone, SUM(sends) AS sends, SUM(clicks) AS clicks FROM SMS_Raw GROUP BY 1) t
)
SELECT 'TOTAL' AS segment, total_sends, total_clicks, click_rate_pct, unique_recipients
FROM total_summary
UNION ALL
SELECT message_segment, total_sends, total_clicks, click_rate_pct, NULL
FROM segment_summary
ORDER BY segment;
```

#### Q2 — Subscriber Count (as of this week)
```sql
SELECT COUNT(DISTINCT a.phone) AS total_subscribers
FROM mk_src.attentive_optstatus a
JOIN mk_src.bq_date_dim b ON substring(a.timestamp,1,10) = b.day_dt
WHERE CAST(b.wk_idnt AS INT) <= CAST('{input_week}' AS INT)
  AND opt_in_status = 'JOIN';
```

### Step A3: Dashboard Output

Render a self-contained HTML dashboard.

**Header:** "SMS Performance — Fiscal Week {input_week} ({week_start} → {week_end})"

**KPI row (4 tiles):** Total Sends · Total Clicks · Click Rate (%) · Total Subscribers

**Segment Breakdown Table:**

| Segment | Sends | Clicks | Click Rate |
|---------|-------|--------|------------|
| Mass | ... | ... | ...% |
| Journey/Trigger | ... | ... | ...% |
| **Total** | ... | ... | ...% |

**Bar Chart:** Sends vs Clicks side-by-side for Mass and Journey/Trigger.

**Design:** Professional, clean, minimal. Neutral palette. Inline CSS — no external dependencies.

---

## PATH B: Individual Campaign Analysis

### Step B1: Confirm campaign

If the user picked from the Step 0 list, use that campaign name. Otherwise ask them to paste the campaign name.

### Step B2: Resolve campaign date range

```sql
SELECT
    MIN(substring(timestamp,1,10)) AS send_start,
    MAX(substring(timestamp,1,10)) AS send_end
FROM mk_src.attentive_general_histunion
WHERE message_name = '{input_campaign}'
  AND type = 'MESSAGE_RECEIPT';
```

Store `send_start` and `send_end`. Attribution window: `send_start` to `send_start + 7 days`.

### Step B3: Run Phase 1 queries in parallel (fast — engagement + click baseline)

Run Q1, Q2a, and Q3a together. These hit only the SMS event table and the opt-status bridge — no transaction scans.

#### Q1 — Campaign Engagement Metrics
```sql
WITH campaign_raw AS (
    SELECT
        phone,
        COUNT(DISTINCT CASE WHEN type = 'MESSAGE_RECEIPT' THEN timestamp END) AS sends,
        COUNT(DISTINCT CASE WHEN type = 'MESSAGE_LINK_CLICK' THEN timestamp END) AS clicks
    FROM mk_src.attentive_general_histunion
    WHERE message_name = '{input_campaign}'
      AND type IN ('MESSAGE_RECEIPT','MESSAGE_LINK_CLICK')
    GROUP BY 1
)
SELECT
    COUNT(DISTINCT phone) AS unique_recipients,
    SUM(sends) AS total_sends,
    SUM(clicks) AS total_clicks,
    COUNT(DISTINCT CASE WHEN clicks > 0 THEN phone END) AS unique_clickers,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN clicks > 0 THEN phone END)
        / NULLIF(COUNT(DISTINCT phone), 0), 2) AS click_rate_pct
FROM campaign_raw;
```

#### Q2a — Resolve Clicker Crafter IDs (fast — SMS tables only)

Run this first. Returns a small set (~thousands of rows) used as the predicate for Q2b.

```sql
SELECT DISTINCT o.crafter_id
FROM mk_src.attentive_general_histunion a
JOIN cdp_unification_mk.enrich_attentive_optstatus o
  ON a.phone = o.phone
WHERE a.message_name = '{input_campaign}'
  AND a.type = 'MESSAGE_LINK_CLICK'
  AND a.substring(timestamp,1,10) BETWEEN '{send_start}' AND '{send_end}'
  AND o.crafter_id IS NOT NULL AND o.crafter_id <> ''
  AND o.opt_in_status = 'join';
```

Store the result as `{clicker_crafter_ids}` — a list of crafter_ids to pass into Q2b.

#### Q3a — Baseline Click Rate (fast — SMS tables only)
```sql
WITH this_campaign AS (
    SELECT COUNT(DISTINCT CASE WHEN type = 'MESSAGE_RECEIPT' THEN phone END) AS this_sends
    FROM mk_src.attentive_general_histunion
    WHERE message_name = '{input_campaign}'
      AND type IN ('MESSAGE_RECEIPT','MESSAGE_LINK_CLICK')
),
peer_sends AS (
    SELECT
        message_name,
        MIN(substring(timestamp,1,10)) AS send_date,
        COUNT(DISTINCT CASE WHEN type = 'MESSAGE_RECEIPT' THEN phone END) AS sends,
        COUNT(DISTINCT CASE WHEN type = 'MESSAGE_LINK_CLICK' THEN phone END) AS clicks
    FROM mk_src.attentive_general_histunion
    WHERE type IN ('MESSAGE_RECEIPT','MESSAGE_LINK_CLICK')
      AND message_name != '{input_campaign}'
      AND (upper(message_name) LIKE 'MASS%' OR upper(message_name) LIKE 'PROMO%')
      AND lower(message_name) NOT LIKE '%welcome%'
      AND lower(message_name) NOT LIKE '%makerplace%'
      AND upper(message_name) NOT LIKE '%_QUE%'
      AND upper(message_name) NOT LIKE '%_CAN%'
      AND substring(timestamp,1,10) >= CAST(DATE_ADD('day', -90, DATE('{send_start}')) AS VARCHAR)
      AND substring(timestamp,1,10) <= '{send_end}'
    GROUP BY 1
),
peers AS (
    SELECT ps.*
    FROM peer_sends ps
    CROSS JOIN this_campaign tc
    WHERE ps.sends BETWEEN tc.this_sends * 0.7 AND tc.this_sends * 1.3
      AND ps.sends > 0
)
SELECT
    COUNT(*) AS peer_campaign_count,
    APPROX_PERCENTILE(ROUND(100.0 * clicks / NULLIF(sends, 0), 2), 0.5) AS median_click_rate_pct,
    AVG(ROUND(100.0 * clicks / NULLIF(sends, 0), 2)) AS avg_click_rate_pct
FROM peers;
```

Store the peer campaign names from the `peers` CTE — needed for Q3b.

---

### Step B4: Render initial dashboard, then run Phase 2 in parallel

Once Q1, Q2a, Q3a complete, render the dashboard with engagement metrics and click rate baseline. Then immediately kick off Q2b and Q3b in parallel — these are the heavy transaction scans.

#### Q2b — Click-Attributed Revenue (uses crafter_ids from Q2a)

Scans `enrich_transactions_behaviour` filtered to the resolved crafter_id set — much smaller join surface than a full table scan.

```sql
SELECT
    COUNT(DISTINCT t.crafter_id) AS customers,
    COUNT(DISTINCT t.transaction_id_number) AS transactions,
    ROUND(SUM(t.total_gross_sales), 2) AS revenue,
    ROUND(SUM(t.total_gross_sales) / NULLIF(COUNT(DISTINCT t.transaction_id_number), 0), 2) AS aov,
    ROUND(SUM(t.total_gross_sales) / NULLIF(COUNT(DISTINCT t.crafter_id), 0), 2) AS rev_per_clicker
FROM cdp_unification_mk.enrich_transactions_behaviour t
WHERE t.crafter_id IN ({clicker_crafter_ids})
  AND t.transaction_time >= '{send_start}'
  AND t.transaction_time <= CAST(DATE_ADD('day', 7, DATE('{send_start}')) AS VARCHAR)
  AND t.total_gross_sales > 0;
```

> **Attribution:** 7-day window from send_start. Returns excluded (`total_gross_sales > 0`).

#### Q3b — Baseline Revenue (uses peer campaign names from Q3a)

Resolves peer clicker crafter_ids then attributes transactions. Scoped to only the peer campaigns identified in Q3a.

```sql
WITH peer_clickers AS (
    SELECT
        a.message_name,
        o.crafter_id,
        MIN(substring(a.timestamp,1,10)) AS send_date
    FROM mk_src.attentive_general_histunion a
    JOIN cdp_unification_mk.enrich_attentive_optstatus o
      ON a.phone = o.phone
    WHERE a.message_name IN ({peer_campaign_names})
      AND a.type = 'MESSAGE_LINK_CLICK'
      AND o.crafter_id IS NOT NULL AND o.crafter_id <> ''
      AND o.opt_in_status = 'join'
    GROUP BY 1, 2
),
peer_txn_agg AS (
    SELECT
        pc.message_name,
        COUNT(DISTINCT t.transaction_id_number) AS transactions,
        COUNT(DISTINCT pc.crafter_id) AS customers,
        ROUND(SUM(t.total_gross_sales), 2) AS revenue
    FROM peer_clickers pc
    JOIN cdp_unification_mk.enrich_transactions_behaviour t
      ON pc.crafter_id = t.crafter_id
     AND t.transaction_time >= pc.send_date
     AND t.transaction_time <= CAST(DATE_ADD('day', 7, DATE(pc.send_date)) AS VARCHAR)
     AND t.total_gross_sales > 0
    GROUP BY 1
)
SELECT
    COUNT(*) AS peer_campaign_count,
    APPROX_PERCENTILE(COALESCE(revenue, 0), 0.5) AS median_revenue,
    AVG(COALESCE(revenue, 0)) AS avg_revenue,
    APPROX_PERCENTILE(ROUND(COALESCE(revenue,0) / NULLIF(COALESCE(customers,0),0), 2), 0.5) AS median_rev_per_clicker,
    APPROX_PERCENTILE(ROUND(COALESCE(revenue,0) / NULLIF(COALESCE(transactions,0),0), 2), 0.5) AS median_aov
FROM peer_txn_agg;
```

---

### Step B5: Update dashboard with revenue data

Once Q2b and Q3b complete, update the dashboard tiles and baseline table with revenue metrics.

**KPI row (5 tiles):** Sends · Unique Clickers · Click Rate · Revenue (7-day click-attributed) · AOV

**Baseline comparison table:**

| Metric | This Campaign | Baseline Median | vs Baseline |
|--------|---------------|-----------------|-------------|
| Click Rate | x% | x% | +/- bps |
| Revenue | $x | $x | +/- % |
| Rev / Clicker | $x | $x | +/- % |
| AOV | $x | $x | +/- % |

Show peer count (e.g. "Baseline: 8 similar campaigns in last 90 days").

**Design:** Professional, clean, minimal. Neutral palette. Inline CSS — no external dependencies.

---

## Calculated Metrics

- **Click Rate:** `Unique Clickers / Sends`
- **Conv Rate:** `Customers / Unique Clickers`
- **Rev / Clicker:** `Revenue / Unique Clickers`
- **AOV:** `Revenue / Transactions`
- **bps delta:** `(campaign% − baseline%) × 100`
- **% delta:** `(campaign − baseline) / baseline × 100`

---

## Data Tables Reference

| Table | Purpose |
|---|---|
| `mk_src.attentive_general_histunion` | Raw SMS events — sends, clicks. Filter on `message_name` + `type`. |
| `mk_src.attentive_optstatus` | Subscriber opt-in status. Use for subscriber count. |
| `cdp_unification_mk.enrich_attentive_optstatus` | Phone → `crafter_id` bridge. Use `opt_in_status = 'join'` only. |
| `cdp_unification_mk.enrich_transactions_behaviour` | Transactions. Has `crafter_id`, `transaction_time` (string), `total_gross_sales`, `transaction_id_number`. |
| `cdp_unification_mk.bq_date_dim` | Calendar / fiscal week dimension. |
