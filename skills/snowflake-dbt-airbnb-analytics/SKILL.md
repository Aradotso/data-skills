---
name: snowflake-dbt-airbnb-analytics
description: Analytics engineering project using Snowflake, dbt, and Streamlit for Inside Airbnb data modeling and visualization
triggers:
  - set up airbnb analytics project with dbt and snowflake
  - build dbt models for inside airbnb data
  - create snowflake data warehouse for airbnb listings
  - configure dbt staging and marts for airbnb analytics
  - load inside airbnb data into snowflake
  - build streamlit dashboard for airbnb data
  - run incremental dbt models for calendar facts
  - troubleshoot dbt snowflake connection issues
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to work with a complete analytics engineering stack: Snowflake data warehouse, dbt transformations, and Streamlit dashboards, using Inside Airbnb open data.

## What This Project Does

A portfolio analytics engineering project demonstrating:

- **Raw data ingestion**: Load CSV/GZIP files into Snowflake internal stages
- **dbt layered modeling**: Staging → Intermediate → Marts (dimensions, facts, aggregates)
- **Incremental processing**: Merge-based incremental fact tables in Snowflake
- **Data quality**: Generic and singular dbt tests
- **Visualization**: Streamlit dashboard connected to analytics marts

**Data layers:**
1. **RAW**: Text-preserving source tables from Inside Airbnb
2. **Staging**: Clean, cast, standardize column names
3. **Intermediate**: Reusable joins, enrichment, revenue proxy logic
4. **Marts**: Analytics-ready dimensions (`dim_listings`, `dim_hosts`), facts (`fct_listing_calendar`, `fct_reviews`), and aggregates

## Installation

### Prerequisites

- Snowflake account with appropriate privileges
- Python 3.8+
- Inside Airbnb dataset files (NYC recommended for classroom use)

### Setup Steps

```bash
# Clone and navigate to project
git clone https://github.com/analyticsdurgesh/Snowflake_DBT_Project.git
cd Snowflake_DBT_Project

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configure Credentials

**Never commit real credentials.** Create local-only configuration files:

```bash
# Copy examples
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

**Edit `profiles.yml`:**

```yaml
airbnb_analytics:
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT
      user: YOUR_USER
      password: YOUR_PASSWORD
      role: YOUR_ROLE
      database: AIRBNB_ANALYTICS
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
  target: dev
```

**Edit `config/local_credentials.json`:**

```json
{
  "account": "YOUR_ACCOUNT",
  "user": "YOUR_USER",
  "password": "YOUR_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_ANALYTICS",
  "schema": "ANALYTICS",
  "role": "YOUR_ROLE"
}
```

### Download Inside Airbnb Data

Place files in `data/raw/`:
- `listings.csv.gz`
- `calendar.csv.gz`
- `reviews.csv.gz`
- `neighbourhoods.csv`

Download from: https://insideairbnb.com/get-the-data/ (NYC recommended)

## Loading Raw Data into Snowflake

The Python loader script automates:
1. Snowflake object creation (database, schemas, stage)
2. File upload to internal stage
3. Raw table creation from CSV headers
4. COPY INTO commands

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What happens:**
- Reads credentials from `config/local_credentials.json`
- Executes `setup/snowflake_setup.sql` (creates `AIRBNB_ANALYTICS` database, `RAW` schema, `INSIDE_AIRBNB_STAGE`)
- Uploads files from `data/raw/` to Snowflake stage
- Creates `RAW.LISTINGS`, `RAW.CALENDAR`, `RAW.REVIEWS`, `RAW.NEIGHBOURHOODS`
- Loads data via `COPY INTO` with CSV format settings

**Script structure:**

