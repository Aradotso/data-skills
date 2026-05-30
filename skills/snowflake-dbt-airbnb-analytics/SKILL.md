---
name: snowflake-dbt-airbnb-analytics
description: Build end-to-end analytics pipelines with Snowflake, dbt, and Streamlit using Inside Airbnb open data
triggers:
  - how do I set up this Airbnb analytics project with Snowflake and dbt
  - load Inside Airbnb data into Snowflake
  - run dbt models for Airbnb staging and marts
  - create incremental fact tables in dbt with Snowflake
  - build a Streamlit dashboard with Snowflake data
  - configure profiles.yml for local dbt Snowflake development
  - test dbt data quality for Airbnb listings and calendar
  - troubleshoot dbt Snowflake connection issues
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build a complete analytics engineering portfolio project using Snowflake as the data warehouse, dbt for transformations, and Streamlit for dashboards. The project demonstrates modern ELT patterns with Inside Airbnb open data, including staging layers, incremental fact tables, data quality tests, and business intelligence reporting.

## What This Project Does

**Snowflake_DBT_Project** is a public analytics engineering reference implementation that:

- Loads Inside Airbnb CSV/GZIP files into Snowflake raw tables
- Transforms raw data through dbt staging → intermediate → marts layers
- Builds incremental fact tables with Snowflake merge strategy
- Validates data quality with generic and singular dbt tests
- Powers a Streamlit dashboard with analytics-ready marts
- Demonstrates public-safe credential management (no committed secrets)

**Key datasets**: listings, calendar (availability proxy for revenue), reviews, neighbourhoods.

**Stack**: Snowflake warehouse, dbt-core 1.x, Python 3.x, Streamlit.

## Installation

### Prerequisites

- Snowflake account (trial or paid)
- Python 3.8+
- dbt-core with dbt-snowflake adapter
- Inside Airbnb data files (download from insideairbnb.com)

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

Expected `requirements.txt` contents:
```
dbt-core>=1.5.0
dbt-snowflake>=1.5.0
streamlit>=1.20.0
snowflake-connector-python>=3.0.0
pandas>=1.5.0
```

### Download Inside Airbnb Data

Place raw files in `data/raw/`:
```
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

Download from: https://insideairbnb.com/get-the-data/ (recommend New York City dataset)

## Configuration

### 1. Snowflake Credentials (Local-Only)

Create `profiles.yml` from example:
```bash
cp profiles.yml.example profiles.yml
```

Edit `profiles.yml`:
```yaml
snowflake_dbt_project:
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT_IDENTIFIER  # e.g., xy12345.us-east-1
      user: YOUR_USERNAME
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"  # or hardcode locally (not committed)
      role: YOUR_ROLE
      database: AIRBNB_DBT
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
  target: dev
```

**Recommended**: Set `SNOWFLAKE_PASSWORD` environment variable:
```bash
export SNOWFLAKE_PASSWORD='your-password'
```

### 2. Streamlit Credentials

Create `config/local_credentials.json` from example:
```bash
cp config/local_credentials.example.json config/local_credentials.json
```

Edit `config/local_credentials.json`:
```json
{
  "account": "YOUR_ACCOUNT_IDENTIFIER",
  "user": "YOUR_USERNAME",
  "password": "YOUR_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DBT",
  "schema": "ANALYTICS",
  "role": "YOUR_ROLE"
}
```

**Security**: Both `profiles.yml` and `config/local_credentials.json` are gitignored.

## Loading Raw Data

### Python Loader Script

The `scripts/load_inside_airbnb_to_snowflake.py` script automates:
1. Snowflake database/schema/stage creation
2. File upload to internal stage
3. Raw table creation from CSV headers
4. COPY INTO commands for data ingestion

**Run the loader**:
```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

### Example Loader Code Pattern

```python
import snowflake.connector
import json
import os

# Load credentials
with open('config/local_credentials.json', 'r') as f:
    creds = json.load(f)

# Connect to Snowflake
conn = snowflake.connector.connect(
    account=creds['account'],
    user=creds['user'],
    password=creds['password'],
    warehouse=creds['warehouse'],
    role=creds['role']
)

cursor = conn.cursor()

# Create database and schema
cursor.execute("CREATE DATABASE IF NOT EXISTS AIRBNB_DBT")
cursor.execute("CREATE SCHEMA IF NOT EXISTS AIRBNB_DBT.RAW")
cursor.execute("USE SCHEMA AIRBNB_DBT.RAW")

# Create internal stage
cursor.execute("""
    CREATE OR REPLACE STAGE INSIDE_AIRBNB_STAGE
    FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
""")

# Upload local file
cursor.execute("""
    PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE
    AUTO_COMPRESS = FALSE
    OVERWRITE = TRUE
""")

# Create raw table (example with text columns for initial load)
cursor.execute("""
    CREATE OR REPLACE TABLE RAW.LISTINGS (
        id TEXT,
        name TEXT,
        host_id TEXT,
        host_name TEXT,
        neighbourhood TEXT,
        room_type TEXT,
        price TEXT,
        minimum_nights TEXT,
        number_of_reviews TEXT,
        last_review TEXT,
        reviews_per_month TEXT,
        calculated_host_listings_count TEXT,
        availability_365 TEXT
    )
""")

# Copy data from stage
cursor.execute("""
    COPY INTO RAW.LISTINGS
    FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
    FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
    ON_ERROR = 'CONTINUE'
""")

cursor.close()
conn.close()
```

