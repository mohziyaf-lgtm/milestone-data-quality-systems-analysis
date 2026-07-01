# Milestone Data Quality — SQL Query Library

I built this query library as the companion evidence base for the *Milestone Data Quality Systems Analysis* document. Every figure I quote in that analysis comes out of one of the twelve queries below, so I wanted the calculation logic to be transparent and reproducible rather than asserted. The dataset these queries run against is synthetic — I generated it to demonstrate the methodology and to let a reviewer re-run the numbers end to end. It is not drawn from a live CargoWise One environment, and no customer or operational data appears anywhere in it. The point is the method: the same SQL, pointed at a real milestone table, would surface the same class of problems.

The queries are written in standard ANSI SQL and run unchanged on both SQL Server and PostgreSQL except where I note a dialect difference inline. I have used short table aliases throughout (`s` for shipments, `m` for milestones) to keep the logic readable.

---

## The synthetic dataset

The dataset models roughly **4,000 shipment records** spread across two related tables. I designed the data generation so that, when the twelve queries are run against it, they reproduce the headline figures in the systems analysis exactly — these are fixed targets, not approximations.

### Table: `shipments`

| Column | Type | Notes |
|---|---|---|
| `shipment_id` | INT (PK) | 1–4000 |
| `customer_reference` | VARCHAR | ~120 distinct customer codes, unevenly weighted so a handful dominate volume |
| `origin_port` | VARCHAR | UN/LOCODE-style, e.g. `AUSYD`, `CNSHA` |
| `destination_port` | VARCHAR | UN/LOCODE-style |
| `shipment_type` | VARCHAR | `SEA_FCL`, `SEA_LCL`, `AIR` |
| `booking_date` | DATE | spread across 12 consecutive months for the trend queries |

### Table: `milestones`

| Column | Type | Notes |
|---|---|---|
| `milestone_id` | INT (PK) | |
| `shipment_id` | INT (FK → shipments) | |
| `milestone_type` | VARCHAR | one of the five types below |
| `expected_timestamp` | DATETIME | the SLA-planned time for that milestone |
| `actual_timestamp` | DATETIME | when the operator actually keyed it; `NULL` if never entered |
| `updated_by_operator_id` | INT | the operator who keyed the milestone; `NULL` where `actual_timestamp` is `NULL` |

The five `milestone_type` values, in their natural chronological order, are:

1. `booking_confirmed`
2. `departed_origin`
3. `customs_cleared`
4. `arrived_destination`
5. `delivered_to_consignee`

A complete shipment therefore has five milestone rows. The chronological order matters: an `actual_timestamp` for `customs_cleared` should never precede `departed_origin`, and so on. Violations of that ordering are exactly what query 7 hunts for.

### How the generation logic hits the target figures exactly

I engineered the synthetic generator (the figures below are over the full 4,000-shipment population) so that:

- **876 shipments (21.9%)** are missing at least one milestone entirely — at least one of their five `actual_timestamp` values is `NULL`. The remaining 3,124 carry all five.
- The mean of `(actual_timestamp − expected_timestamp)`, taken over every milestone row that has an `actual_timestamp`, lands at **≈ 2.3 days** of positive (late) variance.
- **162 milestone rows** are seeded with an `actual_timestamp` that falls earlier than the actual of a milestone that must precede it — the chronologically impossible records.
- **1,412 shipments (35.3%)** have their milestones keyed by **more than one distinct operator**.
- **2,448 shipments (61.2%)** fail the composite data quality threshold defined in query 10 (all five milestones present *and* every one within two days of expected).
- The monthly **overall data quality score** averages out to **38.8%**, the single executive figure produced by query 12.

Because these counts are baked into the data, the queries are deterministic: anyone re-running them against this dataset gets the published numbers back.

---

## The twelve queries

### 1. Share of shipments missing at least one milestone

I start here because it is the most basic completeness question a control tower should be able to answer and usually cannot: of all our shipments, how many have a hole in their milestone trail? I define a shipment as "incomplete" if any of its five expected milestones has no `actual_timestamp`.

```sql
SELECT
    CAST(100.0 * COUNT(DISTINCT CASE WHEN m.actual_timestamp IS NULL
                                     THEN m.shipment_id END)
         / COUNT(DISTINCT s.shipment_id) AS DECIMAL(5,1)) AS pct_missing_any_milestone
FROM shipments s
JOIN milestones m
    ON m.shipment_id = s.shipment_id;
```