```python
import snowflake.connector
import json
from pathlib import Path

# Load credentials
with open('config/local_credentials.json') as f:
    creds = json.load(f)

conn = snowflake.connector.connect(
    account=creds['account'],
    user=creds['user'],
    password=creds['password'],
    warehouse=creds['warehouse'],
    role=creds['role']
)

# Execute setup SQL
with open('setup/snowflake_setup.sql') as f:
    for statement in f.read().split(';'):
        if statement.strip():
            conn.cursor().execute(statement)

# Upload files to stage
conn.cursor().execute("USE SCHEMA AIRBNB_ANALYTICS.RAW")
for file in Path('data/raw').glob('*.csv*'):
    conn.cursor().execute(f"PUT file://{file} @INSIDE_AIRBNB_STAGE")
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
    ├── fct_listing_calendar.sql
    ├── fct_reviews.sql
    ├── agg_listing_monthly_performance.sql
    └── agg_neighbourhood_monthly_performance.sql
```

### Source Configuration

**models/staging/_sources.yml:**

```yaml
version: 2

sources:
  - name: airbnb_raw
    database: AIRBNB_ANALYTICS
    schema: RAW
    tables:
      - name: LISTINGS
        description: Raw listings data from Inside Airbnb
        columns:
          - name: ID
            description: Unique listing identifier
            tests:
              - not_null
              - unique
      - name: CALENDAR
        description: Daily availability and pricing
      - name: REVIEWS
        description: Guest reviews
      - name: NEIGHBOURHOODS
        description: Neighbourhood reference data
```

### Staging Models

**models/staging/stg_airbnb__listings.sql:**

```sql
with source as (
    select * from {{ source('airbnb_raw', 'LISTINGS') }}
),

cleaned as (
    select
        cast(id as integer) as listing_id,
        trim(name) as listing_name,
        cast(host_id as integer) as host_id,
        trim(host_name) as host_name,
        trim(neighbourhood) as neighbourhood,
        trim(room_type) as room_type,
        cast(price as number(10,2)) as price,
        cast(minimum_nights as integer) as minimum_nights,
        cast(availability_365 as integer) as availability_365,
        cast(number_of_reviews as integer) as number_of_reviews,
        cast(latitude as number(10,6)) as latitude,
        cast(longitude as number(10,6)) as longitude,
        to_date(last_review, 'YYYY-MM-DD') as last_review_date,
        current_timestamp() as _loaded_at
    from source
)

select * from cleaned
```

**models/staging/stg_airbnb__calendar.sql:**

```sql
with source as (
    select * from {{ source('airbnb_raw', 'CALENDAR') }}
),

cleaned as (
    select
        cast(listing_id as integer) as listing_id,
        to_date(date, 'YYYY-MM-DD') as calendar_date,
        case when lower(available) = 't' then true else false end as is_available,
        cast(regexp_replace(price, '[^0-9.]', '') as number(10,2)) as price,
        current_timestamp() as _loaded_at
    from source
)

select * from cleaned
```

### Intermediate Models

**models/intermediate/int_airbnb__listing_enriched.sql:**

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
        l.room_type,
        l.price,
        l.minimum_nights,
        l.availability_365,
        l.number_of_reviews,
        l.latitude,
        l.longitude,
        l.last_review_date,
        l._loaded_at
    from listings l
    left join neighbourhoods n
        on l.neighbourhood = n.neighbourhood
)

select * from enriched
```

**models/intermediate/int_airbnb__calendar_enriched.sql:**

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
        c.is_available,
        c.price as calendar_price,
        l.listing_name,
        l.neighbourhood,
        l.neighbourhood_group,
        l.room_type,
        -- Revenue proxy: assume unavailable = booked
        case 
            when not c.is_available then c.price 
            else 0 
        end as estimated_revenue,
        c._loaded_at
    from calendar c
    inner join listings l
        on c.listing_id = l.listing_id
)

select * from enriched
```

### Marts: Incremental Fact Table

**models/marts/fct_listing_calendar.sql:**

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
    calendar_price,
    listing_name,
    neighbourhood,
    neighbourhood_group,
    room_type,
    estimated_revenue,
    _loaded_at
from calendar_enriched

{% if is_incremental() %}
    where calendar_date > (select max(calendar_date) from {{ this }})
{% endif %}
```

### Marts: Aggregates

**models/marts/agg_listing_monthly_performance.sql:**

```sql
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
        date_trunc('month', calendar_date) as month_date,
        count(*) as total_days,
        sum(case when is_available then 1 else 0 end) as available_days,
        sum(case when not is_available then 1 else 0 end) as unavailable_days,
        avg(calendar_price) as avg_price,
        sum(estimated_revenue) as total_estimated_revenue,
        round(avg(case when is_available then 1 else 0 end) * 100, 2) as availability_rate_pct
    from calendar_facts
    group by 1, 2, 3, 4, 5, 6
)

