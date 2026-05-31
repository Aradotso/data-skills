---
name: snowflake-dbt-airbnb-analytics
description: Build analytics engineering pipelines with Snowflake, dbt, and Streamlit using Inside Airbnb data
triggers:
  - set up dbt with Snowflake for Airbnb data
  - load Inside Airbnb data into Snowflake
  - build dbt staging and mart models for analytics
  - create incremental fact tables in dbt
  - configure Snowflake dbt profiles
  - build a Streamlit dashboard with Snowflake
  - run dbt tests for data quality
  - set up analytics engineering project structure
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build complete analytics engineering projects using Snowflake as a data warehouse, dbt for transformations, and Streamlit for dashboards. The project demonstrates loading Inside Airbnb open data, building layered dbt models (staging → intermediate → marts), implementing incremental materializations, and creating production-ready dashboards.

## What This Project Does

- **Data ingestion**: Python script loads Inside Airbnb CSV/GZIP files into Snowflake internal stages and raw tables
- **dbt transformations**: Multi-layer pipeline (staging → intermediate → marts) with dimensions, facts, and aggregates
- **Incremental modeling**: Merge-based incremental fact table for calendar/booking proxy data
- **Data quality**: Generic and singular dbt tests for validation
- **Dashboard**: Streamlit app querying analytics marts for neighbourhood, host, and listing insights
- **Public-safe credentials**: Local config files ignored by git, no environment variable dependencies

## Installation

```bash
# Clone and set up virtual environment
git clone https://github.com/analyticsdurgesh/Snowflake_DBT_Project.git
cd Snowflake_DBT_Project
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

**Required packages** (from requirements.txt):
- `dbt-snowflake` (dbt adapter for Snowflake)
- `snowflake-connector-python` (Python SDK)
- `streamlit` (dashboard framework)
- `pandas` (data manipulation)

## Configuration Setup

### 1. Snowflake Credentials (dbt)

Create `profiles.yml` from the example:

```bash
cp profiles.yml.example profiles.yml
```

Edit `profiles.yml` with real Snowflake credentials:

```yaml
airbnb_analytics:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT_IDENTIFIER  # e.g., xy12345.us-east-1
      user: YOUR_USERNAME
      password: YOUR_PASSWORD
      role: ACCOUNTADMIN
      database: AIRBNB_ANALYTICS
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

**Important**: `profiles.yml` is gitignored. Never commit real credentials.

### 2. Snowflake Credentials (Python Loader & Streamlit)

Create `config/local_credentials.json` from the example:

```bash
cp config/local_credentials.example.json config/local_credentials.json
```

Edit `config/local_credentials.json`:

```json
{
  "account": "YOUR_ACCOUNT_IDENTIFIER",
  "user": "YOUR_USERNAME",
  "password": "YOUR_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_ANALYTICS",
  "schema": "RAW",
  "role": "ACCOUNTADMIN"
}
```

**Important**: `config/local_credentials.json` is gitignored.

### 3. Inside Airbnb Data

