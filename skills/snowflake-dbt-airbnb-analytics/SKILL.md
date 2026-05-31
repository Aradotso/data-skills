---
name: snowflake-dbt-airbnb-analytics
description: Analytics engineering with Snowflake, dbt, and Inside Airbnb data including staging, marts, tests, and Streamlit dashboards
triggers:
  - build airbnb analytics pipeline with dbt and snowflake
  - set up inside airbnb data warehouse project
  - create dbt models for airbnb listings and calendar
  - deploy snowflake dbt analytics dashboard
  - load airbnb data into snowflake with dbt transformations
  - configure incremental dbt models for calendar facts
  - run airbnb analytics tests in dbt
  - build streamlit dashboard from dbt marts
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to work with a complete analytics engineering project that loads Inside Airbnb data into Snowflake, transforms it with dbt (staging → intermediate → marts), validates data quality, and powers a Streamlit dashboard.

## What This Project Does

- **Data Ingestion**: Loads Inside Airbnb CSV/GZIP files (listings, calendar, reviews, neighbourhoods) into Snowflake via internal stages
- **dbt Transformations**: Three-layer dbt architecture (staging → intermediate → marts) with incremental models
- **Data Quality**: Generic and singular dbt tests for primary keys, relationships, accepted values, and business rules
- **Analytics**: Dimension tables, fact tables, and monthly aggregates for neighbourhood and host analysis
- **Dashboard**: Streamlit app connected to analytics marts for visualization

## Project Architecture

**Raw Layer** → Snowflake internal stage → `RAW` schema (text columns)  
**Staging Layer** → `stg_airbnb__*` views (cleaning, casting, standardization)  
**Intermediate Layer** → `int_airbnb__*` tables (joins, enrichment, revenue proxy)  
**Marts Layer** → `dim_*`, `fct_*`, `agg_*` (analytics-ready dimensions, facts, aggregates)  
**Dashboard** → Streamlit app reading from marts

## Installation

```bash
# Clone and set up environment
git clone https://github.com/analyticsdurgesh/Snowflake_DBT_Project.git
cd Snowflake_DBT_Project
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

## Configuration

### Snowflake Credentials (Local Only)

Create local configuration files from examples:

```bash
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

Edit `profiles.yml` with your Snowflake connection:

```yaml
airbnb_snowflake_dbt:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT
      user: YOUR_USER
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"  # or hardcode locally
      role: ACCOUNTADMIN
      database: AIRBNB_DEV
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

Edit `config/local_credentials.json` for Streamlit:

```json
{
  "account": "YOUR_ACCOUNT",
  "user": "YOUR_USER",
  "password": "YOUR_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DEV",
  "schema": "ANALYTICS",
  "role": "ACCOUNTADMIN"
}
```

**Security**: Both files are `.gitignore`d. Never commit credentials.

### Dataset Setup

Download Inside Airbnb data for New York City from [insideairbnb.com/get-the-data](http://insideairbnb.com/get-the-data/) and place in `data/raw/`:

```
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

## Key Commands

### 1. Load Raw Data to Snowflake

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What it does**:
- Creates Snowflake objects (`RAW` schema, warehouse, internal stage)
- Uploads local files to `INSIDE_AIRBNB_STAGE`
- Creates raw tables with text columns
- Copies data from stage to raw tables

**Python loader structure** (`scripts/load_inside_airbnb_to_snowflake.py`):

```python
import snowflake.connector
import json

# Load credentials
with open('config/local_credentials.json') as f:
    creds = json.load(f)

conn = snowflake.connector.connect(
    account=creds['account'],
    user=creds['user'],
    password=creds['password'],
    warehouse=creds['warehouse'],
    database=creds['database'],
    role=creds['role']
)

cursor = conn.cursor()

# Execute setup SQL
with open('setup/snowflake_setup.sql') as f:
    setup_sql = f.read()
    for statement in setup_sql.split(';'):
        if statement.strip():
            cursor.execute(statement)

# Upload files to stage
cursor.execute("USE SCHEMA RAW")
cursor.execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")
cursor.execute("PUT file://data/raw/calendar.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")
cursor.execute("PUT file://data/raw/reviews.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")
cursor.execute("PUT file://data/raw/neighbourhoods.csv @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")

# Copy into raw tables
cursor.execute("""
COPY INTO RAW.LISTINGS
FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
FILE_FORMAT = (TYPE=CSV SKIP_HEADER=1 FIELD_OPTIONALLY_ENCLOSED_BY='"')
""")

conn.close()
```

### 2. Run dbt Transformations

```bash
# Verify connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select fct_listing_calendar+ --profiles-dir .

# Full refresh for incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .
```

