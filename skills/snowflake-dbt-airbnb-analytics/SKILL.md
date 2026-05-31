---
name: snowflake-dbt-airbnb-analytics
description: Build end-to-end Snowflake + dbt analytics pipelines with Inside Airbnb data, incremental models, data quality tests, and Streamlit dashboards
triggers:
  - set up a Snowflake dbt project with Inside Airbnb data
  - load Inside Airbnb data into Snowflake and transform with dbt
  - create dbt staging intermediate and mart layers for analytics
  - build incremental fact tables in dbt with Snowflake merge
  - add dbt tests for data quality and relationship validation
  - create a Streamlit dashboard connected to dbt marts
  - configure local credentials for Snowflake dbt projects
  - run full refresh on dbt incremental models
---

# Snowflake dbt Airbnb Analytics Engineering

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a complete analytics engineering workflow: loading Inside Airbnb open data into Snowflake, transforming it through dbt staging/intermediate/mart layers, validating data quality with dbt tests, and powering a Streamlit dashboard. It showcases incremental fact modeling, modular dbt design, and public-safe credential management.

## What This Project Does

- **Raw data loading**: Python script uploads Inside Airbnb CSV/GZIP files to Snowflake internal stage and creates raw tables
- **dbt transformation layers**: Staging (clean/cast), intermediate (joins/enrichment), and marts (dimensions/facts/aggregates)
- **Incremental modeling**: `fct_listing_calendar` uses Snowflake merge strategy for efficient daily updates
- **Data quality**: Generic and singular dbt tests validate uniqueness, relationships, accepted values, and business rules
- **Streamlit dashboard**: Connects to analytics marts for neighbourhood, host, and listing performance metrics
- **Local credentials**: Uses ignored `profiles.yml` and `config/local_credentials.json` files (never committed)

## Installation

### Prerequisites

- Python 3.8+
- Snowflake account with admin privileges
- dbt Core 1.0+
- Inside Airbnb raw data files (NYC recommended)

### Setup

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

**Requirements.txt** includes:
- `dbt-snowflake`
- `snowflake-connector-python`
- `streamlit`
- `pandas`
- `python-dotenv`

### Configure Local Credentials

```bash
# Copy example credential files (these are git-ignored)
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

Edit `profiles.yml`:

```yaml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT.region
      user: YOUR_USERNAME
      password: YOUR_PASSWORD  # Or use authenticator: externalbrowser
      role: ACCOUNTADMIN
      database: AIRBNB_DEV
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

Edit `config/local_credentials.json`:

```json
{
  "account": "YOUR_ACCOUNT.region",
  "user": "YOUR_USERNAME",
  "password": "YOUR_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DEV",
  "schema": "ANALYTICS",
  "role": "ACCOUNTADMIN"
}
```

**Never commit these files.** The `.gitignore` includes:
```
profiles.yml
config/local_credentials.json
.user.yml
```

## Raw Data Loading

### Download Inside Airbnb Data