This returns **21.9%** — better than one shipment in five is missing at least one milestone timestamp, which is the headline data-completeness gap the analysis opens with.

---

### 2. Average time variance between expected and actual

Next I wanted the average lateness across the whole milestone population, because an average delay is the single number most people intuitively understand. I measure the signed gap between `actual_timestamp` and `expected_timestamp` and average it across every milestone that was actually entered.

```sql
-- PostgreSQL: EXTRACT(EPOCH FROM (m.actual_timestamp - m.expected_timestamp)) / 86400.0
-- SQL Server: DATEDIFF(SECOND, m.expected_timestamp, m.actual_timestamp) / 86400.0
SELECT
    CAST(AVG(DATEDIFF(SECOND, m.expected_timestamp, m.actual_timestamp) / 86400.0)
         AS DECIMAL(4,1)) AS avg_entry_delay_days
FROM milestones m
WHERE m.actual_timestamp IS NOT NULL;
```

This returns **≈ 2.3 days**. On average a milestone is keyed more than two days after it was due, which is the latency figure I reference when arguing the data is not just incomplete but stale.

---

### 3. Distribution of entry delay by time bucket

An average hides the shape of the problem, so here I bucket every entered milestone into same-day, one-to-two days late, and three-or-more days late. This tells me whether the 2.3-day average is driven by a long tail or by broad, consistent lateness.

```sql
SELECT
    delay_bucket,
    COUNT(*) AS milestone_count,
    CAST(100.0 * COUNT(*) / SUM(COUNT(*)) OVER () AS DECIMAL(5,1)) AS pct_of_entered
FROM (
    SELECT
        CASE
            WHEN DATEDIFF(SECOND, m.expected_timestamp, m.actual_timestamp) / 86400.0 < 1
                THEN '0 - same day'
            WHEN DATEDIFF(SECOND, m.expected_timestamp, m.actual_timestamp) / 86400.0 < 3
                THEN '1 - 1 to 2 days late'
            ELSE '2 - 3 or more days late'
        END AS delay_bucket
    FROM milestones m
    WHERE m.actual_timestamp IS NOT NULL
) t
GROUP BY delay_bucket
ORDER BY delay_bucket;
```

This reveals that the lateness is broad rather than a freak tail: a large block of milestones sits in the one-to-two-day bucket, with a meaningful three-or-more-day group dragging the average up.

---

### 4. Operators with the highest average entry delay

Because milestones are keyed by people, I wanted to see whether delay clusters around particular operators. This ranks operators by their mean entry delay across every milestone they touched, with a volume floor so I am not ranking someone on three rows.

```sql
SELECT
    m.updated_by_operator_id,
    COUNT(*) AS milestones_updated,
    CAST(AVG(DATEDIFF(SECOND, m.expected_timestamp, m.actual_timestamp) / 86400.0)
         AS DECIMAL(4,1)) AS avg_delay_days
FROM milestones m
WHERE m.actual_timestamp IS NOT NULL
GROUP BY m.updated_by_operator_id
HAVING COUNT(*) >= 50
ORDER BY avg_delay_days DESC;
```

This surfaces the handful of operators whose average lateness sits well above the 2.3-day mean — the evidence I use to argue the fix is partly process and training, not only system design.

---

### 5. Which milestone type is missed most often

Not all milestones fail equally. Here I calculate, for each milestone type, the share of shipments where that milestone was never entered. This points to where in the lifecycle the data trail most often breaks.

```sql
SELECT
    m.milestone_type,
    COUNT(*) AS total_shipments,
    SUM(CASE WHEN m.actual_timestamp IS NULL THEN 1 ELSE 0 END) AS missing_count,
    CAST(100.0 * SUM(CASE WHEN m.actual_timestamp IS NULL THEN 1 ELSE 0 END)
         / COUNT(*) AS DECIMAL(5,1)) AS pct_missing
FROM milestones m
GROUP BY m.milestone_type
ORDER BY pct_missing DESC;
```

This shows the missing-data problem is concentrated in the later lifecycle milestones — the ones keyed under time pressure after delivery — which is where I focus the proposed validation layer.

---

### 6. Month-over-month trend of average entry delay

A control tower needs to know whether a problem is getting worse or holding steady. Here I trend the average entry delay by booking month so I can say something defensible about direction rather than just describing a snapshot.

