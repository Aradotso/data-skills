---
name: snowflake-dbt-airbnb-analytics
description: Build end-to-end Snowflake data warehouse with dbt transformations, staging to marts, using Inside Airbnb dataset and Streamlit dashboards
triggers:
  - set up snowflake dbt airbnb project
  - load inside airbnb data to snowflake
  - create dbt staging models for airbnb
  - build incremental fact tables in dbt
  - configure snowflake dbt profiles
  - run dbt tests for data quality
  - deploy streamlit dashboard for airbnb analytics
  - troubleshoot dbt snowflake connection
---

# Snowflake dbt Airbnb Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a complete analytics engineering workflow: loading Inside Airbnb open data into Snowflake, transforming it through dbt staging/intermediate/mart layers, validating data quality with dbt tests, and visualizing results in a Streamlit dashboard.

## What This Project Does

- Loads CSV/GZIP files from Inside Airbnb into Snowflake internal stages
- Creates raw text-preserving tables in Snowflake
- Builds dbt staging views with type casting and standardization
- Creates intermediate models with joins and revenue proxy calculations
- Produces analytics-ready dimension and fact tables
- Aggregates monthly performance metrics by listing and neighbourhood
- Powers a Streamlit dashboard with Snowflake connection
- Validates data quality with generic and singular dbt tests

## Installation

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

## Configuration

### 1. Snowflake Credentials

Create local configuration files (ignored by git):

```bash
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

**Edit `profiles.yml`** with your Snowflake connection:

```yaml
airbnb_analytics:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT
      user: YOUR_USERNAME
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"  # Or hardcode locally
      role: AIRBNB_ROLE
      database: AIRBNB_DB
      warehouse: AIRBNB_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

**Edit `config/local_credentials.json`** for Python scripts and Streamlit:

```json
{
  "account": "YOUR_ACCOUNT",
  "user": "YOUR_USERNAME",
  "password": "YOUR_PASSWORD",
  "role": "AIRBNB_ROLE",
  "warehouse": "AIRBNB_WH",
  "database": "AIRBNB_DB",
  "schema": "ANALYTICS"
}
```

### 2. Raw Data Files

