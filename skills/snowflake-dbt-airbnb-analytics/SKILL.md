---
name: snowflake-dbt-airbnb-analytics
description: Build Snowflake data warehouses with dbt for Inside Airbnb analytics - staging, marts, incremental models, and Streamlit dashboards
triggers:
  - set up a Snowflake dbt project for Airbnb data
  - load Inside Airbnb data into Snowflake
  - create dbt staging and mart models for analytics
  - build incremental fact tables in dbt with Snowflake
  - configure dbt profiles for Snowflake connections
  - create a Streamlit dashboard with Snowflake data
  - add dbt tests for data quality validation
  - design analytics engineering pipelines
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end analytics engineering projects using Snowflake as a data warehouse, dbt for transformation, and Streamlit for dashboards. It demonstrates loading open data (Inside Airbnb), modeling in layers (staging, intermediate, marts), incremental patterns, and data quality testing.

## What This Project Does

The `Snowflake_DBT_Project` demonstrates a complete analytics engineering workflow:

1. **Raw data ingestion**: Load CSV/GZIP files from Inside Airbnb into Snowflake internal stages
2. **dbt transformation layers**: Build staging (clean/cast), intermediate (joins/enrichment), and mart models (dimensions, facts, aggregates)
3. **Incremental modeling**: Use Snowflake merge strategy for efficient fact table updates
4. **Data quality**: Generic and singular dbt tests for validation
5. **Dashboard**: Streamlit app connected to Snowflake analytics marts

**Dataset**: Inside Airbnb (NYC recommended) - `listings.csv.gz`, `calendar.csv.gz`, `reviews.csv.gz`, `neighbourhoods.csv`

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

**Requirements.txt typically includes**:
- `dbt-snowflake` - dbt adapter for Snowflake
- `snowflake-connector-python` - Snowflake Python connector
- `streamlit` - Dashboard framework
- `pandas` - Data manipulation

## Configuration

### Snowflake Credentials Setup

**Never commit credentials**. Use local-only config files:

```bash
# Copy examples
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

**profiles.yml** (dbt Snowflake connection):

```yaml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT
      user: YOUR_USERNAME
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"  # Or paste directly (local only)
      role: ACCOUNTADMIN
      database: AIRBNB_DB
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

**config/local_credentials.json** (for Python loader and Streamlit):

```json
{
  "account": "YOUR_ACCOUNT",
  "user": "YOUR_USERNAME",
  "password": "YOUR_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DB",
  "schema": "RAW",
  "role": "ACCOUNTADMIN"
}
```

**Environment variable approach** (recommended for production):

```bash
export SNOWFLAKE_ACCOUNT=xyz12345
export SNOWFLAKE_USER=dbt_user
export SNOWFLAKE_PASSWORD=secure_password
export SNOWFLAKE_WAREHOUSE=COMPUTE_WH
export SNOWFLAKE_DATABASE=AIRBNB_DB
```

Then reference in `profiles.yml`:

```yaml
password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
```

## Loading Raw Data into Snowflake

The project includes a Python loader script:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What the loader does**:
1. Runs `setup/snowflake_setup.sql` to create database, schemas, stage
2. Uploads local files from `data/raw/` to Snowflake internal stage
3. Infers CSV structure and creates raw tables
4. Copies staged data into raw tables

**Example loader code pattern** (`scripts/load_inside_airbnb_to_snowflake.py`):

```python
import json
import snowflake.connector
from pathlib import Path

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
    role=creds.get('role', 'ACCOUNTADMIN')
)

cursor = conn.cursor()

# Execute setup SQL
setup_sql = Path('setup/snowflake_setup.sql').read_text()
for statement in setup_sql.split(';'):
    if statement.strip():
        cursor.execute(statement)

# Upload files to stage
raw_files = [
    'data/raw/listings.csv.gz',
    'data/raw/calendar.csv.gz',
    'data/raw/reviews.csv.gz',
    'data/raw/neighbourhoods.csv'
]

for file_path in raw_files:
    cursor.execute(f"PUT file://{file_path} @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")
    print(f"Uploaded {file_path}")

# Create raw table from staged file
cursor.execute("""
CREATE OR REPLACE TABLE RAW.LISTINGS AS
SELECT $1 AS raw_json
FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
(FILE_FORMAT => 'CSV_FORMAT')
""")

# Copy into raw table with column mapping
cursor.execute("""
COPY INTO RAW.LISTINGS
FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
FILE_FORMAT = (TYPE=CSV SKIP_HEADER=1 FIELD_OPTIONALLY_ENCLOSED_BY='"')
ON_ERROR = CONTINUE
""")

conn.close()
```

