---
name: snowflake-dbt-airbnb-analytics
description: Analytics engineering project that loads Inside Airbnb data into Snowflake, transforms with dbt, and powers a Streamlit dashboard
triggers:
  - build an airbnb analytics project with dbt and snowflake
  - set up inside airbnb data warehouse with dbt
  - create snowflake dbt pipeline for airbnb data
  - load inside airbnb data into snowflake with dbt
  - build analytics dashboard for airbnb listings
  - implement dbt staging and marts for airbnb data
  - create incremental dbt models in snowflake
  - set up dbt project with snowflake for analytics
---

# Snowflake dbt Airbnb Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A complete analytics engineering project that demonstrates loading Inside Airbnb open data into Snowflake, transforming it through dbt staging/intermediate/mart layers, implementing data quality tests, and serving analytics via a Streamlit dashboard.

## What This Project Does

- Loads Inside Airbnb CSV files (listings, calendar, reviews, neighbourhoods) into Snowflake
- Uses Python script to upload to Snowflake internal stage and create raw tables
- Transforms raw data through dbt layers: staging → intermediate → marts
- Implements incremental fact models with Snowflake merge strategy
- Includes generic and singular dbt tests for data quality
- Provides Streamlit dashboard for neighbourhood, host, and listing analytics
- Uses revenue proxy logic (unavailable nights × price) as classroom-friendly substitute for bookings

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

### 1. Set Up Local Credentials

Create local credential files from examples (these are git-ignored):

```bash
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

### 2. Configure profiles.yml

Edit `profiles.yml` with your Snowflake credentials:

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

### 3. Configure local_credentials.json

Edit `config/local_credentials.json` for the Python loader and Streamlit:

```json
{
  "account": "YOUR_ACCOUNT",
  "user": "YOUR_USER",
  "password": "YOUR_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DB",
  "schema": "RAW",
  "role": "YOUR_ROLE"
}
```

**Security Note**: Never commit these files. Use environment variables in production:

```python
import os
password = os.getenv('SNOWFLAKE_PASSWORD')
```

## Data Preparation

Download Inside Airbnb data for your city (e.g., New York City) from [insideairbnb.com/get-the-data](https://insideairbnb.com/get-the-data/).

Place files in `data/raw/`:
- `listings.csv.gz`
- `calendar.csv.gz`
- `reviews.csv.gz`
- `neighbourhoods.csv`

## Loading Raw Data into Snowflake

Run the Python loader script:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

This script:
1. Reads credentials from `config/local_credentials.json`
2. Executes `setup/snowflake_setup.sql` to create database, schemas, and stage
3. Uploads local files to Snowflake internal stage `INSIDE_AIRBNB_STAGE`
4. Creates raw tables with inferred schema from CSV headers
5. Copies data from stage into `RAW` schema tables

### Loader Script Structure

```python
import snowflake.connector
import json
from pathlib import Path

# Load credentials
with open('config/local_credentials.json', 'r') as f:
    creds = json.load(f)

# Connect to Snowflake
conn = snowflake.connector.connect(**creds)
cursor = conn.cursor()

# Execute setup SQL
with open('setup/snowflake_setup.sql', 'r') as f:
    setup_sql = f.read()
    for statement in setup_sql.split(';'):
        if statement.strip():
            cursor.execute(statement)

# Upload files to stage
data_dir = Path('data/raw')
for file_path in data_dir.glob('*'):
    cursor.execute(f"""
        PUT file://{file_path} @INSIDE_AIRBNB_STAGE
        AUTO_COMPRESS=FALSE OVERWRITE=TRUE
    """)

# Create and load tables
cursor.execute("""
    CREATE OR REPLACE TABLE RAW.LISTINGS (
        id TEXT,
        name TEXT,
        host_id TEXT,
        host_name TEXT,
        neighbourhood TEXT,
        latitude TEXT,
        longitude TEXT,
        room_type TEXT,
        price TEXT,
        minimum_nights TEXT,
        availability_365 TEXT
    )
""")

cursor.execute("""
    COPY INTO RAW.LISTINGS
    FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
    FILE_FORMAT = (TYPE=CSV SKIP_HEADER=1 FIELD_OPTIONALLY_ENCLOSED_BY='"')
""")
```

## dbt Project Structure

### dbt Layers

```
models/
├── staging/              # Clean, cast, standardize
│   ├── stg_airbnb__listings.sql
│   ├── stg_airbnb__calendar.sql
│   ├── stg_airbnb__reviews.sql
│   └── stg_airbnb__neighbourhoods.sql
├── intermediate/         # Joins, enrichment, business logic
│   ├── int_airbnb__listing_enriched.sql
│   ├── int_airbnb__calendar_enriched.sql
│   └── int_airbnb__reviews_enriched.sql
└── marts/               # Analytics-ready tables
    ├── dim_listings.sql
    ├── dim_hosts.sql
    ├── fct_listing_calendar.sql  # Incremental
    ├── fct_reviews.sql
    ├── agg_listing_monthly_performance.sql
    └── agg_neighbourhood_monthly_performance.sql