### 3. Test Data Quality

```bash
# Run all tests
dbt test --profiles-dir .

# Test specific model
dbt test --select dim_listings --profiles-dir .

# Run only singular tests
dbt test --select test_type:singular --profiles-dir .
```

### 4. Generate and Serve Documentation

```bash
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

### 5. Run Streamlit Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

## dbt Model Examples

### Staging Model: `stg_airbnb__listings.sql`

```sql
{{
    config(
        materialized='view'
    )
}}

WITH source AS (
    SELECT * FROM {{ source('raw_airbnb', 'listings') }}
),

renamed AS (
    SELECT
        TRY_CAST(id AS INTEGER) AS listing_id,
        TRY_CAST(host_id AS INTEGER) AS host_id,
        TRIM(host_name) AS host_name,
        TRIM(neighbourhood) AS neighbourhood,
        TRIM(neighbourhood_group) AS neighbourhood_group,
        TRIM(room_type) AS room_type,
        TRY_CAST(price AS DECIMAL(10, 2)) AS price,
        TRY_CAST(minimum_nights AS INTEGER) AS minimum_nights,
        TRY_CAST(number_of_reviews AS INTEGER) AS number_of_reviews,
        TRY_TO_DATE(last_review) AS last_review_date,
        TRY_CAST(reviews_per_month AS DECIMAL(10, 2)) AS reviews_per_month,
        TRY_CAST(calculated_host_listings_count AS INTEGER) AS host_listings_count,
        TRY_CAST(availability_365 AS INTEGER) AS availability_365,
        CURRENT_TIMESTAMP() AS loaded_at
    FROM source
)

SELECT * FROM renamed
WHERE listing_id IS NOT NULL
```

### Intermediate Model: `int_airbnb__calendar_enriched.sql`

```sql
{{
    config(
        materialized='table'
    )
}}

WITH calendar AS (
    SELECT * FROM {{ ref('stg_airbnb__calendar') }}
),

listings AS (
    SELECT * FROM {{ ref('int_airbnb__listing_enriched') }}
),

enriched AS (
    SELECT
        c.listing_id,
        c.calendar_date,
        c.is_available,
        c.price,
        c.minimum_nights,
        c.maximum_nights,
        l.host_id,
        l.neighbourhood,
        l.neighbourhood_group,
        l.room_type,
        l.host_is_superhost,
        -- Revenue proxy: if unavailable, assume booked
        CASE
            WHEN c.is_available = FALSE THEN c.price
            ELSE 0
        END AS estimated_revenue,
        c.loaded_at
    FROM calendar c
    INNER JOIN listings l
        ON c.listing_id = l.listing_id
)

SELECT * FROM enriched
```

### Incremental Fact Model: `fct_listing_calendar.sql`

```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        on_schema_change='sync_all_columns',
        cluster_by=['calendar_date']
    )
}}

WITH calendar_enriched AS (
    SELECT * FROM {{ ref('int_airbnb__calendar_enriched') }}
)

SELECT
    listing_id,
    calendar_date,
    is_available,
    price,
    minimum_nights,
    maximum_nights,
    host_id,
    neighbourhood,
    neighbourhood_group,
    room_type,
    estimated_revenue,
    loaded_at
FROM calendar_enriched

{% if is_incremental() %}
WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
{% endif %}
```

**Incremental strategy**: Uses Snowflake merge to update existing rows and insert new rows based on `unique_key`.

### Aggregate Model: `agg_listing_monthly_performance.sql`

```sql
{{
    config(
        materialized='table'
    )
}}

WITH calendar_facts AS (
    SELECT * FROM {{ ref('fct_listing_calendar') }}
),

monthly AS (
    SELECT
        listing_id,
        DATE_TRUNC('month', calendar_date) AS month,
        neighbourhood,
        neighbourhood_group,
        room_type,
        COUNT(*) AS total_days,
        SUM(CASE WHEN is_available THEN 1 ELSE 0 END) AS available_days,
        SUM(CASE WHEN NOT is_available THEN 1 ELSE 0 END) AS unavailable_days,
        AVG(price) AS avg_price,
        SUM(estimated_revenue) AS total_estimated_revenue
    FROM calendar_facts
    GROUP BY 1, 2, 3, 4, 5
)

SELECT
    *,
    ROUND(100.0 * available_days / total_days, 2) AS availability_rate_pct,
    ROUND(100.0 * unavailable_days / total_days, 2) AS occupancy_proxy_pct
FROM monthly
```

## dbt Tests

### Generic Tests in `schema.yml`

```yaml
models:
  - name: stg_airbnb__listings
    columns:
      - name: listing_id
        tests:
          - unique
          - not_null
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
          - relationships:
              to: ref('dim_listings')
              field: listing_id
