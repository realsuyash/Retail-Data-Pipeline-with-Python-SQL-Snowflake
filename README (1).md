# Retail Sales ETL Pipeline with SQL Performance Layer

A end-to-end data engineering project built on the UCI Online Retail dataset. Demonstrates a dual-database ETL architecture using Python, PostgreSQL, and MS SQL Server — with a dedicated query performance optimization layer showing before/after execution plan comparisons.

Built to mirror real-world retail data platform work: staging → transformation → star schema → analytical queries → performance tuning.

---

## Architecture

```
UCI Online Retail Dataset (.xlsx)
            │
            ▼
┌─────────────────────────┐
│  Python: Extract+Clean  │  ← handles nulls, type errors, duplicates, negatives
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  PostgreSQL             │  ← staging layer (stg_retail_sales)
│  Staging Layer          │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Python: Transform      │  ← builds star schema dimensions + fact table
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  MS SQL Server          │  ← analytics layer (star schema)
│  Analytics Layer        │    dim_customer, dim_product, dim_date, fact_sales
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Complex SQL Queries    │  ← CTEs, window functions, joins, subqueries
│  + Performance Tuning   │  ← slow vs optimized, indexes, execution plans
└─────────────────────────┘
```

---

## Tech Stack

| Tool | Version | Role |
|------|---------|------|
| Python | 3.11+ | ETL scripting, data cleaning, automation |
| PostgreSQL | 16 | Staging layer |
| MS SQL Server | 2022 Developer | Analytics layer |
| pandas | 2.1.4 | Data manipulation |
| psycopg2 | 2.9.9 | Python → PostgreSQL connector |
| pyodbc | 5.0.1 | Python → MSSQL connector |
| SQLAlchemy | 2.0.25 | ORM + bulk inserts |

---

## Dataset

**UCI Online Retail Dataset**
Source: https://archive.ics.uci.edu/dataset/352/online+retail

~541,000 rows of UK-based online retail transactions (2010–2011).

| Column | Description |
|--------|-------------|
| InvoiceNo | Transaction ID |
| StockCode | Product/SKU code |
| Description | Product name |
| Quantity | Units purchased |
| InvoiceDate | Transaction timestamp |
| UnitPrice | Price per unit (GBP) |
| CustomerID | Unique customer identifier |
| Country | Customer country |

Download `Online Retail.xlsx` and place it in `data/raw/`.

---

## Project Structure

```
retail-etl/
├── data/
│   ├── raw/                              # Place downloaded dataset here
│   └── cleaned/                          # Auto-generated cleaned CSV
│
├── scripts/
│   ├── extract/
│   │   └── extract_clean.py              # Step 1: Extract + clean raw data
│   ├── load/
│   │   └── load_postgres.py              # Step 2: Load into PostgreSQL staging
│   ├── transform/
│   │   └── transform_push_mssql.py       # Step 3: Build star schema + push to MSSQL
│   └── optimize/
│       └── run_benchmarks.py             # Step 5: Python-side query benchmarking
│
├── sql/
│   ├── staging/
│   │   └── create_staging.sql            # PostgreSQL staging schema (reference)
│   ├── analytics/
│   │   └── complex_queries.sql           # 5 business queries with CTEs + window funcs
│   └── performance/
│       ├── slow_queries.sql              # Deliberately unoptimized query versions
│       ├── optimized_queries.sql         # Fixed versions with indexes
│       └── query_tuning_notes.md         # Documented before/after decisions
│
├── config/
│   └── db_config.py                      # DB connection config (reads from .env)
│
├── .env.example                          # Template — copy to .env and fill in
├── .gitignore
├── requirements.txt
└── README.md
```

---

## Prerequisites (Windows 11)

Install these in order before running anything:

1. **Python 3.11** — https://www.python.org/downloads/
   - Check "Add Python to PATH" during install

2. **PostgreSQL 16** — https://www.enterprisedb.com/downloads/postgres-postgresql-downloads
   - Remember the password you set for `postgres` user
   - Create a database called `retail_staging` in pgAdmin after install

3. **MS SQL Server 2022 Developer** — https://www.microsoft.com/en-us/sql-server/sql-server-downloads
   - Choose Basic install type
   - Also install **SSMS** (SQL Server Management Studio) from the same page
   - Create a database called `retail_analytics` in SSMS after install
   - In SSMS: right-click server → Properties → Security → enable **SQL Server and Windows Authentication mode** → restart SQL Server service

4. **ODBC Driver 17 for SQL Server** — https://learn.microsoft.com/en-us/sql/connect/odbc/download-odbc-driver-for-sql-server
   - Download x64 version, run installer, no extra config needed

---

## Setup

```bash
# Clone or extract the project
cd retail-etl

# Create virtual environment
python -m venv venv
venv\Scripts\activate       # Windows
# source venv/bin/activate  # Mac/Linux

# Install dependencies
pip install -r requirements.txt

# Configure credentials
copy .env.example .env      # Windows
# cp .env.example .env      # Mac/Linux
```

Edit `.env` with your actual database credentials:

```env
PG_HOST=localhost
PG_PORT=5432
PG_DB=retail_staging
PG_USER=postgres
PG_PASSWORD=your_postgres_password

MSSQL_HOST=localhost
MSSQL_PORT=1433
MSSQL_DB=retail_analytics
MSSQL_USER=sa
MSSQL_PASSWORD=your_mssql_password
```