```

### Key dbt Commands

```bash
# Verify connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream
dbt run --select fct_listing_calendar+ --profiles-dir .

# Full refresh for incremental models
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

## dbt Model Examples

### Staging Model: stg_airbnb__listings.sql

```sql
with source as (
    select * from {{ source('airbnb', 'listings') }}
),

cleaned as (
    select
        id::bigint as listing_id,
        name as listing_name,
        host_id::bigint as host_id,
        host_name,
        neighbourhood,
        latitude::float as latitude,
        longitude::float as longitude,
        room_type,
        replace(price, '$', '')::float as price,
        minimum_nights::int as minimum_nights,
        availability_365::int as availability_365
    from source
    where id is not null
)

select * from cleaned
```

### Intermediate Model: int_airbnb__listing_enriched.sql

```sql
with listings as (
    select * from {{ ref('stg_airbnb__listings') }}
),

neighbourhoods as (
    select * from {{ ref('stg_airbnb__neighbourhoods') }}
),

enriched as (
    select
        l.*,
        n.neighbourhood_group,
        case
            when l.price < 50 then 'Budget'
            when l.price < 150 then 'Mid-range'
            else 'Luxury'
        end as price_category
    from listings l
    left join neighbourhoods n
        on l.neighbourhood = n.neighbourhood
)

select * from enriched
```

### Incremental Fact Model: fct_listing_calendar.sql

```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        merge_update_columns=['available', 'price', 'adjusted_price', 'minimum_nights', 'maximum_nights']
    )
}}

with calendar as (
    select * from {{ ref('int_airbnb__calendar_enriched') }}
),

final as (
    select
        listing_id,
        calendar_date,
        available,
        price,
        adjusted_price,
        minimum_nights,
        maximum_nights,
        case
            when available = 'f' then price
            else 0
        end as estimated_revenue
    from calendar
    {% if is_incremental() %}
        where calendar_date > (select max(calendar_date) from {{ this }})
    {% endif %}
)

select * from final
```

### Aggregate Model: agg_listing_monthly_performance.sql

```sql
with calendar_facts as (
    select * from {{ ref('fct_listing_calendar') }}
),

monthly_metrics as (
    select
        listing_id,
        date_trunc('month', calendar_date) as year_month,
        count(*) as total_days,
        sum(case when available = 'f' then 1 else 0 end) as unavailable_days,
        sum(case when available = 't' then 1 else 0 end) as available_days,
        avg(price) as avg_price,
        sum(estimated_revenue) as total_estimated_revenue,
        round(unavailable_days::float / total_days * 100, 2) as occupancy_rate_pct
    from calendar_facts
    group by listing_id, year_month
)

select * from monthly_metrics
```

## dbt Testing

### Generic Tests in schema.yml

```yaml
version: 2

sources:
  - name: airbnb
    database: AIRBNB_DB
    schema: RAW
    tables:
      - name: listings
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

  - name: fct_listing_calendar
    tests:
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns:
            - listing_id
            - calendar_date
```

### Singular Test: tests/no_negative_revenue.sql

```sql
-- Check that estimated revenue is never negative
select
    listing_id,
    calendar_date,
    estimated_revenue
from {{ ref('fct_listing_calendar') }}
where estimated_revenue < 0
```

## Streamlit Dashboard

### Running the Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

### Dashboard Code Example

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
        schema='ANALYTICS',
        role=creds['role']
    )

conn = get_connection()

# Query data
@st.cache_data(ttl=600)
def load_neighbourhood_metrics():
    query = """
        SELECT
            neighbourhood_group,
            COUNT(DISTINCT listing_id) as total_listings,
            AVG(avg_price) as avg_price,
            SUM(total_estimated_revenue) as total_revenue,
            AVG(occupancy_rate_pct) as avg_occupancy
        FROM ANALYTICS.AGG_NEIGHBOURHOOD_MONTHLY_PERFORMANCE
        GROUP BY neighbourhood_group
        ORDER BY total_revenue DESC
    """
    return pd.read_sql(query, conn)

# Display
st.title("🏠 Inside Airbnb Analytics Dashboard")
st.header("Neighbourhood Performance")

df = load_neighbourhood_metrics()
st.dataframe(df)

# Chart
st.bar_chart(df.set_index('neighbourhood_group')['total_revenue'])
```

## Common Patterns

### Full Data Refresh Workflow

When you download a new Inside Airbnb snapshot:

```bash
# 1. Place new files in data/raw/
# 2. Reload raw data
python scripts/load_inside_airbnb_to_snowflake.py

