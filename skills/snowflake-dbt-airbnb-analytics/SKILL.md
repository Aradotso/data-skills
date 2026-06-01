---
name: snowflake-dbt-airbnb-analytics
description: Analytics engineering with Snowflake, dbt, and Inside Airbnb data - staging, marts, incremental models, tests, and Streamlit dashboards
triggers:
  - build airbnb analytics with dbt and snowflake
  - set up inside airbnb data pipeline
  - create dbt staging and mart models
  - configure snowflake dbt incremental loads
  - write dbt tests for data quality
  - build streamlit dashboard on snowflake
  - load inside airbnb csv to snowflake
  - deploy analytics engineering project
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

The Snowflake_DBT_Project is a complete analytics engineering reference implementation that:

- Loads Inside Airbnb open datasets (listings, calendar, reviews, neighbourhoods) into Snowflake
- Uses dbt to transform raw data through staging → intermediate → marts layers
- Implements incremental fact models with merge strategy
- Validates data quality with generic and singular tests
- Powers a Streamlit dashboard for neighbourhood and host analytics
- Demonstrates public-safe credential management (no committed secrets)

**Architecture**: CSV/GZIP files → Python loader → Snowflake internal stage → RAW schema → dbt staging → intermediate enrichment → dimension/fact marts → Streamlit dashboard

**Key Pattern**: Text-preserving raw layer, strongly-typed staging views, reusable intermediate models, analytics-ready marts, incremental calendar facts.

## Installation

### Prerequisites

- Snowflake account with warehouse, database, and schema creation privileges
- Python 3.8+
- dbt-snowflake adapter
- Inside Airbnb CSV files in `data/raw/`:
  - `listings.csv.gz`
  - `calendar.csv.gz`
  - `reviews.csv.gz`
  - `neighbourhoods.csv`

### Setup Steps

```bash
# Clone and enter project
git clone https://github.com/analyticsdurgesh/Snowflake_DBT_Project.git
cd Snowflake_DBT_Project

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configure Credentials (Local Only)

**Never commit real credentials.** Create local files from examples:

```bash
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

**Edit `profiles.yml`:**

```yaml
snowflake_dbt_project:
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT  # e.g., abc12345.us-east-1
      user: YOUR_USERNAME
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"  # or hardcode locally
      role: YOUR_ROLE
      database: AIRBNB_DB
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
  target: dev
```

**Edit `config/local_credentials.json`:**

```json
{
  "account": "YOUR_ACCOUNT.us-east-1",
  "user": "YOUR_USERNAME",
  "password": "YOUR_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DB",
  "schema": "RAW",
  "role": "YOUR_ROLE"
}
```

Both files are `.gitignore`d.

## Loading Raw Data

The Python loader creates Snowflake objects, uploads files to an internal stage, and copies them into raw tables.

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What it does:**

1. Reads credentials from `config/local_credentials.json`
2. Executes `setup/snowflake_setup.sql` (creates database, schemas, stage, file format)
3. Uploads local CSV/GZIP files to `@INSIDE_AIRBNB_STAGE`
4. Creates raw tables from CSV headers (all `VARCHAR`)
5. Copies staged data into `RAW.LISTINGS`, `RAW.CALENDAR`, `RAW.REVIEWS`, `RAW.NEIGHBOURHOODS`

**Key Loader Code Pattern:**

```python
import json
import snowflake.connector
from pathlib import Path

# Read local credentials
with open('config/local_credentials.json') as f:
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

cursor = conn.cursor()

# Execute setup SQL
with open('setup/snowflake_setup.sql') as f:
    for statement in f.read().split(';'):
        if statement.strip():
            cursor.execute(statement)

# Upload file to stage
cursor.execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE OVERWRITE=TRUE")

# Copy into raw table
cursor.execute("""
COPY INTO RAW.LISTINGS
FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
FILE_FORMAT = (FORMAT_NAME = 'AIRBNB_CSV_FORMAT')
ON_ERROR = 'CONTINUE'
""")
```

## dbt Project Structure

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
    ├── fct_listing_calendar.sql  (incremental)
    ├── fct_reviews.sql
    ├── agg_listing_monthly_performance.sql
    └── agg_neighbourhood_monthly_performance.sql
