---
name: snowflake-dbt-airbnb-analytics
description: Build end-to-end Snowflake + dbt analytics pipelines with Inside Airbnb data, incremental models, and Streamlit dashboards
triggers:
  - build a dbt project with Snowflake
  - load Inside Airbnb data into Snowflake
  - create incremental dbt models in Snowflake
  - build analytics marts with dbt and Snowflake
  - set up a Streamlit dashboard with Snowflake
  - configure dbt profiles for Snowflake
  - test data quality in dbt
  - create staging and mart layers in dbt
---

# Snowflake dbt Airbnb Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill teaches you to build production-grade analytics engineering pipelines using Snowflake as the data warehouse and dbt (data build tool) for transformations. The project demonstrates a complete workflow: loading Inside Airbnb open data into Snowflake, transforming it through staging → intermediate → mart layers, validating data quality with tests, and powering a Streamlit dashboard.

## What This Project Does

- **Data Ingestion**: Loads CSV/GZIP files from Inside Airbnb into Snowflake internal stages
- **Layered Transformation**: Implements dbt staging (cleaning), intermediate (enrichment), and mart (analytics-ready) models
- **Incremental Processing**: Uses Snowflake merge strategy for efficient fact table updates
- **Data Quality**: Enforces uniqueness, referential integrity, and business rules via dbt tests
- **Analytics Output**: Dimension tables, fact tables, and monthly aggregates for reporting
- **Visualization**: Streamlit dashboard connected directly to Snowflake marts

## Installation

### Prerequisites

- Snowflake account with CREATE DATABASE privileges
- Python 3.8+
- Inside Airbnb dataset files (listings.csv.gz, calendar.csv.gz, reviews.csv.gz, neighbourhoods.csv)

### Setup Steps

```bash
# Clone and enter the project
git clone https://github.com/analyticsdurgesh/Snowflake_DBT_Project.git
cd Snowflake_DBT_Project

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configuration Files

Create local credential files from examples (these are git-ignored):

```bash
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
      account: YOUR_ACCOUNT.snowflakecomputing.com
      user: YOUR_USERNAME
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"  # Or direct password for local dev
      role: YOUR_ROLE
      database: AIRBNB_DEV
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

**Edit `config/local_credentials.json`:**

```json
{
  "account": "YOUR_ACCOUNT.snowflakecomputing.com",
  "user": "YOUR_USERNAME",
  "password": "YOUR_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DEV",
  "schema": "RAW",
  "role": "YOUR_ROLE"
}
```

**Security Note**: Never commit `profiles.yml` or `config/local_credentials.json`. Use environment variables in production:

```bash
export SNOWFLAKE_PASSWORD="your_password"
export SNOWFLAKE_ACCOUNT="your_account"
```

## Data Loading Workflow

### 1. Prepare Raw Data

Place Inside Airbnb files in `data/raw/`:

```
data/raw/
├── listings.csv.gz
├── calendar.csv.gz
├── reviews.csv.gz
└── neighbourhoods.csv
```

Download from: https://insideairbnb.com/get-the-data/ (New York City recommended)

### 2. Run the Loader Script

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

This script:
- Executes `setup/snowflake_setup.sql` to create database, schemas, stages
- Uploads local files to Snowflake internal stage `INSIDE_AIRBNB_STAGE`
- Creates raw tables with text columns (preserving all data)
- Copies data from stage into `RAW.LISTINGS`, `RAW.CALENDAR`, `RAW.REVIEWS`, `RAW.NEIGHBOURHOODS`

**Key loader code pattern:**

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

# Upload file to stage
cursor.execute(f"PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")

# Copy into raw table
cursor.execute("""
    COPY INTO RAW.LISTINGS
    FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
    FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"')
""")
```

## dbt Workflow

### Project Structure

```
models/
├── staging/
│   ├── sources.yml                    # Define RAW schema sources
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
    ├── fct_listing_calendar.sql       # Incremental merge
    ├── fct_reviews.sql
    ├── agg_listing_monthly_performance.sql
    └── agg_neighbourhood_monthly_performance.sql
```

### Define Sources

**`models/staging/sources.yml`:**

```yaml
version: 2

sources:
  - name: raw_airbnb
    database: AIRBNB_DEV
    schema: RAW
    tables:
      - name: listings
        columns:
          - name: id
            tests:
              - unique
              - not_null
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

### Staging Models (Cleaning Layer)