1. Visit [Inside Airbnb](https://insideairbnb.com/get-the-data/)
2. Download NYC data:
   - `listings.csv.gz`
   - `calendar.csv.gz`
   - `reviews.csv.gz`
   - `neighbourhoods.csv`
3. Place in `data/raw/` directory

### Load to Snowflake

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What this script does:**

1. Reads credentials from `config/local_credentials.json`
2. Executes `setup/snowflake_setup.sql` to create database, schemas, stage
3. Uploads local files to `INSIDE_AIRBNB_STAGE` internal stage
4. Creates raw tables with `COPY INTO` commands
5. Validates row counts

**Script structure:**

```python
import snowflake.connector
import json
import os
from pathlib import Path

def load_credentials():
    with open('config/local_credentials.json') as f:
        return json.load(f)

def create_snowflake_objects(conn):
    with open('setup/snowflake_setup.sql') as f:
        sql_commands = f.read()
    cursor = conn.cursor()
    for statement in sql_commands.split(';'):
        if statement.strip():
            cursor.execute(statement)
    cursor.close()

def upload_files_to_stage(conn, local_dir='data/raw'):
    cursor = conn.cursor()
    for file_path in Path(local_dir).glob('*'):
        if file_path.suffix in ['.csv', '.gz']:
            cursor.execute(f"PUT file://{file_path} @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")
    cursor.close()

def copy_into_raw_tables(conn):
    cursor = conn.cursor()
    
    # Listings
    cursor.execute("""
        COPY INTO RAW.LISTINGS
        FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
        FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
        ON_ERROR = 'CONTINUE'
    """)
    
    # Calendar
    cursor.execute("""
        COPY INTO RAW.CALENDAR
        FROM @INSIDE_AIRBNB_STAGE/calendar.csv.gz
        FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
        ON_ERROR = 'CONTINUE'
    """)
    
    # Reviews
    cursor.execute("""
        COPY INTO RAW.REVIEWS
        FROM @INSIDE_AIRBNB_STAGE/reviews.csv.gz
        FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
        ON_ERROR = 'CONTINUE'
    """)
    
    # Neighbourhoods
    cursor.execute("""
        COPY INTO RAW.NEIGHBOURHOODS
        FROM @INSIDE_AIRBNB_STAGE/neighbourhoods.csv
        FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
        ON_ERROR = 'CONTINUE'
    """)
    
    cursor.close()

def main():
    creds = load_credentials()
    conn = snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        role=creds['role']
    )
    
    create_snowflake_objects(conn)
    upload_files_to_stage(conn)
    copy_into_raw_tables(conn)
    
    conn.close()
    print("✓ Data loaded successfully")

if __name__ == '__main__':
    main()
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

### Source Definitions

**models/staging/_sources.yml:**

```yaml
version: 2

sources:
  - name: airbnb_raw
    description: Raw Inside Airbnb data loaded from CSV files
    database: AIRBNB_DEV
    schema: RAW
    tables:
      - name: LISTINGS
        description: Listing details from Inside Airbnb
        columns:
          - name: id
            description: Unique listing identifier
            tests:
              - unique
              - not_null
      
      - name: CALENDAR
        description: Daily availability and pricing
        columns:
          - name: listing_id
            description: FK to listings
            tests:
              - not_null
          - name: date
            description: Calendar date
            tests:
              - not_null
      
      - name: REVIEWS
        description: Guest reviews
        columns:
          - name: id
            description: Unique review identifier
            tests:
              - unique
              - not_null
      
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
        id::bigint as listing_id,
        name::varchar as listing_name,
        host_id::bigint as host_id,
        host_name::varchar as host_name,
        neighbourhood_cleansed::varchar as neighbourhood,
        latitude::float as latitude,
        longitude::float as longitude,
        room_type::varchar as room_type,
        price::varchar as price_raw,  -- Needs cleaning: "$125.00"
        minimum_nights::int as minimum_nights,
        number_of_reviews::int as number_of_reviews,
        last_review::date as last_review_date,
        reviews_per_month::float as reviews_per_month,
        calculated_host_listings_count::int as host_total_listings_count,
        availability_365::int as availability_365
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
        listing_id::bigint as listing_id,
        date::date as calendar_date,
        available::varchar as available_flag,  -- 't' or 'f'
        price::varchar as price_raw,
        minimum_nights::int as minimum_nights,
        maximum_nights::int as maximum_nights
    from source
)

select * from cleaned
```

**models/staging/stg_airbnb__reviews.sql:**

```sql
with source as (
    select * from {{ source('airbnb_raw', 'REVIEWS') }}
),

cleaned as (
    select
        id::bigint as review_id,
        listing_id::bigint as listing_id,
        date::date as review_date,
        reviewer_id::bigint as reviewer_id,
        reviewer_name::varchar as reviewer_name,
        comments::varchar as review_comments
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
        l.latitude,
        l.longitude,
        l.room_type,
        -- Clean price: remove $ and convert to decimal
        cast(
            replace(replace(l.price_raw, '$', ''), ',', '')
            as decimal(10,2)
        ) as price,
        l.minimum_nights,
        l.number_of_reviews,
        l.last_review_date,
        l.reviews_per_month,
        l.host_total_listings_count,
        l.availability_365,
        case
            when l.host_total_listings_count = 1 then 'Single Listing'
            when l.host_total_listings_count between 2 and 5 then 'Multi Listing'
            else 'Professional'
        end as host_category
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
        c.available_flag,
        case when c.available_flag = 't' then true else false end as is_available,
        cast(
            replace(replace(c.price_raw, '$', ''), ',', '')
            as decimal(10,2)
        ) as price,
        c.minimum_nights,
        c.maximum_nights,
        l.listing_name,
        l.host_id,
        l.neighbourhood,
        l.neighbourhood_group,
        l.room_type,
        -- Revenue proxy: if unavailable, assume booked
        case
            when c.available_flag = 'f' then cast(
                replace(replace(c.price_raw, '$', ''), ',', '')
                as decimal(10,2)
            )
            else 0
        end as estimated_revenue,
        date_trunc('month', c.calendar_date) as month_start_date
    from calendar c
    inner join listings l
        on c.listing_id = l.listing_id
)

