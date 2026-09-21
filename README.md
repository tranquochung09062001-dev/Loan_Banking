# 💳 Credit Portfolio Risk & Concentration Analysis Dashboard | ETL Pipeline, Star Schema & Power BI

![Credit Portfolio Risk & Concentration Analysis Dashboard](https://raw.githubusercontent.com/tranquochung09062001-dev/Loan_Banking/main/Dashboard_Cover.png)

A 2,000-customer credit portfolio analysis for a commercial bank's Risk Management & Data Analytics division, built on a Python + PostgreSQL ETL pipeline (Bronze → Silver → Gold) and delivered as a Power BI dashboard for senior leadership: **is the portfolio still growing, where is exposure concentrated, and is the riskiest segment priced appropriately?**

## 📑 Table of Contents

1. [Background & Overview](#-background--overview)
2. [Dataset Description & Data Structure](#-dataset-description--data-structure)
3. [Design Thinking Process](#-design-thinking-process)
4. [Data Cleaning & Standardization (ETL Pipeline)](#-data-cleaning--standardization-etl-pipeline)
5. [Main Process](#-main-process)
6. [Key Insights & Visualizations](#-key-insights--visualizations)
7. [Skills & Tools Applied](#-skills--tools-applied)
8. [Final Conclusion & Recommendations](#-final-conclusion--recommendations)

## 📌 Background & Overview

**Objective:** the bank's loan book had grown organically for years without a single, comparable view of where credit risk actually sits. This project consolidates raw loan-origination data (2012–2019) into a governed data warehouse and a Power BI report that lets the **Head of Risk Management** answer three questions in one sitting: is growth slowing, is exposure concentrated in a way that would hurt if one segment turned, and is the bank's fixed risk-tier pricing rule behaving the way it was designed to.

**Who this is for:** Risk Management & Data Analytics leadership — reviewed quarterly, not daily; the report is built to be read in one pass rather than queried interactively.

**Key portfolio metrics (2012–2019):**

| Total customers | Total credit limit | Avg. limit / loan | YoY growth | Avg. loans / customer |
|---|---|---|---|---|
| **2,000** | **2.95 billion** | **1.76 million** | **-5.07%** | **1** |

Each customer carries essentially one loan — the product behaves as a one-time transaction rather than a repeat credit relationship, which shapes how growth and retention should be read throughout the report.

## 🗂 Dataset Description & Data Structure

Source data is loan-origination records spanning 2012–2019, ingested from CSV into PostgreSQL and modeled as a star schema before being loaded into Power BI.

<img width="700" alt="Credit Portfolio Data Model" src="https://raw.githubusercontent.com/tranquochung09062001-dev/Loan_Banking/main/model.png" />

**`dwh.fact_loan`** — one row per loan, holding the foreign keys to all five dimensions plus the cleaned numeric measures: `amt_funded`, `amt_loan_balance`, `pct_interest_rate`, `amt_monthly_payment`, `num_duration_years/months`, `amt_total_payments`, `amt_property_value`.

| Dimension | Grain | Key attributes |
|---|---|---|
| `dwh.dim_customer` | 1 row / customer | full_name, num_age, num_employment_length, amt_income, email |
| `dwh.dim_date` | 1 row / calendar day | continuous date range spanning every `funded_date` in the data |
| `dwh.dim_location` | 1 row / city+state | city, state, region |
| `dwh.dim_job` | 1 row / title+position+industry | title, position, industry |
| `dwh.dim_purpose` | 1 row / loan purpose | purpose_name (typo-corrected, case-normalized) |

## 🧠 Design Thinking Process

To keep the report anchored to a real decision-maker rather than "every chart that could be built," the dashboard was scoped using a 4-step Design Thinking pass:

<table>
<tr><td><img src="https://raw.githubusercontent.com/tranquochung09062001-dev/Loan_Banking/main/step1_design_thinking.png" width="480"/></td><td><img src="https://raw.githubusercontent.com/tranquochung09062001-dev/Loan_Banking/main/step2_design_thinking.png" width="480"/></td></tr>
<tr><td><img src="https://raw.githubusercontent.com/tranquochung09062001-dev/Loan_Banking/main/design_thinking_step3.png" width="480"/></td><td><img src="https://raw.githubusercontent.com/tranquochung09062001-dev/Loan_Banking/main/design_thinking_step4.png" width="480"/></td></tr>
</table>

**1. Empathize** — the stakeholder is a Head of Risk Management who reviews the report at quarterly portfolio meetings, never opens raw tables, compares severity *across* regions/industries rather than reading one number in isolation, and thinks in limit-to-income ratio tiers rather than raw amounts.

**2. Define** — *"A Head of Risk Management needs to see whether portfolio growth is slowing, where exposure is concentrated (region, industry, ratio tier), and whether the riskiest borrowers are being priced/sized appropriately — before a downturn in any one segment threatens the whole book."*

**3. Ideate** — six decisions the report has to support: size total exposure & growth trend, find concentration by region/industry, check risk-tier pricing discipline, flag the largest single exposures, segment customers by income/age, and translate all of it into a limit or monitoring action.

**4. Prototype & Review** — the final report reads in exactly four pages, in this order: Overview → Region/Purpose (incl. the 70/30 industry view) → Customer → Risk Tiers — mirroring how the stakeholder actually scans a static report rather than an ad-hoc query tool.

## 🧹 Data Cleaning & Standardization (ETL Pipeline)

Data moves through a 2-stage pipeline into a classic **Bronze → Silver → Gold** medallion architecture: Python handles extract-and-load into a raw layer; all cleaning, modeling and imputation is done in pure SQL (`Ad_Bank.sql`, run in pgAdmin) across the staging and dwh layers.

| Layer (schema) | Role | Data characteristics |
|---|---|---|
| **raw (Bronze)** | Store the CSV exactly as received | every column cast to `TEXT`; a `source_file` column added for traceability |
| **staging (Silver)** | Clean, type-cast, split into a star schema | surrogate (identity) keys, UNIQUE/FK constraints, safe conversion functions |
| **dwh (Gold)** | Final reporting/BI layer | missing values imputed; columns renamed to business-standard prefixes (`amt_`, `num_`, `pct_`) |

**Extract → Raw (Python: `extract.py`, `load.py`, `config.py`, `main.py`)**
- `extract_files()` scans the configured folder for `.csv` files and loads each into a DataFrame — no cleaning happens at this step.
- `load_to_raw()` maps each file to its target raw table via `TABLE_MAPPING` (unmapped files are skipped with a warning), adds a `source_file` column, force-casts the entire DataFrame to string (`astype(str)`) — the Bronze-layer principle of keeping data as raw as possible before any numeric/date casting risks losing rows to format errors — and appends it to PostgreSQL in 1,000-row chunks, so re-runs accumulate rather than overwrite.
- `main.py` checks the database connection first and aborts immediately (`SystemExit`) rather than failing midway through a load.

**Transform → Staging (`Ad_Bank.sql`) — shared cleaning functions**

| Function | Purpose |
|---|---|
| `staging.safe_int(txt)` | text → integer; returns `NULL` on empty/invalid input instead of crashing the whole statement |
| `staging.safe_date(txt, fmt)` | text → date on a given format; `NULL` on failure |
| `staging.clean_thousand_number(txt)` | strips thousand-separator dots (e.g. `"1.234.000"`) into numeric |
| `staging.clean_percent(txt)` | strips `%` and converts to decimal (e.g. `"3%"` → `0.03`) |

**Dimension-building rules:**
- `dim_date` — generated as a **continuous** date series from MIN to MAX `funded_date`, so no calendar day is missing even if no loan funded that day.
- `dim_location` — distinct city/state/region, trimmed and state codes upper-cased.
- `dim_job` — distinct title/position/industry; blank titles default to `"Unknown"` instead of being dropped.
- `dim_purpose` — case-normalized **and typo-corrected** at the source: `"boat"`/`"Boat"` merged into one value; the misspelling `"commerical property"` is mapped to `"Commercial Property"` alongside the correctly-spelled `"commercial property"`.
- `dim_customer` — `birth_date` reassembled from three separate columns (year/month/day); first/middle/last name concatenated into `full_name`; email lower-cased; income de-formatted via `clean_thousand_number`.

**Fact-building:** `staging.fact_loan` is populated by joining the raw data to all five dimension tables on their natural keys (SSN, city+state, title+position+industry, purpose name, funded date) to resolve surrogate keys, with every amount/rate column passed through the safe-conversion functions above.

**Build DWH (Gold):** `dwh.fact_loan` and `dwh.dim_customer` are built from staging via `CREATE TABLE AS SELECT`, applying one imputation rule consistently: `COALESCE(col, ROUND(AVG(col) OVER (), n))` — any `NULL` is filled with the column-wide average — applied to `amt_funded`, `num_duration_years/months`, `pct_interest_rate`, `amt_monthly_payment`, `num_total_past_payments`, `amt_total_payments`, `amt_loan_balance`, `amt_property_value` (fact) and `num_age`, `num_employment_length`, `amt_income` (dim_customer). Columns are simultaneously renamed to business-readable prefixes — `amt_` (money), `num_` (counts/years), `pct_` (percentages) — so a Power BI user can identify a column's data type from its name alone. The remaining four dimension tables are copied unchanged since they hold no numeric columns to impute. The script's final step re-counts `NULL`s in every processed column and confirms zero remain.

**Row lifecycle at a glance:**

| Step | Data state |
|---|---|
| 1. Raw CSV | Raw text, possibly malformatted numbers, typos, missing values |
| 2. `raw.raw_Dataset_Banking` | Same content, all cast to TEXT, `source_file` added |
| 3. staging (Dim + Fact) | Split into tables, correctly typed, deduplicated, typos fixed, surrogate keys + FK constraints |
| 4. dwh (Dim + Fact) | Same as staging but zero NULLs in key numeric columns (mean-imputed), business-standard column names — ready for Power BI |

## ⚒️ Main Process

1. **Extract & Load** — Python reads every CSV in the source folder and appends it into PostgreSQL's `raw` schema as text, with a connection check that fail-fasts before any load begins.
2. **Transform & Model** — pure SQL (`Ad_Bank.sql`) casts types safely, deduplicates, corrects known data-entry typos, and builds a 1-fact / 5-dimension star schema in the `staging` schema with surrogate keys and FK constraints.
3. **Build DWH & Report** — the `dwh` (Gold) layer imputes remaining NULLs with column averages, renames columns to `amt_/num_/pct_` conventions, runs a zero-NULL QA check, and is imported into Power BI (Import mode) from its 6 `dwh` tables only — Power BI auto-detects the 5 one-to-many relationships from the surrogate keys, verified in Model view with single-direction filtering from each Dim to the Fact.

## 📊 Key Insights & Visualizations

### 1. Portfolio Overview

![Overview Dashboard](https://raw.githubusercontent.com/tranquochung09062001-dev/Loan_Banking/main/Overview_Dashboard.png)

- Gender split is close to even: **Female 51.52%** (864 customers) vs. **Male 48.48%** (813).
- After recalibrating income bands to the actual tercile thresholds, customers split almost evenly into three income groups — Under 3M: **35.8%**, 3–6M: **32.9%**, Over 6M: **31.3%** — but the distribution's right tail is steep: the highest recorded income (52.54M) is **12.7×** the median.
- New-customer counts fluctuate 193–227/year across 2012–2019 with no clear expansion trend, consistent with the **-5.07% YoY** growth figure — the portfolio looks flat-to-slowing rather than clearly growing or shrinking.

### 2. Region & Purpose (incl. the 70/30 Industry View)

![Region and Purpose Dashboard](https://raw.githubusercontent.com/tranquochung09062001-dev/Loan_Banking/main/Region_Purpose_Dashboard.png)

| Region | % of customers | Dominant purpose (Investment Property) |
|---|---|---|
| Northeast | 49.9% | 31.12% |
| South | 18.1% | 29.34% |
| West | 15.7% | 33.29% |
| Midwest | 16.3% | 35.52% |

- **Geographic concentration:** Northeast alone holds essentially half the customer base — nearly 3× any other single region — and "Investment Property" is the leading loan purpose in *every* region, not just Northeast.
- **Industry concentration (70/30 rule):** only **12 of 17 industries (70.6%)** account for **70.9%** of total credit limit. Pharmacy leads at 13.26% of total limit, Healthcare brings the cumulative total to 24.07%, and Banking is the 12th industry to cross the 70.87% cumulative mark.
- Pharmacy (214 customers) and Healthcare (177) are also the two industries with the **most customers**, which is a mitigating factor — this concentration sits on a broad, stable customer base rather than a handful of large individual borrowers.
- **Risk to watch:** if the Northeast region or the investment-property segment hits an economic shock, the portfolio is exposed broadly, given how dependent it is on both.

### 3. Customer Analysis

![Customer Dashboard](https://raw.githubusercontent.com/tranquochung09062001-dev/Loan_Banking/main/Customer_Dashboard.png)

- The **highest** individual limit belongs to Jesse Kyle Weaver (**12.5 million**); the **lowest** to Samantha Abigail Nelson (**440 thousand**) — checked and confirmed no ties at either end.
- The customer base skews young, concentrated in the 21–40 age bands, consistent with a product mix dominated by investment-property borrowers still in their asset-accumulation years.

### 4. Risk / Lending Behavior

![Risk Dashboard](https://raw.githubusercontent.com/tranquochung09062001-dev/Loan_Banking/main/Risk_Dashboard.png)

100% of loans in the dataset fall into exactly **5 fixed limit-to-income ratio tiers**, distributed fairly evenly:

| Credit-limit ratio group | # Loans | Share | Avg. limit |
|---|---|---|---|
| 1/5 of income (safest) | 350 | 20.9% | 1.84M |
| 1/4 of income | 344 | 20.5% | 1.79M |
| 1/2 of income | 334 | 19.9% | 1.75M |
| 1/3 of income | 329 | 19.6% | 1.70M |
| Equal to income (riskiest) | 320 | 19.1% | 1.70M |

- **Positive signal:** the highest-risk tier ("Equal to income") is being extended the **lowest** average limit (1.70M) versus the safest tier (1.84M) — the bank's overall risk appetite is directionally sound.
- **Point to monitor:** the Northeast region carries the highest share of "Equal to income" loans (**20.8%** vs. 15–16% elsewhere) — notable because Northeast is also where most of the portfolio already sits.
- The "Human Resources" industry shows the highest risk-tier share, but it sits outside the Top 10 by loan volume — this needs cross-checking against absolute loan counts before it's treated as a systemic risk signal rather than small-sample noise.

## 🛠️ Skills & Tools Applied

| Category | Tools / Techniques |
|---|---|
| Data Ingestion | Python (pandas, SQLAlchemy), chunked `to_sql` loads, config-driven file→table mapping |
| Data Warehousing | PostgreSQL, medallion architecture (raw / staging / dwh), surrogate keys, FK constraints |
| Data Cleaning | Pure SQL: safe type-casting functions, typo correction, deduplication, thousand-separator & percent parsing |
| Data Modeling | Star schema design (1 Fact + 5 Dimensions), `dim_date` via `generate_series` |
| Missing-Value Handling | Window-function imputation (`COALESCE` + `AVG() OVER()`), automated zero-NULL QA check |
| BI & Visualization | Power BI (Import mode, relationship modeling, DAX-ready star schema) |

## 🔎 Final Conclusion & Recommendations

1. **Insight:** YoY growth is negative (-5.07%) and new-customer counts show no clear upward trend across 8 years.
   **Recommendation:** treat customer acquisition as the primary growth lever, since the existing base is essentially "one loan per customer" with little repeat-borrowing behavior to rely on instead.

2. **Insight:** Northeast holds ~50% of the customer base — nearly 3× any other region.
   **Recommendation:** stress-test the portfolio against a Northeast-specific downturn scenario; geographic diversification targets may be worth setting explicitly.

3. **Insight:** 12 of 17 industries account for 70.9% of total limit, led by Pharmacy and Healthcare — but these are also the industries with the most customers.
   **Recommendation:** the concentration is currently supported by customer-count breadth, not a few large exposures — monitor that this stays true rather than restricting new lending in these industries.

4. **Insight:** "Investment Property" is the top loan purpose in every region.
   **Recommendation:** track real-estate-market indicators as a leading signal for portfolio health, given how broadly the book is exposed to this single purpose category.

5. **Insight:** the riskiest ratio tier ("Equal to income") already receives the lowest average limit, but Northeast over-indexes on this tier specifically.
   **Recommendation:** consider a regional overlay on top of the existing ratio-tier policy rather than relying on the ratio tier alone.

6. **Insight:** the Human Resources industry's high risk-tier share is based on a small, outside-Top-10 loan count.
   **Recommendation:** flag for continued monitoring rather than acting on it as a confirmed systemic risk — the sample size does not yet support that conclusion.

---

*All dashboard screenshots above are real captures from the Power BI report. Cover, data model and Design Thinking illustrations are original graphics created for this repository.*
