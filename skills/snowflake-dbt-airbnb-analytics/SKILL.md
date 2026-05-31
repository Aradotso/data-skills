---
name: snowflake-dbt-airbnb-analytics
description: Analytics engineering project using Snowflake, dbt, and Streamlit to transform Inside Airbnb data into marts with data quality tests
triggers:
  - set up dbt with Snowflake for Airbnb analytics
  - load Inside Airbnb data into Snowflake
  - create dbt staging and mart models for Airbnb data
  - build incremental fact tables with dbt
  - configure dbt profiles for Snowflake connection
  - run dbt tests for data quality validation
  - build a Streamlit dashboard on Snowflake data
  - transform raw Airbnb data with dbt layers
---

# Snowflake dbt Airbnb Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a complete analytics engineering pipeline using Snowflake as the data warehouse, dbt for transformation, and Streamlit for visualization. It ingests Inside Airbnb open data (listings, calendar, reviews, neighbourhoods) and builds a layered data model with staging, intermediate, and mart layers following dimensional modeling best practices.

## What It Does

- Loads CSV/GZIP files from Inside Airbnb into Snowflake internal stages
- Creates raw tables preserving original text formats
- Transforms data through dbt staging → intermediate → marts layers
- Builds incremental fact tables using Snowflake merge strategy
- Implements data quality tests (generic and singular)
- Powers a Streamlit dashboard with neighbourhood and host analytics
- Uses calendar availability as a proxy for revenue (since booking data is not public)

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

**Expected dependencies** (create `requirements.txt` if missing):
```
dbt-snowflake>=1.5.0
snowflake-connector-python>=3.0.0
streamlit>=1.20.0
pandas>=1.5.0
```

## Configuration

### 1. Snowflake Credentials

Create `profiles.yml` in the project root (ignored by git):

```yaml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: "{{ env_var('SNOWFLAKE_ROLE', 'ANALYST') }}"
      warehouse: "{{ env_var('SNOWFLAKE_WAREHOUSE', 'COMPUTE_WH') }}"
      database: "{{ env_var('SNOWFLAKE_DATABASE', 'AIRBNB_DB') }}"
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

Or use local file reference (for development):

```yaml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: your_account.region
      user: your_username
      password: your_password
      role: ANALYST
      warehouse: COMPUTE_WH
      database: AIRBNB_DB
      schema: ANALYTICS
      threads: 4
```

### 2. Streamlit Credentials

Create `config/local_credentials.json` (ignored by git):

```json
{
  "account": "your_account.region",
  "user": "your_username",
  "password": "your_password",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DB",
  "schema": "ANALYTICS",
  "role": "ANALYST"
}
```

For production, use environment variables:

```python
# In dashboard/streamlit_app.py
import os
import json

if os.path.exists("config/local_credentials.json"):
    with open("config/local_credentials.json") as f:
        creds = json.load(f)
else:
    creds = {
        "account": os.environ["SNOWFLAKE_ACCOUNT"],
        "user": os.environ["SNOWFLAKE_USER"],
        "password": os.environ["SNOWFLAKE_PASSWORD"],
        "warehouse": os.environ.get("SNOWFLAKE_WAREHOUSE", "COMPUTE_WH"),
        "database": os.environ.get("SNOWFLAKE_DATABASE", "AIRBNB_DB"),
        "schema": os.environ.get("SNOWFLAKE_SCHEMA", "ANALYTICS"),
        "role": os.environ.get("SNOWFLAKE_ROLE", "ANALYST")
    }
```

## Data Loading

### 1. Download Inside Airbnb Data

Download files for New York City from [Inside Airbnb](https://insideairbnb.com/get-the-data/):

```
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

### 2. Load to Snowflake

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

This script:
- Executes `setup/snowflake_setup.sql` to create database, schemas, stage
- Uploads files to Snowflake internal stage
- Creates raw tables with inferred schemas
- Copies data from stage to raw tables

**Key parts of the loader script**:

