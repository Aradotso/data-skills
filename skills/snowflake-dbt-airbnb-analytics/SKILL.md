---
name: snowflake-dbt-airbnb-analytics
description: Load Inside Airbnb data into Snowflake, transform with dbt staging/intermediate/marts, run quality tests, and deploy Streamlit dashboards
triggers:
  - set up Airbnb analytics with Snowflake and dbt
  - load Inside Airbnb data into Snowflake warehouse
  - build dbt staging and mart models for Airbnb listings
  - create incremental fact tables in dbt with Snowflake merge
  - run dbt tests for data quality validation
  - deploy Streamlit dashboard with Snowflake connection
  - configure dbt profiles for Snowflake analytics
  - transform raw Airbnb calendar and review data with dbt
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with a complete analytics engineering project that ingests Inside Airbnb open data into Snowflake, transforms it with dbt (data build tool) using staging, intermediate, and mart layers, validates data quality with tests, and powers a Streamlit reporting dashboard.

## What This Project Does

**Snowflake_DBT_Project** is a public portfolio analytics engineering project demonstrating:

- Loading CSV/GZIP files into Snowflake internal stages
- Raw-zone setup with text-preserving tables
- dbt staging views (clean, cast, standardize)
- dbt intermediate layer (joins, enrichment, revenue proxy logic)
- dbt marts (dimensions, facts, aggregates)
- Incremental fact modeling with Snowflake merge strategy
- Generic and singular dbt tests for data quality
- Streamlit dashboard connected to analytics marts
- Public-safe credential management (no committed secrets)

**Dataset**: Inside Airbnb open dataset (listings, calendar, reviews, neighbourhoods). The project uses calendar unavailability and price as a proxy for revenue since booking transactions are not publicly available.

## Installation

### Prerequisites

- Python 3.8+
- Snowflake account with database creation privileges
- dbt-snowflake adapter
- Streamlit (for dashboard)

### Setup Steps

```bash
# Clone the repository
git clone https://github.com/analyticsdurgesh/Snowflake_DBT_Project.git
cd Snowflake_DBT_Project

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

**requirements.txt** should contain:
```txt
dbt-snowflake>=1.5.0
snowflake-connector-python>=3.0.0
streamlit>=1.20.0
pandas>=1.5.0
```

### Configure Credentials

This project uses **local credential files** (git-ignored) instead of environment variables or committed secrets.

```bash
# Copy example files
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

**profiles.yml** (for dbt):
```yaml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT_LOCATOR    # e.g., xy12345.us-east-1
      user: YOUR_SNOWFLAKE_USER
      password: YOUR_SNOWFLAKE_PASSWORD
      role: YOUR_ROLE                  # e.g., ACCOUNTADMIN
      database: AIRBNB_DEV
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

**config/local_credentials.json** (for Streamlit):
```json
{
  "account": "YOUR_ACCOUNT_LOCATOR",
  "user": "YOUR_SNOWFLAKE_USER",
  "password": "YOUR_SNOWFLAKE_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DEV",
  "schema": "ANALYTICS",
  "role": "YOUR_ROLE"
}
```

**Never commit** `profiles.yml`, `config/local_credentials.json`, or `.user.yml`.

## Data Loading

### Download Inside Airbnb Data

Download the New York City dataset from [Inside Airbnb](https://insideairbnb.com/get-the-data/):

```text
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

### Load Raw Data into Snowflake

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What this does**:
1. Reads `config/local_credentials.json`
2. Executes `setup/snowflake_setup.sql` to create database, schemas, stage
3. Uploads local files to Snowflake internal stage `INSIDE_AIRBNB_STAGE`
4. Creates raw tables from CSV headers (VARCHAR columns)
5. Copies data into `RAW` schema