Download the NYC dataset from [Inside Airbnb](https://insideairbnb.com/get-the-data/) and place files in `data/raw/`:

```text
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

## Key Commands

### Load Raw Data into Snowflake

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What it does**:
1. Creates Snowflake objects (database, schemas, stage) from `setup/snowflake_setup.sql`
2. Uploads local CSV/GZIP files to internal stage `INSIDE_AIRBNB_STAGE`
3. Infers CSV headers and creates raw tables
4. Copies data from stage into `RAW` schema tables

**Example Python loader code pattern**:

```python
import snowflake.connector
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

cursor = conn.cursor()

# Put file to internal stage
cursor.execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")

# Copy into table
cursor.execute("""
COPY INTO RAW.LISTINGS
FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"')
ON_ERROR = CONTINUE
""")

cursor.close()
conn.close()
```

### Run dbt Transformations

```bash
# Verify connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific layer
dbt run --select staging --profiles-dir .
dbt run --select intermediate --profiles-dir .
dbt run --select marts --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Full refresh for incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

**Note**: `--profiles-dir .` tells dbt to use `profiles.yml` in the current directory (not `~/.dbt/`).

### Run Streamlit Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

Dashboard reads from `config/local_credentials.json` and queries analytics marts in Snowflake.

## dbt Model Architecture

### Layer Structure

| Layer | Folder | Materialization | Purpose |
|-------|--------|-----------------|---------|
| **Sources** | `models/staging/sources.yml` | - | Raw table references |
| **Staging** | `models/staging/` | `view` | Clean, cast, rename columns |
| **Intermediate** | `models/intermediate/` | `view` | Joins, enrichment, business logic |
| **Marts** | `models/marts/` | `table` or `incremental` | Analytics-ready dims/facts |
| **Aggregates** | `models/marts/` | `table` | Pre-aggregated reporting metrics |

### Key Models

**Staging Models** (`models/staging/`):
- `stg_airbnb__listings.sql` – Clean listing data with proper types
- `stg_airbnb__calendar.sql` – Calendar availability and pricing
- `stg_airbnb__reviews.sql` – Review events with timestamps
- `stg_airbnb__neighbourhoods.sql` – Neighbourhood dimension

**Intermediate Models** (`models/intermediate/`):
- `int_airbnb__listing_enriched.sql` – Listings joined with neighbourhood data
- `int_airbnb__calendar_enriched.sql` – Calendar with listing attributes and revenue proxy
- `int_airbnb__reviews_enriched.sql` – Reviews with listing context

**Mart Models** (`models/marts/`):
- `dim_listings.sql` – Listing dimension
- `dim_hosts.sql` – Host dimension
- `fct_listing_calendar.sql` – **Incremental fact table** (listing-date grain, revenue proxy)
- `fct_reviews.sql` – Review fact table
- `agg_listing_monthly_performance.sql` – Monthly aggregates per listing
- `agg_neighbourhood_monthly_performance.sql` – Monthly aggregates per neighbourhood

### Example: Staging Model

`models/staging/stg_airbnb__listings.sql`:

```sql
with source as (
    select * from {{ source('inside_airbnb', 'listings') }}
),

cleaned as (
    select
        id::bigint as listing_id,
        name as listing_name,
        host_id::bigint as host_id,
        host_name,
        neighbourhood_cleansed as neighbourhood,
        room_type,
        price::varchar as price_text,
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

### Example: Incremental Fact Table

`models/marts/fct_listing_calendar.sql`:

```sql
{{
  config(
    materialized='incremental',
    unique_key=['listing_id', 'calendar_date'],
    merge_update_columns=['available', 'price', 'adjusted_price', 'minimum_nights', 'maximum_nights', 'estimated_revenue']
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
    adjusted_price,
    minimum_nights,
    maximum_nights,
    estimated_revenue
from calendar_enriched

{% if is_incremental() %}
where calendar_date > (select max(calendar_date) from {{ this }})
{% endif %}
```

**Incremental strategy**: Uses Snowflake `merge` to update existing records and insert new ones based on composite key `(listing_id, calendar_date)`.

### Example: Aggregate Model

`models/marts/agg_neighbourhood_monthly_performance.sql`:

```sql
with listing_monthly as (
    select * from {{ ref('agg_listing_monthly_performance') }}
),

neighbourhood_agg as (
    select
        neighbourhood,
        month_start,
        count(distinct listing_id) as total_listings,
        sum(total_days) as total_calendar_days,
        sum(available_days) as total_available_days,
        sum(unavailable_days) as total_unavailable_days,
        avg(avg_price) as avg_price,
        sum(estimated_revenue) as total_estimated_revenue
    from listing_monthly
    group by neighbourhood, month_start
)

select
    neighbourhood,
    month_start,
    total_listings,
    total_calendar_days,
    total_available_days,
    total_unavailable_days,
    round((total_unavailable_days::float / nullif(total_calendar_days, 0)) * 100, 2) as unavailability_rate,
    round(avg_price, 2) as avg_price,
    round(total_estimated_revenue, 2) as total_estimated_revenue
from neighbourhood_agg
order by neighbourhood, month_start
```

## dbt Sources Configuration

`models/staging/sources.yml`:

```yaml
version: 2

sources:
  - name: inside_airbnb
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

## dbt Tests

### Generic Tests (in model YAML files)

`models/marts/schema.yml`:

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

  - name: fct_listing_calendar
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
      - name: price
        tests:
          - not_null
```

### Singular Tests (custom SQL)

`tests/assert_no_duplicate_listing_dates.sql`:

```sql
-- Check for duplicate listing-date combinations in fact table
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

Run specific test:

```bash
dbt test --select assert_no_duplicate_listing_dates --profiles-dir .
```

## Streamlit Dashboard Pattern

`dashboard/streamlit_app.py`:

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load Snowflake credentials
with open('config/local_credentials.json') as f:
    creds = json.load(f)

# Connect to Snowflake
@st.cache_resource
def get_snowflake_connection():
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema='ANALYTICS',  # Point to analytics schema with marts
        role=creds['role']
    )

conn = get_snowflake_connection()

# Query analytics mart
@st.cache_data
def load_neighbourhood_performance():
    query = """
    SELECT
        neighbourhood,
        month_start,
        total_listings,
        unavailability_rate,
        avg_price,
        total_estimated_revenue
    FROM agg_neighbourhood_monthly_performance
    ORDER BY month_start DESC, total_estimated_revenue DESC
    """
    return pd.read_sql(query, conn)

# Dashboard UI
st.title("Airbnb Analytics Dashboard")
st.subheader("Neighbourhood Performance")

df = load_neighbourhood_performance()
st.dataframe(df)

# Filter by neighbourhood
selected_neighbourhood = st.selectbox(
    "Select Neighbourhood",
    df['NEIGHBOURHOOD'].unique()
)

filtered_df = df[df['NEIGHBOURHOOD'] == selected_neighbourhood]
st.line_chart(
    filtered_df.set_index('MONTH_START')['TOTAL_ESTIMATED_REVENUE']
)
```

## Common Workflows

### Initial Setup (First Time)

```bash
# 1. Install dependencies
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 2. Configure credentials
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
# Edit both files with real Snowflake values

# 3. Download Inside Airbnb data to data/raw/

# 4. Load raw data
python scripts/load_inside_airbnb_to_snowflake.py

# 5. Run dbt pipeline
dbt debug --profiles-dir .
dbt run --profiles-dir .
dbt test --profiles-dir .

# 6. Launch dashboard
streamlit run dashboard/streamlit_app.py
```

### Refresh Data (New Snapshot)

```bash
# 1. Download new Inside Airbnb files to data/raw/

# 2. Reload raw tables
python scripts/load_inside_airbnb_to_snowflake.py

# 3. Full refresh incremental fact table
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 4. Run remaining models and tests
dbt run --exclude fct_listing_calendar --profiles-dir .
dbt test --profiles-dir .
```

### Add a New Mart Model

```bash
# 1. Create SQL file in models/marts/
# Example: models/marts/dim_room_types.sql

# 2. Add model config and tests in models/marts/schema.yml

# 3. Run the new model
dbt run --select dim_room_types --profiles-dir .

# 4. Test it
dbt test --select dim_room_types --profiles-dir .

# 5. Update downstream models if needed
dbt run --select dim_room_types+ --profiles-dir .
```

### Debug a Failing Model

```bash
# Run with verbose logging
dbt run --select failing_model --profiles-dir . --debug

# Compile SQL without running
dbt compile --select failing_model --profiles-dir .
# Check target/compiled/airbnb_analytics/models/.../failing_model.sql

# Run in Snowflake UI
# Copy compiled SQL and test in Snowflake worksheet
```

## Troubleshooting

### "Credentials in profiles.yml not valid"

**Problem**: dbt cannot connect to Snowflake.

**Solution**:
- Verify `profiles.yml` has correct account identifier (format: `xy12345.region`)
- Check Snowflake username/password
- Ensure warehouse, database, and role exist and user has access
- Test connection: `dbt debug --profiles-dir .`

### "Object does not exist" errors during dbt run

**Problem**: Raw tables not found.

**Solution**:
- Run the Python loader first: `python scripts/load_inside_airbnb_to_snowflake.py`
- Verify in Snowflake UI that `AIRBNB_ANALYTICS.RAW` schema has tables
- Check `models/staging/sources.yml` database/schema names match

### Incremental model not updating

**Problem**: New data doesn't appear after `dbt run`.

**Solution**:
- Use `--full-refresh` to rebuild: `dbt run --full-refresh --select fct_listing_calendar --profiles-dir .`
- Check the incremental filter logic in the model's `{% if is_incremental() %}` block
- Verify source data actually has new dates

### Streamlit "Failed to connect to Snowflake"

**Problem**: Dashboard won't load data.

**Solution**:
- Check `config/local_credentials.json` exists and has valid credentials
- Ensure schema in connection is `ANALYTICS` (where marts live, not `RAW`)
- Verify marts exist: `dbt run --select marts --profiles-dir .`

### "File not found" during Python loader

**Problem**: `load_inside_airbnb_to_snowflake.py` fails.

**Solution**:
- Verify files exist in `data/raw/` with exact names:
  - `listings.csv.gz`
  - `calendar.csv.gz`
  - `reviews.csv.gz`
  - `neighbourhoods.csv`
- Check file permissions (readable)
- Ensure gzip files are valid: `gunzip -t data/raw/listings.csv.gz`

### dbt tests fail with "expression error"

**Problem**: SQL syntax error in test.

**Solution**:
- Run `dbt compile --select test_name --profiles-dir .` to see compiled SQL
- Check `target/compiled/...` for the full query
- Test SQL directly in Snowflake UI
- Common issue: column name case mismatch (Snowflake uppercases unquoted identifiers)

## Best Practices

1. **Never commit credentials**: Always use gitignored `profiles.yml` and `config/local_credentials.json`
2. **Use `--profiles-dir .`**: Keep project-specific profiles in repo root
3. **Incremental models for large facts**: Calendar data grows daily; use `incremental` materialization
4. **Test after every run**: `dbt run && dbt test` ensures data quality
5. **Document models**: Add descriptions in YAML files for `dbt docs generate`
6. **Staging layer stays thin**: Only clean and cast; save joins/logic for intermediate
7. **Cache Streamlit queries**: Use `@st.cache_data` to avoid re-querying Snowflake on every interaction

## Real-World Extensions

**Add snapshots for SCD Type 2**:

```sql
-- snapshots/listings_snapshot.sql
{% snapshot listings_snapshot %}
{{
    config(
        target_schema='snapshots',
        unique_key='listing_id',
        strategy='check',
        check_cols=['price_text', 'availability_365']
    )
}}
select * from {{ ref('stg_airbnb__listings') }}
{% endsnapshot %}
```

Run: `dbt snapshot --profiles-dir .`

**Add custom macros**:

```sql
-- macros/cents_to_dollars.sql
{% macro cents_to_dollars(column_name) %}
    ({{ column_name }}::float / 100)::decimal(10,2)
{% endmacro %}
```

Use in model: `{{ cents_to_dollars('price_cents') }} as price_dollars`

**Add exposure for dashboard**:

```yaml
# models/marts/exposures.yml
version: 2

exposures:
  - name: streamlit_dashboard
    type: dashboard
    maturity: medium
    owner:
      name: Analytics Team
      email: analytics@example.com
    depends_on:
      - ref('agg_neighbourhood_monthly_performance')
      - ref('dim_listings')
      - ref('dim_hosts')
```

This skill gives AI agents everything needed to help developers build, extend, and troubleshoot this Snowflake + dbt + Streamlit analytics engineering project.
