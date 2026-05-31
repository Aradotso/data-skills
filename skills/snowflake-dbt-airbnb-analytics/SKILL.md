---
name: snowflake-dbt-airbnb-analytics
description: Build end-to-end analytics engineering pipelines with Snowflake, dbt, and Streamlit using Inside Airbnb data
triggers:
  - how do I set up a dbt project with Snowflake
  - load Inside Airbnb data into Snowflake
  - create dbt staging and mart models
  - build incremental fact tables in dbt
  - connect Streamlit dashboard to Snowflake
  - configure dbt profiles for Snowflake
  - run dbt tests for data quality
  - create analytics aggregations with dbt
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build complete analytics engineering pipelines using Snowflake as a data warehouse, dbt for transformations, and Streamlit for dashboards. The project demonstrates a multi-layered analytics architecture with Inside Airbnb open data.

## What This Project Does

The `Snowflake_DBT_Project` is a portfolio-grade analytics engineering implementation that:

- Loads Inside Airbnb CSV/GZIP files into Snowflake internal stages
- Creates raw, staging, intermediate, and mart layers using dbt
- Implements incremental fact tables with Snowflake merge strategy
- Validates data quality with dbt tests (generic and singular)
- Powers a Streamlit dashboard with analytics-ready marts
- Demonstrates public-safe credential management (no committed secrets)

**Architecture layers:**
1. **Raw zone**: Text-preserving tables loaded from CSV files
2. **Staging**: Clean, cast, and standardize column names
3. **Intermediate**: Reusable joins, enrichment, revenue proxy logic
4. **Marts**: Dimensions (`dim_listings`, `dim_hosts`) and facts (`fct_listing_calendar`, `fct_reviews`)
5. **Aggregates**: Monthly performance metrics by listing and neighbourhood
6. **Dashboard**: Streamlit app connected to mart tables

## Installation

### Prerequisites

- Snowflake account with database creation privileges
- Python 3.8+
- Inside Airbnb raw data files (CSV/GZIP format)

### Setup Steps

```bash
# Clone and navigate to project
cd Snowflake_DBT_Project

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

**Expected `requirements.txt` includes:**
```text
dbt-snowflake>=1.5.0
snowflake-connector-python>=3.0.0
streamlit>=1.20.0
pandas>=1.5.0
```

### Credential Configuration

This project uses local-only files for credentials (never committed):

```bash
# Copy example files
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

**Edit `profiles.yml`:**
```yaml
airbnb_analytics:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT
      user: YOUR_USERNAME
      password: YOUR_PASSWORD
      role: YOUR_ROLE
      database: AIRBNB_ANALYTICS
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

**Edit `config/local_credentials.json`:**
```json
{
  "account": "YOUR_ACCOUNT",
  "user": "YOUR_USERNAME",
  "password": "YOUR_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_ANALYTICS",
  "schema": "ANALYTICS",
  "role": "YOUR_ROLE"
}
```

**For production, use environment variables:**
```python
import os

snowflake_config = {
    "account": os.getenv("SNOWFLAKE_ACCOUNT"),
    "user": os.getenv("SNOWFLAKE_USER"),
    "password": os.getenv("SNOWFLAKE_PASSWORD"),
    "warehouse": os.getenv("SNOWFLAKE_WAREHOUSE"),
    "database": os.getenv("SNOWFLAKE_DATABASE"),
    "role": os.getenv("SNOWFLAKE_ROLE")
}
```

## Data Loading

### Download Inside Airbnb Data

Download the following files for New York City from [insideairbnb.com/get-the-data](https://insideairbnb.com/get-the-data/):

```text
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

### Load to Snowflake

The loader script creates all necessary Snowflake objects and loads data:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What this does:**
1. Executes `setup/snowflake_setup.sql` to create database, schemas, and stage
2. Uploads local files to Snowflake internal stage `INSIDE_AIRBNB_STAGE`
3. Infers CSV headers and creates raw tables
4. Copies data from stage to `RAW` schema tables

**Key loader code pattern:**
```python
import snowflake.connector
import json

# Load credentials
with open('config/local_credentials.json') as f:
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

# Create stage
cursor.execute("""
    CREATE STAGE IF NOT EXISTS INSIDE_AIRBNB_STAGE
    FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1);
""")

# Upload file
cursor.execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE;")

# Copy into table
cursor.execute("""
    COPY INTO RAW.LISTINGS
    FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
    FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1);
""")

conn.close()
```

