# TMG Credit Analytics
**Atharva Gupte** | agupte1312@gmail.com | (619) 587-6109

---

## Why I built this

I work in credit and debt management — helping people dig out of financial difficulty, rebuild their credit profiles, and get back on track. Every day I see the real human story behind a three-digit number. A 580 FICO score isn't just a data point — it's someone who missed payments during a rough stretch and is now paying a higher interest rate on everything they borrow.

What I wanted to understand better was the data side of what we do. We send dispute letters, negotiate settlements, put people on payment plans — but does it actually move the needle on their credit score? That question is what drove this project.

I built a SQL Server database modeled after real consumer credit bureau data — the kind of three-bureau merged credit report we work with at TMG — and wrote queries to answer questions I actually care about.

---

## The dataset

Five tables, 26,779 rows total.

- **Consumers** — 600 client profiles with demographics and income
- **Credit_Reports** — 1,474 credit pulls across those 600 clients over time
- **FICO_Scores** — 4,422 rows, one score per bureau per pull (Experian, TransUnion, Equifax)
- **Tradelines** — 14,730 accounts — credit cards, loans, collections, charge-offs
- **Credit_Inquiries** — 5,553 inquiry records

The data is synthetic but structured to mirror a real Advantage Credit merged infile report. I also intentionally injected about 4% data quality violations — missing fields, invalid scores, illogical values — because real data is never clean and I wanted to practice finding those issues before they corrupt analysis.

---

## The schema

Everything connects through Credit_Reports. You can't get from a consumer to their FICO scores in one step — you go through the report that pulled those scores. That relationship matters when you're writing joins.

```
Consumers
    └── Credit_Reports
            ├── FICO_Scores
            ├── Tradelines  ← this is the fact table, where all the metrics live
            └── Credit_Inquiries
```

---

## Query 1 — Finding the dirty data first

Before I ran any analysis I wanted to know what was broken in the dataset. In credit work, one wrong field cascades — a transposed member ID at registration means a claim gets denied three departments later. Same principle here.

I wrote a validation audit across six business rules:

```sql
SELECT violation_type, COUNT(*) AS violation_count
FROM (
    SELECT 'Balance > Credit Limit' AS violation_type
    FROM Tradelines WHERE balance > credit_limit AND credit_limit > 0
    UNION ALL
    SELECT 'Negative Past Due Amount'
    FROM Tradelines WHERE past_due_amount < 0
    UNION ALL
    SELECT 'Missing Bureau Source'
    FROM Tradelines WHERE bureau_source IS NULL OR bureau_source = ''
    UNION ALL
    SELECT 'Late 60 > Late 30 (Illogical)'
    FROM Tradelines WHERE late_60_count > late_30_count
    UNION ALL
    SELECT 'Invalid FICO Score'
    FROM FICO_Scores WHERE fico_score < 350 OR fico_score > 850
    UNION ALL
    SELECT 'Duplicate Inquiry'
    FROM Credit_Inquiries
    GROUP BY report_id, bureau, inquiry_date, creditor_name
    HAVING COUNT(*) > 1
) violations
GROUP BY violation_type
ORDER BY violation_count DESC;
```

**What came back:**

| Violation | Count |
|---|---|
| Late 60 > Late 30 (illogical sequence) | 671 |
| Balance exceeds credit limit | 359 |
| Negative past due amount | 225 |
| Missing bureau source | 150 |
| FICO score outside valid range | 82 |
| Duplicate inquiry record | 79 |
| **Total** | **1,566** |

1,566 violations out of 26,779 records — about 5.8%. The most common one was late payment sequences that don't make logical sense. You can't have more 60-day lates than 30-day lates — the 30-day bucket feeds the 60-day bucket. When that's reversed it usually means a data entry error upstream in the bureau feed.

I flagged all of these before running any downstream analysis. This is the same discipline you need when working with interface data — whether it's a credit bureau feed or an HL7 message coming through an Epic Bridges interface. Bad data that goes undetected doesn't stay in one place.

---

## Query 2 — Who is in this portfolio and how risky are they

