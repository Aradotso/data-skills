---
name: snowflake-dbt-airbnb-analytics
description: Analytics engineering project loading Inside Airbnb data into Snowflake, transforming with dbt, and visualizing with Streamlit
triggers:
  - how do I set up the Airbnb Snowflake dbt project
  - load Inside Airbnb data into Snowflake
  - run dbt models for Airbnb analytics
  - create Snowflake analytics marts with dbt
  - build Streamlit dashboard for Airbnb data
  - configure dbt incremental models in Snowflake
  - set up analytics engineering pipeline with dbt
  - transform Inside Airbnb data with dbt
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill provides expertise in using the Snowflake_DBT_Project, an analytics engineering portfolio project that ingests Inside Airbnb open data into Snowflake, transforms it with dbt into staging, intermediate, and mart layers, and powers a Streamlit dashboard.

## What This Project Does

- **Data Ingestion**: Loads Inside Airbnb CSV/GZIP files into Snowflake internal stages and raw tables
- **dbt Transformation**: Multi-layer dbt project (staging → intermediate → marts) with incremental fact models
- **Data Quality**: Generic and singular dbt tests for validation
- **Visualization**: Streamlit dashboard connected to analytics marts
- **Public-Safe**: Git-ignored credential files, no committed secrets

## Architecture Overview

```
Inside Airbnb CSVs → Python Loader → Snowflake Stage → RAW Schema
  → dbt Staging Views → dbt Intermediate Models → dbt Marts
  → Streamlit Dashboard
```

**Key dbt Layers:**
- **Staging** (`stg_airbnb__*`): Clean, cast, standardize
- **Intermediate** (`int_airbnb__*`): Joins, enrichment, revenue proxy
- **Marts**: Dimensions (`dim_listings`, `dim_hosts`), facts (`fct_listing_calendar`, `fct_reviews`), aggregates

## Installation

### Prerequisites

```bash
# Snowflake account with credentials
# Python 3.8+
# Inside Airbnb data files in data/raw/
```

### Setup Steps

```bash
# Clone and enter project
cd Snowflake_DBT_Project

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configure Credentials

**Never commit real credentials.** Create local-only files:

```bash
# Copy example files
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
      account: YOUR_ACCOUNT
      user: YOUR_USERNAME
      password: YOUR_PASSWORD  # Use env var in production: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: YOUR_ROLE
      database: AIRBNB_DB
      warehouse: COMPUTE_WH
      schema: DEV
      threads: 4
