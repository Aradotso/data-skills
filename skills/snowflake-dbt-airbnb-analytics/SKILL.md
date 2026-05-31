---
name: snowflake-dbt-airbnb-analytics
description: Build end-to-end analytics pipelines with Snowflake, dbt, and Inside Airbnb data including staging, marts, incremental models, tests, and Streamlit dashboards.
triggers:
  - set up snowflake dbt airbnb project
  - load inside airbnb data to snowflake
  - create dbt marts for airbnb analytics
  - build incremental fact table in dbt
  - configure dbt with snowflake profiles
  - add dbt tests for data quality
  - create streamlit dashboard for snowflake data
  - run dbt models with full refresh
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to work with the Snowflake_DBT_Project, a complete analytics engineering pipeline that:

- Loads Inside Airbnb open dataset into Snowflake via Python
- Transforms raw data through dbt staging → intermediate → marts layers
- Implements incremental fact models with Snowflake merge strategy
- Validates data quality with dbt generic and singular tests
- Powers a Streamlit dashboard with analytics-ready marts

## Project Architecture

**Data Flow:**
1. Inside Airbnb CSV/GZIP files → Python loader
2. Snowflake internal stage → RAW schema tables
3. dbt staging views (clean + cast)
4. dbt intermediate layer (joins + enrichment)
5. dbt marts (dimensions + facts + aggregates)
6. Streamlit dashboard

**Key Models:**
- **Staging:** `stg_airbnb__listings`, `stg_airbnb__calendar`, `stg_airbnb__reviews`, `stg_airbnb__neighbourhoods`
- **Intermediate:** `int_airbnb__listing_enriched`, `int_airbnb__calendar_enriched`, `int_airbnb__reviews_enriched`
- **Marts:** `dim_listings`, `dim_hosts`, `fct_listing_calendar` (incremental), `fct_reviews`, `agg_listing_monthly_performance`, `agg_neighbourhood_monthly_performance`

## Installation & Setup

### 1. Clone and Install Dependencies

```bash
git clone https://github.com/analyticsdurgesh/Snowflake_DBT_Project.git
cd Snowflake_DBT_Project
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Configure Snowflake Credentials

Create local credential files (ignored by git):

```bash
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

**Edit `profiles.yml`:**

```yaml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT  # e.g., abc12345.us-east-1
      user: YOUR_USERNAME
      password: YOUR_PASSWORD
      role: YOUR_ROLE
      database: INSIDE_AIRBNB
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
  "database": "INSIDE_AIRBNB",
  "schema": "ANALYTICS",
  "role": "YOUR_ROLE"
}
```

### 3. Download Inside Airbnb Data