```

### Staging Layer

Clean, cast, and standardize raw data. Use views for freshness.

**`models/staging/stg_airbnb__listings.sql`:**

```sql
with source as (
    select * from {{ source('airbnb_raw', 'listings') }}
),

cleaned as (
    select
        id::bigint as listing_id,
        name as listing_name,
        host_id::bigint as host_id,
        host_name,
        neighbourhood_cleansed as neighbourhood,
        latitude::float as latitude,
        longitude::float as longitude,
        room_type,
        price::varchar as price_text,  -- Clean in intermediate
        minimum_nights::int as minimum_nights,
        availability_365::int as availability_365
    from source
)

select * from cleaned
```

**`models/staging/_sources.yml`:**

```yaml
version: 2

sources:
  - name: airbnb_raw
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

### Intermediate Layer

Join, enrich, and add business logic. Reusable by multiple marts.

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
        l.listing_id,
        l.listing_name,
        l.host_id,
        l.host_name,
        l.neighbourhood,
        n.neighbourhood_group,
        l.latitude,
        l.longitude,
        l.room_type,
        -- Clean price: remove $ and commas, cast to decimal
        regexp_replace(l.price_text, '[^0-9.]', '')::decimal(10,2) as price,
        l.minimum_nights,
        l.availability_365
    from listings l
    left join neighbourhoods n
        on l.neighbourhood = n.neighbourhood
)

select * from enriched
```

**`models/intermediate/int_airbnb__calendar_enriched.sql`:**

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
        c.available,
        c.price,
        l.listing_name,
        l.host_id,
        l.neighbourhood,
        l.neighbourhood_group,
        l.room_type,
        -- Revenue proxy: if not available, assume booked at listed price
        case
            when c.available = 'f' then c.price
            else 0
        end as estimated_revenue
    from calendar c
    inner join listings l
        on c.listing_id = l.listing_id
)

select * from enriched
```

### Marts Layer

Analytics-ready dimensions and facts. Incremental models for large fact tables.

**`models/marts/dim_listings.sql`:**

```sql
{{ config(materialized='table') }}

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
    price as current_price,
    minimum_nights,
    availability_365
from {{ ref('int_airbnb__listing_enriched') }}
```

**`models/marts/fct_listing_calendar.sql` (incremental):**

```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        merge_update_columns=['available', 'price', 'estimated_revenue']
    )
}}

with calendar_data as (
    select
        listing_id,
        calendar_date,
        available,
        price,
        estimated_revenue,
        neighbourhood,
        neighbourhood_group,
        room_type,
        host_id
    from {{ ref('int_airbnb__calendar_enriched') }}

    {% if is_incremental() %}
    where calendar_date >= (select max(calendar_date) from {{ this }})
    {% endif %}
)

select * from calendar_data
```

**`models/marts/agg_listing_monthly_performance.sql`:**

```sql
{{ config(materialized='table') }}

with monthly as (
    select
        listing_id,
        date_trunc('month', calendar_date)::date as month,
        count(*) as total_days,
        sum(case when available = 'f' then 1 else 0 end) as unavailable_days,
        avg(price) as avg_price,
        sum(estimated_revenue) as estimated_revenue
    from {{ ref('fct_listing_calendar') }}
    group by 1, 2
)

select
    listing_id,
    month,
    total_days,
    unavailable_days,
    round(unavailable_days::decimal / total_days, 2) as occupancy_rate,
    avg_price,
    estimated_revenue
from monthly
```

## dbt Commands

Run from project root with `--profiles-dir .` to use local `profiles.yml`:

```bash
# Validate connection and configuration
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select fct_listing_calendar+ --profiles-dir .

# Full refresh incremental models (after new data load)
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Run all tests
dbt test --profiles-dir .

# Test specific model
dbt test --select dim_listings --profiles-dir .

# Generate and serve documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

### Common Workflow for New Data

```bash
# 1. Load new Inside Airbnb snapshot
python scripts/load_inside_airbnb_to_snowflake.py

# 2. Full refresh incremental calendar fact
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# 3. Run downstream aggregates
dbt run --select agg_listing_monthly_performance+ --profiles-dir .