```python
import snowflake.connector
import json
import os

# Load credentials
with open("config/local_credentials.json") as f:
    creds = json.load(f)

conn = snowflake.connector.connect(
    account=creds["account"],
    user=creds["user"],
    password=creds["password"],
    warehouse=creds["warehouse"],
    role=creds["role"]
)

cursor = conn.cursor()

# Execute setup SQL
with open("setup/snowflake_setup.sql") as f:
    for statement in f.read().split(";"):
        if statement.strip():
            cursor.execute(statement)

# Upload files to stage
cursor.execute("USE DATABASE AIRBNB_DB")
cursor.execute("USE SCHEMA RAW")

files = [
    "data/raw/listings.csv.gz",
    "data/raw/calendar.csv.gz",
    "data/raw/reviews.csv.gz",
    "data/raw/neighbourhoods.csv"
]

for file_path in files:
    cursor.execute(f"PUT file://{file_path} @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")

# Create and load raw tables
cursor.execute("""
CREATE OR REPLACE TABLE RAW.LISTINGS (
    id VARCHAR,
    listing_url VARCHAR,
    name VARCHAR,
    host_id VARCHAR,
    host_name VARCHAR,
    neighbourhood_cleansed VARCHAR,
    latitude VARCHAR,
    longitude VARCHAR,
    room_type VARCHAR,
    price VARCHAR,
    minimum_nights VARCHAR,
    number_of_reviews VARCHAR,
    last_review VARCHAR,
    reviews_per_month VARCHAR,
    availability_365 VARCHAR
)
""")

cursor.execute("""
COPY INTO RAW.LISTINGS
FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
ON_ERROR = 'CONTINUE'
""")

cursor.close()
conn.close()
```

## dbt Model Layers

### Project Structure

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
    ├── fct_listing_calendar.sql
    ├── fct_reviews.sql
    ├── agg_listing_monthly_performance.sql
    └── agg_neighbourhood_monthly_performance.sql
```

### Staging Layer Example

**models/staging/_sources.yml**:

```yaml
version: 2

sources:
  - name: raw_airbnb
    database: AIRBNB_DB
    schema: RAW
    tables:
      - name: LISTINGS
        columns:
          - name: id
            tests:
              - not_null
              - unique
      - name: CALENDAR
        columns:
          - name: listing_id
            tests:
              - not_null
          - name: date
            tests:
              - not_null
      - name: REVIEWS
      - name: NEIGHBOURHOODS
```

**models/staging/stg_airbnb__listings.sql**:

```sql
with source as (
    select * from {{ source('raw_airbnb', 'LISTINGS') }}
),

cleaned as (
    select
        cast(id as integer) as listing_id,
        listing_url,
        name as listing_name,
        cast(host_id as integer) as host_id,
        host_name,
        neighbourhood_cleansed as neighbourhood,
        cast(latitude as float) as latitude,
        cast(longitude as float) as longitude,
        room_type,
        cast(replace(replace(price, '$', ''), ',', '') as float) as price,
        cast(minimum_nights as integer) as minimum_nights,
        cast(number_of_reviews as integer) as number_of_reviews,
        to_date(last_review, 'YYYY-MM-DD') as last_review_date,
        cast(reviews_per_month as float) as reviews_per_month,
        cast(availability_365 as integer) as availability_365
    from source
)

select * from cleaned
```

**models/staging/stg_airbnb__calendar.sql**:

```sql
with source as (
    select * from {{ source('raw_airbnb', 'CALENDAR') }}
),

cleaned as (
    select
        cast(listing_id as integer) as listing_id,
        to_date(date, 'YYYY-MM-DD') as calendar_date,
        case 
            when lower(available) = 't' then true 
            when lower(available) = 'f' then false
            else null
        end as is_available,
        cast(replace(replace(price, '$', ''), ',', '') as float) as price,
        cast(minimum_nights as integer) as minimum_nights,
        cast(maximum_nights as integer) as maximum_nights
    from source
)

select * from cleaned
```

### Intermediate Layer Example

**models/intermediate/int_airbnb__calendar_enriched.sql**:

```sql
with calendar as (
    select * from {{ ref('stg_airbnb__calendar') }}
),

listings as (
    select * from {{ ref('int_airbnb__listing_enriched') }}
),

enriched as (
    select
        c.listing_id,
        c.calendar_date,
        c.is_available,
        c.price as calendar_price,
        c.minimum_nights,
        c.maximum_nights,
        l.listing_name,
        l.neighbourhood,
        l.room_type,
        l.host_id,
        l.host_name,
        -- Revenue proxy: if not available, assume it's booked
        case 
            when c.is_available = false then c.price 
            else 0 
        end as estimated_revenue
    from calendar c
    inner join listings l on c.listing_id = l.listing_id
)

select * from enriched
```

### Marts Layer - Incremental Fact Table

**models/marts/fct_listing_calendar.sql**:

```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        on_schema_change='fail'
    )
}}

with calendar_enriched as (
    select * from {{ ref('int_airbnb__calendar_enriched') }}
)

select
    listing_id,
    calendar_date,
    is_available,
    calendar_price,
    minimum_nights,
    maximum_nights,
    neighbourhood,
    room_type,
    host_id,
    estimated_revenue,
    current_timestamp() as dbt_loaded_at