**scripts/load_inside_airbnb_to_snowflake.py** (example structure):
```python
import json
import snowflake.connector
from pathlib import Path

def load_credentials():
    with open('config/local_credentials.json', 'r') as f:
        return json.load(f)

def main():
    creds = load_credentials()
    
    conn = snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        role=creds.get('role', 'ACCOUNTADMIN')
    )
    
    cursor = conn.cursor()
    
    # Run setup SQL
    setup_sql = Path('setup/snowflake_setup.sql').read_text()
    for stmt in setup_sql.split(';'):
        if stmt.strip():
            cursor.execute(stmt)
    
    # Upload files to stage
    cursor.execute(f"USE DATABASE {creds['database']}")
    cursor.execute("USE SCHEMA RAW")
    
    files = [
        'data/raw/listings.csv.gz',
        'data/raw/calendar.csv.gz',
        'data/raw/reviews.csv.gz',
        'data/raw/neighbourhoods.csv'
    ]
    
    for file_path in files:
        cursor.execute(f"PUT file://{file_path} @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")
    
    # Create and load raw tables
    cursor.execute("""
        CREATE OR REPLACE TABLE RAW.LISTINGS AS
        SELECT $1 AS raw_data
        FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
        (FILE_FORMAT => 'CSV_FORMAT')
        LIMIT 0
    """)
    
    cursor.execute("""
        COPY INTO RAW.LISTINGS
        FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
        FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"')
    """)
    
    # Repeat for calendar, reviews, neighbourhoods...
    
    cursor.close()
    conn.close()
    print("✅ Data loaded into Snowflake RAW schema")

if __name__ == '__main__':
    main()
```

## dbt Project Structure

### Model Layers

| Layer | Path | Purpose | Materialization |
|-------|------|---------|----------------|
| **Sources** | `models/staging/sources.yml` | Raw table references | N/A |
| **Staging** | `models/staging/stg_airbnb__*.sql` | Clean, cast, rename | View |
| **Intermediate** | `models/intermediate/int_airbnb__*.sql` | Joins, enrichment | View |
| **Marts** | `models/marts/dim_*.sql`, `fct_*.sql` | Analytics-ready dimensions/facts | Table |
| **Aggregates** | `models/marts/agg_*.sql` | Monthly performance metrics | Table |

### Key dbt Commands

```bash
# Test connection
dbt debug --profiles-dir .

# Compile models (check SQL)
dbt compile --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select stg_airbnb__listings+ --profiles-dir .

# Full refresh incremental model
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Test specific model
dbt test --select dim_listings --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

### Source Configuration

**models/staging/sources.yml**:
```yaml
version: 2

sources:
  - name: airbnb_raw
    database: AIRBNB_DEV
    schema: RAW
    tables:
      - name: LISTINGS
        description: Raw Airbnb listings from Inside Airbnb
        columns:
          - name: ID
            description: Unique listing identifier
            tests:
              - not_null
              - unique
      
      - name: CALENDAR
        description: Daily availability and pricing
        columns:
          - name: LISTING_ID
            tests:
              - not_null
          - name: DATE
            tests:
              - not_null
      
      - name: REVIEWS
        description: Guest reviews
      
      - name: NEIGHBOURHOODS
        description: NYC neighbourhood boundaries
```

### Staging Model Example

**models/staging/stg_airbnb__listings.sql**:
```sql
WITH source AS (
    SELECT * FROM {{ source('airbnb_raw', 'LISTINGS') }}
),

cleaned AS (
    SELECT
        TRY_CAST(id AS INTEGER) AS listing_id,
        TRIM(name) AS listing_name,
        TRIM(host_id) AS host_id,
        TRIM(host_name) AS host_name,
        TRIM(neighbourhood) AS neighbourhood,
        TRIM(neighbourhood_group) AS neighbourhood_group,
        TRY_CAST(latitude AS FLOAT) AS latitude,
        TRY_CAST(longitude AS FLOAT) AS longitude,
        TRIM(room_type) AS room_type,
        TRY_CAST(price AS DECIMAL(10,2)) AS price,
        TRY_CAST(minimum_nights AS INTEGER) AS minimum_nights,
        TRY_CAST(number_of_reviews AS INTEGER) AS number_of_reviews,
        TRY_TO_DATE(last_review) AS last_review_date,
        TRY_CAST(reviews_per_month AS DECIMAL(5,2)) AS reviews_per_month,
        TRY_CAST(calculated_host_listings_count AS INTEGER) AS host_listings_count,
        TRY_CAST(availability_365 AS INTEGER) AS availability_365,
        CURRENT_TIMESTAMP() AS dbt_loaded_at
    FROM source
)

SELECT * FROM cleaned
WHERE listing_id IS NOT NULL
```

### Intermediate Model Example

**models/intermediate/int_airbnb__listing_enriched.sql**:
```sql
WITH listings AS (
    SELECT * FROM {{ ref('stg_airbnb__listings') }}
),

neighbourhoods AS (
    SELECT * FROM {{ ref('stg_airbnb__neighbourhoods') }}
),