select * from enriched
```

### Mart Models: Dimensions

**models/marts/dim_listings.sql:**

```sql
{{ config(materialized='table') }}

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
    host_total_listings_count,
    host_category,
    availability_365,
    current_timestamp() as dbt_loaded_at
from listings
```

**models/marts/dim_hosts.sql:**

```sql
{{ config(materialized='table') }}

with listings as (
    select * from {{ ref('int_airbnb__listing_enriched') }}
),

host_metrics as (
    select
        host_id,
        max(host_name) as host_name,
        count(distinct listing_id) as total_listings,
        avg(price) as avg_listing_price,
        sum(number_of_reviews) as total_reviews,
        max(last_review_date) as most_recent_review,
        max(host_category) as host_category
    from listings
    group by host_id
)

select
    host_id,
    host_name,
    total_listings,
    round(avg_listing_price, 2) as avg_listing_price,
    total_reviews,
    most_recent_review,
    host_category,
    current_timestamp() as dbt_loaded_at
from host_metrics
```

### Mart Models: Incremental Fact Table

**models/marts/fct_listing_calendar.sql:**

```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        merge_update_columns=['is_available', 'price', 'estimated_revenue']
    )
}}

with calendar as (
    select * from {{ ref('int_airbnb__calendar_enriched') }}
)

select
    listing_id,
    calendar_date,
    is_available,
    price,
    minimum_nights,
    maximum_nights,
    neighbourhood,
    neighbourhood_group,
    room_type,
    estimated_revenue,
    month_start_date,
    current_timestamp() as dbt_loaded_at
from calendar

{% if is_incremental() %}
    where calendar_date > (select max(calendar_date) from {{ this }})
{% endif %}
```

**Key incremental features:**
- **unique_key**: Prevents duplicates on `listing_id + calendar_date`
- **merge_update_columns**: Updates only specified columns on conflict (Snowflake merge)
- **is_incremental()**: Only loads new dates after max existing date

### Mart Models: Aggregates

**models/marts/agg_listing_monthly_performance.sql:**

```sql
{{ config(materialized='table') }}

with calendar as (
    select * from {{ ref('fct_listing_calendar') }}
)

select
    listing_id,
    month_start_date,
    neighbourhood,
    neighbourhood_group,
    room_type,
    count(*) as total_days,
    sum(case when is_available then 1 else 0 end) as available_days,
    sum(case when not is_available then 1 else 0 end) as unavailable_days,
    round(
        sum(case when not is_available then 1 else 0 end)::float / count(*),
        3
    ) as occupancy_rate,
    round(avg(price), 2) as avg_price,
    round(sum(estimated_revenue), 2) as total_estimated_revenue
from calendar
group by
    listing_id,
    month_start_date,
    neighbourhood,
    neighbourhood_group,
    room_type
```

**models/marts/agg_neighbourhood_monthly_performance.sql:**

```sql
{{ config(materialized='table') }}

with listing_monthly as (
    select * from {{ ref('agg_listing_monthly_performance') }}
)

select
    neighbourhood,
    neighbourhood_group,
    month_start_date,
    count(distinct listing_id) as total_listings,
    sum(total_days) as total_days,
    sum(available_days) as total_available_days,
    sum(unavailable_days) as total_unavailable_days,
    round(
        sum(unavailable_days)::float / sum(total_days),
        3
    ) as avg_occupancy_rate,
    round(avg(avg_price), 2) as avg_listing_price,
    round(sum(total_estimated_revenue), 2) as total_estimated_revenue,
    round(avg(total_estimated_revenue), 2) as avg_listing_revenue
from listing_monthly
group by
    neighbourhood,
    neighbourhood_group,
    month_start_date
```

## dbt Configuration

**dbt_project.yml:**

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
    +persist_docs:
      relation: true
      columns: true
    
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

## dbt Commands

### Development Workflow

```bash
# Verify connection
dbt debug --profiles-dir .