## dbt Project Structure

### Model Layers

```
models/
├── staging/
│   ├── _sources.yml
│   ├── stg_airbnb__listings.sql
│   ├── stg_airbnb__calendar.sql
│   ├── stg_airbnb__reviews.sql
│   └── stg_airbnb__neighbourhoods.sql
├── intermediate/
│   ├── int_airbnb__listing_enriched.sql
│   ├── int_airbnb__calendar_enriched.sql
│   └── int_airbnb__reviews_enriched.sql
└── marts/
    ├── dim_listings.sql
    ├── dim_hosts.sql
    ├── fct_listing_calendar.sql (incremental)
    ├── fct_reviews.sql
    ├── agg_listing_monthly_performance.sql
    └── agg_neighbourhood_monthly_performance.sql
```

### Source Configuration

`models/staging/_sources.yml`:
```yaml
version: 2

sources:
  - name: raw_airbnb
    database: AIRBNB_DBT
    schema: RAW
    tables:
      - name: listings
        identifier: LISTINGS
      - name: calendar
        identifier: CALENDAR
      - name: reviews
        identifier: REVIEWS
      - name: neighbourhoods
        identifier: NEIGHBOURHOODS
```

### Staging Model Example

`models/staging/stg_airbnb__listings.sql`:
```sql
with source as (
    select * from {{ source('raw_airbnb', 'listings') }}
),

cleaned as (
    select
        id::bigint as listing_id,
        trim(name) as listing_name,
        host_id::bigint as host_id,
        trim(host_name) as host_name,
        trim(neighbourhood) as neighbourhood,
        trim(room_type) as room_type,
        replace(price, '$', '')::decimal(10,2) as price,
        minimum_nights::int as minimum_nights,
        number_of_reviews::int as number_of_reviews,
        try_to_date(last_review, 'YYYY-MM-DD') as last_review_date,
        reviews_per_month::decimal(5,2) as reviews_per_month,
        calculated_host_listings_count::int as host_listings_count,
        availability_365::int as availability_365
    from source
)

select * from cleaned
```

### Incremental Fact Table

`models/marts/fct_listing_calendar.sql`:
```sql
{{
    config(
        materialized='incremental',
        unique_key='calendar_key',
        merge_update_columns=['available', 'price', 'minimum_nights', 'maximum_nights']
    )
}}

with calendar_enriched as (
    select * from {{ ref('int_airbnb__calendar_enriched') }}
)

select
    {{ dbt_utils.generate_surrogate_key(['listing_id', 'calendar_date']) }} as calendar_key,
    listing_id,
    calendar_date,
    available,
    price,
    minimum_nights,
    maximum_nights,
    adjusted_price,
    neighbourhood,
    room_type
from calendar_enriched

{% if is_incremental() %}
    where calendar_date > (select max(calendar_date) from {{ this }})
{% endif %}
```

### Aggregate Model

`models/marts/agg_listing_monthly_performance.sql`:
```sql
with listing_calendar as (
    select * from {{ ref('fct_listing_calendar') }}
)

select
    listing_id,
    date_trunc('month', calendar_date) as month,
    count(*) as total_days,
    sum(case when available = 'f' then 1 else 0 end) as unavailable_days,
    sum(case when available = 'f' then price else 0 end) as estimated_revenue,
    avg(price) as avg_price,
    min(price) as min_price,
    max(price) as max_price
from listing_calendar
group by listing_id, date_trunc('month', calendar_date)
```

## dbt Commands

### Core Workflow

```bash
# Validate configuration and connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select fct_listing_calendar+ --profiles-dir .

# Full refresh for incremental models
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Test specific model
dbt test --select stg_airbnb__listings --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .

# Serve documentation site
dbt docs serve --profiles-dir .
```

### Selective Runs

```bash
# Run only staging models
dbt run --select staging --profiles-dir .

# Run marts and their dependencies
dbt run --select marts+ --profiles-dir .

# Run modified models and downstream
dbt run --select state:modified+ --profiles-dir .
```