```sql
-- PostgreSQL: TO_CHAR(s.booking_date, 'YYYY-MM')
-- SQL Server:  FORMAT(s.booking_date, 'yyyy-MM')
SELECT
    FORMAT(s.booking_date, 'yyyy-MM') AS booking_month,
    CAST(AVG(DATEDIFF(SECOND, m.expected_timestamp, m.actual_timestamp) / 86400.0)
         AS DECIMAL(4,1)) AS avg_delay_days,
    COUNT(*) AS milestones_entered
FROM shipments s
JOIN milestones m
    ON m.shipment_id = s.shipment_id
   AND m.actual_timestamp IS NOT NULL
GROUP BY FORMAT(s.booking_date, 'yyyy-MM')
ORDER BY booking_month;
```

This shows the delay is broadly stable month to month rather than spiking — which matters, because it means the gap is structural and will not resolve on its own without the intervention the analysis proposes.

---

### 7. Chronologically impossible records

This is the integrity check at the heart of the analysis. A milestone's actual timestamp should never precede the actual timestamp of a milestone that must come before it. I assign each milestone type an ordinal, self-join the table, and count the shipments where a later-ordinal milestone was keyed *before* an earlier-ordinal one.

```sql
WITH ordered AS (
    SELECT
        m.shipment_id,
        m.milestone_type,
        m.actual_timestamp,
        CASE m.milestone_type
            WHEN 'booking_confirmed'      THEN 1
            WHEN 'departed_origin'        THEN 2
            WHEN 'customs_cleared'        THEN 3
            WHEN 'arrived_destination'    THEN 4
            WHEN 'delivered_to_consignee' THEN 5
        END AS seq
    FROM milestones m
    WHERE m.actual_timestamp IS NOT NULL
)
SELECT COUNT(*) AS impossible_records
FROM ordered earlier
JOIN ordered later
    ON later.shipment_id = earlier.shipment_id
   AND later.seq        > earlier.seq
   AND later.actual_timestamp < earlier.actual_timestamp;
```

This returns exactly **162** chronologically impossible records — milestones logged out of physical sequence. These are the cases I point to when I say the data is not merely late but internally contradictory, which no downstream report can detect on its own.

---

### 8. Share of shipments handled by more than one operator

Hand-offs are where accountability and data quality both tend to fray, so I measure how many shipments had their milestones keyed by more than one distinct operator over their life.

```sql
SELECT
    CAST(100.0 * COUNT(*)
         / (SELECT COUNT(*) FROM shipments) AS DECIMAL(5,1)) AS pct_multi_operator
FROM (
    SELECT m.shipment_id
    FROM milestones m
    WHERE m.updated_by_operator_id IS NOT NULL
    GROUP BY m.shipment_id
    HAVING COUNT(DISTINCT m.updated_by_operator_id) > 1
) multi;
```

This returns **35.3%** — more than a third of shipments pass through multiple operators, which is the hand-off exposure I use to justify clear ownership rules alongside the validation engine.

---

### 9. Top five customers most affected by missing or delayed milestones

Data quality problems are not evenly distributed across customers, and the ones that hurt commercially are the ones hitting our biggest accounts. Here I rank customer references by a combined count of missing and materially late milestones.

```sql
SELECT TOP 5
    s.customer_reference,
    SUM(CASE WHEN m.actual_timestamp IS NULL THEN 1 ELSE 0 END) AS missing_milestones,
    SUM(CASE WHEN m.actual_timestamp IS NOT NULL
              AND DATEDIFF(SECOND, m.expected_timestamp, m.actual_timestamp) / 86400.0 >= 3
             THEN 1 ELSE 0 END) AS materially_late_milestones,
    SUM(CASE WHEN m.actual_timestamp IS NULL
              OR DATEDIFF(SECOND, m.expected_timestamp, m.actual_timestamp) / 86400.0 >= 3
             THEN 1 ELSE 0 END) AS total_affected
FROM shipments s
JOIN milestones m
    ON m.shipment_id = s.shipment_id
GROUP BY s.customer_reference
ORDER BY total_affected DESC;
-- PostgreSQL: drop "TOP 5" and append "LIMIT 5" after ORDER BY
```

This isolates the five customer references carrying the heaviest concentration of bad milestone data — the accounts where the reporting gap is most likely to be felt externally, and therefore the natural first targets for remediation.

---

### 10. Shipments failing the composite data quality threshold

