---
name: snowflake-dbt-airbnb-analytics
description: Build an end-to-end Snowflake + dbt analytics pipeline for Inside Airbnb data with staging, marts, tests, and Streamlit dashboard
triggers:
  - set up snowflake dbt airbnb project
  - load inside airbnb data to snowflake
  - build dbt models for airbnb analytics
  - create dbt staging and mart layers
  - run dbt incremental models in snowflake
  - build streamlit dashboard for airbnb data
  - test dbt data quality for airbnb
  - configure dbt profiles for snowflake
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build an end-to-end analytics engineering pipeline using Snowflake as the data warehouse, dbt for transformation modeling, and Streamlit for dashboarding. The project demonstrates professional-grade data modeling with the Inside Airbnb open dataset, including staging layers, intermediate enrichment, dimension/fact marts, incremental models, data quality tests, and documentation generation.

## What This Project Does

The Snowflake_DBT_Project loads Inside Airbnb CSV files (listings, calendar, reviews, neighbourhoods) into Snowflake raw tables, transforms them through dbt staging and intermediate layers, materializes dimension and fact tables in analytics marts, runs data quality tests, and powers a Streamlit dashboard for neighbourhood and host insights.

**Key capabilities:**
- Snowflake internal stage loading from local CSV/GZIP files
- dbt staging views with type casting and standardization
- dbt intermediate layer for joins, enrichment, and revenue proxy logic
- dbt marts: `dim_listings`, `dim_hosts`, `fct_listing_calendar` (incremental), `fct_reviews`, and monthly aggregates
- dbt generic and singular tests for data quality validation
- dbt docs generation for model lineage and documentation
- Streamlit dashboard reading from Snowflake analytics marts

## Installation

### Prerequisites
- Snowflake account with appropriate warehouse, database, and schema permissions
- Python 3.8+
- Inside Airbnb dataset files downloaded to `data/raw/`

### Setup Steps

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

**Expected dependencies (requirements.txt):**
```text
dbt-snowflake>=1.5.0
snowflake-connector-python>=3.0.0
streamlit>=1.20.0
pandas>=1.5.0
```

### Configure Credentials

The project uses local, git-ignored credential files:

```bash
# Copy credential templates
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
      account: YOUR_ACCOUNT  # e.g., abc12345.us-east-1
      user: YOUR_USERNAME
      password: YOUR_PASSWORD
      role: YOUR_ROLE
      database: AIRBNB_DB
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

**Edit `config/local_credentials.json`:**
```json
{
  "account": "YOUR_ACCOUNT",
  "user": "YOUR_USERNAME",
  "password": "YOUR_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DB",
  "schema": "ANALYTICS",
  "role": "YOUR_ROLE"
}
```

> **Security:** Never commit these files. They are in `.gitignore`.

## Loading Data into Snowflake

### Download Inside Airbnb Data

Visit [Inside Airbnb](https://insideairbnb.com/get-the-data/) and download the New York City dataset:
- `listings.csv.gz`
- `calendar.csv.gz`
- `reviews.csv.gz`
- `neighbourhoods.csv`

Place files in `data/raw/`.

### Run the Loader Script

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What the loader does:**
1. Reads credentials from `config/local_credentials.json`
2. Executes `setup/snowflake_setup.sql` to create database, schemas (`RAW`, `ANALYTICS`), warehouse, and internal stage
3. Uploads CSV files to Snowflake internal stage `INSIDE_AIRBNB_STAGE`
4. Creates raw tables from CSV headers
5. Copies data from stage into `RAW` schema tables

**Loader script structure (scripts/load_inside_airbnb_to_snowflake.py):**
```python
import json
import snowflake.connector
from pathlib import Path

def load_credentials():
    with open('config/local_credentials.json', 'r') as f:
        return json.load(f)