## Data Quality Tests

### Generic Tests

`models/staging/stg_airbnb__listings.yml`:
```yaml
version: 2

models:
  - name: stg_airbnb__listings
    description: Cleaned and typed listings
    columns:
      - name: listing_id
        description: Unique listing identifier
        tests:
          - unique
          - not_null
      - name: room_type
        description: Type of accommodation
        tests:
          - accepted_values:
              values: ['Entire home/apt', 'Private room', 'Shared room', 'Hotel room']
      - name: price
        description: Nightly price in USD
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
```

### Singular Test

`tests/test_calendar_has_valid_listing_id.sql`:
```sql
-- Check that all calendar entries reference valid listings
select
    c.listing_id,
    c.calendar_date
from {{ ref('stg_airbnb__calendar') }} c
left join {{ ref('stg_airbnb__listings') }} l
    on c.listing_id = l.listing_id
where l.listing_id is null
```

## Streamlit Dashboard

### Dashboard Code

`dashboard/streamlit_app.py`:
```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load credentials
with open('config/local_credentials.json', 'r') as f:
    creds = json.load(f)

# Connect to Snowflake
@st.cache_resource
def get_connection():
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema'],
        role=creds['role']
    )

conn = get_connection()

st.title("Inside Airbnb Analytics Dashboard")

# Query neighbourhood performance
@st.cache_data
def load_neighbourhood_performance():
    query = """
    SELECT
        neighbourhood,
        SUM(estimated_revenue) as total_revenue,
        AVG(avg_price) as avg_price,
        SUM(unavailable_days) as total_unavailable_days
    FROM agg_neighbourhood_monthly_performance
    GROUP BY neighbourhood
    ORDER BY total_revenue DESC
    LIMIT 20
    """
    return pd.read_sql(query, conn)

df_neighbourhoods = load_neighbourhood_performance()

st.subheader("Top Neighbourhoods by Estimated Revenue")
st.bar_chart(df_neighbourhoods.set_index('neighbourhood')['total_revenue'])

st.subheader("Neighbourhood Metrics")
st.dataframe(df_neighbourhoods)

# Query listing performance
@st.cache_data
def load_listing_performance():
    query = """
    SELECT
        l.listing_name,
        l.neighbourhood,
        l.room_type,
        SUM(m.estimated_revenue) as total_revenue,
        AVG(m.avg_price) as avg_price
    FROM agg_listing_monthly_performance m
    JOIN dim_listings l ON m.listing_id = l.listing_id
    GROUP BY l.listing_name, l.neighbourhood, l.room_type
    ORDER BY total_revenue DESC
    LIMIT 50
    """
    return pd.read_sql(query, conn)

df_listings = load_listing_performance()

st.subheader("Top Listings by Revenue")
st.dataframe(df_listings)

# Room type breakdown
st.subheader("Revenue by Room Type")
room_type_revenue = df_listings.groupby('room_type')['total_revenue'].sum()
st.bar_chart(room_type_revenue)
```

### Run Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

Access at `http://localhost:8501`

## Common Patterns

### Pattern 1: Add New Source Table

1. Place CSV in `data/raw/new_table.csv`
2. Update `scripts/load_inside_airbnb_to_snowflake.py` to upload and copy
3. Add source to `models/staging/_sources.yml`:
```yaml
      - name: new_table
        identifier: NEW_TABLE
```
4. Create staging model `models/staging/stg_airbnb__new_table.sql`
5. Run: `dbt run --select stg_airbnb__new_table`

### Pattern 2: Create Custom Aggregate

```sql
-- models/marts/agg_host_performance.sql
with dim_hosts as (
    select * from {{ ref('dim_hosts') }}
),

listing_performance as (
    select * from {{ ref('agg_listing_monthly_performance') }}
),

dim_listings as (
    select * from {{ ref('dim_listings') }}
)

select
    h.host_id,
    h.host_name,
    h.total_listings,
    sum(lp.estimated_revenue) as total_revenue,
    avg(lp.avg_price) as avg_price_across_listings
from dim_hosts h
join dim_listings l on h.host_id = l.host_id
join listing_performance lp on l.listing_id = lp.listing_id
group by h.host_id, h.host_name, h.total_listings
```

### Pattern 3: Add dbt Macro for Reusable Logic

`macros/calculate_occupancy_rate.sql`:
```sql
{% macro calculate_occupancy_rate(unavailable_days, total_days) %}
    case
        when {{ total_days }} > 0 then
            ({{ unavailable_days }}::decimal / {{ total_days }}::decimal) * 100
        else 0
    end
{% endmacro %}
```

Use in model:
```sql
select
    listing_id,
    month,
    {{ calculate_occupancy_rate('unavailable_days', 'total_days') }} as occupancy_rate
from {{ ref('agg_listing_monthly_performance') }}
```

