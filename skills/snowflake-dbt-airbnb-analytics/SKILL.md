---
name: snowflake-dbt-airbnb-analytics
description: Build end-to-end analytics pipelines with Snowflake, dbt, and Streamlit using the Inside Airbnb dataset
triggers:
  - load inside airbnb data into snowflake
  - create dbt models for airbnb data
  - build snowflake analytics with dbt
  - set up airbnb analytics pipeline
  - configure dbt with snowflake for airbnb
  - create streamlit dashboard for airbnb data
  - run dbt incremental models on snowflake
  - test data quality in dbt airbnb project
---

# Snowflake dbt Airbnb Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a complete analytics engineering workflow: loading Inside Airbnb open data into Snowflake, transforming it with dbt through staging/intermediate/mart layers, validating data quality with tests, and visualizing results in a Streamlit dashboard.

## What This Project Does

- Loads Inside Airbnb CSV/GZIP files into Snowflake internal stages
- Creates raw, staging, intermediate, and mart dbt model layers
- Implements incremental fact tables with Snowflake merge strategy
- Runs generic and singular dbt tests for data quality
- Powers a Streamlit dashboard connected to analytics marts
- Demonstrates public-safe credential management with ignored local config files

## Dataset

Inside Airbnb provides open data for cities worldwide. The project expects:

- `data/raw/listings.csv.gz` — property listings
- `data/raw/calendar.csv.gz` — availability and pricing by date
- `data/raw/reviews.csv.gz` — guest reviews
- `data/raw/neighbourhoods.csv` — neighbourhood boundaries