Download Inside Airbnb data from [insideairbnb.com/get-the-data](http://insideairbnb.com/get-the-data/) (recommended: New York City).

Place files in `data/raw/`:

```text
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

## Loading Raw Data to Snowflake

The Python loader creates Snowflake objects, uploads files to internal stage, and copies into raw tables:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What it does:**

1. Reads credentials from `config/local_credentials.json`
2. Executes `setup/snowflake_setup.sql` to create database, schemas, warehouse, stage
3. Uploads local files to `@INSIDE_AIRBNB_STAGE`
4. Creates raw tables with text columns preserving original data
5. Runs `COPY INTO` commands to load data

**Expected output:**

```text
Connected to Snowflake account: YOUR_ACCOUNT
Executing Snowflake setup SQL...
Setup completed.
Uploading data/raw/listings.csv.gz...
Uploading data/raw/calendar.csv.gz...
Uploading data/raw/reviews.csv.gz...
Uploading data/raw/neighbourhoods.csv...
Upload complete.
Creating raw tables from stage...
Raw tables created: LISTINGS, CALENDAR, REVIEWS, NEIGHBOURHOODS
Loading data into raw tables...
Data load complete.
```

## dbt Model Layers

### Staging Models (`models/staging/`)

Clean and type-cast raw data with standard naming:

```sql
-- models/staging/stg_airbnb__listings.sql
WITH source AS (
    SELECT * FROM {{ source('airbnb_raw', 'LISTINGS') }}
)

SELECT
    TRY_CAST(id AS NUMBER) AS listing_id,
    name AS listing_name,
    TRY_CAST(host_id AS NUMBER) AS host_id,
    host_name,
    neighbourhood_cleansed AS neighbourhood,
    latitude::FLOAT AS latitude,
    longitude::FLOAT AS longitude,
    room_type,
    TRY_CAST(price AS NUMBER) AS price,
    TRY_CAST(minimum_nights AS NUMBER) AS minimum_nights,
    TRY_CAST(number_of_reviews AS NUMBER) AS number_of_reviews,
    TRY_CAST(reviews_per_month AS FLOAT) AS reviews_per_month,
    TRY_CAST(availability_365 AS NUMBER) AS availability_365
FROM source
WHERE TRY_CAST(id AS NUMBER) IS NOT NULL
```

### Intermediate Models (`models/intermediate/`)

Join and enrich data with business logic:

```sql
-- models/intermediate/int_airbnb__calendar_enriched.sql
WITH calendar AS (
    SELECT * FROM {{ ref('stg_airbnb__calendar') }}
),

listings AS (
    SELECT * FROM {{ ref('int_airbnb__listing_enriched') }}
)

SELECT
    c.listing_id,
    c.date,
    c.available,
    c.price,
    l.listing_name,
    l.neighbourhood,
    l.room_type,
    l.host_id,
    l.host_name,
    -- Revenue proxy: if unavailable, assume booked at price
    CASE 
        WHEN c.available = FALSE AND c.price > 0 
        THEN c.price 
        ELSE 0 
    END AS estimated_revenue
FROM calendar c
LEFT JOIN listings l 
    ON c.listing_id = l.listing_id
```

### Mart Models (`models/marts/`)

Analytics-ready dimensions and facts:

```sql
-- models/marts/dim_listings.sql
SELECT
    listing_id,
    listing_name,
    host_id,
    neighbourhood,
    room_type,
    latitude,
    longitude,
    minimum_nights,
    number_of_reviews,
    reviews_per_month,
    availability_365
FROM {{ ref('int_airbnb__listing_enriched') }}
```

**Incremental fact table:**

```sql
-- models/marts/fct_listing_calendar.sql
{{ config(
    materialized='incremental',
    unique_key='listing_date_key',
    merge_update_columns=['available', 'price', 'estimated_revenue']
) }}

SELECT
    {{ dbt_utils.generate_surrogate_key(['listing_id', 'date']) }} AS listing_date_key,
    listing_id,
    date,
    available,
    price,
    estimated_revenue
FROM {{ ref('int_airbnb__calendar_enriched') }}

{% if is_incremental() %}
WHERE date > (SELECT MAX(date) FROM {{ this }})
{% endif %}
```

### Aggregate Models (`models/marts/`)

Monthly performance metrics:

```sql
-- models/marts/agg_listing_monthly_performance.sql
SELECT
    listing_id,
    DATE_TRUNC('month', date) AS month,
    COUNT(*) AS total_days,
    SUM(CASE WHEN available = FALSE THEN 1 ELSE 0 END) AS unavailable_days,
    AVG(price) AS avg_price,
    SUM(estimated_revenue) AS total_estimated_revenue
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, DATE_TRUNC('month', date)
```

## Running dbt

```bash
# Test connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select fct_listing_calendar+ --profiles-dir .

# Full refresh for incremental models (after new data load)
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

## dbt Tests

### Generic Tests (in schema.yml)

```yaml
# models/staging/schema.yml
version: 2

sources:
  - name: airbnb_raw
    database: AIRBNB_DB
    schema: RAW
    tables:
      - name: LISTINGS
        columns:
          - name: id
            tests:
              - not_null
              - unique

models:
  - name: stg_airbnb__listings
    columns:
      - name: listing_id
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
```

### Singular Tests (in tests/)

```sql
-- tests/assert_no_duplicate_listing_dates.sql
SELECT
    listing_id,
    date,
    COUNT(*) AS record_count
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, date
HAVING COUNT(*) > 1
```

## Streamlit Dashboard

The dashboard connects to Snowflake and queries mart tables:

```python
# dashboard/streamlit_app.py
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load credentials
with open('config/local_credentials.json') as f:
    creds = json.load(f)

# Connect to Snowflake
@st.cache_resource
def get_connection():
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        role=creds['role'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema']
    )

conn = get_connection()

# Query neighbourhood performance
@st.cache_data(ttl=600)
def load_neighbourhood_performance():
    query = """
    SELECT 
        neighbourhood,
        SUM(total_estimated_revenue) AS total_revenue,
        AVG(avg_price) AS avg_price,
        SUM(unavailable_days) AS total_unavailable_days
    FROM agg_neighbourhood_monthly_performance
    GROUP BY neighbourhood
    ORDER BY total_revenue DESC
    LIMIT 20
    """
    return pd.read_sql(query, conn)

df = load_neighbourhood_performance()

st.title("Airbnb Analytics Dashboard")
st.subheader("Top Neighbourhoods by Estimated Revenue")
st.bar_chart(df.set_index('NEIGHBOURHOOD')['TOTAL_REVENUE'])
st.dataframe(df)
```

Run with:

```bash
streamlit run dashboard/streamlit_app.py
```

## Common Patterns

### Adding a New Staging Model

1. Create SQL file in `models/staging/`:

```sql
-- models/staging/stg_airbnb__new_source.sql
WITH source AS (
    SELECT * FROM {{ source('airbnb_raw', 'NEW_TABLE') }}
)

SELECT
    TRY_CAST(id AS NUMBER) AS record_id,
    name,
    created_at::TIMESTAMP AS created_at
FROM source
```

2. Add to `models/staging/schema.yml`:

```yaml
models:
  - name: stg_airbnb__new_source
    columns:
      - name: record_id
        tests:
          - not_null
          - unique
```

3. Run:

```bash
dbt run --select stg_airbnb__new_source
dbt test --select stg_airbnb__new_source
```

### Creating a Custom Schema Macro

```sql
-- macros/generate_schema_name.sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}
    {%- if custom_schema_name is none -%}
        {{ default_schema }}
    {%- else -%}
        {{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

### Refreshing Data Pipeline

Complete refresh after new Inside Airbnb snapshot:

```bash
# 1. Load new raw data
python scripts/load_inside_airbnb_to_snowflake.py

# 2. Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 3. Run tests
dbt test --profiles-dir .

# 4. Regenerate docs
dbt docs generate --profiles-dir .
```

## Troubleshooting

### Connection Failed

**Issue:** `dbt debug` fails with "Could not connect to Snowflake"

**Solution:**

- Verify Snowflake account identifier format (e.g., `xy12345.us-east-1`)
- Check `profiles.yml` indentation (YAML is whitespace-sensitive)
- Ensure warehouse is running and role has permissions
- Test credentials with Python:

```python
import snowflake.connector
conn = snowflake.connector.connect(
    account='YOUR_ACCOUNT',
    user='YOUR_USER',
    password='YOUR_PASSWORD'
)
print("Connected:", conn.cursor().execute("SELECT CURRENT_USER()").fetchone())
```

### Raw Table Load Fails

**Issue:** `COPY INTO` fails with "File not found"

**Solution:**

- Verify files exist in `data/raw/`
- Check file names match exactly (case-sensitive on some systems)
- Ensure Snowflake stage was created:

```sql
SHOW STAGES LIKE 'INSIDE_AIRBNB_STAGE';
LIST @INSIDE_AIRBNB_STAGE;
```

### dbt Model Compilation Error

**Issue:** `Compilation Error in model ... depends on a node that does not exist`

**Solution:**

- Run `dbt deps` to install dbt packages (if using `packages.yml`)
- Verify source/ref names match exactly
- Check `dbt_project.yml` for correct model paths

### Incremental Model Not Updating

**Issue:** `fct_listing_calendar` not showing new dates

**Solution:**

```bash
# Force full refresh
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Or drop and recreate
dbt run-operation drop_relation --args '{relation: "fct_listing_calendar"}' --profiles-dir .
dbt run --select fct_listing_calendar --profiles-dir .
```

### Streamlit Connection Error

**Issue:** `ProgrammingError: 250001: Could not connect to Snowflake backend`

**Solution:**

- Verify `config/local_credentials.json` exists and has correct values
- Check file path in `streamlit_app.py` (relative to execution directory)
- Use absolute path if needed:

```python
import os
config_path = os.path.join(os.path.dirname(__file__), '..', 'config', 'local_credentials.json')
with open(config_path) as f:
    creds = json.load(f)
```

### Test Failures

**Issue:** dbt tests fail after data refresh

**Solution:**

- Review test output for specific failures
- Check if new data violates accepted values:

```bash
dbt test --select stg_airbnb__listings --profiles-dir .
```

- Update accepted values in `schema.yml` if data changed
- Add new test conditions for edge cases

## Key Files

- `dbt_project.yml` — dbt project configuration, model paths, materializations
- `profiles.yml` — Snowflake connection profile (local, not committed)
- `models/staging/schema.yml` — Source and model definitions with tests
- `scripts/load_inside_airbnb_to_snowflake.py` — Python loader script
- `setup/snowflake_setup.sql` — Snowflake DDL for database, warehouse, stage
- `config/local_credentials.json` — Snowflake credentials for Python/Streamlit (local, not committed)
- `dashboard/streamlit_app.py` — Analytics dashboard app

## Environment Variables (Production)

For production deployments, use environment variables instead of local files:

```bash
export SNOWFLAKE_ACCOUNT=xy12345.us-east-1
export SNOWFLAKE_USER=dbt_user
export SNOWFLAKE_PASSWORD=your_secure_password
export SNOWFLAKE_ROLE=AIRBNB_ROLE
export SNOWFLAKE_WAREHOUSE=AIRBNB_WH
export SNOWFLAKE_DATABASE=AIRBNB_DB
```

Reference in `profiles.yml`:

```yaml
password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
```

Reference in Python:

```python
import os
conn = snowflake.connector.connect(
    account=os.getenv('SNOWFLAKE_ACCOUNT'),
    user=os.getenv('SNOWFLAKE_USER'),
    password=os.getenv('SNOWFLAKE_PASSWORD'),
    # ... other params
)
```