# 4. Test data quality
dbt test --profiles-dir .
```

## Data Quality Tests

### Generic Tests (in `_sources.yml` and model YAML)

```yaml
# models/marts/_marts.yml
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
      - name: current_price
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
```

### Singular Tests (custom SQL)

**`tests/assert_no_duplicate_listing_dates.sql`:**

```sql
-- Ensure fct_listing_calendar has no duplicate listing-date pairs
select
    listing_id,
    calendar_date,
    count(*) as duplicate_count
from {{ ref('fct_listing_calendar') }}
group by 1, 2
having count(*) > 1
```

## Streamlit Dashboard

The dashboard reads Snowflake credentials from `config/local_credentials.json` and queries marts.

**`dashboard/streamlit_app.py`:**

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json
from pathlib import Path

st.set_page_config(page_title="Airbnb Analytics", layout="wide")

@st.cache_resource
def get_snowflake_connection():
    creds_path = Path(__file__).parent.parent / 'config' / 'local_credentials.json'
    with open(creds_path) as f:
        creds = json.load(f)
    
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema='ANALYTICS',  # Marts schema
        role=creds['role']
    )

conn = get_snowflake_connection()

st.title("🏠 Inside Airbnb Analytics Dashboard")

# Top neighbourhoods by estimated revenue
st.subheader("Top 10 Neighbourhoods by Estimated Revenue")

query = """
SELECT
    neighbourhood_group,
    SUM(estimated_revenue) as total_revenue,
    COUNT(DISTINCT listing_id) as listing_count,
    AVG(occupancy_rate) as avg_occupancy
FROM agg_neighbourhood_monthly_performance
GROUP BY 1
ORDER BY 2 DESC
LIMIT 10
"""

df = pd.read_sql(query, conn)
st.dataframe(df)

# Filters
st.sidebar.header("Filters")
selected_neighbourhood = st.sidebar.selectbox(
    "Select Neighbourhood",
    options=df['NEIGHBOURHOOD_GROUP'].tolist()
)

# Monthly trend for selected neighbourhood
st.subheader(f"Monthly Revenue Trend: {selected_neighbourhood}")

trend_query = f"""
SELECT
    month,
    estimated_revenue,
    avg_occupancy
FROM agg_neighbourhood_monthly_performance
WHERE neighbourhood_group = '{selected_neighbourhood}'
ORDER BY month
"""

trend_df = pd.read_sql(trend_query, conn)
st.line_chart(trend_df.set_index('MONTH')['ESTIMATED_REVENUE'])

# Top hosts by listing count
st.subheader("Top 10 Hosts by Number of Listings")

host_query = """
SELECT
    host_id,
    host_name,
    COUNT(DISTINCT listing_id) as listing_count
FROM dim_listings
GROUP BY 1, 2
ORDER BY 3 DESC
LIMIT 10
"""

host_df = pd.read_sql(host_query, conn)
st.dataframe(host_df)
```

**Run dashboard:**

```bash
streamlit run dashboard/streamlit_app.py
```

## Configuration Files

### `dbt_project.yml`

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
      +schema: analytics
```

### Environment Variables (Recommended for CI/CD)

Instead of hardcoding passwords in `profiles.yml`, use:

```yaml
password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
```

Then:

```bash
export SNOWFLAKE_PASSWORD='your_password'
dbt run --profiles-dir .
```

## Common Patterns

### Pattern 1: Adding a New Staging Model

```sql
-- models/staging/stg_airbnb__new_source.sql
with source as (
    select * from {{ source('airbnb_raw', 'new_table') }}
),

cleaned as (
    select
        id::bigint as new_id,
        created_at::timestamp as created_at,
        value::decimal(10,2) as value
    from source
)

select * from cleaned
```

Add to `_sources.yml`:

```yaml
sources:
  - name: airbnb_raw
    tables:
      - name: new_table
        columns:
          - name: id
            tests:
              - not_null
              - unique
```

### Pattern 2: Incremental Model with Surrogate Key

```sql
{{
    config(
        materialized='incremental',
        unique_key='event_key'
    )
}}