Once the data was clean I wanted a picture of the whole portfolio. I used the CFPB standard credit tiers because those are the buckets that actually matter in consumer lending decisions.

This one uses four CTEs chained together. Each one builds on the previous — get the latest report per consumer, match that to a report ID, calculate average FICO, calculate utilization and derogatory stats, then combine and segment.

```sql
WITH LatestReport AS (
    SELECT consumer_id, MAX(report_date) AS latest_report_date
    FROM Credit_Reports GROUP BY consumer_id
),
LatestReportID AS (
    SELECT cr.consumer_id, cr.report_id
    FROM Credit_Reports cr
    INNER JOIN LatestReport lr 
        ON cr.consumer_id = lr.consumer_id 
        AND cr.report_date = lr.latest_report_date
),
AvgFICO AS (
    SELECT lr.consumer_id,
        AVG(CAST(fs.fico_score AS DECIMAL(6,2))) AS avg_fico
    FROM LatestReportID lr
    INNER JOIN FICO_Scores fs ON lr.report_id = fs.report_id
    WHERE fs.fico_score BETWEEN 350 AND 850
    GROUP BY lr.consumer_id
),
UtilizationStats AS (
    SELECT lr.consumer_id,
        AVG(t.credit_utilization_ratio) AS avg_utilization,
        SUM(CASE WHEN t.is_derogatory_flag = 1 THEN 1 ELSE 0 END) AS derogatory_count,
        COUNT(*) AS total_tradelines
    FROM LatestReportID lr
    INNER JOIN Tradelines t ON lr.report_id = t.report_id
    GROUP BY lr.consumer_id
),
Segmented AS (
    SELECT 
        CASE 
            WHEN af.avg_fico < 580 THEN '1. Deep Subprime (<580)'
            WHEN af.avg_fico < 620 THEN '2. Subprime (580-619)'
            WHEN af.avg_fico < 660 THEN '3. Near-Prime (620-659)'
            WHEN af.avg_fico < 720 THEN '4. Prime (660-719)'
            ELSE '5. Super-Prime (720+)'
        END AS fico_band,
        af.avg_fico, us.avg_utilization,
        us.derogatory_count, us.total_tradelines
    FROM AvgFICO af
    INNER JOIN UtilizationStats us ON af.consumer_id = us.consumer_id
)
SELECT 
    fico_band,
    COUNT(*) AS consumer_count,
    ROUND(AVG(avg_fico), 1) AS avg_fico_score,
    ROUND(AVG(avg_utilization) * 100, 1) AS avg_utilization_pct,
    ROUND(AVG(CAST(derogatory_count AS DECIMAL) / 
        NULLIF(total_tradelines, 0)) * 100, 1) AS derogatory_rate_pct
FROM Segmented
GROUP BY fico_band
ORDER BY fico_band;
```

**What came back:**

| FICO Band | Clients | Avg Score | Avg Utilization | Derogatory Rate |
|---|---|---|---|---|
| Deep Subprime (<580) | 148 | 455.6 | 61.4% | 66.5% |
| Subprime (580-619) | 105 | 602.2 | 44.1% | 16.6% |
| Near-Prime (620-659) | 61 | 676.2 | 41.1% | 10.1% |
| Prime (660-719) | 88 | 724.0 | 33.8% | 2.1% |
| Super-Prime (720+) | 198 | 799.3 | 30.5% | 0.0% |

The pattern is exactly what you'd expect in a credit counseling portfolio — the people who need the most help are the ones with the highest utilization and the most derogatory accounts. Deep subprime clients are carrying 61% utilization on average and two thirds of their accounts have derogatory marks. Those are the accounts where a dispute strategy or debt settlement plan has the biggest upside.

---

## Query 3 — The one I found most interesting: are scores actually moving

This is the question I actually wanted to answer. You can send dispute letters and negotiate settlements all day — but is the FICO score moving?

I used a LAG() window function to compare each credit pull against the previous one for the same consumer. LAG() lets you look back at the previous row's value within a partition — in this case, the previous report date for each consumer — without doing a self join.