**Snowflake setup SQL pattern** (`setup/snowflake_setup.sql`):

```sql
-- Database and schemas
CREATE DATABASE IF NOT EXISTS AIRBNB_DB;
USE DATABASE AIRBNB_DB;

CREATE SCHEMA IF NOT EXISTS RAW;
CREATE SCHEMA IF NOT EXISTS ANALYTICS;

-- Internal stage for raw files
CREATE OR REPLACE STAGE INSIDE_AIRBNB_STAGE;

-- File format
CREATE OR REPLACE FILE FORMAT CSV_FORMAT
  TYPE = CSV
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  SKIP_HEADER = 1
  NULL_IF = ('NULL', 'null', '');

-- Raw tables (text-preserving)
CREATE OR REPLACE TABLE RAW.LISTINGS (
  id VARCHAR,
  listing_url VARCHAR,
  name VARCHAR,
  host_id VARCHAR,
  host_name VARCHAR,
  neighbourhood VARCHAR,
  latitude VARCHAR,
  longitude VARCHAR,
  room_type VARCHAR,
  price VARCHAR,
  minimum_nights VARCHAR,
  availability_365 VARCHAR
  -- Add all columns as VARCHAR for raw zone
);
```

## dbt Model Layers

### Project Structure

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

### Staging Layer

**Purpose**: Clean raw data, cast types, standardize column names.

**models/staging/_sources.yml**:

```yaml
version: 2

sources:
  - name: raw_airbnb
    database: AIRBNB_DB
    schema: RAW
    tables:
      - name: listings
        description: Raw Inside Airbnb listings data
        columns:
          - name: id
            tests:
              - not_null
              - unique
      - name: calendar
        description: Raw calendar availability and pricing
      - name: reviews
        description: Raw guest reviews
      - name: neighbourhoods
        description: Neighbourhood boundaries
```

**models/staging/stg_airbnb__listings.sql**:

```sql
{{
  config(
    materialized='view',
    tags=['staging', 'airbnb']
  )
}}

with source as (
  select * from {{ source('raw_airbnb', 'listings') }}
),

cleaned as (
  select
    cast(id as integer) as listing_id,
    trim(name) as listing_name,
    cast(host_id as integer) as host_id,
    trim(host_name) as host_name,
    trim(neighbourhood) as neighbourhood,
    cast(latitude as float) as latitude,
    cast(longitude as float) as longitude,
    trim(room_type) as room_type,
    -- Clean price: remove $ and commas, cast to decimal
    cast(
      replace(replace(price, '$', ''), ',', '') as decimal(10,2)
    ) as price,
    cast(minimum_nights as integer) as minimum_nights,
    cast(availability_365 as integer) as availability_365,
    current_timestamp() as dbt_loaded_at
  from source
  where id is not null
)

select * from cleaned
```

**models/staging/stg_airbnb__calendar.sql**:

```sql
{{
  config(
    materialized='view'
  )
}}

with source as (
  select * from {{ source('raw_airbnb', 'calendar') }}
),

cleaned as (
  select
    cast(listing_id as integer) as listing_id,
    cast(date as date) as calendar_date,
    case
      when lower(trim(available)) in ('t', 'true', '1') then true
      when lower(trim(available)) in ('f', 'false', '0') then false
      else null
    end as is_available,
    cast(
      replace(replace(price, '$', ''), ',', '') as decimal(10,2)
    ) as price,
    cast(
      replace(replace(adjusted_price, '$', ''), ',', '') as decimal(10,2)
    ) as adjusted_price
  from source
  where listing_id is not null
    and date is not null
)

select * from cleaned
```

### Intermediate Layer

**Purpose**: Reusable enriched models with joins and business logic.

**models/intermediate/int_airbnb__listing_enriched.sql**:

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
    l.*,
    n.neighbourhood_group,
    -- Classify price tier
    case
      when l.price < 100 then 'Budget'
      when l.price between 100 and 300 then 'Mid-range'
      when l.price > 300 then 'Luxury'
      else 'Unknown'
    end as price_tier,
    -- Availability category
    case
      when l.availability_365 = 0 then 'Not Available'
      when l.availability_365 < 90 then 'Low Availability'
      when l.availability_365 < 180 then 'Medium Availability'
      else 'High Availability'
    end as availability_category
  from listings l
  left join neighbourhoods n
    on l.neighbourhood = n.neighbourhood
)