enriched AS (
    SELECT
        l.listing_id,
        l.listing_name,
        l.host_id,
        l.host_name,
        l.neighbourhood,
        n.neighbourhood_group,
        l.latitude,
        l.longitude,
        l.room_type,
        l.price,
        l.minimum_nights,
        l.number_of_reviews,
        l.last_review_date,
        l.reviews_per_month,
        l.host_listings_count,
        l.availability_365,
        
        -- Enrichment flags
        CASE 
            WHEN l.number_of_reviews > 50 THEN 'High'
            WHEN l.number_of_reviews > 10 THEN 'Medium'
            ELSE 'Low'
        END AS review_volume_category,
        
        CASE 
            WHEN l.availability_365 > 300 THEN 'High'
            WHEN l.availability_365 > 100 THEN 'Medium'
            ELSE 'Low'
        END AS availability_category,
        
        l.dbt_loaded_at
    FROM listings l
    LEFT JOIN neighbourhoods n
        ON l.neighbourhood = n.neighbourhood
)

SELECT * FROM enriched
```

### Incremental Fact Model

**models/marts/fct_listing_calendar.sql**:
```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        merge_update_columns=['available', 'price', 'adjusted_price']
    )
}}

WITH calendar AS (
    SELECT * FROM {{ ref('stg_airbnb__calendar') }}
),

listings AS (
    SELECT * FROM {{ ref('int_airbnb__listing_enriched') }}
),

calendar_facts AS (
    SELECT
        c.listing_id,
        c.calendar_date,
        c.available,
        c.price,
        c.adjusted_price,
        c.minimum_nights,
        c.maximum_nights,
        l.neighbourhood,
        l.neighbourhood_group,
        l.room_type,
        
        -- Revenue proxy (unavailable nights * price)
        CASE 
            WHEN c.available = 'f' THEN COALESCE(c.adjusted_price, c.price, 0)
            ELSE 0
        END AS estimated_revenue,
        
        CURRENT_TIMESTAMP() AS dbt_loaded_at
    FROM calendar c
    INNER JOIN listings l
        ON c.listing_id = l.listing_id
    
    {% if is_incremental() %}
        WHERE c.calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
    {% endif %}
)

SELECT * FROM calendar_facts
```

### Dimension Model

**models/marts/dim_hosts.sql**:
```sql
WITH listings AS (
    SELECT * FROM {{ ref('int_airbnb__listing_enriched') }}
),

host_summary AS (
    SELECT
        host_id,
        MIN(host_name) AS host_name,
        COUNT(DISTINCT listing_id) AS total_listings,
        ROUND(AVG(price), 2) AS avg_listing_price,
        SUM(number_of_reviews) AS total_reviews,
        MAX(last_review_date) AS most_recent_review_date,
        ROUND(AVG(availability_365), 0) AS avg_availability_365,
        CURRENT_TIMESTAMP() AS dbt_loaded_at
    FROM listings
    GROUP BY host_id
)

SELECT * FROM host_summary
```

### Aggregate Model

**models/marts/agg_listing_monthly_performance.sql**:
```sql
WITH facts AS (
    SELECT * FROM {{ ref('fct_listing_calendar') }}
),

monthly_agg AS (
    SELECT
        listing_id,
        DATE_TRUNC('month', calendar_date) AS month,
        neighbourhood,
        neighbourhood_group,
        room_type,
        
        COUNT(*) AS total_days,
        SUM(CASE WHEN available = 'f' THEN 1 ELSE 0 END) AS unavailable_days,
        SUM(CASE WHEN available = 't' THEN 1 ELSE 0 END) AS available_days,
        
        ROUND(AVG(price), 2) AS avg_price,
        ROUND(SUM(estimated_revenue), 2) AS total_estimated_revenue,
        
        ROUND(
            SUM(CASE WHEN available = 'f' THEN 1 ELSE 0 END)::FLOAT / NULLIF(COUNT(*), 0),
            4
        ) AS occupancy_rate,
        
        CURRENT_TIMESTAMP() AS dbt_loaded_at
    FROM facts
    GROUP BY listing_id, month, neighbourhood, neighbourhood_group, room_type
)

SELECT * FROM monthly_agg
```

### dbt Tests

**models/marts/schema.yml**:
```yaml
version: 2

models:
  - name: dim_listings
    description: Listing dimension with enriched attributes
    columns:
      - name: listing_id
        description: Primary key
        tests:
          - unique
          - not_null
      - name: room_type
        tests:
          - accepted_values:
              values: ['Entire home/apt', 'Private room', 'Shared room', 'Hotel room']
      - name: price
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"

  - name: fct_listing_calendar
    description: Daily listing calendar facts (incremental)
    columns:
      - name: listing_id
        tests:
          - not_null
          - relationships:
              to: ref('dim_listings')
              field: listing_id
      - name: calendar_date
        tests:
          - not_null
    tests:
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns:
            - listing_id
            - calendar_date
