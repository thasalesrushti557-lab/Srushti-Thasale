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
  CTEs, CASE WHEN, date functions
- **Python** — Pandas, NumPy, Matplotlib, Seaborn
- **Google Colab** — notebook environment
- **GenAI** — Claude (Anthropic) used for guided learning

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
Safety Stock = (Max Daily Demand x Max Lead Time) 
             - (Avg Daily Demand x Avg Lead Time)

Reorder Point = (Avg Daily Demand x Avg Lead Time) 
              + Safety Stock

## Project Limitations
- Dataset covers 10 days of consumption data
- Static analysis — a production system would connect to live 
  hospital data and run alerts automatically daily

## Author
Srushti Thasale
BPharm | Data Analytics | MBA Business Analytics — SIMSREE 2026
[LinkedIn](https://www.linkedin.com/in/srushtiyt)