Download from [Inside Airbnb](https://insideairbnb.com/get-the-data/). Recommended classroom dataset: New York City, New York, United States.

## Installation

```bash
# Clone the repository
git clone https://github.com/analyticsdurgesh/Snowflake_DBT_Project.git
cd Snowflake_DBT_Project

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

## Configuration

The project uses local-only credential files that are `.gitignore`d for public safety.

### 1. Create profiles.yml

```bash
cp profiles.yml.example profiles.yml
```

Edit `profiles.yml`:

```yaml
snowflake_dbt_project:
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT
      user: YOUR_USERNAME
      password: YOUR_PASSWORD
      role: YOUR_ROLE
      database: AIRBNB_DEV
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
  target: dev
```

### 2. Create local_credentials.json

```bash
cp config/local_credentials.example.json config/local_credentials.json
```

Edit `config/local_credentials.json`:

```json
{
  "account": "YOUR_ACCOUNT",
  "user": "YOUR_USERNAME",
  "password": "YOUR_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DEV",
  "schema": "ANALYTICS",
  "role": "YOUR_ROLE"
}
```

**Never commit these files.** They are in `.gitignore`.

## Loading Raw Data

The Python loader script handles the entire Snowflake setup:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

This script:
1. Reads `config/local_credentials.json`
2. Executes `setup/snowflake_setup.sql` to create database, schemas, stage
3. Uploads local CSV/GZIP files to Snowflake internal stage
4. Creates raw tables from CSV headers
5. Copies staged data into raw tables

### Raw Layer Tables

```sql
-- Created in RAW schema
RAW.LISTINGS
RAW.CALENDAR
RAW.REVIEWS
RAW.NEIGHBOURHOODS
```

## dbt Project Structure

```
models/
├── staging/
│   ├── _staging_airbnb__sources.yml
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

## Key dbt Commands

```bash
# Test connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select stg_airbnb__listings+ --profiles-dir .

# Run incremental models with full refresh
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Test specific model
dbt test --select stg_airbnb__listings --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .

# Compile without running
dbt compile --profiles-dir .
```

## Model Layer Patterns

### Staging Models

Clean and standardize raw data:

```sql
-- models/staging/stg_airbnb__listings.sql
{{
    config(
        materialized='view'
    )
}}

with source as (
    select * from {{ source('airbnb', 'listings') }}
),

renamed as (
    select
        id::bigint as listing_id,
        name::varchar as listing_name,
        host_id::bigint as host_id,
        host_name::varchar as host_name,
        neighbourhood_cleansed::varchar as neighbourhood,
        room_type::varchar as room_type,
        price::varchar as price_raw,
        -- Remove $ and , then cast to number
        replace(replace(price, '$', ''), ',', '')::decimal(10,2) as price,
        minimum_nights::int as minimum_nights,
        number_of_reviews::int as number_of_reviews,
        last_review::date as last_review_date,
        reviews_per_month::decimal(10,2) as reviews_per_month,
        calculated_host_listings_count::int as host_listings_count,
        availability_365::int as availability_365
    from source
)

select * from renamed
```

### Intermediate Models

Join and enrich data:

```sql
-- models/intermediate/int_airbnb__listing_enriched.sql
{{
    config(
        materialized='view'
    )
}}

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
        l.neighbourhood,
        n.neighbourhood_group,
        l.room_type,
        l.price,
        l.minimum_nights,
        l.number_of_reviews,
        l.last_review_date,
        l.reviews_per_month,
        l.host_listings_count,
        l.availability_365,
        -- Derived fields
        case
            when l.number_of_reviews = 0 then 'No Reviews'
            when l.number_of_reviews < 10 then 'Few Reviews'
            when l.number_of_reviews < 50 then 'Some Reviews'
            else 'Many Reviews'
        end as review_category
    from listings l
    left join neighbourhoods n
        on l.neighbourhood = n.neighbourhood
)

select * from enriched
```

### Incremental Fact Models

Use Snowflake merge for efficient updates:

```sql
-- models/marts/fct_listing_calendar.sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        merge_update_columns=['available', 'price', 'minimum_nights', 'maximum_nights']
    )
}}

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
    adjusted_price,
    listing_name,
    neighbourhood,
    neighbourhood_group,
    room_type,
    -- Revenue proxy: estimated revenue if unavailable
    case
        when available = false then adjusted_price
        else 0
    end as estimated_revenue
from calendar_enriched

{% if is_incremental() %}
    where calendar_date > (select max(calendar_date) from {{ this }})
{% endif %}
```

### Aggregate Models

Pre-compute reporting metrics:

```sql
-- models/marts/agg_listing_monthly_performance.sql
{{
    config(
        materialized='table'
    )
}}

with calendar_facts as (
    select * from {{ ref('fct_listing_calendar') }}
),

monthly_agg as (
    select
        listing_id,
        listing_name,
        neighbourhood,
        neighbourhood_group,
        room_type,
        date_trunc('month', calendar_date)::date as month_date,
        count(*) as total_days,
        sum(case when available = false then 1 else 0 end) as unavailable_days,
        sum(case when available = true then 1 else 0 end) as available_days,
        avg(price) as avg_price,
        sum(estimated_revenue) as total_estimated_revenue,
        -- Availability rate
        sum(case when available = true then 1 else 0 end)::decimal / count(*)::decimal as availability_rate
    from calendar_facts
    group by 1, 2, 3, 4, 5, 6
)

select * from monthly_agg
```

## Data Quality Tests

### Generic Tests

Defined in schema YAML files:

```yaml
# models/staging/_staging_airbnb__models.yml
version: 2

models:
  - name: stg_airbnb__listings
    description: Cleaned and standardized Airbnb listings
    columns:
      - name: listing_id
        description: Unique listing identifier
        tests:
          - unique
          - not_null
      - name: price
        description: Nightly price in USD
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
      - name: room_type
        description: Type of room
        tests:
          - accepted_values:
              values: ['Entire home/apt', 'Private room', 'Shared room', 'Hotel room']
```

### Singular Tests

Custom SQL tests in `tests/`:

```sql
-- tests/assert_no_duplicate_listing_dates.sql
-- Ensure no listing appears twice on the same date in calendar facts

select
    listing_id,
    calendar_date,
    count(*) as occurrences
from {{ ref('fct_listing_calendar') }}
group by listing_id, calendar_date
having count(*) > 1
```

Run all tests:

```bash
dbt test --profiles-dir .
```

## Streamlit Dashboard

Run the interactive dashboard:

```bash
streamlit run dashboard/streamlit_app.py
```

The dashboard reads credentials from `config/local_credentials.json` and connects to Snowflake marts.

### Example Dashboard Code

```python
import streamlit as st
import snowflake.connector
import pandas as pd
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
    database=creds['database'],
    schema=creds['schema'],
    role=creds['role']
)

st.title('Airbnb Analytics Dashboard')

# Neighbourhood performance
query = """
select
    neighbourhood_group,
    neighbourhood,
    sum(total_estimated_revenue) as revenue,
    avg(availability_rate) as avg_availability,
    count(distinct listing_id) as listing_count
from agg_neighbourhood_monthly_performance
group by 1, 2
order by revenue desc
limit 20
"""

df = pd.read_sql(query, conn)
st.subheader('Top Neighbourhoods by Estimated Revenue')
st.dataframe(df)

conn.close()
```

## Common Workflows

### Initial Setup

```bash
# 1. Download Inside Airbnb data to data/raw/
# 2. Configure credentials
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
# Edit both files with your Snowflake credentials

# 3. Load raw data
python scripts/load_inside_airbnb_to_snowflake.py

# 4. Run dbt pipeline
dbt run --profiles-dir .
dbt test --profiles-dir .

# 5. Launch dashboard
streamlit run dashboard/streamlit_app.py
```

### Refresh with New Data Snapshot

```bash
# 1. Download new Inside Airbnb snapshot
# 2. Reload raw data
python scripts/load_inside_airbnb_to_snowflake.py

# 3. Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 4. Run dependent models
dbt run --select agg_listing_monthly_performance+ --profiles-dir .

# 5. Test data quality
dbt test --profiles-dir .
```

### Debug a Model

```bash
# Compile SQL to see what dbt generates
dbt compile --select dim_listings --profiles-dir .
cat target/compiled/snowflake_dbt_project/models/marts/dim_listings.sql

# Run one model
dbt run --select dim_listings --profiles-dir .

# Run model and all downstream
dbt run --select dim_listings+ --profiles-dir .

# Run model and all upstream
dbt run --select +dim_listings --profiles-dir .
```

## Troubleshooting

### Connection Issues

```bash
# Test dbt connection
dbt debug --profiles-dir .
```

If connection fails:
- Verify `profiles.yml` values match your Snowflake account
- Check warehouse is running in Snowflake UI
- Confirm user has necessary privileges
- Test credentials in `config/local_credentials.json`

### Python Loader Errors

```python
# Common error: FileNotFoundError
# Ensure data files exist:
ls -lh data/raw/
# Expected: listings.csv.gz, calendar.csv.gz, reviews.csv.gz, neighbourhoods.csv

# Common error: snowflake.connector.errors.ProgrammingError
# Check credentials in config/local_credentials.json
# Verify warehouse and role exist
```

### Incremental Model Not Updating

```bash
# Force full refresh of incremental model
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Check for unique_key violations
dbt test --select fct_listing_calendar --profiles-dir .
```

### Test Failures

```bash
# Run specific test to see failing rows
dbt test --select stg_airbnb__listings --profiles-dir .

# Check compiled test SQL
cat target/compiled/snowflake_dbt_project/models/staging/_staging_airbnb__models.yml/unique_stg_airbnb__listings_listing_id.sql

# Run compiled test directly in Snowflake to debug
```

### Streamlit Connection Issues

```python
# Verify credentials file exists and has correct values
import json
with open('config/local_credentials.json') as f:
    print(json.load(f))

# Test Snowflake connection manually
import snowflake.connector
conn = snowflake.connector.connect(
    account='YOUR_ACCOUNT',
    user='YOUR_USER',
    password='YOUR_PASSWORD',
    warehouse='COMPUTE_WH',
    database='AIRBNB_DEV',
    schema='ANALYTICS',
    role='YOUR_ROLE'
)
print(conn.cursor().execute("SELECT CURRENT_USER()").fetchone())
```

## Macros and Custom Logic

### Revenue Proxy Logic

Inside Airbnb doesn't provide booking transactions. This project uses calendar availability as a proxy:

```sql
-- If a listing is unavailable, assume it's booked and count the price as revenue
case
    when available = false then adjusted_price
    else 0
end as estimated_revenue
```

### dbt Macros

Example custom macro:

```sql
-- macros/cents_to_dollars.sql
{% macro cents_to_dollars(column_name) %}
    ({{ column_name }} / 100.0)::decimal(10,2)
{% endmacro %}

-- Usage in model:
select {{ cents_to_dollars('price_cents') }} as price_dollars
```

## Best Practices

1. **Never commit credentials** — Use `.gitignore` for `profiles.yml`, `config/local_credentials.json`
2. **Use incremental models** for large fact tables to reduce compute
3. **Test data quality** after every dbt run
4. **Document models** in schema YAML files
5. **Modularize transformations** — staging → intermediate → marts
6. **Use refs, not direct table names** — `{{ ref('stg_airbnb__listings') }}`
7. **Tag models** for selective runs:

```yaml
# dbt_project.yml
models:
  snowflake_dbt_project:
    marts:
      +tags: ["marts", "analytics"]
```

```bash
# Run only marts
dbt run --select tag:marts --profiles-dir .
```

## Example Business Analytics Queries

### Top Revenue Neighbourhoods

```sql
select
    neighbourhood_group,
    neighbourhood,
    sum(total_estimated_revenue) as total_revenue,
    avg(availability_rate) as avg_availability
from {{ ref('agg_neighbourhood_monthly_performance') }}
group by 1, 2
order by total_revenue desc
limit 10;
```

### Host Performance

```sql
select
    host_id,
    host_name,
    count(distinct listing_id) as listing_count,
    avg(price) as avg_price,
    avg(number_of_reviews) as avg_reviews
from {{ ref('dim_listings') }}
group by 1, 2
having count(distinct listing_id) > 5
order by listing_count desc;
```

### Availability Trends by Month

```sql
select
    month_date,
    avg(availability_rate) as avg_availability,
    sum(total_estimated_revenue) as total_revenue
from {{ ref('agg_listing_monthly_performance') }}
group by 1
order by 1;
```

This skill enables AI agents to help developers build, debug, and extend Snowflake + dbt analytics pipelines using the Inside Airbnb dataset as a portfolio-ready example.
