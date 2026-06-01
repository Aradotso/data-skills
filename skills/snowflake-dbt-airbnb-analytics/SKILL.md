---
name: snowflake-dbt-airbnb-analytics
description: Build Snowflake + dbt analytics pipelines with Inside Airbnb data, marts, tests, and Streamlit dashboards
triggers:
  - how do I set up a Snowflake dbt project with Inside Airbnb data
  - load Inside Airbnb data into Snowflake with dbt transformations
  - create dbt staging intermediate and mart models for Airbnb analytics
  - build incremental fact tables in dbt with Snowflake merge
  - add dbt tests for data quality on Airbnb listings and calendar
  - connect a Streamlit dashboard to dbt marts in Snowflake
  - set up dbt profiles for Snowflake with local credentials
  - run dbt full refresh on incremental Airbnb models
---

# Snowflake dbt Airbnb Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end analytics engineering pipelines using Snowflake, dbt, and Inside Airbnb data. The project demonstrates raw data ingestion, multi-layer dbt transformations (staging → intermediate → marts), incremental modeling, data quality testing, and Streamlit dashboards.

## What This Project Does

- Loads Inside Airbnb CSV/GZIP files into Snowflake internal stages
- Creates raw schema with text-preserving tables
- Transforms data through dbt layers: staging (clean/cast), intermediate (joins/enrichment), marts (dimensions/facts/aggregates)
- Implements incremental fact modeling with Snowflake merge strategy
- Validates data quality with dbt generic and singular tests
- Powers a Streamlit dashboard reading from analytics marts

## Installation

```bash
# Clone and set up virtual environment
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Create local credential files (ignored by git)
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

Edit `profiles.yml` with your Snowflake connection:

```yaml
snowflake_dbt_project:
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT
      user: YOUR_USER
      password: YOUR_PASSWORD
      role: YOUR_ROLE
      warehouse: YOUR_WAREHOUSE
      database: AIRBNB_DB
      schema: ANALYTICS
      threads: 4
  target: dev
```

Edit `config/local_credentials.json` for Streamlit:

```json
{
  "account": "YOUR_ACCOUNT",
  "user": "YOUR_USER",
  "password": "YOUR_PASSWORD",
  "warehouse": "YOUR_WAREHOUSE",
  "database": "AIRBNB_DB",
  "schema": "ANALYTICS",
  "role": "YOUR_ROLE"
}
```

## Key Commands

### Load Raw Data into Snowflake

```bash
# Place Inside Airbnb CSV files in data/raw/
# listings.csv.gz, calendar.csv.gz, reviews.csv.gz, neighbourhoods.csv

python scripts/load_inside_airbnb_to_snowflake.py
```

This script:
1. Executes `setup/snowflake_setup.sql` to create database, schemas, stage
2. Uploads files to Snowflake internal stage `INSIDE_AIRBNB_STAGE`
3. Creates raw tables with header inference
4. Copies data into `RAW` schema

### Run dbt Models

```bash
# Test connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific layer
dbt run --select staging --profiles-dir .
dbt run --select intermediate --profiles-dir .
dbt run --select marts --profiles-dir .

# Run specific model and downstream
dbt run --select dim_listings+ --profiles-dir .

# Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Test data quality
dbt test --profiles-dir .

# Generate and serve documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

### Run Streamlit Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

## Project Structure

```
models/
├── staging/          # Clean, cast, standardize
│   ├── _sources.yml
│   ├── stg_airbnb__listings.sql
│   ├── stg_airbnb__calendar.sql
│   ├── stg_airbnb__reviews.sql
│   └── stg_airbnb__neighbourhoods.sql
├── intermediate/     # Joins, enrichment, revenue proxy
│   ├── int_airbnb__listing_enriched.sql
│   ├── int_airbnb__calendar_enriched.sql
│   └── int_airbnb__reviews_enriched.sql
└── marts/            # Dimensions, facts, aggregates
    ├── dim_listings.sql
    ├── dim_hosts.sql
    ├── fct_listing_calendar.sql (incremental)
    ├── fct_reviews.sql
    ├── agg_listing_monthly_performance.sql
    └── agg_neighbourhood_monthly_performance.sql
```