### Pattern 4: Refresh Incremental Model for New Snapshot

```bash
# Load new calendar data
python scripts/load_inside_airbnb_to_snowflake.py

# Full refresh fact table
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Test
dbt test --select fct_listing_calendar --profiles-dir .
```

## Troubleshooting

### Issue: dbt Can't Connect to Snowflake

**Symptoms**:
```
Database Error in model ... (403): Incorrect username or password
```

**Solutions**:
1. Verify `profiles.yml` credentials match Snowflake account
2. Check account identifier format (should be `<org>-<account>` or legacy `<account>.<region>`)
3. Ensure role has access to warehouse and database:
```sql
GRANT USAGE ON WAREHOUSE COMPUTE_WH TO ROLE YOUR_ROLE;
GRANT USAGE ON DATABASE AIRBNB_DBT TO ROLE YOUR_ROLE;
GRANT CREATE SCHEMA ON DATABASE AIRBNB_DBT TO ROLE YOUR_ROLE;
```
4. Test connection with `dbt debug --profiles-dir .`

### Issue: Python Loader Upload Fails

**Symptoms**:
```
FileNotFoundError: [Errno 2] No such file or directory: 'data/raw/listings.csv.gz'
```

**Solutions**:
1. Verify files exist in `data/raw/` directory
2. Check file names match exactly (case-sensitive)
3. Ensure working directory is project root when running script

### Issue: Incremental Model Not Updating

**Symptoms**: New data loaded but `fct_listing_calendar` unchanged

**Solutions**:
1. Check unique_key in config is correct
2. Verify `is_incremental()` filter logic:
```sql
{% if is_incremental() %}
    where calendar_date > (select max(calendar_date) from {{ this }})
{% endif %}
```
3. Force full refresh: `dbt run --full-refresh --select fct_listing_calendar`
4. Inspect raw tables to confirm new data loaded

### Issue: Streamlit Dashboard Shows No Data

**Symptoms**: Dashboard loads but charts/tables empty

**Solutions**:
1. Verify dbt models have run: `dbt run --profiles-dir .`
2. Check schema name in `config/local_credentials.json` matches dbt target schema
3. Query directly in Snowflake to confirm data:
```sql
SELECT COUNT(*) FROM AIRBNB_DBT.ANALYTICS.AGG_NEIGHBOURHOOD_MONTHLY_PERFORMANCE;
```
4. Check Streamlit console for SQL errors
5. Clear Streamlit cache: restart app or add `st.cache_data.clear()`

### Issue: dbt Test Failures

**Symptoms**:
```
Failure in test unique_stg_airbnb__listings_listing_id
Got 5 results, expected 0
```

**Solutions**:
1. Inspect failing rows:
```sql
-- Compiled test query in target/compiled/...
SELECT listing_id, COUNT(*)
FROM {{ ref('stg_airbnb__listings') }}
GROUP BY listing_id
HAVING COUNT(*) > 1
```
2. Fix upstream data or add deduplication logic in staging model
3. For accepted_values failures, check raw data for unexpected values
4. Update test expectations if data has changed

### Issue: Out of Warehouse Credits

**Symptoms**: Queries hang or fail with warehouse suspension

**Solutions**:
1. Check Snowflake warehouse auto-suspend (set to 60 seconds for dev)
2. Use smaller warehouse for dbt development (X-Small sufficient for this dataset)
3. Monitor credit usage in Snowflake UI → Account → Usage
4. Consider trial account for learning (free $400 credits)

## Advanced Usage

### Running Subset of Models

```bash
# Only marts layer
dbt run --select marts --profiles-dir .

# Specific model and parents
dbt run --select +fct_listing_calendar --profiles-dir .

# Tag-based selection (add tags to model configs)
dbt run --select tag:daily --profiles-dir .
```

### Environment-Specific Targets

Extend `profiles.yml`:
```yaml
snowflake_dbt_project:
  outputs:
    dev:
      type: snowflake
      schema: DEV_ANALYTICS
      # ... other dev settings
    prod:
      type: snowflake
      schema: ANALYTICS
      # ... prod settings
  target: dev
```

Run against prod:
```bash
dbt run --target prod --profiles-dir .
```

### Custom dbt Tests with Thresholds

```yaml
# models/marts/fct_listing_calendar.yml
models:
  - name: fct_listing_calendar
    tests:
      - dbt_utils.expression_is_true:
          expression: "price >= 0 AND price <= 10000"
          config:
            severity: warn
      - dbt_utils.recency:
          datepart: day
          field: calendar_date
          interval: 1
```

This skill provides comprehensive guidance for building production-grade analytics pipelines with Snowflake and dbt using real-world open data.
