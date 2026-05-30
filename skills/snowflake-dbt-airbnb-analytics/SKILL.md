---
name: snowflake-dbt-airbnb-analytics
description: Skill for building Snowflake + dbt analytics pipelines with Inside Airbnb data, incremental models, tests, and Streamlit dashboards
triggers:
  - how do I set up a dbt project with Snowflake
  - load Inside Airbnb data into Snowflake
  - create dbt staging and mart models
  - build incremental fact tables in dbt
  - configure dbt profiles for Snowflake
  - create a Streamlit dashboard connected to Snowflake
  - write dbt tests for data quality
  - build analytics engineering pipeline with dbt
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end analytics engineering pipelines using Snowflake as the data warehouse, dbt for transformation, Inside Airbnb as sample data, and Streamlit for dashboards.

## What This Project Does

- **Data ingestion**: Load Inside Airbnb CSV/GZIP files into Snowflake internal stages and raw tables
- **dbt transformation**: Multi-layer dbt project (staging → intermediate → marts) with incremental models
- **Data quality**: Generic and singular dbt tests for validation
- **Analytics**: Dimensional models (dim_listings, dim_hosts) and fact tables (fct_listing_calendar, fct_reviews)
- **Reporting**: Streamlit dashboard connected to analytics marts
- **Security**: Local-only credential files ignored by git

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

### Step 1: Set Up Local Credentials (Not Committed)

```bash
# Copy example files
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

### Step 2: Edit `profiles.yml`

```yaml
airbnb_analytics:
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
  target: dev
```

### Step 3: Edit `config/local_credentials.json`

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

**Important**: Never commit these files. They are in `.gitignore`.

## Data Loading

### Download Inside Airbnb Data

Visit [Inside Airbnb](https://insideairbnb.com/get-the-data/) and download NYC files:

```text
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

### Load to Snowflake

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

This script:
1. Creates Snowflake database, schemas, and internal stage
2. Uploads local files to `INSIDE_AIRBNB_STAGE`
3. Creates raw tables with correct schema
4. Copies data from stage to `RAW` schema tables

### Example: Custom Loader Script

```python
import snowflake.connector
import json
import os
from pathlib import Path

def load_credentials():
    with open('config/local_credentials.json') as f:
        return json.load(f)

def connect_snowflake():
    creds = load_credentials()
    return snowflake.connector.connect(
        user=creds['user'],
        password=creds['password'],
        account=creds['account'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema='RAW'
    )

def upload_file_to_stage(cursor, local_path, stage_name):
    """Upload a local file to Snowflake internal stage"""
    put_sql = f"PUT file://{local_path} @{stage_name} AUTO_COMPRESS=FALSE OVERWRITE=TRUE"
    cursor.execute(put_sql)
    print(f"Uploaded {local_path} to stage {stage_name}")

def copy_into_table(cursor, stage_file, table_name):
    """Copy from stage to table"""
    copy_sql = f"""
    COPY INTO {table_name}
    FROM @INSIDE_AIRBNB_STAGE/{stage_file}
    FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
    ON_ERROR = 'CONTINUE'
    """
    cursor.execute(copy_sql)
    print(f"Copied data into {table_name}")

# Example usage
conn = connect_snowflake()
cursor = conn.cursor()
upload_file_to_stage(cursor, 'data/raw/listings.csv.gz', 'INSIDE_AIRBNB_STAGE')
copy_into_table(cursor, 'listings.csv.gz', 'RAW.LISTINGS')
cursor.close()
conn.close()
```

## dbt Commands

### Basic Workflow

```bash
# Test connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream
dbt run --select dim_listings+ --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

### Incremental Model Refresh

When new data arrives:

```bash
# Load new data
python scripts/load_inside_airbnb_to_snowflake.py

# Full refresh incremental model and downstream
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Validate
dbt test --profiles-dir .
```

## dbt Model Patterns

### Staging Model Example

`models/staging/stg_airbnb__listings.sql`:

```sql
{{
    config(
        materialized='view'
    )
}}

with source as (
    select * from {{ source('airbnb_raw', 'listings') }}
),

cleaned as (
    select
        id::bigint as listing_id,
        name::varchar as listing_name,
        host_id::bigint as host_id,
        host_name::varchar as host_name,
        neighbourhood_cleansed::varchar as neighbourhood,
        latitude::float as latitude,
        longitude::float as longitude,
        room_type::varchar as room_type,
        price::varchar as price_raw,
        minimum_nights::int as minimum_nights,
        number_of_reviews::int as number_of_reviews,
        last_review::date as last_review_date,
        reviews_per_month::float as reviews_per_month,
        calculated_host_listings_count::int as host_listings_count,
        availability_365::int as availability_365
    from source
)