Download the NYC dataset from [Inside Airbnb](https://insideairbnb.com/get-the-data/) and place files in `data/raw/`:

```text
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

## Loading Data to Snowflake

### Python Loader Script

The `scripts/load_inside_airbnb_to_snowflake.py` script automates:
1. Creating Snowflake database, schemas, stage, and file format
2. Uploading local files to internal stage
3. Creating raw tables from CSV headers
4. Copying data into raw tables

**Run the loader:**

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**Key Python patterns from the loader:**

```python
import snowflake.connector
import json
import os

# Load credentials from local file
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

# Create database and schemas
cursor.execute("CREATE DATABASE IF NOT EXISTS INSIDE_AIRBNB")
cursor.execute("USE DATABASE INSIDE_AIRBNB")
cursor.execute("CREATE SCHEMA IF NOT EXISTS RAW")
cursor.execute("CREATE SCHEMA IF NOT EXISTS ANALYTICS")

# Create internal stage
cursor.execute("""
CREATE OR REPLACE STAGE INSIDE_AIRBNB_STAGE
FILE_FORMAT = (
  TYPE = 'CSV'
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  SKIP_HEADER = 1
  NULL_IF = ('', 'NULL', 'null')
  EMPTY_FIELD_AS_NULL = TRUE
)
""")

# Upload file to stage
cursor.execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=TRUE")

# Create raw table and copy data
cursor.execute("""
CREATE OR REPLACE TABLE RAW.LISTINGS (
  id VARCHAR,
  name VARCHAR,
  host_id VARCHAR,
  host_name VARCHAR,
  neighbourhood VARCHAR,
  latitude VARCHAR,
  longitude VARCHAR,
  room_type VARCHAR,
  price VARCHAR,
  minimum_nights VARCHAR,
  number_of_reviews VARCHAR,
  last_review VARCHAR,
  reviews_per_month VARCHAR,
  calculated_host_listings_count VARCHAR,
  availability_365 VARCHAR
)
""")

cursor.execute("COPY INTO RAW.LISTINGS FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz")

cursor.close()
conn.close()
```

## dbt Commands

### Core Workflow

```bash
# Verify dbt connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select dim_listings+ --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

### Incremental Model Refresh

For new data snapshots:

```bash
# Reload raw data
python scripts/load_inside_airbnb_to_snowflake.py

# Full refresh incremental fact table and downstream
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Test updated data
dbt test --profiles-dir .
```

### Model-Specific Commands

```bash
# Run only staging models
dbt run --select staging --profiles-dir .

# Run only intermediate layer
dbt run --select intermediate --profiles-dir .

# Run only marts
dbt run --select marts --profiles-dir .

# Test specific model
dbt test --select dim_listings --profiles-dir .
```

## dbt Model Examples

### Staging Model Pattern

**`models/staging/stg_airbnb__listings.sql`:**

```sql
{{ config(
    materialized='view',
    schema='staging'
) }}

with source as (
    select * from {{ source('airbnb_raw', 'listings') }}
),

cleaned as (
    select
        cast(id as integer) as listing_id,
        name as listing_name,
        cast(host_id as integer) as host_id,
        host_name,
        neighbourhood,
        cast(latitude as float) as latitude,
        cast(longitude as float) as longitude,
        room_type,
        cast(replace(price, '$', '') as float) as price,
        cast(minimum_nights as integer) as minimum_nights,
        cast(number_of_reviews as integer) as number_of_reviews,
        cast(last_review as date) as last_review_date,
        cast(reviews_per_month as float) as reviews_per_month,
        cast(calculated_host_listings_count as integer) as host_listing_count,
        cast(availability_365 as integer) as availability_365
    from source
    where id is not null
)

select * from cleaned
```

### Intermediate Model Pattern

**`models/intermediate/int_airbnb__listing_enriched.sql`:**

```sql
{{ config(
    materialized='view',
    schema='intermediate'
) }}

with listings as (
    select * from {{ ref('stg_airbnb__listings') }}
),

neighbourhoods as (
    select * from {{ ref('stg_airbnb__neighbourhoods') }}
),

enriched as (
    select
        l.listing_id,
        l.listing_name,
        l.host_id,
        l.host_name,
        l.host_listing_count,
        n.neighbourhood_group,
        l.neighbourhood,
        l.latitude,
        l.longitude,
        l.room_type,
        l.price,
        l.minimum_nights,
        l.number_of_reviews,
        l.last_review_date,
        l.reviews_per_month,
        l.availability_365,
        case 
            when l.host_listing_count > 1 then 'Multi-listing host'
            else 'Single-listing host'
        end as host_type
    from listings l
    left join neighbourhoods n
        on l.neighbourhood = n.neighbourhood
)

select * from enriched
```

### Incremental Fact Table

**`models/marts/fct_listing_calendar.sql`:**

```sql
{{ config(
    materialized='incremental',
    unique_key=['listing_id', 'calendar_date'],
    schema='marts',
    on_schema_change='fail'
) }}

with calendar_enriched as (
    select * from {{ ref('int_airbnb__calendar_enriched') }}
)

select
    listing_id,
    calendar_date,
    available,
    price,
    minimum_nights,
    maximum_nights,
    listing_name,
    neighbourhood,
    neighbourhood_group,
    room_type,
    host_id,
    host_name,
    case 
        when available = false then price
        else 0
    end as estimated_revenue
from calendar_enriched

{% if is_incremental() %}
    where calendar_date > (select max(calendar_date) from {{ this }})
{% endif %}
```

### Aggregate Model

**`models/marts/agg_neighbourhood_monthly_performance.sql`:**

```sql
{{ config(
    materialized='table',
    schema='marts'
) }}

with listing_monthly as (
    select * from {{ ref('agg_listing_monthly_performance') }}
),

neighbourhood_agg as (
    select
        neighbourhood_group,
        neighbourhood,
        month_year,
        count(distinct listing_id) as total_listings,
        sum(total_available_days) as total_available_days,
        sum(total_unavailable_days) as total_unavailable_days,
        round(avg(avg_price), 2) as avg_price,
        sum(estimated_monthly_revenue) as estimated_monthly_revenue,
        round(
            sum(total_unavailable_days)::float / 
            nullif(sum(total_available_days + total_unavailable_days), 0) * 100,
            2
        ) as occupancy_rate
    from listing_monthly
    group by neighbourhood_group, neighbourhood, month_year
)

select * from neighbourhood_agg
order by month_year, estimated_monthly_revenue desc
```

## dbt Tests

### Generic Tests in Schema YAML

**`models/staging/schema.yml`:**

```yaml
version: 2

sources:
  - name: airbnb_raw
    database: INSIDE_AIRBNB
    schema: raw
    tables:
      - name: listings
        columns:
          - name: id
            tests:
              - not_null
              - unique
      - name: calendar
        columns:
          - name: listing_id
            tests:
              - not_null
          - name: date
            tests:
              - not_null

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

### Singular Tests

**`tests/assert_no_duplicate_calendar_dates.sql`:**

```sql
-- Ensure no duplicate listing-date combinations in fact table
select
    listing_id,
    calendar_date,
    count(*) as record_count
from {{ ref('fct_listing_calendar') }}
group by listing_id, calendar_date
having count(*) > 1
```

## Streamlit Dashboard

### Running the Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

### Dashboard Code Pattern

**`dashboard/streamlit_app.py`:**

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load Snowflake credentials
with open('config/local_credentials.json', 'r') as f:
    creds = json.load(f)

# Create connection
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

conn = get_snowflake_connection()

# Query neighbourhood performance
@st.cache_data(ttl=600)
def get_neighbourhood_performance():
    query = """
    SELECT
        neighbourhood_group,
        neighbourhood,
        total_listings,
        avg_price,
        estimated_monthly_revenue,
        occupancy_rate
    FROM agg_neighbourhood_monthly_performance
    WHERE month_year = (SELECT MAX(month_year) FROM agg_neighbourhood_monthly_performance)
    ORDER BY estimated_monthly_revenue DESC
    LIMIT 20
    """
    return pd.read_sql(query, conn)

# Streamlit UI
st.title("Inside Airbnb Analytics Dashboard")
st.markdown("### Top 20 Neighbourhoods by Estimated Revenue")

df = get_neighbourhood_performance()
st.dataframe(df, use_container_width=True)

# Bar chart
st.bar_chart(df.set_index('neighbourhood')['estimated_monthly_revenue'])

# Query host performance
@st.cache_data(ttl=600)
def get_top_hosts():
    query = """
    SELECT
        host_id,
        host_name,
        total_listings,
        avg_price,
        total_reviews
    FROM dim_hosts
    ORDER BY total_listings DESC
    LIMIT 10
    """
    return pd.read_sql(query, conn)

st.markdown("### Top 10 Hosts by Listing Count")
hosts_df = get_top_hosts()
st.dataframe(hosts_df, use_container_width=True)
```

## Common Patterns

### Adding a New Staging Model

1. Create SQL file in `models/staging/`:

```sql
-- models/staging/stg_airbnb__new_source.sql
{{ config(materialized='view', schema='staging') }}

with source as (
    select * from {{ source('airbnb_raw', 'new_table') }}
),

cleaned as (
    select
        cast(id as integer) as record_id,
        clean_column_name,
        cast(date_column as date) as event_date
    from source
    where id is not null
)

select * from cleaned
```

2. Add to `models/staging/schema.yml`:

```yaml
  - name: stg_airbnb__new_source
    columns:
      - name: record_id
        tests:
          - not_null
          - unique
```

3. Run model:

```bash
dbt run --select stg_airbnb__new_source --profiles-dir .
dbt test --select stg_airbnb__new_source --profiles-dir .
```

### Creating a New Mart

```sql
-- models/marts/dim_new_dimension.sql
{{ config(materialized='table', schema='marts') }}

with staging as (
    select * from {{ ref('stg_airbnb__new_source') }}
),

intermediate as (
    select * from {{ ref('int_airbnb__enriched_version') }}
),

final as (
    select
        s.record_id,
        i.enriched_field,
        current_timestamp() as created_at
    from staging s
    left join intermediate i on s.record_id = i.record_id
)

select * from final
```

### Custom dbt Macro

**`macros/clean_price.sql`:**

```sql
{% macro clean_price(column_name) %}
    cast(replace(replace({{ column_name }}, '$', ''), ',', '') as float)
{% endmacro %}
```

**Usage in model:**

```sql
select
    listing_id,
    {{ clean_price('price') }} as cleaned_price
from {{ source('airbnb_raw', 'listings') }}
```

## Troubleshooting

### dbt Connection Issues

**Problem:** `dbt debug` fails with authentication error

**Solution:**
```bash
# Verify profiles.yml location
dbt debug --profiles-dir .

# Check Snowflake credentials in profiles.yml
# Ensure account format: abc12345.us-east-1 (not abc12345.snowflakecomputing.com)
```

### Incremental Model Not Updating

**Problem:** New data not appearing in `fct_listing_calendar`

**Solution:**
```bash
# Full refresh to rebuild from scratch
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Check incremental logic in model
# Verify max(calendar_date) predicate is correct
```

### Python Loader Fails

**Problem:** `PUT` command fails with file not found

**Solution:**
```python
# Use absolute paths
import os
file_path = os.path.abspath('data/raw/listings.csv.gz')
cursor.execute(f"PUT file://{file_path} @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=TRUE")
```

### Streamlit Connection Error

**Problem:** Dashboard can't connect to Snowflake

**Solution:**
```python
# Verify config/local_credentials.json exists and has correct values
# Check connection outside Streamlit:
import snowflake.connector
import json

with open('config/local_credentials.json') as f:
    creds = json.load(f)

conn = snowflake.connector.connect(**creds)
print(conn.cursor().execute("SELECT CURRENT_USER()").fetchone())
```

### Test Failures

**Problem:** `dbt test` shows relationship test failures

**Solution:**
```bash
# Run failed test in isolation to see data
dbt test --select test_name --profiles-dir .

# Check for null foreign keys
# Verify source data quality in RAW schema
```

### Schema Change Conflicts

**Problem:** Model fails with schema mismatch error

**Solution:**
```sql
-- Option 1: Allow schema evolution
{{ config(on_schema_change='append_new_columns') }}

-- Option 2: Force full refresh
{{ config(on_schema_change='fail') }}
```

```bash
# Then run with full refresh
dbt run --full-refresh --select model_name --profiles-dir .
```

## Environment Variables (Production Pattern)

For CI/CD or production, use environment variables instead of local files:

**`profiles.yml` with env vars:**

```yaml
snowflake_dbt_project:
  target: prod
  outputs:
    prod:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: "{{ env_var('SNOWFLAKE_ROLE') }}"
      database: "{{ env_var('SNOWFLAKE_DATABASE') }}"
      warehouse: "{{ env_var('SNOWFLAKE_WAREHOUSE') }}"
      schema: ANALYTICS
      threads: 4
```

**Set env vars:**

```bash
export SNOWFLAKE_ACCOUNT=abc12345.us-east-1
export SNOWFLAKE_USER=dbt_user
export SNOWFLAKE_PASSWORD=secret_password
export SNOWFLAKE_ROLE=TRANSFORMER
export SNOWFLAKE_DATABASE=INSIDE_AIRBNB
export SNOWFLAKE_WAREHOUSE=COMPUTE_WH

dbt run --profiles-dir .
```

This skill provides complete coverage for building Snowflake + dbt analytics pipelines with the Inside Airbnb dataset, including data loading, transformation layers, incremental modeling, testing, and dashboard deployment.