```

### Singular Test: `tests/assert_no_duplicate_listing_dates.sql`

```sql
SELECT
    listing_id,
    calendar_date,
    COUNT(*) AS record_count
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, calendar_date
HAVING COUNT(*) > 1
```

Run singular tests:

```bash
dbt test --select test_type:singular --profiles-dir .
```

## Streamlit Dashboard

**`dashboard/streamlit_app.py`** example:

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

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

conn = get_connection()

st.title('Inside Airbnb NYC Analytics')

# Top neighbourhoods by revenue
query_neighbourhoods = """
SELECT
    neighbourhood_group,
    SUM(total_estimated_revenue) AS total_revenue
FROM agg_neighbourhood_monthly_performance
GROUP BY neighbourhood_group
ORDER BY total_revenue DESC
LIMIT 10
"""

df_neighbourhoods = pd.read_sql(query_neighbourhoods, conn)
st.subheader('Top Neighbourhoods by Estimated Revenue')
st.bar_chart(df_neighbourhoods.set_index('NEIGHBOURHOOD_GROUP'))

# Average price by room type
query_room_types = """
SELECT
    room_type,
    AVG(avg_price) AS avg_price
FROM agg_listing_monthly_performance
GROUP BY room_type
ORDER BY avg_price DESC
"""

df_room_types = pd.read_sql(query_room_types, conn)
st.subheader('Average Price by Room Type')
st.bar_chart(df_room_types.set_index('ROOM_TYPE'))

# Monthly availability trend
query_monthly = """
SELECT
    month,
    AVG(availability_rate_pct) AS avg_availability
FROM agg_listing_monthly_performance
GROUP BY month
ORDER BY month
"""

df_monthly = pd.read_sql(query_monthly, conn)
st.subheader('Monthly Availability Trend')
st.line_chart(df_monthly.set_index('MONTH'))
```

Run dashboard:

```bash
streamlit run dashboard/streamlit_app.py
```

## Common Patterns

### Full Refresh Workflow for New Data

When loading a new Inside Airbnb snapshot:

```bash
# 1. Load new raw data
python scripts/load_inside_airbnb_to_snowflake.py

# 2. Full refresh incremental models and downstream
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 3. Validate data quality
dbt test --profiles-dir .

# 4. Regenerate docs
dbt docs generate --profiles-dir .
```

### Selective Model Runs

```bash
# Run only staging models
dbt run --select staging --profiles-dir .

# Run only marts
dbt run --select marts --profiles-dir .

# Run a model and all upstream dependencies
dbt run --select +fct_listing_calendar --profiles-dir .

# Run a model and all downstream dependencies
dbt run --select dim_listings+ --profiles-dir .

# Run models matching a tag
dbt run --select tag:monthly_aggregates --profiles-dir .
```

### Testing Strategies

```bash
# Test sources
dbt test --select source:* --profiles-dir .

# Test specific layer
dbt test --select staging --profiles-dir .

# Test relationships only
dbt test --select test_type:relationships --profiles-dir .

# Fail fast on first error
dbt test --fail-fast --profiles-dir .
```

### dbt Macro for Revenue Proxy

Create `macros/calculate_revenue_proxy.sql`:

```sql
{% macro calculate_revenue_proxy(is_available_column, price_column) %}
    CASE
        WHEN {{ is_available_column }} = FALSE THEN {{ price_column }}
        ELSE 0
    END
{% endmacro %}
```

Use in models:

```sql
SELECT
    listing_id,
    calendar_date,
    {{ calculate_revenue_proxy('is_available', 'price') }} AS estimated_revenue
FROM {{ ref('stg_airbnb__calendar') }}
```

## Troubleshooting

### Connection Issues

**Problem**: `dbt debug` fails with authentication error

**Solution**: 
- Verify `profiles.yml` credentials match Snowflake account
- Check `SNOWFLAKE_PASSWORD` environment variable if using `env_var()`
- Ensure role has access to database/warehouse/schema
- Test connection with Snowflake CLI: `snowsql -a YOUR_ACCOUNT -u YOUR_USER`

### Python Loader Fails

**Problem**: `PUT` command fails with "File not found"

**Solution**:
```bash
# Ensure raw files exist
ls -lh data/raw/

# Check file paths in script are relative to working directory
# Run from project root:
python scripts/load_inside_airbnb_to_snowflake.py
```

**Problem**: `COPY INTO` fails with CSV parsing error