**`models/staging/stg_airbnb__listings.sql`:**

```sql
WITH source AS (
    SELECT * FROM {{ source('raw_airbnb', 'listings') }}
),

cleaned AS (
    SELECT
        TRY_CAST(id AS INTEGER) AS listing_id,
        name AS listing_name,
        TRY_CAST(host_id AS INTEGER) AS host_id,
        host_name,
        LOWER(TRIM(neighbourhood_cleansed)) AS neighbourhood,
        LOWER(TRIM(room_type)) AS room_type,
        TRY_CAST(price AS FLOAT) AS price_raw,
        -- Remove dollar signs and parse price
        TRY_CAST(REPLACE(REPLACE(price, '$', ''), ',', '') AS DECIMAL(10,2)) AS price,
        TRY_CAST(minimum_nights AS INTEGER) AS minimum_nights,
        TRY_CAST(number_of_reviews AS INTEGER) AS number_of_reviews,
        TRY_TO_DATE(last_review) AS last_review_date,
        TRY_CAST(reviews_per_month AS FLOAT) AS reviews_per_month,
        TRY_CAST(availability_365 AS INTEGER) AS availability_365
    FROM source
    WHERE TRY_CAST(id AS INTEGER) IS NOT NULL
)

SELECT * FROM cleaned
```

**`models/staging/stg_airbnb__calendar.sql`:**

```sql
WITH source AS (
    SELECT * FROM {{ source('raw_airbnb', 'calendar') }}
),

cleaned AS (
    SELECT
        TRY_CAST(listing_id AS INTEGER) AS listing_id,
        TRY_TO_DATE(date) AS calendar_date,
        LOWER(TRIM(available)) = 't' AS is_available,
        TRY_CAST(REPLACE(REPLACE(price, '$', ''), ',', '') AS DECIMAL(10,2)) AS price,
        TRY_CAST(REPLACE(REPLACE(adjusted_price, '$', ''), ',', '') AS DECIMAL(10,2)) AS adjusted_price
    FROM source
    WHERE TRY_CAST(listing_id AS INTEGER) IS NOT NULL
      AND TRY_TO_DATE(date) IS NOT NULL
)

SELECT * FROM cleaned
```

### Intermediate Models (Enrichment Layer)

**`models/intermediate/int_airbnb__calendar_enriched.sql`:**

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
        c.is_available,
        c.price,
        l.neighbourhood,
        l.room_type,
        l.host_id,
        -- Revenue proxy: if unavailable, assume booked
        CASE 
            WHEN NOT c.is_available THEN COALESCE(c.price, l.price, 0)
            ELSE 0 
        END AS estimated_revenue
    FROM calendar c
    LEFT JOIN listings l ON c.listing_id = l.listing_id
)

SELECT * FROM enriched
```

### Mart Models (Analytics Layer)

**`models/marts/dim_listings.sql`:**

```sql
WITH listings AS (
    SELECT * FROM {{ ref('int_airbnb__listing_enriched') }}
),

final AS (
    SELECT
        listing_id,
        listing_name,
        host_id,
        host_name,
        neighbourhood,
        room_type,
        price,
        minimum_nights,
        number_of_reviews,
        last_review_date,
        reviews_per_month,
        availability_365,
        CURRENT_TIMESTAMP() AS dbt_updated_at
    FROM listings
)

SELECT * FROM final
```

**`models/marts/fct_listing_calendar.sql` (Incremental):**

```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        merge_update_columns=['is_available', 'price', 'estimated_revenue']
    )
}}

WITH calendar_enriched AS (
    SELECT * FROM {{ ref('int_airbnb__calendar_enriched') }}
),

final AS (
    SELECT
        listing_id,
        calendar_date,
        is_available,
        price,
        neighbourhood,
        room_type,
        host_id,
        estimated_revenue,
        CURRENT_TIMESTAMP() AS dbt_updated_at
    FROM calendar_enriched
    {% if is_incremental() %}
        WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
    {% endif %}
)

SELECT * FROM final
```

**`models/marts/agg_listing_monthly_performance.sql`:**

```sql
WITH calendar_facts AS (
    SELECT * FROM {{ ref('fct_listing_calendar') }}
),

