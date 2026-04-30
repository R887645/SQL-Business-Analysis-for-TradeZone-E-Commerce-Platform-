# 🛒 TradeZone E-Commerce Platform: SQL Business Analysis

> **A full-cycle data analyst project:** data cleaning, business SQL, and an executive memo for a fast-growing Nigerian e-commerce platform.
![My Image Description](https://github.com/R887645/SQL-Business-Analysis-for-TradeZone-E-Commerce-Platform-/blob/main/ecommerce%20(2).png?raw=true)

---

## 📌 Table of Contents

- [Project Overview](#project-overview)
- [Business Problem](#business-problem)
- [My Thinking Process](#my-thinking-process)
- [Dataset & Schema](#dataset--schema)
- [Part A: Data Cleaning](#part-a-data-cleaning)
- [Part B: Business Questions & SQL](#part-b-business-questions--sql)
- [Part C: Key Insights](#part-c-key-insights)
- [Recommendations](#recommendations)
- [What the Data Cannot Tell Us](#what-the-data-cannot-tell-us)
- [Tools Used](#tools-used)
- [Project Structure](#project-structure)
- [How to Run](#how-to-run)

---

## Project Overview

This project simulates the work of a data analyst embedded in the data team at **TradeZone**, a Nigerian e-commerce platform operating across Lagos, Abuja, Kano, Port Harcourt, and Ibadan. The platform has grown quickly over 2023–2024, but leadership suspects that growth is masking deeper operational problems.

My job was to:
1. **Clean the raw dataset** and document every decision
2. **Answer eight business questions** using PostgreSQL
3. **Deliver an analyst memo** translating query results into decisions that leadership can act on

This is not a tutorial project. Every query, every decision, and every recommendation is tied to a real business question the Head of Growth and Head of Seller Operations needed answered before the 2025 planning cycle.

---

## Business Problem

> *"The platform has grown quickly, but that growth is masking some serious operational problems. Customer retention is dropping. Some sellers are dragging down overall platform ratings. Certain product categories are underperforming despite heavy promotion spend."*

When I read this brief, three questions jumped out immediately:

- **Who is actually generating revenue?** If growth looks good in aggregate, is it distributed or concentrated?
- **Are new customers activating?** Sign-up numbers mean nothing if customers never buy.
- **Is the product catalogue working?** High revenue from poorly rated products is a retention time bomb.

These three questions shaped the entire analysis. Everything else, payment methods, seller fulfilment, and quarterly trends — provided supporting context.

---

## My Thinking Process

### Why these metrics?

Most e-commerce analyses go straight to total revenue and top products. I deliberately started from the **customer level** (Query 5: spend segmentation) before touching product or seller data. My reasoning: if the customer base is structurally fragile, every other metric needs to be interpreted through that lens.

The 30-day conversion rate (Query 1) was chosen because it is the most honest signal of whether acquisition spend is actually working. A sign-up without a purchase is a cost, not a result.

Query 7 (ratings vs revenue) was chosen because it tests whether customers are buying products they *trust* or just products that are *visible*. A platform where mid-rated products outsell high-rated ones is a platform quietly losing repeat buyers.

### Why these three findings over the others?

I ran eight queries. I could have led with quarterly revenue growth (Q4 2024 grew 326% vs Q4 2023), that looks impressive. But impressive growth built on a fragile customer base is a risk, not a win. I chose the three findings that together told a coherent story:

1. **Revenue concentration** → the risk
2. **Conversion failure** → the pipeline problem feeding that risk
3. **Ratings vs revenue mismatch** → the retention mechanism making it worse

Each finding connects to the next. That is why they were selected.

---

## Dataset & Schema

The TradeZone database contains **7 tables**:

| Table | Description |
|---|---|
| `customers` | Customer profiles, location, signup date, account status |
| `orders` | Order records with status, dates, and total amount |
| `order_items` | Line-level product details per order |
| `products` | Product catalogue with category and unit price |
| `sellers` | Seller profiles, category, location, account status |
| `payments` | Payment method and amount per order |
| `reviews` | Customer ratings per product and order |

---

## Part A: Data Cleaning

Before writing a single analysis query, the dataset was fully cleaned. Key issues addressed:

### Missing Values

| Table | Column | NULLs Found | Decision |
|---|---|---|---|
| `customers` | `email` | 16 | Replaced with `'not_provided'` — email not needed for analysis |
| `products` | `unit_price` | 4 | Flagged as `MISSING_PRICE` — cannot guess a product price |
| `order_items` | `unit_price`, `line_total` | 97 | Backfilled from `products` where possible; remaining 97 retained as NULL (belong to 4 flagged products) |
| `orders` | `total_amount` | 150 | Recalculated from `order_items` where items existed; 19 remain NULL (cancelled orders with no items) |
| `payments` | `amount` | 155 | Filled from matching `orders.total_amount`; 12 remain NULL (no price source exists) |
| `orders` | `delivery_date` | 1,510 | Retained — only Delivered orders should have a date; all NULLs belong to Cancelled/Shipped/Pending orders |

![My Image Description](https://github.com/R887645/SQL-Business-Analysis-for-TradeZone-E-Commerce-Platform-/blob/main/dATA%20CLEANING_Tradezone.png?raw=true)

### Duplicates
Checked across `customers`, `sellers`, and `orders`. **No duplicate records found.**

### Inconsistent Formatting
- City names standardised using `INITCAP(TRIM())` — fixed issues like `'LAGOS'`, `'Lago s'`, `'PortHarcourt'`, `'Port-Harcourt'`
- Product categories normalised using exact matches for simple categories and `ILIKE` wildcards for complex variations (e.g. `'beautyandpersonalcare'` → `'Beauty & Personal Care'`)

### Data Validation
- **124 orders** flagged where `total_amount` differed from sum of `order_items.line_total` by more than ₦10
- **5 review ratings** found outside the valid 1–5 range and flagged
- **No negative product prices** found
- **No discount column** exists in this dataset — not applicable

![My Image Description](https://github.com/R887645/SQL-Business-Analysis-for-TradeZone-E-Commerce-Platform-/blob/main/Screenshot%202026-04-24%20172031.png?raw=true)

---

## Part B: Business Questions & SQL

### Q1: Customer Acquisition & 30-Day Conversion
*Find the top 5 states by new sign-ups in 2024 and calculate the percentage who made at least one purchase within 30 days of signing up.*

```sql
WITH new_customers AS (
    SELECT customer_id, state, signup_date
    FROM customers
    WHERE EXTRACT(YEAR FROM signup_date) = 2024
),
converted_customers AS (
    SELECT DISTINCT nc.customer_id, nc.state
    FROM new_customers nc
    JOIN orders o ON o.customer_id = nc.customer_id
       AND o.order_date <= nc.signup_date + INTERVAL '30 days'
)
SELECT
    nc.state,
    COUNT(nc.customer_id) AS total_signups,
    COUNT(cc.customer_id) AS converted_customers,
    ROUND((COUNT(cc.customer_id)::decimal / COUNT(nc.customer_id)) * 100, 2) AS conversion_percentage
FROM new_customers nc
LEFT JOIN converted_customers cc ON nc.customer_id = cc.customer_id
GROUP BY nc.state
ORDER BY total_signups DESC
LIMIT 5;
```

**Result:**
| State | Sign-Ups | Converted | Conversion Rate |
|---|---|---|---|
| Lagos | 146 | 72 | 49.32% |
| FCT | 92 | 38 | 41.30% |
| Rivers | 66 | 28 | 42.42% |
| Oyo | 63 | 21 | 33.33% |
| Kano | 58 | 18 | 31.03% |

---

### Q2: Product Performance
*Top 10 products by total revenue in 2024.*

**Result:** All top 10 products were Electronics. HP Pavilion 15 Laptop led with ₦26.7M in revenue across 25 orders. The top 10 ranged from ₦17.7M to ₦26.7M, reflecting strong demand for tech hardware and accessories.

![my image}(https://github.com/R887645/SQL-Business-Analysis-for-TradeZone-E-Commerce-Platform-/blob/main/Q2_Tradezone.png?raw=true)

---

### Q3: Seller Fulfilment Efficiency
*Top 20 fastest sellers by average fulfilment hours, among those with at least 20 completed orders.*

**Result:** Fastest fulfilment: **RunFast NG** at 91.20 hours average. However, RunFast holds only a 3.25 customer rating despite being the quickest — showing that speed alone does not drive satisfaction. **SportsCentral NG** combined competitive speed (98.40 hrs) with a 4.08 rating.

![My Image Description](https://github.com/R887645/SQL-Business-Analysis-for-TradeZone-E-Commerce-Platform-/blob/main/q3_tRADEZONE.png?raw=true)

---

### Q4 — Quarterly Revenue Trends
*Compare quarterly revenue across 2023 and 2024. Identify the strongest growth quarter.*

![My Image Description](https://github.com/R887645/SQL-Business-Analysis-for-TradeZone-E-Commerce-Platform-/blob/main/q4_tradezone.png?raw=true)

**Result:**
| Quarter | Revenue 2023 | Revenue 2024 | Growth | Growth % |
|---|---|---|---|---|
| Q1 | ₦1,000,181 | ₦56,520,465 | ₦55,520,285 | 5,551% |
| Q2 | ₦11,616,690 | ₦76,818,510 | ₦65,201,820 | 561% |
| Q3 | ₦25,942,715 | ₦117,442,048 | ₦91,499,333 | 353% |
| **Q4** | **₦40,084,104** | **₦170,586,322** | **₦130,502,218** | **326%** |

**Q4 2024 recorded the strongest absolute revenue growth** at ₦130.5M — flagged as the Strongest Growth Quarter.

---

### Q5: Customer Spend Segmentation
*Segment 2024 customers into High, Medium, and Low Spenders.*

![My Image description](https://github.com/R887645/SQL-Business-Analysis-for-TradeZone-E-Commerce-Platform-/blob/main/q5_tradezone.png?raw=true)

**Result:**
| Segment | Customers | Avg Spend | Total Revenue |
|---|---|---|---|
| High Spenders (≥₦100k) | 603 | ₦1,436,927 | ₦866,466,981 |
| Medium Spenders (₦50k–₦99,999) | 25 | ₦69,076 | ₦1,726,908 |
| Low Spenders (<₦50k) | 49 | ₦22,969 | ₦1,125,480 |

High Spenders account for **99.6% of all 2024 revenue.**

---

### Q6: Payment Method Preferences by State
*Transaction count and total amount per payment method, per state.*

**Result:** Card dominates in Lagos (371 transactions) and Rivers. Mobile Money leads in Kano. Cash on Delivery leads in Oyo. Urban states favour digital payments; others still rely on traditional methods.

---

### Q7: Review Ratings and Sales Performance
*Group products by average rating. Calculate product count, total revenue and average unit price per group.*

**Result:**
| Rating Category | Products | Total Revenue | Avg Unit Price |
|---|---|---|---|
| High Rated (≥4.0) | 113 | ₦1,136,119,134 | ₦45,763 |
| Mid Rated (3.0–3.99) | 116 | ₦1,686,546,232 | ₦64,905 |
| Low Rated (<3.0) | 32 | ₦349,490,173 | ₦52,009 |

Mid-Rated products outsell High-Rated products by approximately ₦550 million.

---

### Q8 — Top Seller Bonus Qualification
*Top 10 sellers in 2024 by revenue, with at least 10 completed orders and average rating ≥ 4.0.*

**Result:** 10 sellers qualified. **SportsCentral NG** led with ₦11.3M revenue and 4.08 rating. **WellnessHub NG** had the highest rating at 4.44. All 10 completed between 10 and 19 orders — strong candidates for the 2025 bonus programme.

---

## Part C: Key Insights

### Insight 1: Revenue Is Dangerously Concentrated in 603 Customers
*(Source: Query 5)*

603 High Spenders generated ₦866.5M, 99.6% of all 2024 revenue — at an average of ₦1,436,927 per customer. The remaining 74 customers contributed less than ₦3M combined. If just 10% of High Spenders churned in 2025, the platform would lose approximately **₦86.6 million overnight** with no realistic buffer from lower segments.

This is not a tiered customer base. It is a single segment holding up the entire platform.

### Insight 2: More Than Half of New Customers Are Not Converting Within 30 Days
*(Source: Query 1)*

Lagos, the top acquisition state, converted only 49.32% of new sign-ups within 30 days. Kano converted just 31.03%. Across all five states, the platform is losing between half and seven-in-ten new customers before they ever transact. Every unactivated customer is a lost relationship, and the pipeline for growing the high-spend segment is leaking at the very first step.

### Insight 3: Mid-Rated Products Outsell High-Rated Products by ₦550 Million
*(Source: Query 7)*

Products rated 3.0–3.99 generated ₦1.69B in revenue vs ₦1.14B for products rated 4.0 and above. Customers are buying based on price, availability, or promotions, not satisfaction. For a platform already facing dropping retention, this pattern will compound: customers who are only moderately happy with their purchases will not come back.

---

## Recommendations

### Recommendation 1: Launch a High-Spender Retention Programme
**Owner:** Head of Growth — CRM Sub-team
**Action:** Design and deploy a dedicated retention programme for the 603 High Spenders within 30 days. This should include personalised communication, loyalty incentives, priority customer service, and purchase-behaviour-based offers.
**Expected outcome (60–90 days):** Measurable improvement in repeat purchase rate among High Spenders and a reduction in churn within this segment. Protecting even 10% of at-risk high spenders shields approximately ₦86.6M in revenue.
**Traceable to:** Query 5

### Recommendation 2: Implement a 30-Day New Customer Activation Campaign
**Owner:** Head of Growth:Acquisition & Onboarding Sub-team
**Action:** Introduce a structured onboarding sequence for all new sign-ups triggered at registration, including a time-limited first-purchase incentive (e.g., delivery waiver valid for 14 days), curated product recommendations by state, and at least two follow-up touchpoints within the 30-day window.
**Expected outcome (60–90 days):** Lift the average 30-day conversion rate from the current 31–49% range to above 55% across all five top states. Improved activation directly grows the pipeline toward medium and high-spend tiers.
**Traceable to:** Query 1

---

## What the Data Cannot Tell Us

**Question leadership will ask: "How much profit is TradeZone actually generating from these sales?"**

The current dataset contains only revenue data. It cannot answer whether that revenue is profitable. A product or seller ranking highly by revenue may contribute very little to the bottom line if costs of goods sold are high, commission rates are low, or fulfilment expenses are significant.

To answer this, the data team would need:
- Product-level cost of goods sold (COGS)
- Seller commission or platform fee structure per transaction
- Fulfilment and logistics cost per order, by state

With these inputs, net profit per product, per seller, and per customer segment could all be calculated, giving leadership a complete picture before committing the 2025 budget to any category or incentive programme.

---

## Tools Used

| Tool | Purpose |
|---|---|
| **PostgreSQL 17** | All data cleaning and business queries |
| **pgAdmin** | Query execution and result validation |
| **SQL (CTEs, Window Functions, CASE, JOINS)** | Core query logic |

---

## Project Structure

```
HNG14-DA-S2/
│
├── Part 1.sql               # Part A: Full data cleaning script
│
├── Q1.sql                   # Customer Acquisition & 30-Day Conversion
├── Q2.sql                   # Product Performance
├── Q3.sql                   # Seller Fulfilment Efficiency
├── Q4.sql                   # Quarterly Revenue Trends
├── Q5.sql                   # Customer Spend Segmentation
├── Q6.sql                   # Payment Method Preferences by State
├── Q7.sql                   # Review Ratings and Sales Performance
├── Q8.sql                   # Top Seller Bonus Qualification
│
├── cleaned_dump.sql         # Full cleaned database dump
│
└── Analyst_Memo.pdf         # Part C: Executive memo for leadership
```

---

## How to Run

1. **Restore the database dump**
```bash
psql -U postgres -d tradezone_db -f cleaned_dump.sql
```

2. **Run the cleaning script first** (if starting from raw data)
```bash
psql -U postgres -d tradezone_db -f "Part 1.sql"
```

3. **Run any query file**
```bash
psql -U postgres -d tradezone_db -f Q1.sql
```

Or open any `.sql` file directly in pgAdmin and execute.

---

## Key Takeaways

If I were hired as a full-time analyst at TradeZone today, the first thing I would do is protect the 603 high-value customers who are holding up the entire revenue structure, before touching any growth or acquisition budget. Growth metrics look strong on the surface. The data underneath tells a more urgent story.

---

*Project completed as part of HNG Internship Stage 2: Data Analytics Track.*