```

**Singular Test** (`tests/assert_no_negative_prices.sql`):
```sql
SELECT
    listing_id,
    calendar_date,
    price
FROM {{ ref('fct_listing_calendar') }}
WHERE price < 0
```

## Streamlit Dashboard

### Dashboard Code

**dashboard/streamlit_app.py**:
```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json
from pathlib import Path

@st.cache_resource
def get_connection():
    creds_path = Path(__file__).parent.parent / 'config' / 'local_credentials.json'
    with open(creds_path, 'r') as f:
        creds = json.load(f)
    
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema'],
        role=creds.get('role', 'ACCOUNTADMIN')
    )

@st.cache_data(ttl=600)
def query_data(_conn, sql):
    return pd.read_sql(sql, _conn)

def main():
    st.set_page_config(page_title="Airbnb Analytics", layout="wide")
    st.title("🏠 Inside Airbnb Analytics Dashboard")
    
    conn = get_connection()
    
    # Sidebar filters
    st.sidebar.header("Filters")
    
    neighbourhoods_df = query_data(conn, """
        SELECT DISTINCT neighbourhood_group
        FROM dim_listings
        ORDER BY neighbourhood_group
    """)
    
    selected_neighbourhood = st.sidebar.selectbox(
        "Neighbourhood Group",
        options=['All'] + neighbourhoods_df['NEIGHBOURHOOD_GROUP'].tolist()
    )
    
    # Top neighbourhoods by revenue
    st.header("Top Neighbourhoods by Estimated Revenue")
    
    neighbourhood_filter = ""
    if selected_neighbourhood != 'All':
        neighbourhood_filter = f"WHERE neighbourhood_group = '{selected_neighbourhood}'"
    
    revenue_sql = f"""
        SELECT
            neighbourhood_group,
            neighbourhood,
            SUM(total_estimated_revenue) AS total_revenue,
            AVG(occupancy_rate) AS avg_occupancy_rate,
            COUNT(DISTINCT listing_id) AS total_listings
        FROM agg_listing_monthly_performance
        {neighbourhood_filter}
        GROUP BY neighbourhood_group, neighbourhood
        ORDER BY total_revenue DESC
        LIMIT 10
    """
    
    revenue_df = query_data(conn, revenue_sql)
    st.dataframe(revenue_df, use_container_width=True)
    
    # Top hosts
    st.header("Top Hosts by Listings")
    
    hosts_sql = """
        SELECT
            host_id,
            host_name,
            total_listings,
            avg_listing_price,
            total_reviews,
            most_recent_review_date
        FROM dim_hosts
        ORDER BY total_listings DESC
        LIMIT 10
    """
    
    hosts_df = query_data(conn, hosts_sql)
    st.dataframe(hosts_df, use_container_width=True)
    
    # Monthly trend
    st.header("Monthly Revenue Trend")
    
    trend_sql = f"""
        SELECT
            month,
            SUM(total_estimated_revenue) AS total_revenue,
            AVG(occupancy_rate) AS avg_occupancy
        FROM agg_listing_monthly_performance
        {neighbourhood_filter}
        GROUP BY month
        ORDER BY month
    """
    
    trend_df = query_data(conn, trend_sql)
    st.line_chart(trend_df.set_index('MONTH')['TOTAL_REVENUE'])

if __name__ == '__main__':
    main()
```

### Run Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

The dashboard automatically reads credentials from `config/local_credentials.json`.

## Common Patterns

### Full Refresh After New Data Load

```bash
# Load new snapshot
python scripts/load_inside_airbnb_to_snowflake.py

# Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Run tests
dbt test --profiles-dir .
```

### Selective Model Runs

```bash
# Run only staging models
dbt run --select staging.* --profiles-dir .

# Run marts and downstream only
dbt run --select marts.* --profiles-dir .

# Run one model and its parents
dbt run --select +fct_listing_calendar --profiles-dir .

# Run one model and its children
dbt run --select fct_listing_calendar+ --profiles-dir .
```

### Testing Specific Models

```bash
# Test all marts
dbt test --select marts.* --profiles-dir .

# Test sources
dbt test --select source:* --profiles-dir .