# Install dependencies (if using packages)
dbt deps --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and its downstream dependencies
dbt run --select int_airbnb__calendar_enriched+ --profiles-dir .

# Run only staging models
dbt run --select staging.* --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Run tests for specific model
dbt test --select dim_listings --profiles-dir .

# Generate and serve documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

### Incremental Model Refresh

```bash
# Full refresh for incremental model (rebuilds from scratch)
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Full refresh after new data load
python scripts/load_inside_airbnb_to_snowflake.py
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .
dbt test --profiles-dir .
```

### Production Patterns

```bash
# Run with specific target
dbt run --target prod --profiles-dir .

# Run with vars
dbt run --vars '{"target_date": "2024-01-01"}' --profiles-dir .

# Fail fast on first error
dbt run --fail-fast --profiles-dir .

# Run changed models only (requires state)
dbt run --select state:modified+ --state ./target --profiles-dir .
```

## Data Quality Tests

### Generic Tests (in _sources.yml and schema.yml)

**models/marts/schema.yml:**

```yaml
version: 2

models:
  - name: dim_listings
    description: Dimension table of Airbnb listings
    columns:
      - name: listing_id
        description: Primary key
        tests:
          - unique
          - not_null
      - name: room_type
        description: Type of room offered
        tests:
          - accepted_values:
              values: ['Entire home/apt', 'Private room', 'Shared room', 'Hotel room']
      - name: price
        description: Nightly price
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
  
  - name: fct_listing_calendar
    description: Daily listing availability and pricing fact table
    columns:
      - name: listing_id
        description: FK to dim_listings
        tests:
          - not_null
          - relationships:
              to: ref('dim_listings')
              field: listing_id
      - name: calendar_date
        description: Date of availability record
        tests:
          - not_null
```

### Singular Tests

**tests/assert_no_duplicate_listing_dates.sql:**

```sql
-- Check for duplicate listing_id + calendar_date combinations in fact table
select
    listing_id,
    calendar_date,
    count(*) as record_count
from {{ ref('fct_listing_calendar') }}
group by listing_id, calendar_date
having count(*) > 1
```

**tests/assert_price_within_range.sql:**

```sql
-- Ensure prices are reasonable (not outliers)
select *
from {{ ref('dim_listings') }}
where price < 10 or price > 10000
```

**tests/assert_calendar_coverage.sql:**

```sql
-- Ensure calendar covers expected date range
with date_range as (
    select
        min(calendar_date) as min_date,
        max(calendar_date) as max_date,
        datediff('day', min(calendar_date), max(calendar_date)) as date_span
    from {{ ref('fct_listing_calendar') }}
)

select *
from date_range
where date_span < 365  -- Expect at least 1 year of data
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
def load_credentials():
    creds_path = Path(__file__).parent.parent / 'config' / 'local_credentials.json'
    with open(creds_path) as f:
        return json.load(f)

# Connect to Snowflake
@st.cache_resource
def get_snowflake_connection():
    creds = load_credentials()
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema'],
        role=creds['role']
    )

# Query helper
def run_query(query):
    conn = get_snowflake_connection()
    return pd.read_sql(query, conn)

st.set_page_config(page_title="Airbnb Analytics", layout="wide")
st.title("🏠 Inside Airbnb Analytics Dashboard")

# Sidebar filters
st.sidebar.header("Filters")
neighbourhood_group = st.sidebar.selectbox(
    "Neighbourhood Group",
    ["All"] + run_query("SELECT DISTINCT neighbourhood_group FROM agg_neighbourhood_monthly_performance ORDER BY 1")['NEIGHBOURHOOD_GROUP'].tolist()
)

# Top neighbourhoods by revenue
st.header("Top Neighbourhoods by Estimated Revenue")
query = """
SELECT
    neighbourhood,
    neighbourhood_group,
    SUM(total_estimated_revenue) as total_revenue,
    AVG(avg_occupancy_rate) as avg_occupancy,
    COUNT(DISTINCT month_start_date) as months_tracked
FROM agg_neighbourhood_monthly_performance
"""
if neighbourhood_group != "All":
    query += f"WHERE neighbourhood_group = '{neighbourhood_group}'"
query += """
GROUP BY neighbourhood, neighbourhood_group
ORDER BY total_revenue DESC
LIMIT 10
"""

df_neighbourhoods = run_query(query)
st.dataframe(df_neighbourhoods, use_container_width=True)

# Monthly revenue trend
st.header("Monthly Revenue Trend")
query_trend = """
SELECT
    month_start_date,
    SUM(total_estimated_revenue) as monthly_revenue,
    AVG(avg_occupancy_rate) as avg_occupancy
FROM agg_neighbourhood_monthly_performance
"""
if neighbourhood_group != "All":
    query_trend += f"WHERE neighbourhood_group = '{neighbourhood_group}'"
query_trend += """
GROUP BY month_start_date
ORDER BY month_start_date
"""

df_trend = run_query(query_trend)
st.line_chart(df_trend.set_index('MONTH_START_DATE')['MONTHLY_REVENUE'])

# Host analysis
st.header("Top Hosts by Listing Count")
query_hosts = """
SELECT
    host_id,
    host_name,
    total_listings,
    avg_listing_price,
    total_reviews,
    host_category
FROM dim_hosts
ORDER BY total_listings DESC
LIMIT 10
"""
df_hosts = run_query(query_hosts)
st.dataframe(df_hosts, use_container_width=True)

# Room type distribution
st.header("Revenue by Room Type")
query_room = """
SELECT
    room_type,
    SUM(total_estimated_revenue) as total_revenue,
    COUNT(DISTINCT listing_id) as listing_count
FROM agg_listing_monthly_performance
GROUP BY room_type
ORDER BY total_revenue DESC
"""
df_room = run_query(query_room)
st.bar_chart(df_room.set_index('ROOM_TYPE')['TOTAL_REVENUE'])
```