---

## Execution — Step by Step

### Step 1: Extract + Clean

```bash
python scripts/extract/extract_clean.py
```

Reads `Online Retail.xlsx`, applies cleaning rules, saves `data/cleaned/retail_clean.csv`.

Cleaning operations applied:
- Drop rows with null CustomerID, Description, Quantity, UnitPrice
- Remove cancellations (Quantity ≤ 0) and invalid prices (UnitPrice ≤ 0)
- Deduplicate on InvoiceNo + StockCode + CustomerID
- Type coercion: dates, numerics, strings
- Derive `TotalAmount = Quantity × UnitPrice`
- Normalize text fields to uppercase

Expected: ~541K raw rows → ~406K clean rows retained.

---

### Step 2: Load into PostgreSQL (Staging)

```bash
python scripts/load/load_postgres.py
```

Creates `stg_retail_sales` table in PostgreSQL, inserts clean data in chunks of 5,000 rows, then runs a verification query.

---

### Step 3: Transform + Push to MS SQL Server (Analytics)

```bash
python scripts/transform/transform_push_mssql.py
```

Reads from PostgreSQL staging, builds a star schema, loads into MSSQL:

| Table | Type | Rows (approx) |
|-------|------|---------------|
| dim_customer | Dimension | ~4,400 |
| dim_product | Dimension | ~3,700 |
| dim_date | Dimension | ~400 |
| fact_sales | Fact | ~406,000 |

---

### Step 4: Run Complex Analytical Queries (SSMS)

Open `sql/analytics/complex_queries.sql` in SSMS. Run with F5.

Queries included:

| Query | Techniques Used |
|-------|----------------|
| Monthly revenue with MoM growth | CTE + `LAG()` window function |
| Top 10 SKUs by revenue | `RANK()` + cumulative `SUM() OVER()` |
| Revenue by country | `DENSE_RANK()` + multi-table join |
| Customer RFM segmentation | Nested CTEs + `NTILE()` |
| Quarterly revenue pivot | Conditional aggregation (`CASE WHEN`) |

---

### Step 5: Performance Tuning (SSMS)

Open `sql/performance/slow_queries.sql` in SSMS.

`SET STATISTICS TIME ON` and `SET STATISTICS IO ON` are already at the top. Run with F5, then check the **Messages tab** — note the logical reads and elapsed time for each query.

Then open `sql/performance/optimized_queries.sql`. Run the index creation section first, then the optimized queries. Compare logical reads to what you saw before.

See `sql/performance/query_tuning_notes.md` for documented decisions with before/after metrics.

---

### Step 6: Python Benchmark

```bash
python scripts/optimize/run_benchmarks.py
```

Runs slow and fast query pairs from Python and prints a comparison table.

---

## Performance Optimization Summary

| Problem | Fix | Impact |
|---------|-----|--------|
| Correlated subquery in SELECT (N+1) | Aggregated CTE + single JOIN | ~40× fewer logical reads |
| `UPPER()` / `CONVERT()` on indexed column | Remove function, normalize data at load time | Index seek instead of full scan |
| Leading wildcard `LIKE '%term%'` | Trailing wildcard or Full-Text Search index | Eliminates full table scan |
| Multiple correlated subqueries per row | Single CTE with conditional `SUM(CASE WHEN ...)` | One pass instead of N×2 passes |

Indexes created:

```sql
-- Covering index for customer-based aggregations
CREATE NONCLUSTERED INDEX idx_fact_customer_covering
ON fact_sales (customer_key) INCLUDE (total_amount, invoice_no, invoice_date);

-- Covering index for date-based aggregations  
CREATE NONCLUSTERED INDEX idx_fact_date_amount
ON fact_sales (date_key) INCLUDE (total_amount, invoice_no);

-- Country filter on dim_customer
CREATE NONCLUSTERED INDEX idx_customer_country
ON dim_customer (country);
```

---

## Key Concepts Demonstrated

- Dual-database ETL architecture (PostgreSQL staging → MSSQL analytics)
- Star schema design (fact + dimension tables)
- Python data pipeline with chunked inserts and verification
- Complex SQL: CTEs, window functions (`LAG`, `RANK`, `NTILE`, `SUM OVER`), multi-table joins
- SQL performance tuning: sargable predicates, covering indexes, correlated subquery elimination
- Before/after execution plan analysis using `SET STATISTICS IO ON`
- RFM customer segmentation (real retail analytics pattern)

---

## Troubleshooting

**`pyodbc` driver error**
Verify ODBC Driver 17 is installed: open Start → search "ODBC Data Sources (64-bit)" → Drivers tab → check for "ODBC Driver 17 for SQL Server".

**MSSQL login failed**
In SSMS: right-click server → Properties → Security → set to "SQL Server and Windows Authentication mode". Then restart SQL Server service via Services app.

**PostgreSQL connection refused**
Check pgAdmin is running and the `retail_staging` database exists. Verify `.env` password matches what you set during PostgreSQL install.

**`FileNotFoundError` on xlsx**
Confirm the file is named exactly `Online Retail.xlsx` (with space) and placed in `data/raw/`.