def get_snowflake_connection(creds):
    return snowflake.connector.connect(
        user=creds['user'],
        password=creds['password'],
        account=creds['account'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema'],
        role=creds['role']
    )

def execute_setup_sql(conn):
    with open('setup/snowflake_setup.sql', 'r') as f:
        sql_commands = f.read().split(';')
    cursor = conn.cursor()
    for cmd in sql_commands:
        if cmd.strip():
            cursor.execute(cmd)
    cursor.close()

def upload_to_stage(conn, local_path, stage_name):
    cursor = conn.cursor()
    cursor.execute(f"PUT file://{local_path} @{stage_name} AUTO_COMPRESS=FALSE OVERWRITE=TRUE")
    cursor.close()

def copy_into_table(conn, stage_file, table_name):
    cursor = conn.cursor()
    cursor.execute(f"""
        COPY INTO {table_name}
        FROM @INSIDE_AIRBNB_STAGE/{stage_file}
        FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
    """)
    cursor.close()

def main():
    creds = load_credentials()
    conn = get_snowflake_connection(creds)
    
    execute_setup_sql(conn)
    
    data_files = [
        ('data/raw/listings.csv.gz', 'listings.csv.gz', 'RAW.LISTINGS'),
        ('data/raw/calendar.csv.gz', 'calendar.csv.gz', 'RAW.CALENDAR'),
        ('data/raw/reviews.csv.gz', 'reviews.csv.gz', 'RAW.REVIEWS'),
        ('data/raw/neighbourhoods.csv', 'neighbourhoods.csv', 'RAW.NEIGHBOURHOODS')
    ]
    
    for local_file, stage_file, table in data_files:
        print(f"Uploading {local_file}...")
        upload_to_stage(conn, local_file, 'INSIDE_AIRBNB_STAGE')
        print(f"Copying into {table}...")
        copy_into_table(conn, stage_file, table)
    
    conn.close()
    print("Data load complete!")

if __name__ == '__main__':
    main()
```

## dbt Project Structure

```text
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

## Key dbt Commands

```bash
# Test connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model
dbt run --select stg_airbnb__listings --profiles-dir .

# Run model and all downstream dependencies
dbt run --select stg_airbnb__listings+ --profiles-dir .

# Run only mart layer
dbt run --select marts --profiles-dir .

# Full refresh for incremental models
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Test specific model
dbt test --select dim_listings --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .

# Serve documentation site
dbt docs serve --profiles-dir .

# Compile without running
dbt compile --profiles-dir .

# Show model lineage
dbt ls --select +fct_listing_calendar+ --profiles-dir .
```

## dbt Model Examples

### Staging Layer: stg_airbnb__listings.sql

```sql
-- models/staging/stg_airbnb__listings.sql
with source as (
    select * from {{ source('raw_airbnb', 'listings') }}
),

renamed as (
    select
        id::bigint as listing_id,
        name::varchar as listing_name,
        host_id::bigint as host_id,
        host_name::varchar as host_name,
        neighbourhood::varchar as neighbourhood,
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

select * from renamed
```

### Staging Layer: stg_airbnb__calendar.sql

```sql
-- models/staging/stg_airbnb__calendar.sql
with source as (
    select * from {{ source('raw_airbnb', 'calendar') }}
),

renamed as (
    select
        listing_id::bigint as listing_id,
        date::date as calendar_date,
        available::varchar as available_raw,
        price::varchar as price_raw
    from source
)

select * from renamed
```

### Intermediate Layer: int_airbnb__listing_enriched.sql

```sql
-- models/intermediate/int_airbnb__listing_enriched.sql
with listings as (
    select * from {{ ref('stg_airbnb__listings') }}
),

neighbourhoods as (
    select * from {{ ref('stg_airbnb__neighbourhoods') }}
),

cleaned as (
    select
        listing_id,
        listing_name,
        host_id,
        host_name,
        listings.neighbourhood,
        neighbourhoods.neighbourhood_group,
        latitude,
        longitude,
        room_type,
        -- Clean price: remove $ and convert to numeric
        regexp_replace(price_raw, '[^0-9.]', '')::float as price,
        minimum_nights,
        number_of_reviews,
        last_review_date,
        reviews_per_month,
        host_listings_count,
        availability_365,
        case 
            when availability_365 > 0 then 'available'
            else 'unavailable'
        end as availability_status
    from listings
    left join neighbourhoods
        on listings.neighbourhood = neighbourhoods.neighbourhood
)

select * from cleaned
```

### Intermediate Layer: int_airbnb__calendar_enriched.sql

```sql
-- models/intermediate/int_airbnb__calendar_enriched.sql
with calendar as (
    select * from {{ ref('stg_airbnb__calendar') }}
),

listings as (
    select * from {{ ref('int_airbnb__listing_enriched') }}
),

enriched as (
    select
        calendar.listing_id,
        calendar_date,
        case 
            when lower(available_raw) = 't' then true
            when lower(available_raw) = 'f' then false
            else null
        end as is_available,
        -- Clean price
        regexp_replace(price_raw, '[^0-9.]', '')::float as price,
        listings.listing_name,
        listings.neighbourhood,
        listings.neighbourhood_group,
        listings.room_type,
        listings.host_id,
        listings.host_name,
        -- Revenue proxy: if unavailable, assume booked
        case
            when lower(available_raw) = 'f' 
            then regexp_replace(price_raw, '[^0-9.]', '')::float
            else 0
        end as estimated_revenue
    from calendar
    inner join listings
        on calendar.listing_id = listings.listing_id
)

select * from enriched
```

### Mart: dim_listings.sql

```sql
-- models/marts/dim_listings.sql
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
    host_name,
    neighbourhood,
    neighbourhood_group,
    latitude,
    longitude,
    room_type,
    price,
    minimum_nights,
    number_of_reviews,
    last_review_date,
    reviews_per_month,
    host_listings_count,
    availability_365,
    availability_status,
    current_timestamp() as dbt_loaded_at
from listings
```

### Mart: fct_listing_calendar.sql (Incremental)

```sql
-- models/marts/fct_listing_calendar.sql
{{
    config(
        materialized='incremental',
        unique_key='calendar_key',
        incremental_strategy='merge',
        on_schema_change='fail'
    )
}}

with calendar_enriched as (
    select * from {{ ref('int_airbnb__calendar_enriched') }}
)

select
    {{ dbt_utils.generate_surrogate_key(['listing_id', 'calendar_date']) }} as calendar_key,
    listing_id,
    calendar_date,
    is_available,
    price,
    estimated_revenue,
    neighbourhood,
    neighbourhood_group,
    room_type,
    host_id,
    current_timestamp() as dbt_loaded_at
from calendar_enriched

{% if is_incremental() %}
    where calendar_date > (select max(calendar_date) from {{ this }})
{% endif %}
```

### Mart: agg_listing_monthly_performance.sql

```sql
-- models/marts/agg_listing_monthly_performance.sql
{{
    config(
        materialized='table'
    )
}}

with calendar_facts as (
    select * from {{ ref('fct_listing_calendar') }}
)

select
    listing_id,
    date_trunc('month', calendar_date) as month_start,
    neighbourhood,
    neighbourhood_group,
    room_type,
    count(*) as total_days,
    sum(case when is_available = false then 1 else 0 end) as unavailable_days,
    sum(case when is_available = true then 1 else 0 end) as available_days,
    round(avg(price), 2) as avg_price,
    round(sum(estimated_revenue), 2) as total_estimated_revenue,
    round(sum(case when is_available = false then 1 else 0 end)::float / count(*) * 100, 2) as occupancy_rate
from calendar_facts
group by 1, 2, 3, 4, 5
```

### Mart: agg_neighbourhood_monthly_performance.sql

```sql
-- models/marts/agg_neighbourhood_monthly_performance.sql
{{
    config(
        materialized='table'
    )
}}

with listing_monthly as (
    select * from {{ ref('agg_listing_monthly_performance') }}
)

select
    neighbourhood_group,
    neighbourhood,
    month_start,
    count(distinct listing_id) as total_listings,
    sum(total_days) as total_listing_days,
    sum(unavailable_days) as total_unavailable_days,
    round(avg(avg_price), 2) as avg_listing_price,
    round(sum(total_estimated_revenue), 2) as total_estimated_revenue,
    round(avg(occupancy_rate), 2) as avg_occupancy_rate
from listing_monthly
group by 1, 2, 3
order by month_start, total_estimated_revenue desc
```

## dbt Source Configuration

**models/staging/_sources.yml:**
```yaml
version: 2

sources:
  - name: raw_airbnb
    description: Raw Inside Airbnb data loaded into Snowflake
    database: AIRBNB_DB
    schema: RAW
    tables:
      - name: listings
        description: Raw listings data
        columns:
          - name: id
            description: Unique listing identifier
            tests:
              - not_null
              - unique
      
      - name: calendar
        description: Raw calendar availability and pricing
        columns:
          - name: listing_id
            description: Foreign key to listings
            tests:
              - not_null
          - name: date
            description: Calendar date
            tests:
              - not_null
      
      - name: reviews
        description: Raw reviews data
        columns:
          - name: listing_id
            description: Foreign key to listings
            tests:
              - not_null
          - name: id
            description: Unique review identifier
            tests:
              - not_null
              - unique
      
      - name: neighbourhoods
        description: Neighbourhood reference data
        columns:
          - name: neighbourhood
            description: Neighbourhood name
            tests:
              - not_null
              - unique
```

## dbt Tests

### Generic Tests (schema.yml)

**models/marts/schema.yml:**
```yaml
version: 2

models:
  - name: dim_listings
    description: Dimension table for Airbnb listings
    columns:
      - name: listing_id
        description: Unique listing identifier
        tests:
          - not_null
          - unique
      - name: room_type
        description: Type of room
        tests:
          - accepted_values:
              values: ['Entire home/apt', 'Private room', 'Shared room', 'Hotel room']
      - name: price
        description: Daily listing price
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0
              inclusive: true

  - name: fct_listing_calendar
    description: Fact table for listing calendar availability
    columns:
      - name: calendar_key
        description: Surrogate key for listing-date combination
        tests:
          - not_null
          - unique
      - name: listing_id
        description: Foreign key to dim_listings
        tests:
          - not_null
          - relationships:
              to: ref('dim_listings')
              field: listing_id
      - name: calendar_date
        description: Calendar date
        tests:
          - not_null
      - name: price
        description: Price for this date
        tests:
          - dbt_utils.accepted_range:
              min_value: 0
              inclusive: true
```

### Singular Tests

**tests/assert_no_duplicate_listing_dates.sql:**
```sql
-- Test that there are no duplicate listing-date combinations
select
    listing_id,
    calendar_date,
    count(*) as occurrences
from {{ ref('fct_listing_calendar') }}
group by listing_id, calendar_date
having count(*) > 1
```

**tests/assert_positive_estimated_revenue.sql:**
```sql
-- Test that estimated revenue is non-negative
select *
from {{ ref('fct_listing_calendar') }}
where estimated_revenue < 0
```

## Streamlit Dashboard

**dashboard/streamlit_app.py:**
```python
import streamlit as st
import pandas as pd
import snowflake.connector
import json
from pathlib import Path

def load_credentials():
    with open('config/local_credentials.json', 'r') as f:
        return json.load(f)

@st.cache_resource
def get_snowflake_connection():
    creds = load_credentials()
    return snowflake.connector.connect(
        user=creds['user'],
        password=creds['password'],
        account=creds['account'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema'],
        role=creds['role']
    )

def run_query(query):
    conn = get_snowflake_connection()
    cursor = conn.cursor()
    cursor.execute(query)
    df = cursor.fetch_pandas_all()
    cursor.close()
    return df

st.set_page_config(page_title="Airbnb Analytics", layout="wide")

st.title("🏠 Inside Airbnb Analytics Dashboard")
st.markdown("Powered by Snowflake + dbt")

# Sidebar filters
st.sidebar.header("Filters")
selected_month = st.sidebar.selectbox(
    "Select Month",
    options=run_query("SELECT DISTINCT MONTH_START FROM AGG_NEIGHBOURHOOD_MONTHLY_PERFORMANCE ORDER BY MONTH_START DESC")['MONTH_START']
)

# Top neighbourhoods by revenue
st.header("Top Neighbourhoods by Estimated Revenue")
neighbourhood_revenue_query = f"""
SELECT
    NEIGHBOURHOOD,
    NEIGHBOURHOOD_GROUP,
    TOTAL_LISTINGS,
    TOTAL_ESTIMATED_REVENUE,
    AVG_OCCUPANCY_RATE
FROM AGG_NEIGHBOURHOOD_MONTHLY_PERFORMANCE
WHERE MONTH_START = '{selected_month}'
ORDER BY TOTAL_ESTIMATED_REVENUE DESC
LIMIT 10
"""
df_neighbourhoods = run_query(neighbourhood_revenue_query)
st.dataframe(df_neighbourhoods, use_container_width=True)

# Top hosts by listing count
st.header("Top Hosts by Listing Count")
host_query = """
SELECT
    HOST_ID,
    HOST_NAME,
    COUNT(DISTINCT LISTING_ID) AS TOTAL_LISTINGS,
    AVG(PRICE) AS AVG_PRICE
FROM DIM_LISTINGS
GROUP BY HOST_ID, HOST_NAME
ORDER BY TOTAL_LISTINGS DESC
LIMIT 10
"""
df_hosts = run_query(host_query)
st.dataframe(df_hosts, use_container_width=True)

# Room type breakdown
st.header("Listings by Room Type")
room_type_query = """
SELECT
    ROOM_TYPE,
    COUNT(*) AS LISTING_COUNT,
    ROUND(AVG(PRICE), 2) AS AVG_PRICE
FROM DIM_LISTINGS
GROUP BY ROOM_TYPE
ORDER BY LISTING_COUNT DESC
"""
df_room_types = run_query(room_type_query)
st.bar_chart(df_room_types.set_index('ROOM_TYPE')['LISTING_COUNT'])

# Monthly trend
st.header("Monthly Revenue Trend")
monthly_trend_query = """
SELECT
    MONTH_START,
    SUM(TOTAL_ESTIMATED_REVENUE) AS TOTAL_REVENUE
FROM AGG_NEIGHBOURHOOD_MONTHLY_PERFORMANCE
GROUP BY MONTH_START
ORDER BY MONTH_START
"""
df_trend = run_query(monthly_trend_query)
st.line_chart(df_trend.set_index('MONTH_START'))
```

**Run the dashboard:**
```bash
streamlit run dashboard/streamlit_app.py
```

## Common Patterns

### Full Refresh Workflow

When loading new Inside Airbnb snapshot data:

```bash
# 1. Download new data to data/raw/
# 2. Reload into Snowflake
python scripts/load_inside_airbnb_to_snowflake.py

# 3. Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 4. Run tests
dbt test --profiles-dir .

# 5. Regenerate docs
dbt docs generate --profiles-dir .
```

### Selective Model Execution

```bash
# Run only staging layer
dbt run --select staging --profiles-dir .

# Run a specific model and its downstream dependencies
dbt run --select stg_airbnb__calendar+ --profiles-dir .

# Run all models upstream of a specific model
dbt run --select +agg_neighbourhood_monthly_performance --profiles-dir .

# Run models matching a tag
dbt run --select tag:daily --profiles-dir .
```

### Testing Strategy

```bash
# Test all models
dbt test --profiles-dir .

# Test sources
dbt test --select source:* --profiles-dir .

# Test specific layer
dbt test --select marts --profiles-dir .

# Test one model and its relationships
dbt test --select fct_listing_calendar --profiles-dir .
```

## Configuration

### dbt_project.yml

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

### packages.yml (Optional)

For utilities like `dbt_utils`:

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1
```

Install packages:
```bash
dbt deps --profiles-dir .
```

## Troubleshooting

### Connection Issues

**Problem:** `dbt debug` fails with authentication error

**Solution:**
- Verify `profiles.yml` credentials match your Snowflake account
- Test connection directly:
  ```bash
  dbt debug --profiles-dir .
  ```
- Check Snowflake account identifier format: `<account>.<region>` or `<orgname>-<account>`

### Loader Script Fails

**Problem:** `load_inside_airbnb_to_snowflake.py` raises permission error

**Solution:**
- Ensure Snowflake role has `CREATE DATABASE`, `CREATE SCHEMA`, `CREATE TABLE`, `CREATE STAGE` privileges
- Run setup SQL manually in Snowflake UI first:
  ```sql
  CREATE DATABASE IF NOT EXISTS AIRBNB_DB;
  CREATE SCHEMA IF NOT EXISTS AIRBNB_DB.RAW;
  CREATE WAREHOUSE IF NOT EXISTS COMPUTE_WH WITH WAREHOUSE_SIZE='XSMALL';
  ```

### Incremental Model Not Updating

**Problem:** `fct_listing_calendar` doesn't update with new data

**Solution:**
- Force full refresh:
  ```bash
  dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
  ```
- Check incremental logic in model (date filter)
- Verify new data exists in staging layer:
  ```sql
  SELECT MAX(calendar_date) FROM ANALYTICS.int_airbnb__calendar_enriched;
  ```

### Test Failures

**Problem:** `dbt test` fails on relationship test

**Solution:**
- Check if referenced table exists:
  ```bash
  dbt run --select dim_listings --profiles-dir .
  ```
- Inspect test results:
  ```bash
  dbt test --store-failures --profiles-dir .
  ```
- Query stored failures:
  ```sql
  SELECT * FROM ANALYTICS.dim_listings__relationships_test;
  ```

### Streamlit Dashboard Errors

**Problem:** Dashboard can't connect to Snowflake

**Solution:**
- Verify `config/local_credentials.json` exists and has correct values
- Test connection outside Streamlit:
  ```python
  import snowflake.connector
  import json
  with open('config/local_credentials.json') as f:
      creds = json.load(f)
  conn = snowflake.connector.connect(**creds)
  print(conn.cursor().execute("SELECT CURRENT_USER()").fetchone())
  ```

### Performance Issues

**Problem:** dbt models run slowly

**Solution:**
- Use larger Snowflake warehouse for dev target in `profiles.yml`:
  ```yaml
  warehouse: COMPUTE_WH_MEDIUM
  ```
- Optimize incremental models with clustering:
  ```sql
  {{ config(
      materialized='incremental',
      cluster_by=['calendar_date']
  ) }}
  ```
- Check query profile in Snowflake UI

## Best Practices

1. **Credential Management:** Never commit `profiles.yml` or `local_credentials.json`
2. **Incremental Strategy:** Use `merge` strategy for Snowflake incremental models
3. **Testing:** Add tests for all primary keys, foreign keys, and business logic
4. **Documentation:** Add descriptions to all models in `schema.yml` files
5. **Modularity:** Use intermediate models for reusable transformations
6. **Version Control:** Tag dbt releases and document data lineage changes

## Advanced Usage

### Custom Macros

**macros/clean_price.sql:**
```sql
{% macro clean_price(price_column) %}
    regexp_replace({{ price_column }}, '[^0-9.]', '')::float
{% endmacro %}
```

Use in models:
```sql
select
    listing_id,
    {{ clean_price('price_raw') }} as price
from {{ ref('stg_airbnb__listings') }}
```

### Exposures

Document dashboard dependencies in **models/exposures.yml:**
```yaml
version: 2

exposures:
  - name: airbnb_streamlit_dashboard
    type: dashboard
    maturity: medium
    url: http://localhost:8501
    description: Streamlit dashboard for Airbnb neighbourhood analytics
    depends_on:
      - ref('dim_listings')
      - ref('dim_hosts')
      - ref('agg_neighbourhood_monthly_performance')
    owner:
      name: Durgesh Yadav
      email: durgesh@example.com
```

---

This skill provides comprehensive guidance for deploying a production-grade Snowflake + dbt analytics pipeline with the Inside Airbnb dataset, including data loading, transformation modeling, testing, documentation, and dashboard deployment.