select * from monthly_agg
```

## Running dbt

### Basic Commands

```bash
# Verify connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select stg_airbnb__listings+ --profiles-dir .

# Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Test specific model
dbt test --select stg_airbnb__listings --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .

# Serve docs locally
dbt docs serve --profiles-dir .
```

### Workflow for New Data Snapshot

```bash
# 1. Load new raw data
python scripts/load_inside_airbnb_to_snowflake.py

# 2. Full refresh incremental fact and downstream
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 3. Validate data quality
dbt test --profiles-dir .
```

### Model Selection Syntax

```bash
# Run model and all downstream
dbt run --select stg_airbnb__listings+

# Run model and all upstream
dbt run --select +fct_listing_calendar

# Run model, upstream, and downstream
dbt run --select +fct_listing_calendar+

# Run all staging models
dbt run --select staging.*

# Run by tag
dbt run --select tag:daily
```

## Data Quality Tests

### Generic Tests in Schema Files

**models/staging/_schema.yml:**

```yaml
version: 2

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
```

### Singular Tests

**tests/assert_no_duplicate_listing_dates.sql:**

```sql
-- Test for duplicate listing-date combinations in fact table
with calendar_facts as (
    select
        listing_id,
        calendar_date,
        count(*) as duplicate_count
    from {{ ref('fct_listing_calendar') }}
    group by 1, 2
    having count(*) > 1
)

select * from calendar_facts
```

**tests/assert_calendar_listings_exist.sql:**

```sql
-- Test that all calendar listings exist in dimension
with calendar_listings as (
    select distinct listing_id 
    from {{ ref('fct_listing_calendar') }}
),

dim_listings as (
    select listing_id 
    from {{ ref('dim_listings') }}
),

orphaned as (
    select c.listing_id
    from calendar_listings c
    left join dim_listings d
        on c.listing_id = d.listing_id
    where d.listing_id is null
)

select * from orphaned
```

## Streamlit Dashboard

**dashboard/streamlit_app.py:**

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json
from pathlib import Path

# Load credentials
with open('config/local_credentials.json') as f:
    creds = json.load(f)

@st.cache_resource
def get_connection():
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema'],
        role=creds['role']
    )

@st.cache_data
def load_neighbourhood_performance():
    conn = get_connection()
    query = """
    SELECT
        neighbourhood_group,
        neighbourhood,
        SUM(total_estimated_revenue) as total_revenue,
        AVG(availability_rate_pct) as avg_availability_rate,
        COUNT(DISTINCT listing_id) as listing_count
    FROM agg_neighbourhood_monthly_performance
    GROUP BY 1, 2
    ORDER BY 3 DESC
    LIMIT 20
    """
    return pd.read_sql(query, conn)

st.title("Airbnb Analytics Dashboard")

st.header("Top 20 Neighbourhoods by Estimated Revenue")
df = load_neighbourhood_performance()
st.dataframe(df)

st.bar_chart(df.set_index('neighbourhood')['total_revenue'])
```

**Run dashboard:**

```bash
streamlit run dashboard/streamlit_app.py
```

## Common Patterns

### Adding a New Staging Model

```sql
-- models/staging/stg_airbnb__new_source.sql
with source as (
    select * from {{ source('airbnb_raw', 'NEW_SOURCE') }}
),

cleaned as (
    select
        cast(id as integer) as record_id,
        trim(name) as record_name,
        to_date(created_at, 'YYYY-MM-DD') as created_date,
        current_timestamp() as _loaded_at
    from source
)

select * from cleaned
```

### Creating Reusable Macros

**macros/clean_price.sql:**

```sql
{% macro clean_price(column_name) %}
    cast(regexp_replace({{ column_name }}, '[^0-9.]', '') as number(10,2))
{% endmacro %}
```

