---
name: snowflake-dbt-airbnb-analytics
description: Analytics engineering with Snowflake, dbt, and Inside Airbnb data including staging, marts, incremental models, tests, and Streamlit dashboards.
triggers:
  - build airbnb analytics with dbt and snowflake
  - set up inside airbnb dbt project
  - create dbt marts for airbnb data
  - load airbnb data into snowflake with dbt
  - configure dbt incremental models in snowflake
  - build streamlit dashboard from dbt marts
  - test dbt models for data quality
  - deploy analytics engineering pipeline
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill provides expertise in the Snowflake_DBT_Project, a complete analytics engineering pipeline that loads Inside Airbnb data into Snowflake, transforms it with dbt across staging/intermediate/mart layers, validates data quality with tests, and powers a Streamlit dashboard.

## What This Project Does

- **Data ingestion**: Loads Inside Airbnb CSV/GZIP files into Snowflake internal stages
- **Raw layer**: Text-preserving raw tables in Snowflake `RAW` schema
- **dbt transformation**: Staging views → intermediate enrichment → dimension/fact marts
- **Incremental models**: `fct_listing_calendar` uses Snowflake merge strategy
- **Data quality**: Generic and singular dbt tests for validation
- **Documentation**: dbt docs generation with persisted descriptions
- **Visualization**: Streamlit dashboard consuming analytics marts

## Installation

### Prerequisites

```bash
# Python 3.8+, Snowflake account
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

**requirements.txt** typically includes:
```text
dbt-snowflake>=1.5.0
snowflake-connector-python>=3.0.0
streamlit>=1.20.0
pandas>=1.5.0
```

### Credential Configuration

This project uses **local credential files** (NOT environment variables):

```bash
# Copy example files
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
      user: YOUR_USER
      password: YOUR_PASSWORD
      role: YOUR_ROLE
      database: AIRBNB_DB
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

**config/local_credentials.json** (Streamlit connection):
```json
{
  "account": "YOUR_ACCOUNT",
  "user": "YOUR_USER",
  "password": "YOUR_PASSWORD",
  "role": "YOUR_ROLE",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DB",
  "schema": "ANALYTICS"
}
```

⚠️ **These files are git-ignored**. Never commit real credentials.

## Data Setup

### Download Inside Airbnb Data