select
    {{ dbt_utils.generate_surrogate_key(['listing_id', 'event_date']) }} as event_key,
    listing_id,
    event_date,
    event_type
from {{ ref('stg_events') }}

{% if is_incremental() %}
where event_date > (select max(event_date) from {{ this }})
{% endif %}
```

### Pattern 3: Custom Generic Test Macro

```sql
-- macros/test_positive_value.sql
{% test positive_value(model, column_name) %}

select *
from {{ model }}
where {{ column_name }} < 0

{% endtest %}
```

Use in model YAML:

```yaml
columns:
  - name: price
    tests:
      - positive_value
```

## Troubleshooting

### Connection Issues

**Error**: `250001: Could not connect to Snowflake backend`

- Verify `account` format: `abc12345.us-east-1` (include region)
- Check network/firewall settings
- Confirm credentials in `profiles.yml` or `local_credentials.json`

**Debug command:**

```bash
dbt debug --profiles-dir .
```

### Incremental Model Not Updating

**Issue**: New data not appearing after `dbt run`

**Solution**: Use `--full-refresh` after loading new raw data:

```bash
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

### Test Failures

**Error**: `unique` test fails on `listing_id`

**Diagnosis**:

```bash
dbt test --select dim_listings --profiles-dir .
```

Check for duplicates:

```sql
SELECT listing_id, COUNT(*)
FROM ANALYTICS.dim_listings
GROUP BY 1
HAVING COUNT(*) > 1;
```

**Fix**: Review staging model for distinct logic or add `group by` in mart model.

### Price Cleaning Issues

**Problem**: Price field has `$`, commas, or nulls

**Intermediate model fix:**

```sql
select
    coalesce(
        regexp_replace(price_text, '[^0-9.]', '')::decimal(10,2),
        0
    ) as price
```

### Stage File Upload Fails

**Error**: `PUT file://data/raw/listings.csv.gz` fails

**Check**:

```bash
ls -lh data/raw/listings.csv.gz  # Verify file exists
```

**Manual Snowflake test**:

```sql
USE DATABASE AIRBNB_DB;
USE SCHEMA RAW;
LIST @INSIDE_AIRBNB_STAGE;
```

### Dashboard Connection Errors

**Error**: `config/local_credentials.json` not found

**Fix**:

```bash
cp config/local_credentials.example.json config/local_credentials.json
# Edit with real credentials
```

Ensure Streamlit script uses correct path:

```python
creds_path = Path(__file__).parent.parent / 'config' / 'local_credentials.json'
```

## Advanced: CI/CD Integration

For GitHub Actions or similar:

```yaml
# .github/workflows/dbt_run.yml
name: dbt Run

on:
  push:
    branches: [main]

jobs:
  dbt:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: 3.9
      
      - name: Install dependencies
        run: pip install -r requirements.txt
      
      - name: Run dbt
        env:
          SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
          SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
          SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
        run: |
          dbt run --profiles-dir . --target prod
          dbt test --profiles-dir . --target prod
```

Update `profiles.yml` for prod target:

```yaml
snowflake_dbt_project:
  outputs:
    dev:
      # ... local dev config
    prod:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: TRANSFORMER
      database: AIRBNB_PROD
      warehouse: PROD_WH
      schema: ANALYTICS
      threads: 8
  target: dev
```

## Key Takeaways

1. **Layered Transformation**: Raw → Staging (clean/cast) → Intermediate (join/enrich) → Marts (analytics-ready)
2. **Incremental Strategy**: Use `materialized='incremental'` with `unique_key` for large fact tables
3. **Credential Safety**: Never commit `profiles.yml` or `local_credentials.json`; use examples and `.gitignore`
4. **Testing**: Combine generic tests (not_null, unique, accepted_values) with singular SQL tests
5. **Business Logic**: Encode domain knowledge in intermediate models (e.g., revenue proxy from availability)
6. **Dashboard Integration**: Streamlit queries marts directly via Python connector
7. **Documentation**: `dbt docs generate` creates lineage graph and column-level docs

This skill enables AI agents to scaffold, extend, and troubleshoot Snowflake + dbt + Streamlit analytics pipelines using the Inside Airbnb pattern.