select * from enriched
```

**models/intermediate/int_airbnb__calendar_enriched.sql**:

```sql
{{
  config(
    materialized='view'
  )
}}

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
    date_trunc('month', c.calendar_date) as calendar_month,
    c.is_available,
    c.price,
    c.adjusted_price,
    -- Revenue proxy: if not available, assume booking
    case
      when c.is_available = false then coalesce(c.adjusted_price, c.price, 0)
      else 0
    end as estimated_revenue,
    l.neighbourhood,
    l.neighbourhood_group,
    l.room_type,
    l.host_id
  from calendar c
  inner join listings l
    on c.listing_id = l.listing_id
)

select * from enriched
```

### Marts Layer - Dimensions

**models/marts/dim_listings.sql**:

```sql
{{
  config(
    materialized='table',
    tags=['dimension', 'listings']
  )
}}

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
  price_tier,
  minimum_nights,
  availability_365,
  availability_category,
  dbt_loaded_at
from {{ ref('int_airbnb__listing_enriched') }}
```

**models/marts/dim_hosts.sql**:

```sql
{{
  config(
    materialized='table'
  )
}}

with listings as (
  select * from {{ ref('dim_listings') }}
),

host_summary as (
  select
    host_id,
    max(host_name) as host_name,
    count(distinct listing_id) as total_listings,
    avg(price) as avg_listing_price,
    sum(availability_365) as total_availability,
    count(distinct neighbourhood) as neighbourhoods_count
  from listings
  group by host_id
)

select
  host_id,
  host_name,
  total_listings,
  round(avg_listing_price, 2) as avg_listing_price,
  total_availability,
  neighbourhoods_count,
  case
    when total_listings = 1 then 'Single Listing'
    when total_listings between 2 and 5 then 'Small Host'
    else 'Large Host'
  end as host_category
from host_summary
```

### Marts Layer - Incremental Facts

**models/marts/fct_listing_calendar.sql**:

```sql
{{
  config(
    materialized='incremental',
    unique_key=['listing_id', 'calendar_date'],
    on_schema_change='fail',
    tags=['fact', 'incremental']
  )
}}

select
  listing_id,
  calendar_date,
  calendar_month,
  is_available,
  price,
  adjusted_price,
  estimated_revenue,
  neighbourhood,
  neighbourhood_group,
  room_type,
  host_id,
  current_timestamp() as dbt_updated_at
from {{ ref('int_airbnb__calendar_enriched') }}

{% if is_incremental() %}
  -- Only load new dates
  where calendar_date > (select max(calendar_date) from {{ this }})
{% endif %}
```

**Incremental strategy**: Snowflake uses `merge` by default. On `dbt run`, only new records (dates beyond existing max) are processed.

**Full refresh**:

```bash
dbt run --full-refresh --select fct_listing_calendar
```

### Marts Layer - Aggregates

**models/marts/agg_listing_monthly_performance.sql**:

```sql
{{
  config(
    materialized='table'
  )
}}

with calendar_facts as (
  select * from {{ ref('fct_listing_calendar') }}
),

monthly_summary as (
  select
    listing_id,
    calendar_month,
    count(*) as total_days,
    sum(case when is_available = false then 1 else 0 end) as unavailable_days,
    sum(case when is_available = true then 1 else 0 end) as available_days,
    avg(price) as avg_price,
    sum(estimated_revenue) as total_estimated_revenue,
    max(neighbourhood) as neighbourhood,
    max(room_type) as room_type
  from calendar_facts
  group by listing_id, calendar_month
)

select
  listing_id,
  calendar_month,
  total_days,
  unavailable_days,
  available_days,
  round(
    (unavailable_days::float / nullif(total_days, 0)) * 100,
    2
  ) as occupancy_rate_pct,
  round(avg_price, 2) as avg_price,
  round(total_estimated_revenue, 2) as total_estimated_revenue,
  neighbourhood,
  room_type
from monthly_summary
```

**models/marts/agg_neighbourhood_monthly_performance.sql**:

```sql
{{
  config(
    materialized='table'
  )
}}

select
  neighbourhood,
  calendar_month,
  count(distinct listing_id) as active_listings,
  sum(total_days) as total_listing_days,
  sum(unavailable_days) as total_unavailable_days,
  round(
    avg(occupancy_rate_pct),
    2
  ) as avg_occupancy_rate_pct,
  round(avg(avg_price), 2) as avg_price,
  round(sum(total_estimated_revenue), 2) as total_estimated_revenue