Download from [Inside Airbnb](https://insideairbnb.com/get-the-data/) (New York City recommended):

```bash
mkdir -p data/raw
# Place files:
# data/raw/listings.csv.gz
# data/raw/calendar.csv.gz
# data/raw/reviews.csv.gz
# data/raw/neighbourhoods.csv
```

### Load to Snowflake

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What this does**:
1. Reads credentials from `config/local_credentials.json`
2. Executes `setup/snowflake_setup.sql` (creates database, schemas, stage)
3. Uploads files to Snowflake internal stage `INSIDE_AIRBNB_STAGE`
4. Creates raw tables with `INFER_SCHEMA`
5. Copies data from stage to `RAW.LISTINGS`, `RAW.CALENDAR`, `RAW.REVIEWS`, `RAW.NEIGHBOURHOODS`

**Key loader code pattern**:
```python
import snowflake.connector
import json

# Load credentials
with open('config/local_credentials.json', 'r') as f:
    creds = json.load(f)

conn = snowflake.connector.connect(
    account=creds['account'],
    user=creds['user'],
    password=creds['password'],
    role=creds['role'],
    warehouse=creds['warehouse']
)

cursor = conn.cursor()

# Upload to internal stage
cursor.execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")

# Create table from schema inference
cursor.execute("""
CREATE OR REPLACE TABLE RAW.LISTINGS
USING TEMPLATE (
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*))
    FROM TABLE(INFER_SCHEMA(
        LOCATION=>'@INSIDE_AIRBNB_STAGE/listings.csv.gz',
        FILE_FORMAT=>'CSV_WITH_HEADER'
    ))
)
""")

# Copy data
cursor.execute("""
COPY INTO RAW.LISTINGS
FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"')
ON_ERROR = 'CONTINUE'
""")
```

## dbt Project Structure

```
models/
├── staging/
│   ├── _staging.yml              # Sources + staging docs
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
    ├── fct_listing_calendar.sql          # INCREMENTAL
    ├── fct_reviews.sql
    ├── agg_listing_monthly_performance.sql
    └── agg_neighbourhood_monthly_performance.sql
```

### Source Definition

**models/staging/_staging.yml**:
```yaml
version: 2

sources:
  - name: raw_airbnb
    database: AIRBNB_DB
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
      - name: reviews
      - name: neighbourhoods
```

### Staging Layer Example

**models/staging/stg_airbnb__listings.sql**:
```sql
WITH source AS (
    SELECT * FROM {{ source('raw_airbnb', 'listings') }}
),

cleaned AS (
    SELECT
        CAST(id AS INTEGER) AS listing_id,
        name AS listing_name,
        host_id,
        host_name,
        neighbourhood_cleansed AS neighbourhood,
        room_type,
        CAST(price AS DECIMAL(10,2)) AS price,
        minimum_nights,
        number_of_reviews,
        last_review,
        reviews_per_month,
        calculated_host_listings_count,
        availability_365,
        number_of_reviews_ltm,
        license
    FROM source
)

SELECT * FROM cleaned
```

### Intermediate Layer Example

**models/intermediate/int_airbnb__calendar_enriched.sql**:
```sql
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
        c.available,
        c.price AS daily_price,
        l.listing_name,
        l.neighbourhood,
        l.room_type,
        l.host_id,
        l.host_name,
        -- Revenue proxy: unavailable nights assumed booked
        CASE
            WHEN c.available = 'f' THEN c.price
            ELSE 0
        END AS estimated_revenue
    FROM calendar c
    LEFT JOIN listings l
        ON c.listing_id = l.listing_id
)

SELECT * FROM enriched
```

### Incremental Fact Table

**models/marts/fct_listing_calendar.sql**:
```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        merge_update_columns=['available', 'daily_price', 'estimated_revenue']
    )
}}

WITH calendar_enriched AS (
    SELECT * FROM {{ ref('int_airbnb__calendar_enriched') }}
)

SELECT
    listing_id,
    calendar_date,
    available,
    daily_price,
    estimated_revenue,
    listing_name,
    neighbourhood,
    room_type,
    host_id
FROM calendar_enriched

{% if is_incremental() %}
    -- Only process new/changed dates
    WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
{% endif %}
```

### Aggregate Mart Example

**models/marts/agg_neighbourhood_monthly_performance.sql**:
```sql
WITH listing_monthly AS (
    SELECT * FROM {{ ref('agg_listing_monthly_performance') }}
),

neighbourhood_agg AS (
    SELECT
        year_month,
        neighbourhood,
        COUNT(DISTINCT listing_id) AS total_listings,
        SUM(total_available_nights) AS total_available_nights,
        SUM(total_unavailable_nights) AS total_unavailable_nights,
        ROUND(AVG(avg_daily_price), 2) AS avg_price,
        SUM(estimated_monthly_revenue) AS estimated_monthly_revenue
    FROM listing_monthly
    GROUP BY 1, 2
)

SELECT
    *,
    ROUND(
        total_unavailable_nights::DECIMAL / NULLIF(total_available_nights + total_unavailable_nights, 0) * 100,
        2
    ) AS occupancy_rate_pct
FROM neighbourhood_agg
ORDER BY year_month DESC, estimated_monthly_revenue DESC
```

## Running dbt

### Key Commands

```bash
# Verify connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select stg_airbnb__listings+ --profiles-dir .

# Run only marts
dbt run --select marts --profiles-dir .

# Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Test specific model
dbt test --select dim_listings --profiles-dir .

# Generate and serve documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

### Typical Development Workflow

```bash
# 1. Load new data
python scripts/load_inside_airbnb_to_snowflake.py

# 2. Full refresh incremental fact
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 3. Run all downstream models
dbt run --select fct_listing_calendar+ --profiles-dir .

# 4. Test everything
dbt test --profiles-dir .

# 5. Generate docs
dbt docs generate --profiles-dir .
```

## Testing Patterns

### Generic Tests

**models/marts/_marts.yml**:
```yaml
version: 2

models:
  - name: dim_listings
    description: Dimension table for Airbnb listings
    columns:
      - name: listing_id
        description: Primary key
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

**tests/assert_no_duplicate_listing_dates.sql**:
```sql
-- Ensure fct_listing_calendar has unique listing_id + calendar_date
SELECT
    listing_id,
    calendar_date,
    COUNT(*) AS duplicate_count
FROM {{ ref('fct_listing_calendar') }}
GROUP BY 1, 2
HAVING COUNT(*) > 1
```

**tests/assert_positive_prices.sql**:
```sql
SELECT *
FROM {{ ref('fct_listing_calendar') }}
WHERE daily_price < 0
```

## Streamlit Dashboard

**dashboard/streamlit_app.py**:
```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load credentials
with open('config/local_credentials.json', 'r') as f:
    creds = json.load(f)

@st.cache_resource
def get_snowflake_connection():
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        role=creds['role'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema']
    )

def run_query(query):
    conn = get_snowflake_connection()
    return pd.read_sql(query, conn)

st.title("Inside Airbnb Analytics Dashboard")

# Neighbourhood performance
st.header("Top Neighbourhoods by Revenue")
neighbourhood_query = """
SELECT
    neighbourhood,
    SUM(estimated_monthly_revenue) AS total_revenue,
    AVG(occupancy_rate_pct) AS avg_occupancy
FROM agg_neighbourhood_monthly_performance
WHERE year_month >= DATEADD('month', -6, CURRENT_DATE())
GROUP BY 1
ORDER BY total_revenue DESC
LIMIT 10
"""
df_neighbourhoods = run_query(neighbourhood_query)
st.dataframe(df_neighbourhoods)
st.bar_chart(df_neighbourhoods.set_index('NEIGHBOURHOOD')['TOTAL_REVENUE'])

# Top hosts
st.header("Top Hosts by Listing Count")
host_query = """
SELECT
    host_id,
    host_name,
    COUNT(DISTINCT listing_id) AS listing_count
FROM dim_listings
GROUP BY 1, 2
ORDER BY listing_count DESC
LIMIT 10
"""
df_hosts = run_query(host_query)
st.dataframe(df_hosts)

# Room type pricing
st.header("Average Price by Room Type")
room_type_query = """
SELECT
    room_type,
    ROUND(AVG(price), 2) AS avg_price
FROM dim_listings
GROUP BY 1
ORDER BY avg_price DESC
"""
df_room_types = run_query(room_type_query)
st.bar_chart(df_room_types.set_index('ROOM_TYPE'))
```

**Run the dashboard**:
```bash
streamlit run dashboard/streamlit_app.py
```

## Common Patterns

### Adding a New Staging Model

```sql
-- models/staging/stg_airbnb__new_source.sql
WITH source AS (
    SELECT * FROM {{ source('raw_airbnb', 'new_source') }}
),

cleaned AS (
    SELECT
        CAST(id AS INTEGER) AS record_id,
        TRIM(name) AS record_name,
        -- Cast and clean fields
        CURRENT_TIMESTAMP() AS loaded_at
    FROM source
)

SELECT * FROM cleaned
```

Update `_staging.yml`:
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

### Creating a New Mart

```sql
-- models/marts/dim_new_dimension.sql
{{ config(materialized='table') }}

WITH source AS (
    SELECT * FROM {{ ref('int_airbnb__new_intermediate') }}
),

final AS (
    SELECT
        dimension_key,
        attribute_1,
        attribute_2,
        -- Surrogate key if needed
        {{ dbt_utils.generate_surrogate_key(['natural_key_1', 'natural_key_2']) }} AS surrogate_key
    FROM source
)

SELECT * FROM final
```

### Macro for Revenue Calculation

**macros/calculate_revenue.sql**:
```sql
{% macro calculate_revenue(available_col, price_col) %}
    CASE
        WHEN {{ available_col }} = 'f' THEN {{ price_col }}
        ELSE 0
    END
{% endmacro %}
```

**Usage in model**:
```sql
SELECT
    listing_id,
    calendar_date,
    {{ calculate_revenue('available', 'price') }} AS estimated_revenue
FROM {{ ref('stg_airbnb__calendar') }}
```

## Troubleshooting

### Connection Issues

```bash
# Test Snowflake connection
dbt debug --profiles-dir .

# Common fixes:
# - Check profiles.yml account format (no https://, no .snowflakecomputing.com)
# - Verify role has access to database/warehouse
# - Check warehouse is running
```

### Incremental Model Not Updating

```bash
# Force full refresh
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Check unique_key is correct
# Verify merge_update_columns list
```

### Stage Upload Fails

```python
# Check file exists
import os
assert os.path.exists('data/raw/listings.csv.gz')

# Verify Snowflake stage
cursor.execute("LIST @INSIDE_AIRBNB_STAGE")
print(cursor.fetchall())

# Re-create stage if corrupted
cursor.execute("DROP STAGE IF EXISTS INSIDE_AIRBNB_STAGE")
cursor.execute("CREATE STAGE INSIDE_AIRBNB_STAGE")
```

### Test Failures

```bash
# Run specific test with verbose output
dbt test --select dim_listings --profiles-dir . --store-failures

# Query failed rows (stored in dbt_test__audit schema)
# SELECT * FROM dbt_test__audit.unique_dim_listings_listing_id

# Common fixes:
# - Check raw data quality
# - Verify relationships between tables
# - Update accepted_values if new categories exist
```

### Dashboard Connection Error

```python
# Verify credentials file
import json
with open('config/local_credentials.json') as f:
    creds = json.load(f)
    print(f"Connecting as {creds['user']} to {creds['account']}")

# Test direct connection
import snowflake.connector
conn = snowflake.connector.connect(**creds)
print(conn.cursor().execute("SELECT CURRENT_USER()").fetchone())
```

### Performance Optimization

```sql
-- Add clustering key to fact table (dbt_project.yml)
models:
  snowflake_dbt_project:
    marts:
      fct_listing_calendar:
        +cluster_by: ['calendar_date']

-- Partition incremental loads by date range
{% if is_incremental() %}
    WHERE calendar_date BETWEEN 
        (SELECT MAX(calendar_date) FROM {{ this }})
        AND CURRENT_DATE()
{% endif %}
```

## Best Practices

1. **Always use `--profiles-dir .`** to reference local profiles.yml
2. **Never commit** `profiles.yml`, `config/local_credentials.json`, `target/`, `logs/`
3. **Test incrementally** with `--select model_name` during development
4. **Document models** in schema YAML files
5. **Use intermediate layer** for reusable business logic
6. **Full refresh facts** when raw data is reloaded completely
7. **Version control dbt_project.yml** configuration changes
8. **Cache Streamlit queries** with `@st.cache_resource` or `@st.cache_data`

This skill enables you to build production-grade analytics engineering pipelines with Snowflake and dbt following dimensional modeling best practices.