from calendar_enriched

{% if is_incremental() %}
    where calendar_date > (select max(calendar_date) from {{ this }})
{% endif %}
```

**models/marts/dim_listings.sql**:

```sql
with listings as (
    select * from {{ ref('int_airbnb__listing_enriched') }}
)

select
    listing_id,
    listing_name,
    listing_url,
    host_id,
    host_name,
    neighbourhood,
    latitude,
    longitude,
    room_type,
    price,
    minimum_nights,
    number_of_reviews,
    last_review_date,
    reviews_per_month,
    availability_365,
    current_timestamp() as dbt_loaded_at
from listings
```

**models/marts/agg_neighbourhood_monthly_performance.sql**:

```sql
with listing_performance as (
    select * from {{ ref('agg_listing_monthly_performance') }}
)

select
    neighbourhood,
    year_month,
    count(distinct listing_id) as listing_count,
    sum(total_available_days) as total_available_days,
    sum(total_unavailable_days) as total_unavailable_days,
    avg(avg_price) as avg_price,
    sum(estimated_monthly_revenue) as total_estimated_revenue,
    avg(availability_rate) as avg_availability_rate
from listing_performance
group by neighbourhood, year_month
```

## dbt Tests

**models/marts/schema.yml**:

```yaml
version: 2

models:
  - name: dim_listings
    description: Dimension table for Airbnb listings
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

  - name: fct_listing_calendar
    description: Fact table for listing daily calendar and availability
    tests:
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns:
            - listing_id
            - calendar_date
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
      - name: estimated_revenue
        tests:
          - dbt_utils.expression_is_true:
              expression: ">= 0"
```

**Singular test** - `tests/assert_no_duplicate_listing_dates.sql`:

```sql
-- Ensure no duplicate listing_id + calendar_date combinations in fact table
select
    listing_id,
    calendar_date,
    count(*) as record_count
from {{ ref('fct_listing_calendar') }}
group by listing_id, calendar_date
having count(*) > 1
```

## dbt Commands

```bash
# Verify connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select fct_listing_calendar+ --profiles-dir .

# Full refresh of incremental models
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Test specific model
dbt test --select dim_listings --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .

# Serve documentation
dbt docs serve --profiles-dir .

# Compile without running
dbt compile --profiles-dir .

# Build (run + test)
dbt build --profiles-dir .
```

## Streamlit Dashboard

**dashboard/streamlit_app.py** (simplified example):

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json
import os

# Load credentials
if os.path.exists("config/local_credentials.json"):
    with open("config/local_credentials.json") as f:
        creds = json.load(f)
else:
    creds = {
        "account": os.environ["SNOWFLAKE_ACCOUNT"],
        "user": os.environ["SNOWFLAKE_USER"],
        "password": os.environ["SNOWFLAKE_PASSWORD"],
        "warehouse": os.environ.get("SNOWFLAKE_WAREHOUSE", "COMPUTE_WH"),
        "database": os.environ.get("SNOWFLAKE_DATABASE", "AIRBNB_DB"),
        "schema": os.environ.get("SNOWFLAKE_SCHEMA", "ANALYTICS"),
        "role": os.environ.get("SNOWFLAKE_ROLE", "ANALYST")
    }

@st.cache_resource
def get_connection():
    return snowflake.connector.connect(
        account=creds["account"],
        user=creds["user"],
        password=creds["password"],
        warehouse=creds["warehouse"],
        database=creds["database"],
        schema=creds["schema"],
        role=creds["role"]
    )

@st.cache_data
def run_query(query):
    conn = get_connection()
    return pd.read_sql(query, conn)

st.title("Inside Airbnb Analytics Dashboard")

# Neighbourhood performance
st.header("Top Neighbourhoods by Estimated Revenue")

query = """
SELECT 
    neighbourhood,
    SUM(total_estimated_revenue) as total_revenue,
    AVG(avg_availability_rate) as avg_availability,
    COUNT(DISTINCT listing_id) as listing_count
FROM agg_neighbourhood_monthly_performance
GROUP BY neighbourhood
ORDER BY total_revenue DESC
LIMIT 10
"""

df = run_query(query)
st.dataframe(df)
st.bar_chart(df.set_index("NEIGHBOURHOOD")["TOTAL_REVENUE"])

# Room type analysis
st.header("Average Price by Room Type")

query = """
SELECT 
    room_type,
    AVG(price) as avg_price,
    COUNT(*) as listing_count
FROM dim_listings
GROUP BY room_type
ORDER BY avg_price DESC
"""

df_rooms = run_query(query)
st.dataframe(df_rooms)

# Host rankings
st.header("Top Hosts by Number of Listings")

query = """
SELECT 
    host_id,
    host_name,
    COUNT(*) as listing_count,
    AVG(price) as avg_price
FROM dim_listings
GROUP BY host_id, host_name
ORDER BY listing_count DESC
LIMIT 10
"""

df_hosts = run_query(query)
st.dataframe(df_hosts)
```