# Test singular tests only
dbt test --select test_type:singular --profiles-dir .
```

### Generate and Serve Documentation

```bash
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir . --port 8080
```

Access at `http://localhost:8080` to browse lineage, column descriptions, and tests.

## Troubleshooting

### "Profile not found"

**Problem**: dbt can't find `profiles.yml`.

**Solution**: Always use `--profiles-dir .` to point to the local profiles file:
```bash
dbt run --profiles-dir .
```

### "Object does not exist" errors

**Problem**: Raw tables not loaded.

**Solution**: Run the data loader first:
```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

### Incremental model not updating

**Problem**: New data not appearing in `fct_listing_calendar`.

**Solution**: Use `--full-refresh`:
```bash
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

### Streamlit connection error

**Problem**: Dashboard can't connect to Snowflake.

**Solution**: Verify `config/local_credentials.json` exists and contains correct credentials:
```bash
cat config/local_credentials.json
```

### Test failures on unique constraints

**Problem**: Duplicate listing-date combinations.

**Solution**: Check raw data for duplicates, or adjust incremental model merge strategy:
```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        merge_update_columns=['available', 'price']
    )
}}
```

### Snowflake timeout during large loads

**Problem**: `PUT` command times out for large files.

**Solution**: Increase warehouse size or split files:
```sql
-- In Snowflake
ALTER WAREHOUSE COMPUTE_WH SET WAREHOUSE_SIZE = 'MEDIUM';
```

### dbt compilation errors

**Problem**: Jinja syntax errors in models.

**Solution**: Use `dbt compile` to check SQL generation:
```bash
dbt compile --select model_name --profiles-dir .
# Check compiled SQL in target/compiled/
```

## Key Configuration Files

### dbt_project.yml

```yaml
name: 'snowflake_dbt_project'
version: '1.0.0'
config-version: 2

profile: 'snowflake_dbt_project'

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

target-path: "target"
clean-targets:
  - "target"
  - "dbt_packages"

models:
  snowflake_dbt_project:
    staging:
      +materialized: view
      +schema: staging
    intermediate:
      +materialized: view
      +schema: intermediate
    marts:
      +materialized: table
      +schema: analytics
```

### setup/snowflake_setup.sql

```sql
-- Database setup
CREATE DATABASE IF NOT EXISTS AIRBNB_DEV;
USE DATABASE AIRBNB_DEV;

-- Schemas
CREATE SCHEMA IF NOT EXISTS RAW;
CREATE SCHEMA IF NOT EXISTS STAGING;
CREATE SCHEMA IF NOT EXISTS INTERMEDIATE;
CREATE SCHEMA IF NOT EXISTS ANALYTICS;

-- File format
CREATE OR REPLACE FILE FORMAT CSV_FORMAT
  TYPE = CSV
  FIELD_DELIMITER = ','
  SKIP_HEADER = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  NULL_IF = ('NULL', 'null', '')
  EMPTY_FIELD_AS_NULL = TRUE;

-- Internal stage
CREATE OR REPLACE STAGE INSIDE_AIRBNB_STAGE
  FILE_FORMAT = CSV_FORMAT;
```

## Business Questions Examples

### Neighbourhood Revenue Ranking

```sql
SELECT
    neighbourhood_group,
    SUM(total_estimated_revenue) AS total_revenue,
    AVG(occupancy_rate) AS avg_occupancy
FROM agg_neighbourhood_monthly_performance
GROUP BY neighbourhood_group
ORDER BY total_revenue DESC;
```

### Room Type Analysis

```sql
SELECT
    room_type,
    COUNT(DISTINCT listing_id) AS total_listings,
    AVG(price) AS avg_price,
    AVG(availability_365) AS avg_availability
FROM dim_listings
GROUP BY room_type
ORDER BY total_listings DESC;
```

### Host Performance

```sql
SELECT
    host_name,
    total_listings,
    total_reviews,
    avg_listing_price
FROM dim_hosts
WHERE total_listings >= 5
ORDER BY total_reviews DESC
LIMIT 20;
```

## Summary

This skill covers:

- ✅ Credential setup with local files (no committed secrets)
- ✅ Loading Inside Airbnb data into Snowflake
- ✅ dbt staging, intermediate, and mart model patterns
- ✅ Incremental fact tables with merge strategy
- ✅ Generic and singular dbt tests
- ✅ Streamlit dashboard with Snowflake connection
- ✅ Common dbt CLI workflows and troubleshooting

Use this skill to help developers build analytics engineering projects with Snowflake, dbt, and Streamlit following modern data warehouse patterns.