# 3. Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 4. Run tests
dbt test --profiles-dir .

# 5. Regenerate docs
dbt docs generate --profiles-dir .
```

### Custom Macro Example

Create reusable SQL logic in `macros/`:

```sql
-- macros/calculate_occupancy_rate.sql
{% macro calculate_occupancy_rate(unavailable_col, total_col) %}
    round(
        {{ unavailable_col }}::float / nullif({{ total_col }}, 0) * 100,
        2
    )
{% endmacro %}
```

Usage in models:

```sql
select
    listing_id,
    {{ calculate_occupancy_rate('unavailable_days', 'total_days') }} as occupancy_rate_pct
from calendar_summary
```

### Snapshot Example

Track slowly changing dimensions:

```sql
-- snapshots/listings_snapshot.sql
{% snapshot listings_snapshot %}
    {{
        config(
            target_schema='SNAPSHOTS',
            unique_key='listing_id',
            strategy='timestamp',
            updated_at='updated_at'
        )
    }}
    select * from {{ ref('stg_airbnb__listings') }}
{% endsnapshot %}
```

Run with:

```bash
dbt snapshot --profiles-dir .
```

## Troubleshooting

### Connection Issues

**Problem**: `dbt debug` fails with authentication error

**Solution**: Check `profiles.yml` credentials match Snowflake account. Verify role has access to database:

```sql
USE ROLE YOUR_ROLE;
SHOW DATABASES;
```

### Loader Script Fails

**Problem**: Python script cannot find files

**Solution**: Verify data files exist in `data/raw/`:

```bash
ls -la data/raw/
```

**Problem**: Stage upload fails

**Solution**: Check Snowflake permissions:

```sql
SHOW GRANTS ON STAGE INSIDE_AIRBNB_STAGE;
GRANT READ, WRITE ON STAGE INSIDE_AIRBNB_STAGE TO ROLE YOUR_ROLE;
```

### Incremental Model Not Updating

**Problem**: New calendar data not appearing in `fct_listing_calendar`

**Solution**: Full refresh the model:

```bash
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

### Test Failures

**Problem**: Unique test fails on `fct_listing_calendar`

**Solution**: Check for duplicate listing-date combinations:

```sql
SELECT
    listing_id,
    calendar_date,
    COUNT(*) as cnt
FROM ANALYTICS.FCT_LISTING_CALENDAR
GROUP BY listing_id, calendar_date
HAVING COUNT(*) > 1;
```

### Streamlit Dashboard Empty

**Problem**: Dashboard shows no data

**Solution**: Verify schema in credentials matches dbt target:

```python
# Should be 'ANALYTICS', not 'RAW'
creds['schema'] = 'ANALYTICS'
```

### Memory Issues with Large Files

**Problem**: Python loader runs out of memory

**Solution**: Use Snowflake external stage with S3/Azure instead of local PUT:

```sql
CREATE STAGE external_stage
    URL='s3://your-bucket/airbnb-data/'
    CREDENTIALS=(AWS_KEY_ID='...' AWS_SECRET_KEY='...');

COPY INTO RAW.LISTINGS
FROM @external_stage/listings.csv.gz
FILE_FORMAT = (TYPE=CSV SKIP_HEADER=1);
```

## Advanced Usage

### Custom Source Freshness

Add to `models/staging/sources.yml`:

```yaml
sources:
  - name: airbnb
    database: AIRBNB_DB
    schema: RAW
    freshness:
      warn_after: {count: 7, period: day}
      error_after: {count: 14, period: day}
    loaded_at_field: _loaded_at
    tables:
      - name: listings
```

Check freshness:

```bash
dbt source freshness --profiles-dir .
```

### Exposures for Downstream Tools

Document dashboard dependencies in `models/marts/exposures.yml`:

```yaml
version: 2

exposures:
  - name: neighbourhood_dashboard
    type: dashboard
    maturity: high
    url: http://localhost:8501
    description: Streamlit dashboard showing neighbourhood metrics
    depends_on:
      - ref('dim_listings')
      - ref('agg_neighbourhood_monthly_performance')
    owner:
      name: Durgesh Yadav
      email: durgesh@example.com
```

### CI/CD Integration

Example GitHub Actions workflow:

```yaml
name: dbt CI
on: [pull_request]
jobs:
  dbt-test:
    runs-on: ubuntu-latest
    env:
      SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-python@v2
        with:
          python-version: '3.9'
      - run: pip install dbt-snowflake
      - run: dbt deps --profiles-dir .
      - run: dbt run --profiles-dir .
      - run: dbt test --profiles-dir .
```

This skill provides comprehensive guidance for working with the Snowflake dbt Airbnb Analytics project, covering installation, configuration, data loading, dbt development, testing, dashboard deployment, and troubleshooting.
