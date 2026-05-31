---
name: snowflake-dbt-airbnb-analytics
description: Load Inside Airbnb data into Snowflake, transform with dbt staging/intermediate/mart layers, and build analytics dashboards with Streamlit
triggers:
  - set up airbnb analytics warehouse with snowflake and dbt
  - load inside airbnb data into snowflake with dbt transformations
  - build dbt staging intermediate and mart models for airbnb
  - create incremental fact tables in snowflake with dbt
  - configure dbt profiles for snowflake analytics project
  - run dbt tests and generate documentation for airbnb models
  - build streamlit dashboard on top of dbt marts
  - troubleshoot dbt incremental merge strategy in snowflake
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a complete analytics engineering workflow: loading Inside Airbnb open data into Snowflake, transforming it through dbt staging/intermediate/mart layers with incremental models, validating data quality with tests, and serving metrics through a Streamlit dashboard.

## What This Project Does

- **Data Loading**: Python script uploads Inside Airbnb CSV/GZIP files to Snowflake internal stage and RAW schema
- **dbt Transformation**: Multi-layer dbt project (staging → intermediate → marts) with incremental fact tables
- **Data Quality**: Generic and singular dbt tests for uniqueness, relationships, accepted values, non-negative prices
- **Analytics**: Dimensional models (listings, hosts) and fact tables (calendar, reviews) with monthly aggregates
- **Visualization**: Streamlit dashboard reading from analytics marts

The project uses Inside Airbnb data (listings, calendar, reviews, neighbourhoods) as a classroom-friendly dataset since real booking transactions are not available.

## Installation

```bash
# Clone and navigate to project
git clone <repository-url>
cd Snowflake_DBT_Project

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

**Requirements.txt includes:**
- `dbt-snowflake`
- `snowflake-connector-python`
- `streamlit`
- `pandas`
- `python-dotenv`

## Configuration

### 1. Create Local Credentials (Never Commit)

```bash
# Copy example files
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

### 2. Configure profiles.yml

```yaml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT  # e.g., xy12345.us-east-1
      user: YOUR_USERNAME
      password: YOUR_PASSWORD
      role: YOUR_ROLE  # e.g., ACCOUNTADMIN
      database: AIRBNB_ANALYTICS
      warehouse: COMPUTE_WH
      schema: DEV
      threads: 4
      client_session_keep_alive: False
```

**Environment variable alternative:**

```yaml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: "{{ env_var('SNOWFLAKE_ROLE') }}"
      database: AIRBNB_ANALYTICS
      warehouse: COMPUTE_WH
      schema: DEV
      threads: 4
```

### 3. Configure local_credentials.json (for Streamlit)

```json
{
  "snowflake": {
    "account": "YOUR_ACCOUNT",
    "user": "YOUR_USERNAME",
    "password": "YOUR_PASSWORD",
    "warehouse": "COMPUTE_WH",
    "database": "AIRBNB_ANALYTICS",
    "schema": "MARTS",
    "role": "YOUR_ROLE"
  }
}
```

**Never commit these files** — they are in `.gitignore`.

## Data Acquisition

Download Inside Airbnb data for your city (recommended: New York City):

```bash
# Create data directory
mkdir -p data/raw

# Download from http://insideairbnb.com/get-the-data/
# Expected files:
# data/raw/listings.csv.gz
# data/raw/calendar.csv.gz
# data/raw/reviews.csv.gz
# data/raw/neighbourhoods.csv
```

## Loading Data into Snowflake

The Python loader creates Snowflake objects, uploads files to internal stage, and copies into RAW schema:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What the loader does:**

1. Reads credentials from `config/local_credentials.json`
2. Executes `setup/snowflake_setup.sql` to create database, schemas, stage, warehouse
3. Uploads local CSV/GZIP files to `INSIDE_AIRBNB_STAGE`
4. Creates RAW tables with inferred schemas
5. Copies stage data into RAW tables using `COPY INTO`

**Key code pattern from loader:**

```python
import snowflake.connector
import json

# Load credentials
with open('config/local_credentials.json') as f:
    config = json.load(f)

# Connect to Snowflake
conn = snowflake.connector.connect(
    account=config['snowflake']['account'],
    user=config['snowflake']['user'],
    password=config['snowflake']['password'],
    role=config['snowflake']['role'],
    warehouse=config['snowflake']['warehouse']
)

# Upload file to stage
conn.cursor().execute(f"""
    PUT file://data/raw/listings.csv.gz 
    @AIRBNB_ANALYTICS.RAW.INSIDE_AIRBNB_STAGE
    AUTO_COMPRESS=FALSE
    OVERWRITE=TRUE
""")

# Copy into table
conn.cursor().execute("""
    COPY INTO AIRBNB_ANALYTICS.RAW.LISTINGS
    FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
    FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"')
""")
```

## dbt Project Structure

```
models/
├── staging/
│   ├── _sources.yml           # Source definitions
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
    ├── fct_listing_calendar.sql      # Incremental
    ├── fct_reviews.sql
    ├── agg_listing_monthly_performance.sql
    └── agg_neighbourhood_monthly_performance.sql
```