## dbt Project Structure

### Model Layers

**Staging models** (`models/staging/`):
```sql
-- stg_airbnb__listings.sql
WITH source AS (
    SELECT * FROM {{ source('airbnb_raw', 'listings') }}
)

SELECT
    id::BIGINT AS listing_id,
    name AS listing_name,
    host_id::BIGINT AS host_id,
    host_name,
    neighbourhood_cleansed AS neighbourhood,
    room_type,
    price::FLOAT AS price,
    minimum_nights::INT AS minimum_nights,
    number_of_reviews::INT AS number_of_reviews,
    last_review::DATE AS last_review_date,
    reviews_per_month::FLOAT AS reviews_per_month,
    availability_365::INT AS availability_365
FROM source
WHERE id IS NOT NULL
```

**Intermediate models** (`models/intermediate/`):
```sql
-- int_airbnb__listing_enriched.sql
WITH listings AS (
    SELECT * FROM {{ ref('stg_airbnb__listings') }}
),

neighbourhoods AS (
    SELECT * FROM {{ ref('stg_airbnb__neighbourhoods') }}
)

SELECT
    l.listing_id,
    l.listing_name,
    l.host_id,
    l.host_name,
    l.neighbourhood,
    n.neighbourhood_group,
    l.room_type,
    l.price,
    l.minimum_nights,
    l.number_of_reviews,
    l.last_review_date,
    l.reviews_per_month,
    l.availability_365,
    CASE
        WHEN l.number_of_reviews = 0 THEN 'New'
        WHEN l.number_of_reviews < 10 THEN 'Low Activity'
        WHEN l.number_of_reviews < 50 THEN 'Medium Activity'
        ELSE 'High Activity'
    END AS activity_level
FROM listings l
LEFT JOIN neighbourhoods n ON l.neighbourhood = n.neighbourhood
```

**Incremental fact table** (`models/marts/`):
```sql
-- fct_listing_calendar.sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        on_schema_change='fail'
    )
}}

WITH calendar AS (
    SELECT * FROM {{ ref('int_airbnb__calendar_enriched') }}
)

SELECT
    listing_id,
    calendar_date,
    available,
    price,
    minimum_nights,
    maximum_nights,
    CASE
        WHEN NOT available THEN price
        ELSE 0
    END AS estimated_revenue
FROM calendar

{% if is_incremental() %}
    WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
{% endif %}
```

**Dimension table:**
```sql
-- dim_listings.sql
SELECT
    listing_id,
    listing_name,
    host_id,
    host_name,
    neighbourhood,
    neighbourhood_group,
    room_type,
    price,
    minimum_nights,
    availability_365,
    activity_level
FROM {{ ref('int_airbnb__listing_enriched') }}
```

**Aggregate model:**
```sql
-- agg_listing_monthly_performance.sql
WITH calendar_facts AS (
    SELECT * FROM {{ ref('fct_listing_calendar') }}
),

listings AS (
    SELECT * FROM {{ ref('dim_listings') }}
)

SELECT
    cf.listing_id,
    l.listing_name,
    l.neighbourhood,
    l.room_type,
    DATE_TRUNC('month', cf.calendar_date) AS month,
    COUNT(*) AS total_days,
    SUM(CASE WHEN cf.available THEN 1 ELSE 0 END) AS available_days,
    SUM(CASE WHEN NOT cf.available THEN 1 ELSE 0 END) AS unavailable_days,
    AVG(cf.price) AS avg_price,
    SUM(cf.estimated_revenue) AS total_estimated_revenue
FROM calendar_facts cf
LEFT JOIN listings l ON cf.listing_id = l.listing_id
GROUP BY 1, 2, 3, 4, 5
```

### Sources Configuration

**`models/staging/sources.yml`:**
```yaml
version: 2

sources:
  - name: airbnb_raw
    database: AIRBNB_ANALYTICS
    schema: RAW
    tables:
      - name: listings
        description: Raw Airbnb listings data
        columns:
          - name: id
            description: Unique listing identifier
            tests:
              - not_null
              - unique

      - name: calendar
        description: Daily availability and pricing
        columns:
          - name: listing_id
            tests:
              - not_null
          - name: date
            tests:
              - not_null

      - name: reviews
        description: Guest reviews
        columns:
          - name: id
            tests:
              - not_null
              - unique

      - name: neighbourhoods
        description: Neighbourhood reference data
```