monthly_agg AS (
    SELECT
        listing_id,
        neighbourhood,
        room_type,
        DATE_TRUNC('month', calendar_date) AS performance_month,
        COUNT(*) AS total_days,
        SUM(CASE WHEN is_available THEN 1 ELSE 0 END) AS available_days,
        SUM(CASE WHEN NOT is_available THEN 1 ELSE 0 END) AS unavailable_days,
        AVG(price) AS avg_daily_price,
        SUM(estimated_revenue) AS estimated_monthly_revenue,
        ROUND(100.0 * SUM(CASE WHEN is_available THEN 1 ELSE 0 END) / COUNT(*), 2) AS availability_rate
    FROM calendar_facts
    GROUP BY listing_id, neighbourhood, room_type, DATE_TRUNC('month', calendar_date)
)

SELECT * FROM monthly_agg
```

### Running dbt

```bash
# Verify connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select fct_listing_calendar+ --profiles-dir .

# Full refresh for incremental models
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Test data quality
dbt test --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

## Data Quality Tests

### Schema Tests (YAML)

**`models/marts/schema.yml`:**

```yaml
version: 2

models:
  - name: dim_listings
    description: "Dimension table of unique Airbnb listings"
    columns:
      - name: listing_id
        description: "Unique listing identifier"
        tests:
          - unique
          - not_null
      - name: room_type
        tests:
          - accepted_values:
              values: ['entire home/apt', 'private room', 'shared room', 'hotel room']
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
      - name: calendar_date
        tests:
          - not_null
```

### Singular Tests (SQL)

**`tests/listing_calendar_no_duplicates.sql`:**

```sql
-- Ensure no duplicate listing-date combinations
SELECT
    listing_id,
    calendar_date,
    COUNT(*) AS row_count
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, calendar_date
HAVING COUNT(*) > 1
```

## Streamlit Dashboard

**`dashboard/streamlit_app.py`:**

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

st.set_page_config(page_title="Airbnb Analytics", layout="wide")

# Load credentials
@st.cache_resource
def get_connection():
    with open('config/local_credentials.json') as f:
        creds = json.load(f)
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema='ANALYTICS',
        role=creds['role']
    )

conn = get_connection()

st.title("🏠 Inside Airbnb Analytics Dashboard")

# Top neighbourhoods by revenue
st.header("Top 10 Neighbourhoods by Estimated Revenue")
query = """
SELECT
    neighbourhood,
    SUM(estimated_monthly_revenue) AS total_revenue,
    AVG(availability_rate) AS avg_availability_rate
FROM ANALYTICS.AGG_NEIGHBOURHOOD_MONTHLY_PERFORMANCE
GROUP BY neighbourhood
ORDER BY total_revenue DESC
LIMIT 10
"""
df = pd.read_sql(query, conn)
st.bar_chart(df.set_index('neighbourhood')['TOTAL_REVENUE'])

# Room type distribution
st.header("Average Price by Room Type")
query = """
SELECT
    room_type,
    AVG(price) AS avg_price,
    COUNT(DISTINCT listing_id) AS listing_count
FROM ANALYTICS.DIM_LISTINGS
GROUP BY room_type
ORDER BY avg_price DESC
"""
df_room = pd.read_sql(query, conn)
st.dataframe(df_room)

# Monthly trends
st.header("Monthly Revenue Trend")
query = """
SELECT
    performance_month,
    SUM(estimated_monthly_revenue) AS total_revenue
FROM ANALYTICS.AGG_LISTING_MONTHLY_PERFORMANCE
GROUP BY performance_month
ORDER BY performance_month
"""
df_monthly = pd.read_sql(query, conn)
st.line_chart(df_monthly.set_index('PERFORMANCE_MONTH'))
```

**Run dashboard:**

```bash
streamlit run dashboard/streamlit_app.py
```

## Common Patterns

### Pattern: Incremental Model with Merge Strategy

```sql
{{
    config(
        materialized='incremental',
        unique_key='unique_id',
        merge_update_columns=['column_to_update'],
        on_schema_change='append_new_columns'
    )
}}