from {{ ref('agg_listing_monthly_performance') }}
group by neighbourhood, calendar_month
```

## dbt Tests

### Generic Tests (in .yml)

**models/marts/_marts.yml**:

```yaml
version: 2

models:
  - name: dim_listings
    description: Dimension table for Airbnb listings
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

### Singular Tests

**tests/no_duplicate_listing_dates.sql**:

```sql
-- Check for duplicate listing-date combinations in fact table
select
  listing_id,
  calendar_date,
  count(*) as occurrence_count
from {{ ref('fct_listing_calendar') }}
group by listing_id, calendar_date
having count(*) > 1
```

**tests/valid_revenue_proxy.sql**:

```sql
-- Ensure revenue is only non-zero when listing is unavailable
select *
from {{ ref('fct_listing_calendar') }}
where is_available = true
  and estimated_revenue > 0
```

## Running dbt

```bash
# Verify connection
dbt debug --profiles-dir .

# Compile models (check SQL)
dbt compile --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select fct_listing_calendar+ --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .

# Serve docs locally
dbt docs serve --profiles-dir .
```

**Workflow for new data**:

```bash
# 1. Load new raw files
python scripts/load_inside_airbnb_to_snowflake.py

# 2. Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 3. Run tests
dbt test --profiles-dir .
```

## Streamlit Dashboard

**dashboard/streamlit_app.py**:

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load credentials
@st.cache_resource
def get_snowflake_connection():
    with open('config/local_credentials.json') as f:
        creds = json.load(f)
    
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema='ANALYTICS',  # dbt target schema
        role=creds.get('role', 'ACCOUNTADMIN')
    )

@st.cache_data
def load_neighbourhood_performance():
    conn = get_snowflake_connection()
    query = """
    SELECT
      neighbourhood,
      calendar_month,
      active_listings,
      avg_occupancy_rate_pct,
      avg_price,
      total_estimated_revenue
    FROM agg_neighbourhood_monthly_performance
    ORDER BY calendar_month DESC, total_estimated_revenue DESC
    """
    df = pd.read_sql(query, conn)
    conn.close()
    return df

st.title('🏠 Inside Airbnb Analytics Dashboard')

st.header('Neighbourhood Performance')

df_neighbourhood = load_neighbourhood_performance()

# Filter by month
months = df_neighbourhood['CALENDAR_MONTH'].unique()
selected_month = st.selectbox('Select Month', months)

df_filtered = df_neighbourhood[
    df_neighbourhood['CALENDAR_MONTH'] == selected_month
]

st.metric(
    'Total Revenue (Estimated)',
    f"${df_filtered['TOTAL_ESTIMATED_REVENUE'].sum():,.2f}"
)

st.dataframe(
    df_filtered.sort_values('TOTAL_ESTIMATED_REVENUE', ascending=False),
    use_container_width=True
)

st.bar_chart(
    df_filtered.set_index('NEIGHBOURHOOD')['TOTAL_ESTIMATED_REVENUE']
)
```

**Run dashboard**:

```bash
streamlit run dashboard/streamlit_app.py
```

## Common Patterns

### Adding a New Source

1. **Upload to Snowflake**:

```python
cursor.execute("PUT file://data/raw/new_file.csv @INSIDE_AIRBNB_STAGE")
cursor.execute("""
CREATE OR REPLACE TABLE RAW.NEW_TABLE (
  id VARCHAR,
  field1 VARCHAR,
  field2 VARCHAR
)
""")
cursor.execute("COPY INTO RAW.NEW_TABLE FROM @INSIDE_AIRBNB_STAGE/new_file.csv ...")
```

2. **Add to `_sources.yml`**:

```yaml
- name: new_table
  description: New data source
  columns:
    - name: id
      tests:
        - not_null
```

3. **Create staging model** `stg_airbnb__new_table.sql`:

```sql
select
  cast(id as integer) as new_id,
  trim(field1) as field1,
  cast(field2 as date) as field2
from {{ source('raw_airbnb', 'new_table') }}
```

### Creating a Snapshot (SCD Type 2)

**snapshots/listings_snapshot.sql**:

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

select * from {{ ref('stg_airbnb__listings') }}

{% endsnapshot %}
```

**Run**:

```bash
dbt snapshot --profiles-dir .
```

### Using Macros

**macros/clean_price.sql**:

```sql
{% macro clean_price(column_name) %}
  cast(
    replace(replace({{ column_name }}, '$', ''), ',', '')
    as decimal(10,2)
  )
{% endmacro %}
```

**Usage in model**:

```sql
select
  listing_id,
  {{ clean_price('price') }} as price
from source
```

### Exposures for Dashboards

**models/marts/_exposures.yml**:

```yaml
version: 2

exposures:
  - name: streamlit_dashboard
    type: dashboard
    maturity: high
    owner:
      name: Durgesh Yadav
      email: durgesh@example.com
    depends_on:
      - ref('agg_neighbourhood_monthly_performance')
      - ref('agg_listing_monthly_performance')
      - ref('dim_listings')
      - ref('dim_hosts')
    description: |
      Streamlit dashboard showing neighbourhood and host analytics
```

## Troubleshooting

### dbt Connection Issues

**Error**: `Could not connect to Snowflake`

**Solution**:

```bash
# Check profiles.yml syntax
dbt debug --profiles-dir .

# Verify account identifier (e.g., xy12345.us-east-1)
# Ensure password has no YAML special characters (wrap in quotes)

# Test direct connection
python -c "
import snowflake.connector
conn = snowflake.connector.connect(
  account='YOUR_ACCOUNT',
  user='YOUR_USER',
  password='YOUR_PASS'
)
print('Connected!')
"
```

### Incremental Model Not Updating

**Issue**: New data not appearing after `dbt run`

**Solution**:

```bash
# Full refresh to rebuild
dbt run --full-refresh --select fct_listing_calendar

# Check incremental logic
dbt compile --select fct_listing_calendar
# Review compiled SQL in target/compiled/
```

### Test Failures

**Error**: `relationships test failed: foreign key violation`

**Solution**:

```sql
-- Debug missing listings
select distinct c.listing_id
from {{ ref('fct_listing_calendar') }} c
left join {{ ref('dim_listings') }} l
  on c.listing_id = l.listing_id
where l.listing_id is null
```

### Streamlit Can't Find Credentials

**Error**: `FileNotFoundError: config/local_credentials.json`

**Solution**:

```bash
# Run Streamlit from project root
cd /path/to/Snowflake_DBT_Project
streamlit run dashboard/streamlit_app.py

# Or use absolute path in code
import os
config_path = os.path.join(os.path.dirname(__file__), '..', 'config', 'local_credentials.json')
```

### Price Parsing Errors

**Issue**: Prices show as NULL after cleaning

**Solution**:

```sql
-- Debug raw values
select
  price,
  replace(replace(price, '$', ''), ',', '') as cleaned
from {{ source('raw_airbnb', 'listings') }}
where price is not null
limit 10;

-- Handle edge cases
cast(
  nullif(
    regexp_replace(price, '[^0-9.]', ''),
    ''
  ) as decimal(10,2)
) as price
```

### Performance - Large Calendar Table

**Optimize incremental loads**:

```sql
-- Partition by month in Snowflake
alter table fct_listing_calendar
cluster by (calendar_month);

-- Limit incremental window
{% if is_incremental() %}
  where calendar_date > (select max(calendar_date) from {{ this }})
    and calendar_date <= current_date()
{% endif %}
```

## Advanced Patterns

### dbt Cloud Integration

**dbt_project.yml**:

```yaml
dispatch:
  - macro_namespace: dbt_utils
    search_order: ['dbt_utils', 'dbt']

vars:
  revenue_multiplier: 1.0  # Adjust revenue proxy sensitivity
```

**Use in model**:

```sql
select
  estimated_revenue * {{ var('revenue_multiplier') }} as adjusted_revenue
from {{ ref('int_airbnb__calendar_enriched') }}
```

### Custom Schema Names

**dbt_project.yml**:

```yaml
models:
  snowflake_dbt_project:
    marts:
      +schema: analytics
    staging:
      +schema: staging
```

**Generate schemas**: `AIRBNB_DB.ANALYTICS`, `AIRBNB_DB.STAGING`

### CI/CD with GitHub Actions

**.github/workflows/dbt_ci.yml**:

```yaml
name: dbt CI
on: [pull_request]

jobs:
  dbt_run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.9'
      - run: pip install -r requirements.txt
      - name: dbt test
        env:
          SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
          SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
          SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
        run: dbt test --profiles-dir .
```

This skill provides comprehensive guidance for building Snowflake + dbt analytics pipelines with real-world patterns for staging, marts, incremental models, testing, and dashboards.