### Data Quality Tests

**Generic tests in model YAML:**
```yaml
version: 2

models:
  - name: dim_listings
    description: Listing dimension table
    columns:
      - name: listing_id
        description: Primary key
        tests:
          - not_null
          - unique
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
    description: Daily calendar facts
    tests:
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns:
            - listing_id
            - calendar_date
```

**Singular test** (`tests/assert_no_negative_revenue.sql`):
```sql
SELECT
    listing_id,
    calendar_date,
    estimated_revenue
FROM {{ ref('fct_listing_calendar') }}
WHERE estimated_revenue < 0
```

## Key dbt Commands

```bash
# Verify connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select dim_listings+ --profiles-dir .

# Run only staging models
dbt run --select staging.* --profiles-dir .

# Full refresh incremental model
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Run all tests
dbt test --profiles-dir .

# Test specific model
dbt test --select dim_listings --profiles-dir .

# Generate and serve documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .

# Compile SQL without running
dbt compile --profiles-dir .

# Show compiled SQL for a model
dbt show --select dim_listings --profiles-dir .
```

## Streamlit Dashboard

### Dashboard Code Pattern

**`dashboard/streamlit_app.py`:**
```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load credentials
with open('config/local_credentials.json') as f:
    creds = json.load(f)

# Connect to Snowflake
@st.cache_resource
def get_snowflake_connection():
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema'],
        role=creds['role']
    )

# Query helper
@st.cache_data(ttl=600)
def run_query(query):
    conn = get_snowflake_connection()
    df = pd.read_sql(query, conn)
    return df

# Dashboard layout
st.title("Inside Airbnb Analytics")

# Neighbourhood performance
st.header("Top Neighbourhoods by Revenue")

neighbourhood_query = """
    SELECT
        neighbourhood,
        SUM(total_estimated_revenue) AS total_revenue,
        AVG(avg_price) AS avg_price,
        COUNT(DISTINCT listing_id) AS listing_count
    FROM agg_neighbourhood_monthly_performance
    GROUP BY neighbourhood
    ORDER BY total_revenue DESC
    LIMIT 10
"""

df_neighbourhoods = run_query(neighbourhood_query)
st.bar_chart(df_neighbourhoods.set_index('NEIGHBOURHOOD')['TOTAL_REVENUE'])

# Room type analysis
st.header("Room Type Distribution")

room_type_query = """
    SELECT
        room_type,
        COUNT(*) AS listing_count,
        AVG(price) AS avg_price
    FROM dim_listings
    GROUP BY room_type
"""

df_room_types = run_query(room_type_query)
st.dataframe(df_room_types)

# Monthly trends
st.header("Monthly Revenue Trends")

monthly_query = """
    SELECT
        month,
        SUM(total_estimated_revenue) AS revenue
    FROM agg_listing_monthly_performance
    GROUP BY month
    ORDER BY month
"""

df_monthly = run_query(monthly_query)
st.line_chart(df_monthly.set_index('MONTH')['REVENUE'])
```

### Run Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

## Common Patterns

### Pattern: Add a New Source Table

1. **Place raw file:**
   ```bash
   # Download file to data/raw/
   cp ~/Downloads/new_table.csv data/raw/
   ```

2. **Update loader script:**
   ```python
   # In scripts/load_inside_airbnb_to_snowflake.py
   cursor.execute("PUT file://data/raw/new_table.csv @INSIDE_AIRBNB_STAGE;")
   cursor.execute("COPY INTO RAW.NEW_TABLE FROM @INSIDE_AIRBNB_STAGE/new_table.csv;")
   ```

3. **Add to dbt sources:**
   ```yaml
   # models/staging/sources.yml
   sources:
     - name: airbnb_raw
       tables:
         - name: new_table
           columns:
             - name: id
               tests:
                 - not_null
   ```

4. **Create staging model:**
   ```sql
   -- models/staging/stg_airbnb__new_table.sql
   SELECT
       id::BIGINT AS new_table_id,
       field1,
       field2::DATE AS field2_date
   FROM {{ source('airbnb_raw', 'new_table') }}
   ```