SELECT * FROM {{ ref('source_model') }}
{% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

### Pattern: Reusable Source Test

```yaml
sources:
  - name: raw_airbnb
    tables:
      - name: listings
        tests:
          - dbt_utils.expression_is_true:
              expression: "id IS NOT NULL"
```

### Pattern: Custom Macro for Price Parsing

**`macros/parse_price.sql`:**

```sql
{% macro parse_price(column_name) %}
    TRY_CAST(
        REPLACE(REPLACE({{ column_name }}, '$', ''), ',', '') 
        AS DECIMAL(10,2)
    )
{% endmacro %}
```

**Usage:**

```sql
SELECT {{ parse_price('price') }} AS clean_price
FROM {{ source('raw_airbnb', 'listings') }}
```

### Pattern: Date Spine for Missing Dates

```sql
WITH date_spine AS (
    SELECT DATEADD(DAY, ROW_NUMBER() OVER (ORDER BY SEQ4()) - 1, '2024-01-01') AS calendar_date
    FROM TABLE(GENERATOR(ROWCOUNT => 365))
)
SELECT * FROM date_spine
```

## Troubleshooting

### Issue: `dbt debug` fails with "Could not connect to Snowflake"

**Solution:**
- Verify `profiles.yml` is in project root (not in `~/.dbt/`)
- Check `--profiles-dir .` flag in all dbt commands
- Test Snowflake credentials directly:

```python
import snowflake.connector
conn = snowflake.connector.connect(
    account='YOUR_ACCOUNT',
    user='YOUR_USER',
    password='YOUR_PASSWORD'
)
print(conn.cursor().execute("SELECT CURRENT_VERSION()").fetchone())
```

### Issue: Loader script fails with "File not found"

**Solution:**
- Ensure `data/raw/` contains all four files
- Check file names match exactly: `listings.csv.gz`, `calendar.csv.gz`, etc.
- Verify Snowflake stage exists:

```sql
SHOW STAGES IN SCHEMA RAW;
LIST @RAW.INSIDE_AIRBNB_STAGE;
```

### Issue: Incremental model not updating

**Solution:**
- Run full refresh:

```bash
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

- Verify unique_key constraint:

```sql
SELECT listing_id, calendar_date, COUNT(*)
FROM ANALYTICS.FCT_LISTING_CALENDAR
GROUP BY listing_id, calendar_date
HAVING COUNT(*) > 1
```

### Issue: Price columns showing NULL

**Solution:**
- Raw data may have inconsistent formatting
- Inspect source:

```sql
SELECT price, REPLACE(REPLACE(price, '$', ''), ',', '') AS cleaned
FROM RAW.LISTINGS
LIMIT 10;
```

- Update staging model to handle edge cases:

```sql
COALESCE(
    TRY_CAST(REPLACE(REPLACE(NULLIF(price, ''), '$', ''), ',', '') AS DECIMAL(10,2)),
    0
) AS price
```

### Issue: Tests failing on `accepted_values`

**Solution:**
- Check for case sensitivity and whitespace:

```sql
SELECT DISTINCT LOWER(TRIM(room_type)) FROM RAW.LISTINGS;
```

- Update test to match actual values or add `quote: false` for case-insensitive matching

### Issue: Streamlit dashboard slow

**Solution:**
- Add indexes in Snowflake:

```sql
-- Snowflake automatically clusters, but you can optimize:
ALTER TABLE ANALYTICS.FCT_LISTING_CALENDAR CLUSTER BY (calendar_date);
```

- Use `@st.cache_data` for expensive queries:

```python
@st.cache_data(ttl=600)
def load_data(query):
    return pd.read_sql(query, conn)
```

## Key Commands Reference

```bash
# Installation
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# Data loading
python scripts/load_inside_airbnb_to_snowflake.py

# dbt workflow
dbt debug --profiles-dir .
dbt deps --profiles-dir .
dbt run --profiles-dir .
dbt test --profiles-dir .
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .

# Selective runs
dbt run --select staging --profiles-dir .
dbt run --select fct_listing_calendar+ --profiles-dir .
dbt test --select dim_listings --profiles-dir .

# Full refresh
dbt run --full-refresh --profiles-dir .

# Dashboard
streamlit run dashboard/streamlit_app.py
```

## Best Practices

1. **Never commit credentials**: Always use `.gitignore` for `profiles.yml`, `config/local_credentials.json`
2. **Use environment variables in production**: Replace hardcoded passwords with `{{ env_var('SNOWFLAKE_PASSWORD') }}`
3. **Test early and often**: Run `dbt test` after every model change
4. **Document models**: Add descriptions in `schema.yml` for all tables and key columns
5. **Incremental models**: Use for large fact tables (>1M rows); full tables for dimensions
6. **Stage files before transformation**: Keep raw data immutable in `RAW` schema
7. **Use intermediate models**: Don't repeat complex joins in multiple marts

This skill provides a complete reference for building Snowflake + dbt analytics pipelines with real-world patterns for data loading, transformation, testing, and visualization.
