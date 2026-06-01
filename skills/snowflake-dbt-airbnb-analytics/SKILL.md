---
name: snowflake-dbt-airbnb-analytics
description: Build analytics pipelines with Snowflake, dbt, and Inside Airbnb data including staging, marts, tests, and Streamlit dashboards
triggers:
  - set up snowflake dbt project with inside airbnb
  - create dbt models for airbnb data warehouse
  - load inside airbnb data to snowflake
  - build dbt staging and mart models
  - configure dbt incremental models in snowflake
  - create streamlit dashboard for dbt marts
  - test dbt models with data quality checks
  - structure analytics engineering project with dbt
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to work with a complete analytics engineering project that loads Inside Airbnb open data into Snowflake, transforms it with dbt into staging/intermediate/mart layers, validates data quality, and powers a Streamlit dashboard.

## What This Project Does

- Loads Inside Airbnb CSV files (listings, calendar, reviews, neighbourhoods) into Snowflake RAW schema
- Transforms raw data through dbt staging → intermediate → marts layers
- Implements incremental fact tables with Snowflake merge strategy
- Validates data quality with dbt generic and singular tests
- Generates dbt documentation
- Provides Streamlit dashboard for neighbourhood, host, and listing analytics

## Installation

```bash
# Clone and set up environment
git clone https://github.com/analyticsdurgesh/Snowflake_DBT_Project.git
cd Snowflake_DBT_Project
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

## Configuration

### 1. Snowflake Credentials (profiles.yml)

Create `profiles.yml` from the example:

```bash
cp profiles.yml.example profiles.yml
```

Edit `profiles.yml`:

```yaml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT
      user: YOUR_USER
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: YOUR_ROLE
      database: AIRBNB_DB
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

### 2. Loader Credentials (local_credentials.json)

Create `config/local_credentials.json` from the example:

```bash
cp config/local_credentials.example.json config/local_credentials.json
```

Edit `config/local_credentials.json`:

```json
{
  "snowflake": {
    "account": "YOUR_ACCOUNT",
    "user": "YOUR_USER",
    "password": "YOUR_PASSWORD",
    "warehouse": "COMPUTE_WH",
    "database": "AIRBNB_DB",
    "schema": "RAW",
    "role": "YOUR_ROLE"
  }
}
```

### 3. Environment Variables (Alternative)

```bash
export SNOWFLAKE_PASSWORD="your_password"
export SNOWFLAKE_ACCOUNT="your_account"
export SNOWFLAKE_USER="your_user"
```

## Data Preparation