## Key dbt Commands

```bash
# Verify connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select dim_listings+ --profiles-dir .

# Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Test specific model
dbt test --select stg_airbnb__listings --profiles-dir .

# Generate and serve documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

## dbt Model Examples

### Staging Model: stg_airbnb__listings.sql

```sql
{{ config(
    materialized='view'
) }}

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
        price::varchar as price_text,
        minimum_nights::int as minimum_nights,
        number_of_reviews::int as number_of_reviews,
        last_review::date as last_review_date,
        reviews_per_month::float as reviews_per_month,
        calculated_host_listings_count::int as host_total_listings_count,
        availability_365::int as availability_365
    from source
)

select * from renamed
```

### Intermediate Model: int_airbnb__listing_enriched.sql

```sql
{{ config(
    materialized='view'
) }}

with listings as (
    select * from {{ ref('stg_airbnb__listings') }}
),

neighbourhoods as (
    select * from {{ ref('stg_airbnb__neighbourhoods') }}
),

enriched as (
    select
        l.*,
        -- Clean price: remove $ and commas, cast to float
        replace(replace(l.price_text, '$', ''), ',', '')::float as price,
        n.neighbourhood_group,
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

### Incremental Fact Model: fct_listing_calendar.sql

```sql
{{ config(
    materialized='incremental',
    unique_key=['listing_id', 'calendar_date'],
    merge_update_columns=['available', 'price', 'minimum_nights', 'maximum_nights']
) }}

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
        -- Revenue proxy: unavailable nights * price
        case
            when available = false and price > 0
            then price
            else 0
        end as estimated_revenue
    from calendar
    {% if is_incremental() %}
        where calendar_date > (select max(calendar_date) from {{ this }})
    {% endif %}
)

select * from final
```

**Incremental strategy:** Uses Snowflake `MERGE` to upsert based on `listing_id + calendar_date` composite key.

### Aggregate Model: agg_listing_monthly_performance.sql

```sql
{{ config(
    materialized='table'
) }}

with calendar_facts as (
    select * from {{ ref('fct_listing_calendar') }}
),

monthly_metrics as (
    select
        listing_id,
        date_trunc('month', calendar_date) as month_start,
        count(*) as total_days,
        sum(case when available = false then 1 else 0 end) as unavailable_days,
        avg(price) as avg_daily_price,
        sum(estimated_revenue) as total_estimated_revenue,
        round(
            (sum(case when available = false then 1 else 0 end)::float / count(*)) * 100,
            2
        ) as occupancy_rate_pct
    from calendar_facts
    group by listing_id, date_trunc('month', calendar_date)
)

select * from monthly_metrics
```

## dbt Tests

### Schema Tests (models/staging/schema.yml)

```yaml
version: 2

models:
  - name: stg_airbnb__listings
    description: Cleaned and typed listing records
    columns:
      - name: listing_id
        description: Unique listing identifier
        tests:
          - unique
          - not_null
      - name: room_type
        description: Type of accommodation
        tests:
          - accepted_values:
              values: ['Entire home/apt', 'Private room', 'Shared room', 'Hotel room']

  - name: stg_airbnb__calendar
    columns:
      - name: listing_id
        tests:
          - not_null
          - relationships:
              to: ref('stg_airbnb__listings')
              field: listing_id
      - name: calendar_date
        tests:
          - not_null
```

### Singular Test (tests/assert_no_negative_prices.sql)

```sql
-- Ensure no listings have negative prices in enriched layer
select
    listing_id,
    price
from {{ ref('int_airbnb__listing_enriched') }}
where price < 0
```

This test fails if any rows are returned.

## Running the Streamlit Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

**Dashboard code pattern:**

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load credentials
with open('config/local_credentials.json') as f:
    config = json.load(f)['snowflake']

@st.cache_resource
def get_snowflake_connection():
    return snowflake.connector.connect(
        account=config['account'],
        user=config['user'],
        password=config['password'],
        warehouse=config['warehouse'],
        database=config['database'],
        schema=config['schema'],
        role=config['role']
    )

# Query aggregates
conn = get_snowflake_connection()
query = """
    SELECT
        month_start,
        SUM(total_estimated_revenue) as monthly_revenue,
        AVG(occupancy_rate_pct) as avg_occupancy
    FROM agg_listing_monthly_performance
    GROUP BY month_start
    ORDER BY month_start
"""
df = pd.read_sql(query, conn)

# Visualize
st.line_chart(df.set_index('month_start')['monthly_revenue'])
```

## Common Workflows

### Initial Setup and Full Run

```bash
# 1. Install and configure
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
# Edit profiles.yml and local_credentials.json with real values

# 2. Load data
python scripts/load_inside_airbnb_to_snowflake.py

# 3. Run dbt
dbt debug --profiles-dir .
dbt run --profiles-dir .
dbt test --profiles-dir .
dbt docs generate --profiles-dir .

# 4. Launch dashboard
streamlit run dashboard/streamlit_app.py
```

### Updating with New Snapshot

```bash
# 1. Download new Inside Airbnb files to data/raw/

# 2. Reload raw tables
python scripts/load_inside_airbnb_to_snowflake.py

# 3. Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 4. Test
dbt test --profiles-dir .
```

### Incremental-Only Update

```bash
# If only new calendar dates arrived (append-only scenario)
dbt run --select fct_listing_calendar --profiles-dir .
```

### Debugging a Model

```bash
# Compile SQL without running
dbt compile --select dim_listings --profiles-dir .
# Check target/compiled/snowflake_dbt_project/models/marts/dim_listings.sql

# Run with verbose logging
dbt run --select dim_listings --profiles-dir . --debug

# Show query in Snowflake history
dbt show --select dim_listings --profiles-dir .
```

## Troubleshooting

### Connection Errors

**Problem:** `dbt debug` fails with "Invalid account" or authentication error.

**Solution:**
1. Verify `profiles.yml` account format: `<account_locator>.<region>` (e.g., `xy12345.us-east-1`)
2. Check user has correct role and warehouse access
3. Test connection directly:

```python
import snowflake.connector
conn = snowflake.connector.connect(
    account='YOUR_ACCOUNT',
    user='YOUR_USER',
    password='YOUR_PASSWORD'
)
print(conn.cursor().execute("SELECT CURRENT_USER()").fetchone())
```

### Incremental Model Not Updating

**Problem:** `fct_listing_calendar` shows old data after `dbt run`.

**Solution:**
1. Check `is_incremental()` filter — it may be excluding all new rows
2. Force full refresh: `dbt run --full-refresh --select fct_listing_calendar`
3. Verify unique key matches source data: `listing_id` + `calendar_date` should be truly unique

### Data Loader Fails to Upload

**Problem:** `PUT` command fails with "File not found" error.

**Solution:**
1. Verify files exist: `ls -lh data/raw/`
2. Check file paths are absolute or relative to script location
3. Ensure Snowflake stage exists: `SHOW STAGES IN AIRBNB_ANALYTICS.RAW;`

### Tests Failing on Relationships

**Problem:** `relationships` test fails for `calendar.listing_id → listings.listing_id`.

**Solution:**
1. Calendar may have orphaned listing IDs from old data
2. Add filter to staging model:

```sql
select * from source
where listing_id in (select listing_id from {{ ref('stg_airbnb__listings') }})
```

3. Or accept the test failure and document it as data quality issue

### Streamlit Can't Connect

**Problem:** Dashboard shows "Unable to connect to Snowflake".

**Solution:**
1. Verify `config/local_credentials.json` exists and has correct values
2. Test connection manually:

```python
import json
import snowflake.connector

with open('config/local_credentials.json') as f:
    config = json.load(f)['snowflake']

conn = snowflake.connector.connect(**config)
print(conn.cursor().execute("SELECT CURRENT_DATABASE()").fetchone())
```

3. Check Snowflake firewall allows your IP

### Performance Issues with Large Calendar Table

**Problem:** `fct_listing_calendar` takes too long to rebuild.

**Solution:**
1. Use incremental materialization (already configured)
2. Partition by month in Snowflake:

```sql
{{ config(
    materialized='incremental',
    unique_key=['listing_id', 'calendar_date'],
    cluster_by=['calendar_date']
) }}
```

3. Limit calendar to next 365 days in staging:

```sql
where calendar_date <= current_date + interval '365 days'
```

## Advanced Patterns

### Using dbt Macros for Repeated Logic

Create `macros/clean_price.sql`:

```sql
{% macro clean_price(price_column) %}
    replace(replace({{ price_column }}, '$', ''), ',', '')::float
{% endmacro %}
```

Use in models:

```sql
select
    listing_id,
    {{ clean_price('price_text') }} as price
from {{ ref('stg_airbnb__listings') }}
```

### Environment-Specific Schemas

Update `dbt_project.yml`:

```yaml
models:
  snowflake_dbt_project:
    staging:
      +schema: staging
    intermediate:
      +schema: intermediate
    marts:
      +schema: marts
```

This creates `DEV_STAGING`, `DEV_INTERMEDIATE`, `DEV_MARTS` in dev target.

### CI/CD Integration

```bash
# Install dbt in GitHub Actions
- name: Install dbt
  run: pip install dbt-snowflake

# Run dbt with environment variables
- name: Run dbt
  env:
    SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
    SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
    SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
  run: |
    dbt run --profiles-dir . --target prod
    dbt test --profiles-dir . --target prod
```

## Key Takeaways

- **Separation of concerns**: RAW (immutable) → STAGING (clean) → INTERMEDIATE (enrich) → MARTS (analytics)
- **Incremental strategy**: Use `unique_key` + `merge_update_columns` for Snowflake merge upserts
- **Security**: Never commit `profiles.yml` or `local_credentials.json` — use env vars in CI/CD
- **Testing**: Combine generic tests (uniqueness, relationships) with singular tests (business logic)
- **Documentation**: `dbt docs generate` produces interactive lineage graphs from model relationships