```

**Edit `config/local_credentials.json`:**

```json
{
  "account": "YOUR_ACCOUNT",
  "user": "YOUR_USERNAME",
  "password": "YOUR_PASSWORD",
  "role": "YOUR_ROLE",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DB",
  "schema": "DEV"
}
```

## Data Loading

### Load Inside Airbnb Data into Snowflake

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What this does:**
1. Creates Snowflake database, schemas, stage, file format (from `setup/snowflake_setup.sql`)
2. Uploads `data/raw/*.csv.gz` and `*.csv` to internal stage
3. Creates raw tables with text columns (preserves raw data)
4. Copies staged files into `RAW.LISTINGS`, `RAW.CALENDAR`, `RAW.REVIEWS`, `RAW.NEIGHBOURHOODS`

**Expected raw files:**
```
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

Download from [Inside Airbnb](https://insideairbnb.com/get-the-data/) (NYC recommended).

## dbt Commands

### Basic Workflow

```bash
# Verify connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .

# Serve documentation
dbt docs serve --profiles-dir .
```

### Incremental Model Refresh

For new Inside Airbnb snapshots:

```bash
# Reload raw data
python scripts/load_inside_airbnb_to_snowflake.py

# Full refresh of incremental fact and downstream models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Validate
dbt test --profiles-dir .
```

### Selective Runs

```bash
# Run only staging models
dbt run --select staging --profiles-dir .

# Run marts and downstream
dbt run --select marts+ --profiles-dir .

# Run a specific model and its children
dbt run --select dim_listings+ --profiles-dir .

# Test a specific model
dbt test --select fct_listing_calendar --profiles-dir .
```

## Key dbt Models

### Staging Models

**`models/staging/stg_airbnb__listings.sql`:**
```sql
with source as (
    select * from {{ source('raw_airbnb', 'listings') }}
),

cleaned as (
    select
        cast(id as integer) as listing_id,
        cast(host_id as integer) as host_id,
        trim(name) as listing_name,
        trim(neighbourhood) as neighbourhood_name,
        cast(latitude as float) as latitude,
        cast(longitude as float) as longitude,
        trim(room_type) as room_type,
        cast(price as float) as price,
        cast(minimum_nights as integer) as minimum_nights,
        cast(number_of_reviews as integer) as number_of_reviews,
        cast(reviews_per_month as float) as reviews_per_month,
        cast(calculated_host_listings_count as integer) as host_listings_count,
        cast(availability_365 as integer) as availability_365
    from source
)

select * from cleaned
```

### Intermediate Models

**`models/intermediate/int_airbnb__listing_enriched.sql`:**
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
        n.neighbourhood_group
    from listings l
    left join neighbourhoods n
        on l.neighbourhood_name = n.neighbourhood
)

select * from enriched
```

### Incremental Fact Model

**`models/marts/fct_listing_calendar.sql`:**
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
    price,
    minimum_nights,
    maximum_nights,
    neighbourhood_name,
    neighbourhood_group,
    room_type,
    case
        when is_available = false then price
        else 0
    end as estimated_revenue
from calendar_enriched

{% if is_incremental() %}
    where calendar_date > (select max(calendar_date) from {{ this }})
{% endif %}
```

### Aggregate Model

**`models/marts/agg_listing_monthly_performance.sql`:**
```sql
with calendar_facts as (
    select * from {{ ref('fct_listing_calendar') }}
),

monthly_agg as (
    select
        listing_id,
        date_trunc('month', calendar_date) as month,
        count(*) as total_days,
        sum(case when is_available = false then 1 else 0 end) as unavailable_days,
        sum(estimated_revenue) as estimated_monthly_revenue,
        avg(price) as avg_daily_price,
        max(neighbourhood_name) as neighbourhood_name,
        max(neighbourhood_group) as neighbourhood_group,
        max(room_type) as room_type
    from calendar_facts
    group by listing_id, date_trunc('month', calendar_date)
)

select * from monthly_agg
```

## Configuration

### dbt Project Configuration

**`dbt_project.yml`:**
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
      +materialized: view
      +schema: intermediate
    marts:
      +materialized: table
      +schema: marts
```

### Source Configuration

**`models/staging/sources.yml`:**
```yaml
version: 2

sources:
  - name: raw_airbnb
    database: airbnb_db
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
      - name: reviews
      - name: neighbourhoods
```

## Data Quality Tests

### Generic Tests

**In model YAML:**
```yaml
version: 2

models:
  - name: dim_listings
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

**`tests/listing_calendar_unique.sql`:**
```sql
-- Test that listing_id and calendar_date combinations are unique
select
    listing_id,
    calendar_date,
    count(*) as occurrences
from {{ ref('fct_listing_calendar') }}
group by listing_id, calendar_date
having count(*) > 1
```

## Streamlit Dashboard

### Run Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

### Dashboard Connection Pattern

**`dashboard/streamlit_app.py`:**
```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load credentials from git-ignored file
with open('config/local_credentials.json', 'r') as f:
    creds = json.load(f)

# Create Snowflake connection
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

# Query marts
@st.cache_data
def load_neighbourhood_performance():
    query = """
    SELECT
        neighbourhood_name,
        neighbourhood_group,
        SUM(estimated_monthly_revenue) as total_revenue,
        AVG(avg_daily_price) as avg_price,
        COUNT(DISTINCT listing_id) as listing_count
    FROM marts.agg_listing_monthly_performance
    GROUP BY neighbourhood_name, neighbourhood_group
    ORDER BY total_revenue DESC
    """
    return pd.read_sql(query, conn)

df = load_neighbourhood_performance()

st.title("Airbnb Analytics Dashboard")
st.dataframe(df)

# Visualizations
st.bar_chart(df.set_index('neighbourhood_name')['total_revenue'].head(10))
```

## Common Patterns

### Adding a New Staging Model

1. **Create SQL file** in `models/staging/`:
```sql
-- models/staging/stg_airbnb__new_source.sql
with source as (
    select * from {{ source('raw_airbnb', 'new_source') }}
),

cleaned as (
    select
        cast(id as integer) as record_id,
        trim(name) as record_name,
        cast(created_at as timestamp) as created_at
    from source
)

select * from cleaned
```

2. **Add to sources.yml**:
```yaml
sources:
  - name: raw_airbnb
    tables:
      - name: new_source
        columns:
          - name: id
            tests:
              - not_null
              - unique
```

3. **Run and test**:
```bash
dbt run --select stg_airbnb__new_source
dbt test --select stg_airbnb__new_source
```

### Creating a New Mart

```sql
-- models/marts/dim_hosts.sql
with listings as (
    select * from {{ ref('int_airbnb__listing_enriched') }}
),

host_agg as (
    select
        host_id,
        count(distinct listing_id) as total_listings,
        avg(price) as avg_listing_price,
        sum(number_of_reviews) as total_reviews,
        max(host_listings_count) as host_listings_count
    from listings
    group by host_id
)

select * from host_agg
```

### Using Macros

**`macros/cents_to_dollars.sql`:**
```sql
{% macro cents_to_dollars(column_name, precision=2) %}
    round({{ column_name }} / 100.0, {{ precision }})
{% endmacro %}
```

**Usage in model:**
```sql
select
    listing_id,
    {{ cents_to_dollars('price_cents') }} as price_dollars
from source
```

## Troubleshooting

### Connection Issues

**Problem**: `dbt debug` fails with authentication error

**Solution**:
```bash
# Verify profiles.yml is in project root
ls -la profiles.yml

# Check Snowflake credentials
snowsql -a YOUR_ACCOUNT -u YOUR_USERNAME

# Test Python connection
python -c "import snowflake.connector; print('OK')"
```

### Incremental Model Issues

**Problem**: Incremental model not updating

**Solution**:
```bash
# Full refresh the model
dbt run --full-refresh --select fct_listing_calendar

# Check for schema changes
dbt run --select fct_listing_calendar --full-refresh
```

**Problem**: Duplicate records in incremental fact

**Solution**:
```sql
-- Add deduplication in model
with deduped as (
    select *,
        row_number() over (
            partition by listing_id, calendar_date
            order by _loaded_at desc
        ) as rn
    from source
)
select * from deduped where rn = 1
```

### Test Failures

**Problem**: `accepted_values` test fails

**Solution**:
```sql
-- Investigate actual values
select distinct room_type
from {{ ref('stg_airbnb__listings') }}
where room_type not in ('Entire home/apt', 'Private room', 'Shared room', 'Hotel room')
```

### Loader Script Issues

**Problem**: `load_inside_airbnb_to_snowflake.py` fails

**Solution**:
```bash
# Check file exists
ls -lh data/raw/listings.csv.gz

# Test Snowflake connection
python -c "
import json
import snowflake.connector
with open('config/local_credentials.json') as f:
    creds = json.load(f)
conn = snowflake.connector.connect(**creds)
print(conn.cursor().execute('SELECT CURRENT_VERSION()').fetchone())
"

# Run with verbose logging
python scripts/load_inside_airbnb_to_snowflake.py 2>&1 | tee load.log
```

### Streamlit Connection Issues

**Problem**: Dashboard can't connect to Snowflake

**Solution**:
```python
# Verify credentials file
import json
with open('config/local_credentials.json') as f:
    creds = json.load(f)
    print(f"Account: {creds['account']}")
    print(f"Database: {creds['database']}")

# Test query manually
import snowflake.connector
conn = snowflake.connector.connect(**creds)
cur = conn.cursor()
cur.execute("SELECT CURRENT_DATABASE(), CURRENT_SCHEMA()")
print(cur.fetchone())
```

## Best Practices

1. **Never commit credentials** — use git-ignored files
2. **Use staging for cleaning** — keep raw data untouched
3. **Document models** — add descriptions in YAML
4. **Test incrementally** — add tests as you build
5. **Use `--select`** — run only what changed
6. **Full-refresh sparingly** — only when schema changes
7. **Version Inside Airbnb data** — keep download dates in filenames
8. **Monitor warehouse costs** — use appropriate sizes

## Environment Variables (Production)

For production deployments, use environment variables instead of JSON:

**`profiles.yml` (production):**
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
      schema: prod
      threads: 4
```

**Set in CI/CD:**
```bash
export SNOWFLAKE_ACCOUNT=your_account
export SNOWFLAKE_USER=your_user
export SNOWFLAKE_PASSWORD=your_password
export SNOWFLAKE_ROLE=your_role
export SNOWFLAKE_DATABASE=AIRBNB_DB
export SNOWFLAKE_WAREHOUSE=COMPUTE_WH
```

This skill equips AI agents to help developers set up, configure, run, and troubleshoot the Snowflake_DBT_Project for analytics engineering workflows with Inside Airbnb data.