### Pattern: Create Custom dbt Macro

**`macros/cents_to_dollars.sql`:**
```sql
{% macro cents_to_dollars(column_name) %}
    ({{ column_name }} / 100.0)::DECIMAL(10,2)
{% endmacro %}
```

**Usage in model:**
```sql
SELECT
    listing_id,
    {{ cents_to_dollars('price_cents') }} AS price_dollars
FROM {{ ref('stg_airbnb__listings') }}
```

### Pattern: Incremental Model with Delete+Insert

```sql
{{
    config(
        materialized='incremental',
        unique_key='listing_id',
        incremental_strategy='delete+insert'
    )
}}

SELECT
    listing_id,
    listing_name,
    CURRENT_TIMESTAMP() AS updated_at
FROM {{ ref('stg_airbnb__listings') }}

{% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

### Pattern: Use dbt_utils for Testing

**Install package** (`packages.yml`):
```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1
```

```bash
dbt deps --profiles-dir .
```

**Use in tests:**
```yaml
models:
  - name: fct_listing_calendar
    tests:
      - dbt_utils.recency:
          datepart: day
          field: calendar_date
          interval: 7
```

## Troubleshooting

### Connection Issues

**Problem:** `dbt debug` fails with "Invalid account name"

**Solution:**
- Verify `account` in `profiles.yml` matches Snowflake format: `organization-account_name` (e.g., `myorg-myaccount`)
- Check with `SHOW ORGANIZATIONS;` and `SHOW ACCOUNTS;` in Snowflake

**Problem:** "Object does not exist" errors

**Solution:**
```sql
-- In Snowflake, verify:
USE ROLE YOUR_ROLE;
SHOW DATABASES;
SHOW SCHEMAS IN DATABASE AIRBNB_ANALYTICS;
```

### Data Loading Issues

**Problem:** CSV upload fails with special characters

**Solution:**
```python
# Specify encoding in PUT command
cursor.execute("""
    PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE
    FILE_FORMAT = (
        TYPE = 'CSV'
        FIELD_OPTIONALLY_ENCLOSED_BY = '"'
        ENCODING = 'UTF-8'
        SKIP_HEADER = 1
    );
""")
```

**Problem:** Incremental model not updating

**Solution:**
```bash
# Force full refresh
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Check unique_key configuration
dbt compile --select fct_listing_calendar --profiles-dir .
# Review compiled SQL in target/compiled/
```

### Test Failures

**Problem:** Uniqueness test fails on composite key

**Solution:**
```yaml
# Use dbt_utils
tests:
  - dbt_utils.unique_combination_of_columns:
      combination_of_columns:
        - listing_id
        - calendar_date
```

**Problem:** Accepted values test fails after new data load

**Solution:**
```sql
-- Check actual values
SELECT DISTINCT room_type
FROM {{ ref('stg_airbnb__listings') }};

-- Update test
tests:
  - accepted_values:
      values: ['Entire home/apt', 'Private room', 'Shared room', 'Hotel room', 'Tiny home']
```

## Best Practices

1. **Always use `--profiles-dir .`** to reference local profiles.yml
2. **Never commit credentials** — add to `.gitignore`:
   ```
   profiles.yml
   config/local_credentials.json
   target/
   logs/
   .venv/
   ```
3. **Use incremental models for large fact tables** (calendar, reviews)
4. **Document models and columns** in YAML files for `dbt docs`
5. **Run tests after every data load** to catch quality issues early
6. **Use environment variables in production** instead of JSON files

## Advanced: CI/CD Integration

**GitHub Actions example** (`.github/workflows/dbt.yml`):
```yaml
name: dbt CI
on: [push]
jobs:
  dbt-run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      - run: pip install -r requirements.txt
      - name: dbt run and test
        env:
          SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
          SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
          SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
        run: |
          dbt run --profiles-dir .
          dbt test --profiles-dir .
```

**Dynamic `profiles.yml` for CI:**
```yaml
airbnb_analytics:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: ANALYTICS_ROLE
      database: AIRBNB_ANALYTICS
      warehouse: COMPUTE_WH
      schema: ANALYTICS
```

This skill equips AI agents to guide developers through complete analytics engineering workflows with Snowflake, dbt, and Streamlit.