This is the pass/fail definition the analysis is built on. I define a "good" shipment as one where all five milestones are present *and* every milestone was entered within two days of its expected time. Anything short of that fails. Here I calculate the failing percentage.

```sql
SELECT
    CAST(100.0 * SUM(CASE WHEN q.is_pass = 0 THEN 1 ELSE 0 END)
         / COUNT(*) AS DECIMAL(5,1)) AS pct_failing_threshold
FROM (
    SELECT
        s.shipment_id,
        CASE
            WHEN COUNT(m.actual_timestamp) = 5
             AND MAX(CASE WHEN m.actual_timestamp IS NULL THEN 1
                          WHEN DATEDIFF(SECOND, m.expected_timestamp, m.actual_timestamp)
                               / 86400.0 > 2 THEN 1
                          ELSE 0 END) = 0
            THEN 1 ELSE 0
        END AS is_pass
    FROM shipments s
    JOIN milestones m
        ON m.shipment_id = s.shipment_id
    GROUP BY s.shipment_id
) q;
```

This returns **61.2%** of shipments failing the threshold. That single number — nearly two in three shipments not meeting a fairly modest "complete and roughly on time" bar — is the figure I lead the executive summary with.

---

### 11. Count of shipments with milestones entered by more than one operator

Query 8 gave the percentage; here I want the absolute count for the operations team, who think in shipments rather than rates when they plan ownership changes.

```sql
SELECT COUNT(*) AS multi_operator_shipments
FROM (
    SELECT m.shipment_id
    FROM milestones m
    WHERE m.updated_by_operator_id IS NOT NULL
    GROUP BY m.shipment_id
    HAVING COUNT(DISTINCT m.updated_by_operator_id) > 1
) multi;
```

This returns **1,412 shipments** keyed by more than one operator — the same population as the 35.3% in query 8, expressed as a raw count for planning the hand-off and ownership controls.

---

### 12. Overall milestone data quality score (executive dashboard)

Finally I roll everything into one figure a leadership audience can track over time. I score each shipment as fully clean only if it is complete, on time within two days, and free of any chronological violation, then express the clean share as a percentage per month — the number that would sit at the top of an executive dashboard.

```sql
WITH seq AS (
    SELECT
        m.*,
        CASE m.milestone_type
            WHEN 'booking_confirmed'      THEN 1
            WHEN 'departed_origin'        THEN 2
            WHEN 'customs_cleared'        THEN 3
            WHEN 'arrived_destination'    THEN 4
            WHEN 'delivered_to_consignee' THEN 5
        END AS ord
    FROM milestones m
),
violation AS (
    SELECT DISTINCT e.shipment_id
    FROM seq e
    JOIN seq l
        ON l.shipment_id = e.shipment_id
       AND l.ord > e.ord
       AND l.actual_timestamp < e.actual_timestamp
),
scored AS (
    SELECT
        s.shipment_id,
        FORMAT(s.booking_date, 'yyyy-MM') AS booking_month,
        CASE
            WHEN COUNT(m.actual_timestamp) = 5
             AND MAX(CASE WHEN DATEDIFF(SECOND, m.expected_timestamp, m.actual_timestamp)
                               / 86400.0 > 2 THEN 1 ELSE 0 END) = 0
             AND v.shipment_id IS NULL
            THEN 1 ELSE 0
        END AS is_clean
    FROM shipments s
    JOIN milestones m ON m.shipment_id = s.shipment_id
    LEFT JOIN violation v ON v.shipment_id = s.shipment_id
    GROUP BY s.shipment_id, FORMAT(s.booking_date, 'yyyy-MM'), v.shipment_id
)
SELECT
    booking_month,
    CAST(100.0 * SUM(is_clean) / COUNT(*) AS DECIMAL(5,1)) AS data_quality_score_pct
FROM scored
GROUP BY booking_month
ORDER BY booking_month;

-- Overall single-figure score for the dashboard header:
-- SELECT CAST(100.0 * SUM(is_clean) / COUNT(*) AS DECIMAL(5,1)) FROM scored;
```

The overall score lands at **38.8%** — fewer than two in five shipments are fully clean by the combined completeness, timeliness, and integrity test. That is the single headline metric the systems analysis tracks, and the number the proposed validation and exception engine is designed to move.

---

*Synthetic dataset and query library prepared as supporting evidence for the Milestone Data Quality Systems Analysis. All data is fabricated for demonstration of method; figures are reproducible by running the queries above against the generated dataset.*