## dbt Model Patterns

### Staging Model Example

```sql
-- models/staging/stg_airbnb__listings.sql
with source as (
    select * from {{ source('raw_airbnb', 'listings') }}
),

cleaned as (
    select
        id::bigint as listing_id,
        name as listing_name,
        host_id::bigint as host_id,
        host_name,
        neighbourhood,
        room_type,
        price::text as price_text,
        minimum_nights::int as minimum_nights,
        number_of_reviews::int as number_of_reviews,
        last_review::date as last_review_date,
        reviews_per_month::float as reviews_per_month,
        availability_365::int as availability_365
    from source
)

select * from cleaned
```

### Intermediate Join Example

```sql
-- models/intermediate/int_airbnb__listing_enriched.sql
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
        l.room_type,
        l.neighbourhood as neighbourhood_name,
        n.neighbourhood_group,
        regexp_replace(l.price_text, '[^0-9.]', '')::float as price_cleaned,
        l.minimum_nights,
        l.number_of_reviews,
        l.last_review_date,
        l.reviews_per_month,
        l.availability_365
    from listings l
    left join neighbourhoods n
        on l.neighbourhood = n.neighbourhood
)

select * from enriched
```

### Incremental Fact Table

```sql
-- models/marts/fct_listing_calendar.sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        on_schema_change='sync_all_columns',
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
    neighbourhood_name,
    neighbourhood_group,
    room_type
from calendar_enriched

{% if is_incremental() %}
where calendar_date > (select max(calendar_date) from {{ this }})
{% endif %}
```

### Aggregate Mart Example

```sql
-- models/marts/agg_neighbourhood_monthly_performance.sql
with listing_monthly as (
    select * from {{ ref('agg_listing_monthly_performance') }}
),

neighbourhood_agg as (
    select
        neighbourhood_name,
        neighbourhood_group,
        year_month,
        count(distinct listing_id) as total_listings,
        sum(total_unavailable_nights) as total_unavailable_nights,
        sum(estimated_revenue) as total_estimated_revenue,
        avg(avg_price) as avg_neighbourhood_price,
        avg(availability_rate) as avg_availability_rate
    from listing_monthly
    group by 1, 2, 3
)

select * from neighbourhood_agg
order by year_month desc, total_estimated_revenue desc
```

## dbt Testing Patterns

### Generic Tests in schema.yml

```yaml
# models/staging/_schema.yml
version: 2

models:
  - name: stg_airbnb__listings
    columns:
      - name: listing_id
        tests:
          - unique
          - not_null
      - name: room_type
        tests:
          - accepted_values:
              values: ['Entire home/apt', 'Private room', 'Shared room', 'Hotel room']
      - name: price_cleaned
        tests:
          - dbt_utils.not_negative
```

### Singular Test Example

```sql
-- tests/assert_no_duplicate_listing_dates.sql
select
    listing_id,
    calendar_date,
    count(*) as occurrence_count
from {{ ref('fct_listing_calendar') }}
group by 1, 2
having count(*) > 1
```

## Streamlit Dashboard Pattern

```python
# dashboard/streamlit_app.py
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load credentials from ignored local file
with open('config/local_credentials.json', 'r') as f:
    creds = json.load(f)

conn = snowflake.connector.connect(
    account=creds['account'],
    user=creds['user'],
    password=creds['password'],
    warehouse=creds['warehouse'],
    database=creds['database'],
    schema=creds['schema'],
    role=creds['role']
)

st.title("Airbnb Analytics Dashboard")

# Query dbt mart
query = """
SELECT
    neighbourhood_name,
    total_estimated_revenue,
    avg_neighbourhood_price,
    avg_availability_rate
FROM agg_neighbourhood_monthly_performance
WHERE year_month = (SELECT MAX(year_month) FROM agg_neighbourhood_monthly_performance)
ORDER BY total_estimated_revenue DESC
LIMIT 10
"""

df = pd.read_sql(query, conn)
st.dataframe(df)
st.bar_chart(df.set_index('neighbourhood_name')['total_estimated_revenue'])

conn.close()
```

## Common Workflows

### Initial Setup and First Run