```sql
WITH FICOByReport AS (
    SELECT cr.consumer_id, cr.report_date,
        AVG(CAST(fs.fico_score AS DECIMAL(6,2))) AS avg_fico
    FROM Credit_Reports cr
    INNER JOIN FICO_Scores fs ON cr.report_id = fs.report_id
    WHERE fs.fico_score BETWEEN 350 AND 850
    GROUP BY cr.consumer_id, cr.report_date
),
FICOWithLag AS (
    SELECT consumer_id, report_date,
        avg_fico AS current_fico,
        LAG(avg_fico) OVER (
            PARTITION BY consumer_id 
            ORDER BY report_date
        ) AS previous_fico
    FROM FICOByReport
)
SELECT 
    CASE 
        WHEN (current_fico - previous_fico) > 20  
            THEN '1. Significant Improvement (>20 pts)'
        WHEN (current_fico - previous_fico) > 0   
            THEN '2. Slight Improvement (1-20 pts)'
        WHEN (current_fico - previous_fico) = 0   
            THEN '3. No Change'
        WHEN (current_fico - previous_fico) > -20 
            THEN '4. Slight Decline (1-20 pts)'
        ELSE '5. Significant Decline (>20 pts)'
    END AS score_movement,
    COUNT(*) AS occurrences,
    ROUND(AVG(current_fico), 1) AS avg_current_fico,
    ROUND(AVG(current_fico - previous_fico), 1) AS avg_point_change
FROM FICOWithLag
WHERE previous_fico IS NOT NULL
GROUP BY 
    CASE 
        WHEN (current_fico - previous_fico) > 20  
            THEN '1. Significant Improvement (>20 pts)'
        WHEN (current_fico - previous_fico) > 0   
            THEN '2. Slight Improvement (1-20 pts)'
        WHEN (current_fico - previous_fico) = 0   
            THEN '3. No Change'
        WHEN (current_fico - previous_fico) > -20 
            THEN '4. Slight Decline (1-20 pts)'
        ELSE '5. Significant Decline (>20 pts)'
    END
ORDER BY score_movement;
```

**What came back:**

| Movement | Cases | Avg FICO | Avg Point Change |
|---|---|---|---|
| Significant Improvement (>20 pts) | 213 | 654.3 | +29.1 |
| Slight Improvement (1-20 pts) | 324 | 671.0 | +9.7 |
| No Change | 7 | 669.0 | 0.0 |
| Slight Decline (1-20 pts) | 280 | 645.2 | -8.6 |
| Significant Decline (>20 pts) | 50 | 626.4 | -26.7 |

537 out of 874 tracked cases showed improvement — 61% of the portfolio is moving in the right direction. But 50 cases showed significant decline, dropping an average of 26.7 points between pulls. Those are the accounts worth a deep dive — something happened between the two pulls that hurt the score. New derogatory entry, missed payment, spike in utilization, new collections. The dispute team should pull those 50 accounts and investigate before the slide continues.

That's the story I was looking for when I started this project.

---

## What I learned building this

The data quality audit changed how I think about analysis. I almost ran the FICO trend query first — it's the interesting one. But the bureau comparison self join I ran later showed that the biggest score discrepancies between Experian and Equifax were actually the invalid FICO scores from Query 1. If I hadn't flagged those first, the trend analysis would have been polluted by garbage values and I wouldn't have known it.

Data quality isn't a box you check. It's the foundation everything else stands on.

---

## Technical stack

- Microsoft SQL Server / SSMS
- CTEs, Window Functions (LAG, ROW_NUMBER), UNION ALL
- INNER / LEFT / RIGHT / FULL OUTER / SELF JOINs
- Star schema design, surrogate keys, foreign key relationships
- Data validation patterns, NULLIF, CASE statements

---

## A note on process

I used AI tools (Claude) to help structure queries, debug syntax errors, 
and think through the analysis — the same way a developer uses Stack Overflow 
or a senior colleague to sanity-check their work. Every query in this project 
was run by me against a real SQL Server database, and every result and 
business interpretation is my own.