Download Inside Airbnb data for New York City from [insideairbnb.com/get-the-data](http://insideairbnb.com/get-the-data/):

```bash
mkdir -p data/raw
# Place downloaded files:
# data/raw/listings.csv.gz
# data/raw/calendar.csv.gz
# data/raw/reviews.csv.gz
# data/raw/neighbourhoods.csv
```

## Key Commands

### Load Raw Data to Snowflake

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

This script:
1. Creates Snowflake database, schemas (RAW, ANALYTICS), and internal stage
2. Uploads local CSV/GZIP files to Snowflake stage
3. Creates raw tables from CSV headers
4. Copies data into RAW schema tables

### dbt Workflow

```bash
# Verify dbt connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select dim_listings+ --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Full refresh (rebuild incremental models)
dbt run --full-refresh --profiles-dir .

# Full refresh specific model
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .

# Serve documentation site
dbt docs serve --profiles-dir .
```

### Streamlit Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

## dbt Model Structure

### Staging Layer (`models/staging/`)

Clean and standardize raw data:

**stg_airbnb__listings.sql**
```sql
{{
    config(
        materialized='view'
    )
}}

with source as (
    select * from {{ source('airbnb_raw', 'listings') }}
),

renamed as (
    select
        id::bigint as listing_id,
        name as listing_name,
        host_id::bigint as host_id,
        host_name,
        neighbourhood_cleansed as neighbourhood,
        latitude::float as latitude,
        longitude::float as longitude,
        room_type,
        price,
        minimum_nights::int as minimum_nights,
        number_of_reviews::int as number_of_reviews,
        last_review::date as last_review_date,
        reviews_per_month::float as reviews_per_month,
        calculated_host_listings_count::int as host_listings_count,
        availability_365::int as availability_365
    from source
)

select * from renamed
```

**stg_airbnb__calendar.sql**
```sql
{{
    config(
        materialized='view'
    )
}}

with source as (
    select * from {{ source('airbnb_raw', 'calendar') }}
),

cleaned as (
    select
        listing_id::bigint as listing_id,
        date::date as calendar_date,
        available = 't' as is_available,
        -- Remove $ and commas from price string
        nullif(
            replace(replace(price, '$', ''), ',', ''),
            ''
        )::float as price,
        minimum_nights::int as minimum_nights,
        maximum_nights::int as maximum_nights
    from source
)

select * from cleaned
```

### Intermediate Layer (`models/intermediate/`)

Join and enrich staging models:

**int_airbnb__calendar_enriched.sql**
```sql
{{
    config(
        materialized='ephemeral'
    )
}}

with calendar as (
    select * from {{ ref('stg_airbnb__calendar') }}
),

listings as (
    select * from {{ ref('int_airbnb__listing_enriched') }}
),

joined as (
    select
        calendar.*,
        listings.neighbourhood,
        listings.room_type,
        -- Revenue proxy: if unavailable, assume booked
        case
            when not calendar.is_available
                then coalesce(calendar.price, listings.listing_price, 0)
            else 0
        end as estimated_revenue
    from calendar
    left join listings
        on calendar.listing_id = listings.listing_id
)

select * from joined
```

### Marts Layer (`models/marts/`)

Analytics-ready dimensions and facts:

**dim_listings.sql**
```sql
{{
    config(
        materialized='table'
    )
}}

with listings_enriched as (
    select * from {{ ref('int_airbnb__listing_enriched') }}
)

select
    listing_id,
    listing_name,
    host_id,
    host_name,
    neighbourhood,
    neighbourhood_group,
    latitude,
    longitude,
    room_type,
    listing_price,
    minimum_nights,
    number_of_reviews,
    last_review_date,
    reviews_per_month,
    host_listings_count,
    availability_365
from listings_enriched
```

**fct_listing_calendar.sql (Incremental)**
```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        merge_update_columns=['is_available', 'price', 'estimated_revenue']
    )
}}

with calendar_enriched as (
    select * from {{ ref('int_airbnb__calendar_enriched') }}
)

select
    listing_id,
    calendar_date,
    is_available,
    price,
    minimum_nights,
    maximum_nights,
    neighbourhood,
    room_type,
    estimated_revenue
from calendar_enriched

{% if is_incremental() %}
    where calendar_date > (select max(calendar_date) from {{ this }})
{% endif %}
```

**agg_listing_monthly_performance.sql**
```sql
{{
    config(
        materialized='table'
    )
}}

with calendar_fact as (
    select * from {{ ref('fct_listing_calendar') }}
),

listings as (
    select * from {{ ref('dim_listings') }}
),

monthly_agg as (
    select
        calendar_fact.listing_id,
        date_trunc('month', calendar_date) as month_date,
        count(*) as total_days,
        sum(case when is_available then 1 else 0 end) as available_days,
        sum(case when not is_available then 1 else 0 end) as unavailable_days,
        avg(price) as avg_price,
        sum(estimated_revenue) as total_estimated_revenue
    from calendar_fact
    group by 1, 2
)

select
    monthly_agg.*,
    listings.listing_name,
    listings.host_id,
    listings.host_name,
    listings.neighbourhood,
    listings.room_type,
    round(
        (unavailable_days::float / nullif(total_days, 0)) * 100,
        2
    ) as occupancy_rate_pct
from monthly_agg
left join listings
    on monthly_agg.listing_id = listings.listing_id
```

## Source Configuration

**models/staging/sources.yml**
```yaml
version: 2

sources:
  - name: airbnb_raw
    database: AIRBNB_DB
    schema: RAW
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
      
      - name: reviews
        columns:
          - name: listing_id
            tests:
              - not_null
          - name: id
            tests:
              - not_null
              - unique
      
      - name: neighbourhoods
```

## Data Quality Tests

### Generic Tests (in model YAML)

**models/marts/schema.yml**
```yaml
version: 2

models:
  - name: dim_listings
    description: "Dimension table for Airbnb listings"
    columns:
      - name: listing_id
        description: "Unique listing identifier"
        tests:
          - not_null
          - unique
      
      - name: room_type
        description: "Type of room"
        tests:
          - accepted_values:
              values: ['Entire home/apt', 'Private room', 'Shared room', 'Hotel room']
      
      - name: listing_price
        description: "Listing price in USD"
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
  
  - name: fct_listing_calendar
    description: "Daily calendar facts for listings"
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
```

### Singular Tests (in tests/)

**tests/no_duplicate_listing_dates.sql**
```sql
-- Check for duplicate listing-date combinations
select
    listing_id,
    calendar_date,
    count(*) as record_count
from {{ ref('fct_listing_calendar') }}
group by listing_id, calendar_date
having count(*) > 1
```

**tests/valid_revenue_calculation.sql**
```sql
-- Verify estimated revenue is only non-zero when unavailable
select
    listing_id,
    calendar_date,
    is_available,
    estimated_revenue
from {{ ref('fct_listing_calendar') }}
where is_available = true
    and estimated_revenue > 0
```

## Streamlit Dashboard

**dashboard/streamlit_app.py**
```python
import streamlit as st
import pandas as pd
import snowflake.connector
import json
from pathlib import Path

# Load credentials
creds_path = Path(__file__).parent.parent / 'config' / 'local_credentials.json'
with open(creds_path) as f:
    creds = json.load(f)['snowflake']

# Snowflake connection
@st.cache_resource
def get_connection():
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema='ANALYTICS',
        role=creds['role']
    )

conn = get_connection()

st.title("🏠 Inside Airbnb Analytics Dashboard")

# Top neighbourhoods by revenue
st.header("Top 10 Neighbourhoods by Estimated Revenue")
query = """
select
    neighbourhood,
    sum(total_estimated_revenue) as total_revenue,
    avg(occupancy_rate_pct) as avg_occupancy_rate,
    count(distinct listing_id) as listing_count
from agg_neighbourhood_monthly_performance
group by neighbourhood
order by total_revenue desc
limit 10
"""
df = pd.read_sql(query, conn)
st.bar_chart(df.set_index('NEIGHBOURHOOD')['TOTAL_REVENUE'])
st.dataframe(df)

# Room type analysis
st.header("Room Type Performance")
query = """
select
    room_type,
    count(distinct listing_id) as listing_count,
    avg(avg_price) as avg_price,
    sum(total_estimated_revenue) as total_revenue
from agg_listing_monthly_performance
group by room_type
order by total_revenue desc
"""
df_rooms = pd.read_sql(query, conn)
st.dataframe(df_rooms)

# Top hosts
st.header("Top 10 Hosts by Listing Count")
query = """
select
    host_id,
    host_name,
    count(distinct listing_id) as listing_count,
    sum(total_estimated_revenue) as total_revenue
from agg_listing_monthly_performance
group by host_id, host_name
order by listing_count desc
limit 10
"""
df_hosts = pd.read_sql(query, conn)
st.dataframe(df_hosts)
```

## Common Patterns

### Full Data Refresh Workflow

When loading a new Inside Airbnb snapshot:

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

### Selective Model Execution

```bash
# Run only staging models
dbt run --select staging.* --profiles-dir .

# Run a model and all downstream dependencies
dbt run --select dim_listings+ --profiles-dir .

# Run a model and all upstream dependencies
dbt run --select +fct_listing_calendar --profiles-dir .

# Run models matching a tag
dbt run --select tag:monthly --profiles-dir .
```

### Adding a New Mart

1. Create model file `models/marts/your_mart.sql`
2. Add model documentation in `models/marts/schema.yml`
3. Add tests in the YAML or create singular test in `tests/`
4. Run: `dbt run --select your_mart --profiles-dir .`
5. Test: `dbt test --select your_mart --profiles-dir .`

### Custom dbt Macro

**macros/cents_to_dollars.sql**
```sql
{% macro cents_to_dollars(column_name) %}
    ({{ column_name }} / 100.0)
{% endmacro %}
```

Use in model:
```sql
select
    listing_id,
    {{ cents_to_dollars('price_cents') }} as price_usd
from {{ ref('stg_prices') }}
```

## Troubleshooting

### Connection Issues

**Problem**: `dbt debug` fails with "Could not connect to Snowflake"

**Solution**:
```bash
# Verify profiles.yml is in project root
ls -la profiles.yml

# Test Snowflake connection directly
python -c "
import snowflake.connector
conn = snowflake.connector.connect(
    account='YOUR_ACCOUNT',
    user='YOUR_USER',
    password='YOUR_PASSWORD',
    warehouse='COMPUTE_WH'
)
print('Connected!')
conn.close()
"
```

### Incremental Model Not Updating

**Problem**: `fct_listing_calendar` doesn't update with new data

**Solution**:
```bash
# Check if incremental filter is working
dbt run --select fct_listing_calendar --profiles-dir . --debug

# Force full refresh
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Verify max date in target table
dbt run-operation run_query --args "sql: 'select max(calendar_date) from {{ ref(\"fct_listing_calendar\") }}'" --profiles-dir .
```

### Test Failures

**Problem**: Relationship test fails between `fct_listing_calendar` and `dim_listings`

**Solution**:
```sql
-- Run query to find orphaned records
select distinct calendar.listing_id
from analytics.fct_listing_calendar as calendar
left join analytics.dim_listings as listings
    on calendar.listing_id = listings.listing_id
where listings.listing_id is null;
```

### Data Loading Errors

**Problem**: Python loader fails to upload files

**Solution**:
```bash
# Verify file paths
ls -lh data/raw/*.csv* 

# Check Snowflake stage
python -c "
import snowflake.connector
import json
with open('config/local_credentials.json') as f:
    c = json.load(f)['snowflake']
conn = snowflake.connector.connect(**c)
cursor = conn.cursor()
cursor.execute('LIST @INSIDE_AIRBNB_STAGE')
print(cursor.fetchall())
"

# Manually remove stage files if needed
# Then re-run loader
```

### Dashboard Connection Errors

**Problem**: Streamlit can't connect to Snowflake

**Solution**:
```python
# Add error handling to dashboard
try:
    conn = snowflake.connector.connect(**creds)
    st.success("Connected to Snowflake")
except Exception as e:
    st.error(f"Connection failed: {e}")
    st.stop()
```

## Project Structure Reference

```
.
├── config/
│   ├── local_credentials.json          # Snowflake credentials (gitignored)
│   └── local_credentials.example.json
├── dashboard/
│   └── streamlit_app.py               # Streamlit dashboard
├── data/raw/                          # Inside Airbnb CSV files
│   ├── listings.csv.gz
│   ├── calendar.csv.gz
│   ├── reviews.csv.gz
│   └── neighbourhoods.csv
├── models/
│   ├── staging/                       # stg_airbnb__* views
│   ├── intermediate/                  # int_airbnb__* ephemeral
│   └── marts/                         # dim_*, fct_*, agg_* tables
├── scripts/
│   └── load_inside_airbnb_to_snowflake.py
├── setup/
│   └── snowflake_setup.sql           # DDL for database/schemas
├── tests/                            # Singular dbt tests
├── dbt_project.yml                   # dbt project config
├── profiles.yml                      # dbt Snowflake connection (gitignored)
├── profiles.yml.example
└── requirements.txt
```

## Key dbt Project Configuration

**dbt_project.yml**
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
      +materialized: ephemeral
    marts:
      +materialized: table
      +schema: analytics
```

This skill provides everything needed to work with Snowflake, dbt, and Inside Airbnb data in a production-ready analytics engineering project.