**Usage in model:**

```sql
select
    listing_id,
    {{ clean_price('price') }} as price
from source
```

### Incremental Model with Delete+Insert

```sql
{{
    config(
        materialized='incremental',
        unique_key='listing_id',
        incremental_strategy='delete+insert'
    )
}}

select * from {{ ref('stg_airbnb__listings') }}

{% if is_incremental() %}
    where _loaded_at > (select max(_loaded_at) from {{ this }})
{% endif %}
```

### Using dbt Utils Package

**packages.yml:**

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1
```

```bash
dbt deps --profiles-dir .
```

**In models:**

```sql
select
    {{ dbt_utils.generate_surrogate_key(['listing_id', 'calendar_date']) }} as calendar_key,
    listing_id,
    calendar_date
from {{ ref('stg_airbnb__calendar') }}
```

## Troubleshooting

### Connection Issues

**Error: "Could not connect to Snowflake"**

1. Verify `profiles.yml` and `config/local_credentials.json` have correct values
2. Test connection manually:

```python
import snowflake.connector
conn = snowflake.connector.connect(
    account='YOUR_ACCOUNT',
    user='YOUR_USER',
    password='YOUR_PASSWORD'
)
print(conn.cursor().execute("SELECT CURRENT_VERSION()").fetchone())
```

3. Check Snowflake account identifier format: `account.region` or `account.region.cloud`

### dbt Debug Failures

```bash
dbt debug --profiles-dir .
```

Common fixes:
- Ensure `profiles.yml` is in project root or use `--profiles-dir .`
- Verify `profile` name in `dbt_project.yml` matches `profiles.yml`
- Check Snowflake warehouse is running

### Incremental Model Not Updating

**Full refresh:**

```bash
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

**Check `is_incremental()` logic:**

```sql
{% if is_incremental() %}
    -- Ensure filter is correct
    where calendar_date > (select coalesce(max(calendar_date), '1900-01-01') from {{ this }})
{% endif %}
```

### Test Failures

**View failing rows:**

```bash
dbt test --select stg_airbnb__listings --store-failures
```

Query stored failures in Snowflake:

```sql
SELECT * FROM AIRBNB_ANALYTICS.ANALYTICS.DBT_TEST_FAILURES
WHERE test_name = 'unique_stg_airbnb__listings_listing_id';
```

### Data Type Mismatches

**Error: "Numeric value 'N/A' is not recognized"**

Add null handling in staging:

```sql
select
    case 
        when price = 'N/A' then null
        else cast(regexp_replace(price, '[^0-9.]', '') as number(10,2))
    end as price
from source
```

### Stage File Upload Fails

**Error: "PUT command failed"**

Check file paths are absolute or relative to script location:

```python
from pathlib import Path
file_path = Path('data/raw/listings.csv.gz').resolve()
conn.cursor().execute(f"PUT file://{file_path} @INSIDE_AIRBNB_STAGE")
```

### Performance: Slow Incremental Builds

1. Ensure `unique_key` is indexed in Snowflake
2. Use partitioning for large tables:

```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        cluster_by=['calendar_date']
    )
}}
```

3. Consider switching to `merge` strategy:

```sql
{{
    config(
        incremental_strategy='merge'
    )
}}
```

## Configuration Reference

### dbt_project.yml

```yaml
name: 'airbnb_analytics'
version: '1.0.0'
config-version: 2

profile: 'airbnb_analytics'

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
  airbnb_analytics:
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

### Environment-Specific Targets

**profiles.yml:**

```yaml
airbnb_analytics:
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: DEV_ROLE
      database: AIRBNB_ANALYTICS_DEV
      warehouse: DEV_WH
      schema: ANALYTICS
      threads: 4
    prod:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: PROD_ROLE
      database: AIRBNB_ANALYTICS_PROD
      warehouse: PROD_WH
      schema: ANALYTICS
      threads: 8
  target: dev
```

```bash
dbt run --target prod --profiles-dir .
```

This skill provides complete coverage for building, testing, and deploying analytics engineering pipelines with Snowflake, dbt, and Streamlit.