### Run Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

Dashboard reads from `config/local_credentials.json` (git-ignored).

## Common Patterns

### Adding a New Source Table

1. Load data with Python script or `COPY INTO`
2. Add source definition in `models/staging/_sources.yml`
3. Create staging model `stg_airbnb__newsource.sql`
4. Add tests in source YAML
5. Run `dbt run --select stg_airbnb__newsource`

### Creating Custom Macros

**macros/clean_price.sql:**

```sql
{% macro clean_price(column_name) %}
    cast(
        replace(replace({{ column_name }}, '$', ''), ',', '')
        as decimal(10,2)
    )
{% endmacro %}
```

**Usage in model:**

```sql
select
    listing_id,
    {{ clean_price('price_raw') }} as price
from {{ ref('stg_airbnb__listings') }}
```

### Updating Incremental Model After Schema Change

```bash
# Drop and rebuild
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Or manually drop in Snowflake
# DROP TABLE AIRBNB_DEV.ANALYTICS.FCT_LISTING_CALENDAR;
# dbt run --select fct_listing_calendar --profiles-dir .
```

### Running Tests in CI/CD

```bash
# GitHub Actions workflow snippet
- name: Run dbt tests
  run: |
    dbt deps --profiles-dir .
    dbt run --profiles-dir .
    dbt test --profiles-dir . || exit 1
```

## Troubleshooting

### Connection Error: "Invalid account name"

**Cause**: Snowflake account identifier incorrect in `profiles.yml`

**Fix**: Use format `ACCOUNT.REGION` (e.g., `xy12345.us-east-1`)

```yaml
account: xy12345.us-east-1  # Not just xy12345
```

### dbt Error: "Compilation Error in model"

**Cause**: SQL syntax error or undefined ref/source

**Fix**: Check model SQL and verify references exist

```bash
dbt compile --select problematic_model --profiles-dir .
# Review compiled SQL in target/compiled/
```

### Incremental Model Not Updating

**Cause**: `is_incremental()` filter prevents new data

**Fix**: Run full-refresh to rebuild

```bash
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

### Test Failure: "Relationship test failed"

**Cause**: Orphaned foreign keys (calendar.listing_id not in listings)

**Fix**: Add inner join in staging or intermediate layer

```sql
-- In int_airbnb__calendar_enriched.sql
inner join listings l  -- Changed from left join
    on c.listing_id = l.listing_id
```

### Streamlit: "Could not find credentials file"

**Cause**: `config/local_credentials.json` not created

**Fix**: Copy from example and populate

```bash
cp config/local_credentials.example.json config/local_credentials.json
# Edit with real credentials
```

### dbt Docs: "Relation not found"

**Cause**: Models not yet run in target database

**Fix**: Run models before generating docs

```bash
dbt run --profiles-dir .
dbt docs generate --profiles-dir .
```