**Solution**:
```sql
-- Check file format in Snowflake
SELECT $1, $2, $3 FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz LIMIT 10;

-- Adjust file format parameters
FILE_FORMAT = (
    TYPE=CSV 
    SKIP_HEADER=1 
    FIELD_OPTIONALLY_ENCLOSED_BY='"'
    ESCAPE_UNENCLOSED_FIELD='NONE'
    ENCODING='UTF8'
)
```

### Incremental Model Issues

**Problem**: Incremental model not picking up new data

**Solution**:
```bash
# Check max date in target table
# In Snowflake:
SELECT MAX(calendar_date) FROM ANALYTICS.fct_listing_calendar;

# Full refresh to rebuild
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

**Problem**: Duplicate key errors on incremental merge

**Solution**:
- Verify `unique_key` in model config matches actual business key
- Check for duplicate rows in source:
```sql
SELECT listing_id, calendar_date, COUNT(*)
FROM {{ ref('int_airbnb__calendar_enriched') }}
GROUP BY 1, 2
HAVING COUNT(*) > 1
```

### Test Failures

**Problem**: Relationship test fails between `fct_listing_calendar` and `dim_listings`

**Solution**:
```bash
# Run models in dependency order
dbt run --select dim_listings fct_listing_calendar --profiles-dir .

# Check for orphaned records
SELECT DISTINCT c.listing_id
FROM fct_listing_calendar c
LEFT JOIN dim_listings l ON c.listing_id = l.listing_id
WHERE l.listing_id IS NULL
```

### Streamlit Dashboard Errors

**Problem**: `snowflake.connector.errors.ProgrammingError: Object does not exist`

**Solution**:
- Ensure dbt models have run: `dbt run --profiles-dir .`
- Verify schema in `local_credentials.json` matches dbt target schema
- Check table names are uppercase in SQL queries:
```python
query = "SELECT * FROM AGG_NEIGHBOURHOOD_MONTHLY_PERFORMANCE"
```

**Problem**: Connection timeout

**Solution**:
```python
# Add connection timeout and retry
conn = snowflake.connector.connect(
    **creds,
    login_timeout=10,
    network_timeout=10
)
```

## Project Structure Reference

```
.
├── analyses/              # Ad-hoc analysis queries (not built)
├── config/
│   └── local_credentials.json  # Snowflake creds for Streamlit (gitignored)
├── dashboard/
│   └── streamlit_app.py        # Analytics dashboard
├── data/raw/                    # Inside Airbnb CSV/GZIP files (gitignored)
├── dbt_project.yml              # dbt project config
├── macros/                      # Reusable SQL macros
├── models/
│   ├── staging/                 # stg_airbnb__* views
│   ├── intermediate/            # int_airbnb__* tables
│   └── marts/                   # dim_*, fct_*, agg_* tables
├── profiles.yml                 # dbt Snowflake connection (gitignored)
├── scripts/
│   └── load_inside_airbnb_to_snowflake.py  # Python data loader
├── setup/
│   └── snowflake_setup.sql      # DDL for warehouse, schema, stage
├── tests/                       # Singular dbt tests
└── requirements.txt             # Python dependencies
```

## Key Dependencies

From `requirements.txt`:

```txt
dbt-snowflake>=1.5.0
snowflake-connector-python>=3.0.0
streamlit>=1.20.0
pandas>=1.5.0
```

## Environment Variables (Optional)

If using environment variables for credentials:

```bash
export SNOWFLAKE_ACCOUNT=your_account
export SNOWFLAKE_USER=your_user
export SNOWFLAKE_PASSWORD=your_password
export SNOWFLAKE_WAREHOUSE=COMPUTE_WH
export SNOWFLAKE_DATABASE=AIRBNB_DEV
export SNOWFLAKE_SCHEMA=ANALYTICS
export SNOWFLAKE_ROLE=ACCOUNTADMIN
```

Reference in `profiles.yml`:

```yaml
password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
```

## Best Practices

1. **Never commit credentials**: Use `.gitignore` for `profiles.yml`, `local_credentials.json`, `.user.yml`
2. **Use incremental models for large fact tables**: `fct_listing_calendar` demonstrates merge strategy
3. **Test early and often**: Run `dbt test` after every model change
4. **Document models**: Add descriptions in `schema.yml` for dbt docs
5. **Tag models**: Use tags for selective runs (`dbt run --select tag:hourly`)
6. **Version control dbt packages**: Pin versions in `packages.yml`
7. **Cluster large tables**: Use `cluster_by` for date/time columns in Snowflake
8. **Cache Streamlit connections**: Use `@st.cache_resource` for database connections

This skill provides complete guidance for operating the Snowflake dbt Airbnb Analytics project, from raw data loading through dashboard deployment.