Run the dashboard:

```bash
streamlit run dashboard/streamlit_app.py
```

## Common Patterns

### Adding a New Staging Model

1. Define source in `models/staging/_sources.yml`
2. Create staging model SQL file
3. Add tests in schema.yml

```sql
-- models/staging/stg_airbnb__new_source.sql
with source as (
    select * from {{ source('raw_airbnb', 'NEW_TABLE') }}
),

cleaned as (
    select
        cast(id as integer) as record_id,
        cast(created_at as timestamp) as created_at,
        value
    from source
)

select * from cleaned
```

### Creating a New Aggregate

```sql
-- models/marts/agg_daily_summary.sql
with facts as (
    select * from {{ ref('fct_listing_calendar') }}
)

select
    calendar_date,
    count(distinct listing_id) as total_listings,
    sum(case when is_available = false then 1 else 0 end) as booked_listings,
    sum(estimated_revenue) as total_revenue,
    avg(calendar_price) as avg_price
from facts
group by calendar_date
order by calendar_date
```

### Using dbt Macros

**macros/cents_to_dollars.sql**:

```sql
{% macro cents_to_dollars(column_name) %}
    ({{ column_name }} / 100.0)
{% endmacro %}
```

**Usage in model**:

```sql
select
    listing_id,
    {{ cents_to_dollars('price_cents') }} as price_dollars
from source
```

### Full Pipeline Refresh

```bash
# Re-load raw data
python scripts/load_inside_airbnb_to_snowflake.py

# Full refresh all incremental models
dbt run --full-refresh --profiles-dir .

# Run all tests
dbt test --profiles-dir .

# Regenerate docs
dbt docs generate --profiles-dir .
```

## Troubleshooting

### Connection Issues

```bash
# Test Snowflake connection
dbt debug --profiles-dir .

# Common fixes:
# - Verify SNOWFLAKE_ACCOUNT format (account.region)
# - Check warehouse is running
# - Verify role has database access
# - Ensure user/password are correct
```

### Incremental Model Not Updating

```bash
# Force full refresh
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Check for unique_key violations
dbt test --select fct_listing_calendar --profiles-dir .
```

### Test Failures

```bash
# Run specific test
dbt test --select dim_listings --profiles-dir .

# View compiled test SQL
dbt compile --select test_name --profiles-dir .
# Check target/compiled/project_name/tests/
```

### Data Loading Errors

Check Snowflake load history:

```sql
SELECT * 
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME=>'LISTINGS',
    START_TIME=>DATEADD(hours, -1, CURRENT_TIMESTAMP())
));
```

### Streamlit Connection Timeout

```python
# Add connection pooling
import snowflake.connector
from snowflake.connector import DictCursor

conn = snowflake.connector.connect(
    account=creds["account"],
    user=creds["user"],
    password=creds["password"],
    warehouse=creds["warehouse"],
    database=creds["database"],
    schema=creds["schema"],
    role=creds["role"],
    client_session_keep_alive=True,  # Keep connection alive
    login_timeout=30,
    network_timeout=30
)
```

### dbt Compilation Errors

```bash
# Check for syntax errors
dbt compile --profiles-dir .

# View compiled SQL
cat target/compiled/snowflake_dbt_project/models/marts/dim_listings.sql

# Parse project structure
dbt parse --profiles-dir .
```

## Key Environment Variables

```bash
export SNOWFLAKE_ACCOUNT="your_account.region"
export SNOWFLAKE_USER="your_username"
export SNOWFLAKE_PASSWORD="your_password"
export SNOWFLAKE_ROLE="ANALYST"
export SNOWFLAKE_WAREHOUSE="COMPUTE_WH"
export SNOWFLAKE_DATABASE="AIRBNB_DB"
export SNOWFLAKE_SCHEMA="ANALYTICS"
```

## Additional Resources

- [dbt Snowflake adapter docs](https://docs.getdbt.com/reference/warehouse-setups/snowflake-setup)
- [Inside Airbnb data dictionary](http://insideairbnb.com/data-dictionary.html)
- [dbt best practices](https://docs.getdbt.com/guides/best-practices)
- [Snowflake COPY INTO](https://docs.snowflake.com/en/sql-reference/sql/copy-into-table.html)