```bash
# 1. Download Inside Airbnb data for NYC
# Place in data/raw/

# 2. Load to Snowflake
python scripts/load_inside_airbnb_to_snowflake.py

# 3. Run dbt
dbt run --profiles-dir .
dbt test --profiles-dir .
dbt docs generate --profiles-dir .

# 4. Launch dashboard
streamlit run dashboard/streamlit_app.py
```

### Refresh with New Data Snapshot

```bash
# 1. Replace files in data/raw/
# 2. Re-load to Snowflake
python scripts/load_inside_airbnb_to_snowflake.py

# 3. Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 4. Run tests
dbt test --profiles-dir .
```

### Develop New Mart Model

```bash
# 1. Create SQL file in models/marts/
# 2. Run just that model
dbt run --select my_new_mart --profiles-dir .

# 3. Add tests in _schema.yml
# 4. Test
dbt test --select my_new_mart --profiles-dir .

# 5. Check lineage
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

## Configuration Best Practices

### dbt_project.yml

```yaml
name: 'snowflake_dbt_project'
version: '1.0.0'

profile: 'snowflake_dbt_project'

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
macro-paths: ["macros"]
docs-paths: ["docs"]

target-path: "target"
clean-targets:
  - "target"
  - "dbt_packages"
  - "logs"

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
      +schema: marts
```

### Source Definition

```yaml
# models/staging/_sources.yml
version: 2

sources:
  - name: raw_airbnb
    database: AIRBNB_DB
    schema: RAW
    tables:
      - name: listings
        columns:
          - name: id
            tests:
              - unique
              - not_null
      - name: calendar
        columns:
          - name: listing_id
            tests:
              - not_null
      - name: reviews
      - name: neighbourhoods
```

## Troubleshooting

### dbt Connection Fails

```bash
# Test connection
dbt debug --profiles-dir .

# Common issues:
# - Wrong account format (use org-account format)
# - Incorrect warehouse/role permissions
# - profiles.yml indentation errors
```

### Incremental Model Not Updating

```bash
# Force full refresh
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Check incremental logic
dbt run --select fct_listing_calendar --profiles-dir . --debug
```

### Snowflake Stage Upload Fails

```python
# In load_inside_airbnb_to_snowflake.py
# Verify file paths exist
import os
data_dir = "data/raw"
files = ["listings.csv.gz", "calendar.csv.gz", "reviews.csv.gz", "neighbourhoods.csv"]
for f in files:
    path = os.path.join(data_dir, f)
    if not os.path.exists(path):
        print(f"Missing: {path}")
```

### Dashboard Connection Error

```python
# Verify credentials file exists and is valid JSON
import json
with open('config/local_credentials.json', 'r') as f:
    creds = json.load(f)
    print(creds.keys())  # Should have account, user, password, etc.
```

### Test Failures on Room Types

```bash
# Check actual room_type values in source
dbt run-operation print_source_values --args '{source: raw_airbnb, table: listings, column: room_type}'

# Update accepted_values test in _schema.yml accordingly
```

## Advanced Patterns

### Custom Generic Test Macro

```sql
-- macros/test_revenue_not_negative.sql
{% macro test_revenue_not_negative(model, column_name) %}

select *
from {{ model }}
where {{ column_name }} < 0

{% endmacro %}
```

### dbt Exposures for Dashboard

```yaml
# models/marts/_exposures.yml
version: 2

exposures:
  - name: streamlit_airbnb_dashboard
    type: dashboard
    maturity: high
    owner:
      name: Durgesh Yadav
      email: durgesh@example.com
    depends_on:
      - ref('dim_listings')
      - ref('dim_hosts')
      - ref('agg_neighbourhood_monthly_performance')
    description: >
      Streamlit dashboard showing neighbourhood performance,
      host analytics, and listing trends.
```

### Environment-Specific Targets

```yaml
# profiles.yml
snowflake_dbt_project:
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      database: AIRBNB_DB
      schema: DEV
    prod:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      database: AIRBNB_DB
      schema: ANALYTICS
  target: dev
```

```bash
# Run against prod
dbt run --target prod --profiles-dir .
```

This skill equips AI agents to guide developers through building production-ready Snowflake + dbt analytics pipelines with proper layering, incremental modeling, testing, and dashboard integration.
