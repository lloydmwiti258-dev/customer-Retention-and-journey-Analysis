# 📊 Customer Retention & Business Intelligence Dashboard

<p>
  <img src="https://img.shields.io/badge/Python-3776AB?style=flat-square&logo=python&logoColor=white" alt="Python"/>
  <img src="https://img.shields.io/badge/pandas-150458?style=flat-square&logo=pandas&logoColor=white" alt="pandas"/>
  <img src="https://img.shields.io/badge/Flask-000000?style=flat-square&logo=flask&logoColor=white" alt="Flask"/>
  <img src="https://img.shields.io/badge/Chart.js-FF6384?style=flat-square&logo=chartdotjs&logoColor=white" alt="Chart.js"/>
  <img src="https://img.shields.io/badge/Google%20Sheets%20API-34A853?style=flat-square&logo=googlesheets&logoColor=white" alt="Google Sheets API"/>
  <img src="https://img.shields.io/badge/Vercel-000000?style=flat-square&logo=vercel&logoColor=white" alt="Vercel"/>
  <img src="https://img.shields.io/badge/Source-Private%20%C2%B7%20Showcase-2C5364?style=flat-square" alt="Showcase"/>
</p>

A full-stack analytics platform for a multi-branch retail business with shops across Kenya, the diaspora (Tanzania, Uganda) and an online store. It turns raw point-of-sale records kept in Google Sheets into a live decision-support dashboard. The dashboard covers **customer retention, repeat purchasing, customer acquisition cost (CAC), lifetime value (LTV), regional performance, cross-shop customer migration, recency/churn risk and product-bundling opportunities**.