select * from cleaned
```

### Intermediate Model Example

`models/intermediate/int_airbnb__listing_enriched.sql`:

```sql
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
        l.latitude,
        l.longitude,
        l.room_type,
        -- Clean price: remove $ and commas, cast to number
        replace(replace(l.price_raw, '$', ''), ',', '')::float as price,
        l.minimum_nights,
        l.number_of_reviews,
        l.last_review_date,
        l.reviews_per_month,
        l.host_listings_count,
        l.availability_365,
        current_timestamp() as dbt_loaded_at
    from listings l
    left join neighbourhoods n
        on l.neighbourhood = n.neighbourhood
)

select * from enriched
```

### Incremental Fact Model Example

`models/marts/fct_listing_calendar.sql`:

```sql
{{
    config(
        materialized='incremental',
        unique_key='calendar_key',
        merge_update_columns=['available', 'price', 'adjusted_price', 'minimum_nights', 'maximum_nights']
    )
}}

with calendar as (
    select * from {{ ref('int_airbnb__calendar_enriched') }}
)

select
    {{ dbt_utils.generate_surrogate_key(['listing_id', 'calendar_date']) }} as calendar_key,
    listing_id,
    calendar_date,
    available,
    price,
    adjusted_price,
    minimum_nights,
    maximum_nights,
    -- Revenue proxy: if unavailable, assume booked
    case
        when available = false then coalesce(adjusted_price, price, 0)
        else 0
    end as estimated_revenue,
    current_timestamp() as dbt_loaded_at
from calendar

{% if is_incremental() %}
where calendar_date >= (select max(calendar_date) from {{ this }})
{% endif %}
```

### Dimensional Model Example

`models/marts/dim_listings.sql`:

```sql
{{
    config(
        materialized='table'
    )
}}

with listings as (
    select * from {{ ref('int_airbnb__listing_enriched') }}
)

select
    listing_id,
    listing_name,
    host_id,
    neighbourhood,
    neighbourhood_group,
    latitude,
    longitude,
    room_type,
    price,
    minimum_nights,
    availability_365,
    dbt_loaded_at
from listings
```

### Aggregate Model Example

`models/marts/agg_listing_monthly_performance.sql`:

```sql
{{
    config(
        materialized='table'
    )
}}

with calendar_fact as (
    select * from {{ ref('fct_listing_calendar') }}
),

monthly_agg as (
    select
        listing_id,
        date_trunc('month', calendar_date) as month,
        count(*) as total_days,
        sum(case when available = true then 1 else 0 end) as available_days,
        sum(case when available = false then 1 else 0 end) as booked_days,
        avg(price) as avg_price,
        sum(estimated_revenue) as total_estimated_revenue
    from calendar_fact
    group by 1, 2
)

select
    listing_id,
    month,
    total_days,
    available_days,
    booked_days,
    round(booked_days::float / total_days, 2) as occupancy_rate,
    avg_price,
    total_estimated_revenue
from monthly_agg
```

## dbt Tests

### Source Tests

`models/staging/sources.yml`:

```yaml
version: 2

sources:
  - name: airbnb_raw
    database: AIRBNB_ANALYTICS
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
```

### Model Tests

`models/marts/schema.yml`:

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
      - name: price
        description: "Nightly price"
        tests:
          - not_null

  - name: fct_listing_calendar
    description: "Fact table for listing calendar availability and revenue proxy"
    columns:
      - name: calendar_key
        description: "Surrogate key"
        tests:
          - not_null
          - unique
      - name: listing_id
        description: "Foreign key to dim_listings"
        tests:
          - not_null
          - relationships:
              to: ref('dim_listings')
              field: listing_id
```

### Singular Test Example

`tests/assert_positive_prices.sql`:

```sql
select
    listing_id,
    price
from {{ ref('dim_listings') }}
where price < 0
```

## Streamlit Dashboard

### Basic Connection

`dashboard/streamlit_app.py`:

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

