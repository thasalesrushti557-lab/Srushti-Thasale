[Healthcare queires.sql](https://github.com/user-attachments/files/28940329/Healthcare.queires.sql)
[GENAI analytics - Colab.pdf](https://github.com/user-attachments/files/28940304/GENAI.analytics.-.Colab.pdf)
# Srushti-Thasale
End-to-end hospital-- ============================================================
-- Healthcare Supply Chain & Inventory Optimization
-- Oracle SQL Analytical Queries
-- Author: Srushti Thasale
-- ============================================================


-- ============================================================
-- QUERY 1: Average Daily Demand
-- Purpose: Calculate average daily consumption per item
-- Used as baseline input for Reorder Point formula
-- ============================================================

SELECT 
    i.ITEM_NAME,
    AVG(c.QT_CONSUMED) AS avg_daily_demand
FROM consumption_fact c
JOIN inventory i ON c.ITEM_ID = i.ITEM_ID
GROUP BY i.ITEM_NAME;


-- ============================================================
-- QUERY 2: Maximum Daily Demand
-- Purpose: Identify worst-case daily consumption per item
-- Used in Safety Stock formula to account for demand spikes
-- ============================================================

SELECT 
    i.ITEM_NAME,
    MAX(c.QT_CONSUMED) AS max_daily_demand
FROM consumption_fact c
JOIN inventory i ON c.ITEM_ID = i.ITEM_ID
GROUP BY i.ITEM_NAME;


-- ============================================================
-- QUERY 3: Average and Maximum Supplier Lead Time
-- Purpose: Determine how many days each supplier takes to deliver
-- Both avg and max lead time needed for Safety Stock formula
-- ============================================================

SELECT 
    i.ITEM_NAME,
    AVG(s.SUPPLIER_LEAD_TIME) AS avg_lead_time,
    MAX(s.SUPPLIER_LEAD_TIME) AS max_lead_time
FROM stock_fact s
JOIN inventory i ON s.ITEM_ID = i.ITEM_ID
GROUP BY i.ITEM_NAME;


-- ============================================================
-- QUERY 4: Safety Stock and Reorder Point Calculation (CTE)
-- Purpose: Calculate exact reorder trigger levels per item
-- Formula:
--   Safety Stock = (Max Daily Demand x Max Lead Time) 
--                - (Avg Daily Demand x Avg Lead Time)
--   Reorder Point = (Avg Daily Demand x Avg Lead Time) 
--                 + Safety Stock
-- ============================================================

WITH demand AS (
    SELECT 
        i.ITEM_ID,
        i.ITEM_NAME,
        AVG(c.QT_CONSUMED) AS avg_daily_demand,
        MAX(c.QT_CONSUMED) AS max_daily_demand
    FROM consumption_fact c
    JOIN inventory i ON c.ITEM_ID = i.ITEM_ID
    GROUP BY i.ITEM_ID, i.ITEM_NAME
),
lead AS (
    SELECT 
        ITEM_ID,
        AVG(SUPPLIER_LEAD_TIME) AS avg_lead_time,
        MAX(SUPPLIER_LEAD_TIME) AS max_lead_time
    FROM stock_fact
    GROUP BY ITEM_ID
),
calculations AS (
    SELECT 
        d.ITEM_NAME,
        d.avg_daily_demand,
        d.max_daily_demand,
        l.avg_lead_time,
        l.max_lead_time,
        (d.max_daily_demand * l.max_lead_time) - 
        (d.avg_daily_demand * l.avg_lead_time) AS safety_stock,
        (d.avg_daily_demand * l.avg_lead_time) + 
        ((d.max_daily_demand * l.max_lead_time) - 
        (d.avg_daily_demand * l.avg_lead_time)) AS reorder_point
    FROM demand d
    JOIN lead l ON d.ITEM_ID = l.ITEM_ID
)
SELECT * FROM calculations;


-- ============================================================
-- QUERY 5: Stockout Alert
-- Purpose: Flag items where current stock has already fallen
-- below the calculated reorder point — ORDER NOW items
-- ============================================================

WITH demand AS (
    SELECT 
        i.ITEM_ID,
        i.ITEM_NAME,
        AVG(c.QT_CONSUMED) AS avg_daily_demand,
        MAX(c.QT_CONSUMED) AS max_daily_demand
    FROM consumption_fact c
    JOIN inventory i ON c.ITEM_ID = i.ITEM_ID
    GROUP BY i.ITEM_ID, i.ITEM_NAME
),
lead AS (
    SELECT 
        ITEM_ID,
        AVG(SUPPLIER_LEAD_TIME) AS avg_lead_time,
        MAX(SUPPLIER_LEAD_TIME) AS max_lead_time
    FROM stock_fact
    GROUP BY ITEM_ID
),
reorder AS (
    SELECT 
        d.ITEM_ID,
        d.ITEM_NAME,
        (d.avg_daily_demand * l.avg_lead_time) + 
        ((d.max_daily_demand * l.max_lead_time) - 
        (d.avg_daily_demand * l.avg_lead_time)) AS reorder_point
    FROM demand d
    JOIN lead l ON d.ITEM_ID = l.ITEM_ID
)
SELECT 
    r.ITEM_NAME,
    s.CURRENT_STOCK,
    ROUND(r.reorder_point, 2) AS reorder_point,
    'ORDER NOW' AS alert
FROM reorder r
JOIN stock_fact s ON r.ITEM_ID = s.ITEM_ID
WHERE s.CURRENT_STOCK < r.reorder_point;


-- ============================================================
-- QUERY 6: Expiry Risk Analysis
-- Purpose: Identify batches expiring within 30/60/90 days
-- and calculate total rupee value at risk of loss
-- ============================================================

SELECT 
    i.ITEM_NAME,
    TO_CHAR(s.BATCH_EXP_DATE, 'YYYY-MM-DD') AS batch_exp_date,
    s.CURRENT_STOCK,
    i.UNIT_COST,
    ROUND(s.CURRENT_STOCK * i.UNIT_COST, 2) AS rupee_value_at_risk,
    CASE
        WHEN s.BATCH_EXP_DATE <= SYSDATE + 30 THEN 'Expires in 30 days'
        WHEN s.BATCH_EXP_DATE <= SYSDATE + 60 THEN 'Expires in 60 days'
        WHEN s.BATCH_EXP_DATE <= SYSDATE + 90 THEN 'Expires in 90 days'
        ELSE 'Safe'
    END AS expiry_risk
FROM stock_fact s
JOIN inventory i ON s.ITEM_ID = i.ITEM_ID
ORDER BY s.BATCH_EXP_DATE ASC;


-- ============================================================
-- QUERY 7: ABC Classification
-- Purpose: Rank items by total annual spend
-- Class A = high value items needing tight control
-- Class B = moderate value
-- Class C = low value, bulk ordering acceptable
-- ============================================================

SELECT 
    i.ITEM_NAME,
    i.UNIT_COST,
    SUM(c.QT_CONSUMED) AS total_consumed,
    ROUND(SUM(c.QT_CONSUMED) * i.UNIT_COST, 2) AS total_spend,
    CASE
        WHEN ROUND(SUM(c.QT_CONSUMED) * i.UNIT_COST, 2) > 100000 
            THEN 'Class A'
        WHEN ROUND(SUM(c.QT_CONSUMED) * i.UNIT_COST, 2) > 10000  
            THEN 'Class B'
        ELSE 'Class C'
    END AS abc_class
FROM consumption_fact c
JOIN inventory i ON c.ITEM_ID = i.ITEM_ID
GROUP BY i.ITEM_NAME, i.UNIT_COST
ORDER BY total_spend DESC;
 inventory optimization system using Oracle SQL and Python — identifying pharmaceutical stockouts and expiry risks across 15 medicines and equipment items.
# Healthcare Supply Chain & Inventory Optimization

## Project Overview
An end-to-end hospital inventory management system built using 
Oracle SQL and Python to eliminate pharmaceutical stockouts and 
minimize financial loss from expired medicines and equipment.

## Business Problem
Hospitals face two critical operational risks:
- **Stockouts** of life-critical medicines causing medical emergencies
  (e.g. a doctor needing insulin at 2 AM with an empty pharmacy)
- **Expiry losses** from medicines sitting unused until they expire —
  100% financial loss with no recovery possible

This project builds an automated system that calculates exactly 
when to reorder each item and flags high-value batches at risk 
of expiring before they are used.

## Tools & Technologies
- **Oracle SQL** — relational database design, 7 analytical queries,
  CTEs, window functions, CASE WHEN, date functions
- **Python** — Pandas, NumPy, Matplotlib, Seaborn
- **Google Colab** — notebook environment
- **GenAI** — Claude (Anthropic) used for guided learning and 
  query assistance

## Database Schema
3 normalized tables with Primary Keys and Foreign Keys:

| Table | Purpose |
|---|---|
| inventory | Master list of 15 items (medicines + equipment) |
| stock_fact | Current batch stock levels and expiry dates |
| consumption_fact | 20 daily usage transactions across 3 departments |

## SQL Queries
7 analytical queries covering:
1. Average Daily Demand per item
2. Maximum Daily Demand (worst-case scenario)
3. Average and Maximum Supplier Lead Time
4. Safety Stock + Reorder Point calculation (using CTEs)
5. Stockout Alert — items already below danger level
6. Expiry Risk Analysis — rupee value at risk by urgency bucket
7. ABC Classification — items ranked by total spend

## Key Findings
- **Critical Stockout:** Insulin Glargine current stock (80 units)
  already below reorder point (84 units) — immediate order required
- **₹74,960 in expiry risk** identified across 10 items expiring
  within 30 days
- **ICU** identified as highest-consumption ward for critical 
  medications (Insulin, Morphine)
- **All items classified Class C** in ABC analysis due to 10-day 
  dataset — with full year data, Ventilator (₹8.5L) and Dialysis 
  Machine (₹12L) would be Class A

## Visualizations
1. Expiry Risk Bar Chart — rupee value at risk by expiry status
2. ABC Classification Bar Chart — items ranked by total spend
3. Stockout Alert Chart — current stock vs reorder point comparison
4. Department Consumption Heatmap — which ward uses what most

## Formulas Used
## Project Limitations
- Dataset covers 10 days of consumption data — ABC classification
  requires minimum 6-12 months for meaningful results
- Static analysis — a production system would connect to live 
  hospital data and run alerts automatically daily

## Author
Srushti Thasale
BPharm | Data Analytics | SIMSREE MBA (Business Analytics) 2026
[LinkedIn](https://www.linkedin.com/in/srushtiyt)