**Author:** [Lloyd Mwiti Mugo](https://www.linkedin.com/in/lloyd-mwiti-mugo-94066a243/). I designed and built the whole pipeline myself: data ingestion, cleaning, metric engineering, the REST API, the interactive BI front end and cloud deployment.

---

## 🧭 Table of Contents
1. [Business Problem](#-business-problem)
2. [Solution Architecture](#-solution-architecture)
3. [Skills Demonstrated](#-skills-demonstrated)
   - [Data Engineering](#1-data-engineering)
   - [Data Analysis](#2-data-analysis)
   - [Business Intelligence & Visualization](#3-business-intelligence--visualization)
   - [Software Engineering & Deployment](#4-software-engineering--deployment)
4. [KPI Dictionary](#-kpi-dictionary)
5. [API Reference](#-api-reference)
6. [Tech Stack](#-tech-stack)
7. [Source Code & Access](#-source-code--access)
8. [Testing & Data Validation](#-testing--data-validation)
9. [Project Structure](#-project-structure)

---

## 💼 Business Problem

Management had thousands of transaction rows spread across 20+ branches in a shared spreadsheet, with no reliable way to answer questions like:

- *Are customers coming back, and how often?*
- *Which shops and regions bring in the most repeat revenue?*
- *When we open a new branch (e.g. KTDA, Kisii, Busia, Rongai), is it reaching **new** customers or just moving **existing** ones from other shops?*
- *How much does it cost to acquire a customer, and is that cost going up or down?*
- *Which customers have gone quiet and should get a win-back campaign?*
- *Which products are bought together and could be bundled?*

This project answers each of these with automated, refreshable analytics.

---

## 🏗 Solution Architecture

```
 ┌──────────────────────┐      ┌───────────────────────────────────────────────┐      ┌──────────────────────────┐
 │   DATA SOURCES       │      │          PYTHON / FLASK BACKEND (ETL + API)     │      │   BI FRONT END           │
 │                      │      │                                               │      │                          │
 │ • Google Sheets      │─────▶│ EXTRACT   gspread (service account) OR        │      │ • KPI cards              │
 │   (POS transactions) │      │           public CSV export with retry/backoff│      │ • 15+ Chart.js visuals   │
 │ • CSV uploads        │─────▶│ TRANSFORM pandas cleaning, normalization,     │─JSON▶│ • Region/shop/period     │
 │                      │◀─────│           feature engineering, KPI models     │      │   filters & drill-downs  │
 │  (write-back via API)│      │ LOAD      2-tier in-memory cache (30 min TTL) │      │ • Excel exports          │
 └──────────────────────┘      │ SERVE     15 REST endpoints, Excel exports    │      │ • CSV upload             │
                               └───────────────────────────────────────────────┘      └──────────────────────────┘
                                              Deployed on Vercel / Gunicorn
```

---

## 🛠 Skills Demonstrated

### 1. Data Engineering

| Skill | How it's applied in this project |
|---|---|
| **ETL pipeline design** | `get_customer_data()` runs a complete Extract → Transform → Load flow from Google Sheets into an analysis-ready pandas DataFrame. |
| **Multiple ingestion strategies** | Supports authenticated **Google Sheets API** access (service-account file *or* inline JSON credentials for cloud) and a **public CSV-export URL** fallback. Sheet URLs are rewritten into export URLs automatically. |
| **Resilient data extraction** | HTTP retry strategy (`urllib3.Retry`) with exponential backoff on `429/5xx` responses, request timeouts, and **graceful degradation to stale cache** if the source is unreachable. |
| **Data cleaning & standardization** | Reconciles schema drift (`Location` → `Shop`), normalizes shop names (trim + Title Case), strips currency symbols and text from `Price` and `MARKETING EXPENSE` with regex, coerces bad dates and numbers safely (`errors='coerce'`), and filters out B2B/organization records and out-of-scope years. |
| **Master data / dimension modelling** | Builds a **shop → region dimension** (`SHOP_REGION_MAP`) and a region color palette, so 20+ branches roll up into 9 business regions (Nairobi CBD, Metropolitan, Coastal, Western & Nyanza, Rift Valley, Central, Diaspora, Online, Rejects). |
| **Entity resolution** | Uses the phone number as a surrogate **Customer_ID** to track the same customer across shops and over time. |
| **Feature engineering** | Pre-computes each customer's `First_Visit_Date` once, at ingestion, so new-vs-returning and CAC calculations run fast downstream. |
| **Caching layer** | Two-level caching: a **raw-data cache** and a **computed-API-response cache** (30-minute TTL), plus a manual `/api/refresh-now` invalidation endpoint. |
| **Performance optimization** | Replaced an O(n × m) per-customer scan with **vectorized pre-grouping** (O(m log m)), computed purchase cycles with vectorized `diff()` + boundary masks instead of loops, and **lazy-loads** shop-level data only when a shop is selected. |
| **Reverse ETL / write-back** | `/api/upload` validates an uploaded CSV against a required schema and **appends it to the Google Sheet**, then invalidates both caches. |
| **Secrets & configuration management** | All credentials and settings come from environment variables via `python-dotenv`, with a documented `.env.example`. No secrets or key-file paths are hard-coded anywhere in the codebase, and a `.gitignore` keeps `.env`, service-account keys, customer data exports and logs out of version control. |

### 2. Data Analysis

| Analytical technique | Implementation |
|---|---|
| **Customer retention analysis** | Period-over-period retention (customers in period *t* who were also active in *t-1*) at **daily, weekly, monthly, quarterly, semi-annual and yearly** grain. |
| **Cohort-style cumulative retention** | Starting from a fixed launch date (April 2025), each month's customers are split into **New vs Returning**, with the revenue share of each group. |
| **Repeat-purchase behaviour** | Classifies customers as **one-timers vs repeat** by *distinct visit days* (not transaction count), which avoids inflating repeat rates with same-day multi-item baskets. |
| **Customer lifespan & purchase cycle** | Average days between first and last purchase, and the **mean inter-purchase interval** (mean of each customer's own average). |
| **Unit economics: CAC & LTV** | **CAC** = marketing spend ÷ newly acquired customers; **LTV proxy** = revenue ÷ active customers; both tracked with period-over-period **growth rates**. |
| **Growth analytics** | Repeat-customer growth, spend growth, lifespan growth, CAC growth and LTV growth at monthly, quarterly, semi-annual and annual levels. |
| **Recency segmentation (RFM-style)** | Customers are bucketed into **Monthly / Quarterly / Semi-Annual / Annual / Inactive (>1yr)** recency brackets, which feeds churn-risk and win-back targeting. |
| **Cross-shop migration / cannibalization analysis** | For new branches (KTDA, Kisii, Busia, Rongai, Website, Rejects): what share of customers are **genuinely new** and what share came from **other shops**, which shops they came from, and the revenue contribution of each source shop. |
| **Market basket analysis** | Product **co-occurrence (pair) mining** with `itertools.combinations`, broken down by gender, to recommend **product bundles**. |
| **Segmentation** | Performance cuts by **region, shop, gender and product** (top 10 products by revenue, average price, revenue per sale, unique buyers). |
| **Data reconciliation** | Regional totals are deliberately built as the **sum of shop-level figures**, so drill-downs always add back up to the parent total. |
| **Target benchmarking** | Every KPI is checked against a business target band (e.g. retention 15–20%, repeat rate 20–30%) and shown with a ✓ / ✗ status. |

### 3. Business Intelligence & Visualization

- **Interactive single-page BI dashboard**  with sections for:
  - 📈 General Overview (KPI cards)
  - 📢 Marketing & Acquisition Analysis (spend vs. new customers vs. CAC)
  - 🕵️ Customer Retention & Recency (bracket distribution + searchable at-risk list)
  - 📊 Periodic Performance Metrics (growth KPIs)
  - 🎯 Overall Performance Breakdown & 👥 Customer Distribution
  - 🌍 Regional Performance Analysis with shop-level drill-down
  - 🏪 Monthly Performance per Shop
  - 🔍 New-Branch Customer Origin breakdowns (KTDA, Kisii, Busia, Rongai, Rejects, Website)
  - 🛍️ Product Bundling Recommendations & 📦 Product Performance
- **Chart.js visualizations**: bar, line, doughnut, pie and radar charts, including dual-axis combined charts.
- **Slicers and filters**: unified period selector, region, shop, recency bracket, days-inactive threshold, and bundle filters by shop, region and timeline.
- **Self-service data exports**: one-click **Excel (.xlsx)** downloads of new, repeat and existing customer lists and recency/win-back lists, so marketing teams can act on them directly.
- **Self-service data loading**: CSV upload from the UI straight into the source system.
- **Storytelling with KPIs**: target-band status indicators, color-coded regions, and revenue-contribution percentages that turn the numbers into business actions.

### 4. Software Engineering & Deployment

- **RESTful API design** with Flask: 15 JSON and file endpoints, query-parameter filtering (`shop`, `region`, `granularity`, `period`, `start_date`, `end_date`), proper HTTP status codes (400/404/500) and structured error payloads.
- **Reusable, parameterized analytics functions**: one generic `calculate_shop_customer_classification(df, shop_name)` serves every branch instead of duplicated code.
- **Defensive programming**: divide-by-zero guards, NaN handling, flexible column detection (`Product` / `Item` / `Product Name` …), and tracebacks logged server-side.
- **Cloud deployment**: Vercel serverless config and Gunicorn for WSGI hosting, with pinned dependencies.
- **Test-driven validation**: mock-data verification scripts for core business logic (see [Testing](#-testing--data-validation)).

---

## 📖 KPI Dictionary

| KPI | Definition |
|---|---|
| **Unique Customers** | Distinct `Customer_ID` (phone number) in the period |
| **Visit** | A unique *(customer, calendar day)* pair |
| **Avg Spend per Visit** | Total revenue ÷ visits |
| **Avg Spend per Customer** | Total revenue ÷ unique customers |
| **One-Timer** | Customer with exactly 1 distinct visit day in the period |
| **Repeat Customer** | Customer with more than 1 distinct visit day in the period |
| **Repeat Rate** | Repeat customers ÷ unique customers (target: **20–30%**) |
| **Repeat Revenue %** | Revenue from repeat customers ÷ total revenue |
| **Retention Rate** | Customers active in both *t* and *t-1* ÷ customers in *t-1* (target: **15–20%**) |
| **Avg Lifespan** | Mean days between first and last purchase (customers with lifespan > 0) |
| **Avg Purchase Cycle** | Mean of each customer's average days between consecutive purchases |
| **New Customer** | Customer whose first-ever purchase falls in the period |
| **CAC** | Marketing expense ÷ new customers |
| **LTV (proxy)** | Period revenue ÷ active customers |
| **Recency Bracket** | Days since last visit: <30 Monthly · <90 Quarterly · <180 Semi-Annual · <365 Annual · ≥365 Inactive |
| **Cross-shop "Existing" Customer** | Customer of a branch who has also bought at any other branch |

---

## 🔌 API Reference

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/` | Dashboard UI |
| GET | `/api/data` | Full company-wide KPI payload (all time grains, regions, gender, products, marketing, periodic growth) |
| GET | `/api/shop-data?shop=` | Lazy-loaded KPI payload for one shop |
| GET | `/api/{ktda\|kisii\|busia\|rongai\|rejects\|website}-customer-analysis` | New vs. existing customer origin analysis (supports `start_date`, `end_date`) |
| GET | `/api/bundles?shop=&region=&granularity=&period=` | Top-10 product pair recommendations with gender split |
| GET | `/api/customer-recency?shop=&bracket=&days_inactive=&export=` | Recency distribution and at-risk customer list (JSON or Excel) |
| GET | `/api/export-customers?shop=&type=&logic=` | Excel export of new/repeat customers (`cross`-shop or `internal` logic) |
| POST | `/api/upload` | Validate a CSV and append it to the Google Sheet |
| POST | `/api/refresh-now` | Clear the cache and recompute from the live source |

---

## 🧰 Tech Stack

| Layer | Tools |
|---|---|
| Language | Python 3.13, JavaScript, HTML/CSS |
| Data processing | **pandas**, **NumPy**, itertools, collections |
| Data source / integration | **Google Sheets API** (gspread, google-auth), REST/CSV via requests |
| Backend / API | **Flask**, Gunicorn |
| Visualization | **Chart.js** |
| Reporting / export | openpyxl, XlsxWriter (Excel) |
| Config & security | python-dotenv, environment-based secrets |
| Deployment | **Vercel** (serverless Python) |

---

## 🔒 Source Code & Access

> This repository is a **project showcase**. The production source code is kept **private** because it runs on a live business's customer data.
>
> 📩 **A code walkthrough or live demo is available on request.** Reach me on [LinkedIn](https://www.linkedin.com/in/lloyd-mwiti-mugo-94066a243/).

**Source data model** (columns ingested from the POS sheet): `Date`, `First Name`, `Phone`, `Gender`, `Shop` (or `Location`), `Price`, `Product`, `MARKETING EXPENSE`.

**Deployment:** Python 3.13 · Flask · Gunicorn · Vercel · secrets supplied through environment variables (`.env`), never committed.

---

## ✅ Testing & Data Validation

Business logic is checked with **mock datasets that have known expected outputs**, which isolates the calculations from the live data source:

| Script | Validates |
|---|---|
| `verify_retention_logic.py` | Cumulative new vs. returning retention |
| `verify_performance_metrics.py` | CAC, LTV and period-over-period growth |
| `verify_marketing_logic.py` | Marketing spend to acquisition attribution |
| `verify_bundles.py` | Product co-occurrence / bundle mining |
| `verify_busia_mocked.py`, `verify_busia.py`, `verify_kisii.py` | Cross-shop new vs. existing classification |
| `verify_api.py`, `test_api.py` | Endpoint health and payload shape |

Data-profiling and diagnostic utilities (`inspect_worksheets.py`, `debug_columns.py`, `diagnose_data.py`, `check_scale.py`, `inspect_cycle.py`, `sample_marketing_data.py`) were used to explore the schema, find data-quality issues (e.g. inconsistent shop spellings, currency-formatted prices) and check dataset scale before building the metrics.

---

## 📁 Project Structure

```
CUSTOMERS DATA/
├── APP.PY                    # Flask app: ETL, KPI engine, REST API, exports
├── templates/
│   └── index.html            # Interactive BI dashboard (Chart.js)
├── requirements.txt          # Pinned dependencies
├── vercel.json               # Serverless deployment config
├── .env.example              # Environment variable template
├── verify_*.py / test_*.py   # Business-logic & API validation scripts
└── inspect_* / debug_* / diagnose_* / check_*.py   # Data profiling utilities
```

---

## 🎯 Business Impact

- **One source of truth** for retention and revenue KPIs across all branches and regions, replacing manual spreadsheet work.
- **Expansion decisions backed by data**: shows whether new branches bring in new customers or move existing ones over from other shops.
- **Marketing ROI visibility** through CAC and LTV trends.
- **Action-ready customer lists** for win-back and loyalty campaigns, exported straight to Excel.
- **Revenue uplift opportunities** from data-driven product bundling.

---

## 👤 Author

**Lloyd Mwiti Mugo**: Data Analyst · Data Engineer · Business Intelligence Developer

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Lloyd%20Mwiti%20Mugo-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/lloyd-mwiti-mugo-94066a243/)

I'm open to connecting about data analytics, data engineering and BI opportunities.

---

*Designed and built end-to-end by Lloyd Mwiti Mugo as a data analytics, data engineering and business intelligence project.*