@st.cache_resource
def get_snowflake_connection():
    with open('config/local_credentials.json') as f:
        creds = json.load(f)
    return snowflake.connector.connect(
        user=creds['user'],
        password=creds['password'],
        account=creds['account'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema'],
        role=creds['role']
    )

@st.cache_data(ttl=600)
def run_query(query):
    conn = get_snowflake_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

st.title("Airbnb Analytics Dashboard")

# Example: Top neighbourhoods by revenue
query = """
SELECT
    neighbourhood_group,
    SUM(total_estimated_revenue) as total_revenue,
    AVG(occupancy_rate) as avg_occupancy
FROM ANALYTICS.agg_neighbourhood_monthly_performance
GROUP BY 1
ORDER BY 2 DESC
LIMIT 10
"""

df = run_query(query)
st.bar_chart(df.set_index('NEIGHBOURHOOD_GROUP')['TOTAL_REVENUE'])
```

### Running the Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

## Common Patterns

### Pattern: Reusable Macros

`macros/clean_price.sql`:

```sql
{% macro clean_price(price_column) %}
    replace(replace({{ price_column }}, '$', ''), ',', '')::float
{% endmacro %}
```

Usage in model:

```sql
select
    listing_id,
    {{ clean_price('price_raw') }} as price
from {{ ref('stg_airbnb__listings') }}
```

### Pattern: Custom Schema Names

`dbt_project.yml`:

```yaml
models:
  airbnb_analytics:
    staging:
      +schema: staging
      +materialized: view
    intermediate:
      +schema: intermediate
      +materialized: view
    marts:
      +schema: analytics
      +materialized: table
```

### Pattern: dbt Variables

```bash
# Run with custom variable
dbt run --vars '{"min_price": 100}' --profiles-dir .
```

In model:

```sql
where price >= {{ var('min_price', 0) }}
```

## Troubleshooting

### Error: "Object does not exist"

**Problem**: dbt cannot find raw tables.

**Solution**: Ensure data is loaded and source paths are correct:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
dbt run --profiles-dir .
```

### Error: "Compilation Error in model"

**Problem**: SQL syntax or Jinja error.

**Solution**: Use `dbt compile` to see compiled SQL:

```bash
dbt compile --select problematic_model --profiles-dir .
# Check target/compiled/airbnb_analytics/models/...
```

### Error: "Incremental model not updating"

**Problem**: Incremental logic not triggering.

**Solution**: Force full refresh:

```bash
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

### Error: "Streamlit cannot connect to Snowflake"

**Problem**: Credentials mismatch or missing file.

**Solution**: Verify `config/local_credentials.json` exists and has correct values:

```python
import json
with open('config/local_credentials.json') as f:
    print(json.load(f))
```

### Performance: Slow dbt Runs

**Solution**: Increase Snowflake warehouse size or add more threads:

```yaml
# profiles.yml
threads: 8  # Increase parallelism
```

### Data Quality: Failed Tests

**Problem**: Tests failing on `dbt test`.

**Solution**: Investigate failures:

```bash
dbt test --store-failures --profiles-dir .
# Check target/run_results.json for details
```

Query failed test results:

```sql
SELECT * FROM AIRBNB_ANALYTICS.ANALYTICS_DBT_TEST__AUDIT.not_null_dim_listings_listing_id;
```

## Advanced Usage

### Snapshot Example (SCD Type 2)

`snapshots/listings_snapshot.sql`:

```sql
{% snapshot listings_snapshot %}

{{
    config(
      target_schema='snapshots',
      unique_key='listing_id',
      strategy='timestamp',
      updated_at='dbt_loaded_at'
    )
}}

select * from {{ ref('dim_listings') }}

{% endsnapshot %}
```

Run snapshots:

```bash
dbt snapshot --profiles-dir .
```

### Custom Test Example

`tests/generic/test_revenue_positive.sql`:

```sql
{% test revenue_positive(model, column_name) %}

select *
from {{ model }}
where {{ column_name }} < 0

{% endtest %}
```

Usage:

```yaml
models:
  - name: fct_listing_calendar
    columns:
      - name: estimated_revenue
        tests:
          - revenue_positive
```

### Environment-Specific Runs

```bash
# Development
dbt run --target dev --profiles-dir .

# Production
dbt run --target prod --profiles-dir .
```

`profiles.yml`:

```yaml
airbnb_analytics:
  outputs:
    dev:
      # ... dev config
    prod:
      # ... prod config
  target: dev
```

## Key Files Reference

| File | Purpose |
|------|---------|
| `profiles.yml` | dbt Snowflake connection (local, not committed) |
| `dbt_project.yml` | dbt project configuration |
| `config/local_credentials.json` | Streamlit Snowflake credentials (local, not committed) |
| `scripts/load_inside_airbnb_to_snowflake.py` | Data loader script |
| `models/staging/` | Staging models (cleaning, casting) |
| `models/intermediate/` | Intermediate models (joins, enrichment) |
| `models/marts/` | Analytics-ready dimensions, facts, aggregates |
| `tests/` | Singular tests |
| `dashboard/streamlit_app.py` | Streamlit dashboard |

## Security Best Practices

1. **Never commit credentials**: Use `.gitignore` for `profiles.yml`, `config/local_credentials.json`
2. **Use environment variables** for CI/CD:
   ```bash
   export SNOWFLAKE_USER=${{ secrets.SNOWFLAKE_USER }}
   export SNOWFLAKE_PASSWORD=${{ secrets.SNOWFLAKE_PASSWORD }}
   ```
3. **Use Snowflake roles** with least privilege
4. **Enable MFA** on Snowflake accounts
5. **Rotate passwords** regularly

This skill provides comprehensive guidance for building production-grade analytics pipelines with Snowflake, dbt, and Streamlit using the Inside Airbnb dataset as a classroom-friendly foundation.
