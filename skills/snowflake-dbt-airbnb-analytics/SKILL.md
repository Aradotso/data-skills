---
name: snowflake-dbt-airbnb-analytics
description: Build end-to-end analytics pipelines with Snowflake, dbt, and Streamlit using Inside Airbnb data
triggers:
  - set up snowflake dbt project
  - load inside airbnb data to snowflake
  - create dbt staging models for airbnb
  - build dbt marts for analytics
  - run incremental dbt models in snowflake
  - create streamlit dashboard with snowflake
  - configure dbt profiles for snowflake
  - test dbt data quality
---

# Snowflake dbt Airbnb Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build complete analytics engineering pipelines using Snowflake as the data warehouse, dbt for transformation, and Streamlit for visualization. The project demonstrates modern analytics patterns with the Inside Airbnb open dataset including staging, intermediate, and mart layers, incremental models, data quality tests, and dashboard development.

## What This Project Does

- **Data Loading**: Python script uploads Inside Airbnb CSV/GZIP files to Snowflake internal stages
- **Raw Layer**: Text-preserving raw tables in Snowflake RAW schema
- **dbt Transformations**: Staging → Intermediate → Marts pipeline with dimensional and fact models
- **Incremental Modeling**: Merge-based incremental strategy for large fact tables
- **Data Quality**: Generic and singular dbt tests for validation
- **Analytics Dashboard**: Streamlit app connected to dbt marts for reporting

## Installation

```bash
# Clone and set up virtual environment
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Create local credential files (git-ignored)
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

## Configuration

### profiles.yml

Configure dbt connection to Snowflake:

```yaml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT
      user: YOUR_USER
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: AIRBNB_ROLE
      database: AIRBNB_DB
      warehouse: AIRBNB_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

### config/local_credentials.json

Used by Python loader and Streamlit dashboard:

```json
{
  "account": "YOUR_ACCOUNT",
  "user": "YOUR_USER",
  "password": "YOUR_PASSWORD",
  "role": "AIRBNB_ROLE",
  "warehouse": "AIRBNB_WH",
  "database": "AIRBNB_DB",
  "schema": "RAW"
}
```

**Security**: Never commit these files. Use environment variables in production:

```python
import os
credentials = {
    "account": os.getenv("SNOWFLAKE_ACCOUNT"),
    "user": os.getenv("SNOWFLAKE_USER"),
    "password": os.getenv("SNOWFLAKE_PASSWORD"),
    # ...
}
```

## Data Loading

### Load Inside Airbnb Data to Snowflake

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

The loader script performs:
1. Creates Snowflake warehouse, database, schemas, and internal stage
2. Uploads local CSV/GZIP files to `@INSIDE_AIRBNB_STAGE`
3. Infers schema from CSV headers
4. Copies data into RAW tables (LISTINGS, CALENDAR, REVIEWS, NEIGHBOURHOODS)

### Python Loader Pattern

```python
import snowflake.connector
import json
from pathlib import Path

# Load credentials
with open("config/local_credentials.json") as f:
    creds = json.load(f)

# Connect to Snowflake
conn = snowflake.connector.connect(
    account=creds["account"],
    user=creds["user"],
    password=creds["password"],
    role=creds["role"],
    warehouse=creds["warehouse"],
    database=creds["database"],
    schema=creds["schema"]
)

cursor = conn.cursor()

# Upload to internal stage
cursor.execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")

# Create table from staged file
cursor.execute("""
CREATE OR REPLACE TABLE RAW.LISTINGS
USING TEMPLATE (
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*))
    FROM TABLE(
        INFER_SCHEMA(
            LOCATION=>'@INSIDE_AIRBNB_STAGE/listings.csv.gz',
            FILE_FORMAT=>'CSV_FORMAT'
        )
    )
)
""")

# Copy into table
cursor.execute("""
COPY INTO RAW.LISTINGS
FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"')
ON_ERROR = 'CONTINUE'
""")

cursor.close()
conn.close()
```

## dbt Model Development

### Project Structure

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

### Staging Model Example

```sql
-- models/staging/stg_airbnb__listings.sql
WITH source AS (
    SELECT * FROM {{ source('raw_airbnb', 'LISTINGS') }}
),

cleaned AS (
    SELECT
        id::INTEGER AS listing_id,
        name AS listing_name,
        host_id::INTEGER AS host_id,
        host_name,
        neighbourhood_cleansed AS neighbourhood,
        latitude::FLOAT AS latitude,
        longitude::FLOAT AS longitude,
        room_type,
        price::VARCHAR AS price_text,
        minimum_nights::INTEGER AS minimum_nights,
        number_of_reviews::INTEGER AS number_of_reviews,
        last_review::DATE AS last_review_date,
        reviews_per_month::FLOAT AS reviews_per_month,
        calculated_host_listings_count::INTEGER AS host_listings_count,
        availability_365::INTEGER AS availability_365
    FROM source
)

SELECT * FROM cleaned
```

### Intermediate Model with Joins

```sql
-- models/intermediate/int_airbnb__listing_enriched.sql
WITH listings AS (
    SELECT * FROM {{ ref('stg_airbnb__listings') }}
),

neighbourhoods AS (
    SELECT * FROM {{ ref('stg_airbnb__neighbourhoods') }}
),

enriched AS (
    SELECT
        l.listing_id,
        l.listing_name,
        l.host_id,
        l.host_name,
        l.neighbourhood,
        n.neighbourhood_group,
        l.latitude,
        l.longitude,
        l.room_type,
        -- Clean price: remove $ and commas
        CAST(REPLACE(REPLACE(l.price_text, '$', ''), ',', '') AS FLOAT) AS price,
        l.minimum_nights,
        l.number_of_reviews,
        l.last_review_date,
        l.reviews_per_month,
        l.host_listings_count,
        l.availability_365
    FROM listings l
    LEFT JOIN neighbourhoods n
        ON l.neighbourhood = n.neighbourhood
)

SELECT * FROM enriched
```

### Incremental Fact Table

```sql
-- models/marts/fct_listing_calendar.sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        merge_update_columns=['available', 'price', 'minimum_nights', 'maximum_nights']
    )
}}

WITH calendar AS (
    SELECT * FROM {{ ref('int_airbnb__calendar_enriched') }}
),

final AS (
    SELECT
        listing_id,
        calendar_date,
        available,
        price,
        minimum_nights,
        maximum_nights,
        -- Revenue proxy: assume unavailable nights generate price revenue
        CASE WHEN available = FALSE THEN price ELSE 0 END AS estimated_revenue
    FROM calendar
    {% if is_incremental() %}
        WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
    {% endif %}
)

SELECT * FROM final
```

### Aggregate Mart

```sql
-- models/marts/agg_neighbourhood_monthly_performance.sql
WITH listing_monthly AS (
    SELECT * FROM {{ ref('agg_listing_monthly_performance') }}
),

listings AS (
    SELECT * FROM {{ ref('dim_listings') }}
),

neighbourhood_agg AS (
    SELECT
        l.neighbourhood,
        l.neighbourhood_group,
        lm.year_month,
        COUNT(DISTINCT lm.listing_id) AS active_listings,
        SUM(lm.total_nights) AS total_nights,
        SUM(lm.available_nights) AS total_available_nights,
        SUM(lm.unavailable_nights) AS total_unavailable_nights,
        AVG(lm.avg_price) AS avg_price,
        SUM(lm.estimated_revenue) AS total_estimated_revenue,
        AVG(lm.availability_rate) AS avg_availability_rate
    FROM listing_monthly lm
    JOIN listings l ON lm.listing_id = l.listing_id
    GROUP BY l.neighbourhood, l.neighbourhood_group, lm.year_month
)

SELECT * FROM neighbourhood_agg
```

## dbt Commands

```bash
# Test connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select dim_listings+ --profiles-dir .

# Full refresh for incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Test specific model
dbt test --select stg_airbnb__listings --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .

# Compile without running
dbt compile --profiles-dir .

# Show compiled SQL
dbt show --select dim_listings --profiles-dir .
```

## Data Quality Tests

### Generic Tests in Schema YAML

```yaml
# models/staging/_staging.yml
version: 2

models:
  - name: stg_airbnb__listings
    description: Cleaned and typed listing dimension
    columns:
      - name: listing_id
        description: Unique listing identifier
        tests:
          - unique
          - not_null
      
      - name: room_type
        description: Type of room offered
        tests:
          - accepted_values:
              values: ['Entire home/apt', 'Private room', 'Shared room', 'Hotel room']
      
      - name: price
        description: Nightly price in USD
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
```

### Singular Test

```sql
-- tests/no_duplicate_listing_dates.sql
-- Verify no duplicate listing-date combinations in fact table
SELECT
    listing_id,
    calendar_date,
    COUNT(*) AS record_count
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, calendar_date
HAVING COUNT(*) > 1
```

## Streamlit Dashboard

### Dashboard Structure

```python
# dashboard/streamlit_app.py
import streamlit as st
import snowflake.connector
import pandas as pd
import json
from pathlib import Path

# Load credentials
config_path = Path(__file__).parent.parent / "config" / "local_credentials.json"
with open(config_path) as f:
    creds = json.load(f)

# Connect to Snowflake
@st.cache_resource
def get_connection():
    return snowflake.connector.connect(
        account=creds["account"],
        user=creds["user"],
        password=creds["password"],
        role=creds["role"],
        warehouse=creds["warehouse"],
        database=creds["database"],
        schema="ANALYTICS"  # dbt mart schema
    )

conn = get_connection()

# Query mart
@st.cache_data(ttl=600)
def load_neighbourhood_performance():
    query = """
    SELECT 
        neighbourhood,
        neighbourhood_group,
        year_month,
        active_listings,
        total_estimated_revenue,
        avg_price,
        avg_availability_rate
    FROM agg_neighbourhood_monthly_performance
    ORDER BY year_month DESC, total_estimated_revenue DESC
    """
    return pd.read_sql(query, conn)

# Build dashboard
st.title("🏠 Inside Airbnb Analytics")

df = load_neighbourhood_performance()

# Filters
selected_group = st.selectbox(
    "Neighbourhood Group",
    ["All"] + sorted(df["neighbourhood_group"].dropna().unique().tolist())
)

if selected_group != "All":
    df = df[df["neighbourhood_group"] == selected_group]

# Metrics
col1, col2, col3 = st.columns(3)
with col1:
    st.metric("Total Listings", f"{df['active_listings'].sum():,.0f}")
with col2:
    st.metric("Est. Revenue", f"${df['total_estimated_revenue'].sum():,.0f}")
with col3:
    st.metric("Avg Price", f"${df['avg_price'].mean():.2f}")

# Chart
st.subheader("Revenue by Neighbourhood")
top_neighbourhoods = (
    df.groupby("neighbourhood")["total_estimated_revenue"]
    .sum()
    .sort_values(ascending=False)
    .head(10)
)
st.bar_chart(top_neighbourhoods)

# Table
st.subheader("Monthly Performance")
st.dataframe(
    df[["neighbourhood", "year_month", "active_listings", "total_estimated_revenue", "avg_price"]],
    use_container_width=True
)
```

### Run Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

## Common Patterns

### Reloading New Data Snapshot

```bash
# 1. Download new Inside Airbnb snapshot to data/raw/
# 2. Load to Snowflake
python scripts/load_inside_airbnb_to_snowflake.py

# 3. Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 4. Run tests
dbt test --profiles-dir .

# 5. Regenerate docs
dbt docs generate --profiles-dir .
```

### Adding a New Mart

```sql
-- models/marts/dim_room_types.sql
{{ config(materialized='table') }}

WITH listings AS (
    SELECT * FROM {{ ref('dim_listings') }}
),

room_type_summary AS (
    SELECT
        room_type,
        COUNT(DISTINCT listing_id) AS listing_count,
        AVG(price) AS avg_price,
        MIN(price) AS min_price,
        MAX(price) AS max_price,
        AVG(number_of_reviews) AS avg_reviews,
        AVG(availability_365) AS avg_availability
    FROM listings
    GROUP BY room_type
)

SELECT * FROM room_type_summary
```

Then run:
```bash
dbt run --select dim_room_types --profiles-dir .
```

### Using dbt Macros

```sql
-- macros/clean_price.sql
{% macro clean_price(price_column) %}
    CAST(
        REPLACE(REPLACE({{ price_column }}, '$', ''), ',', '')
        AS FLOAT
    )
{% endmacro %}
```

Use in model:
```sql
SELECT
    listing_id,
    {{ clean_price('price_text') }} AS price
FROM {{ ref('stg_airbnb__listings') }}
```

### Creating Custom Tests

```sql
-- macros/test_positive_price.sql
{% test positive_price(model, column_name) %}
    SELECT *
    FROM {{ model }}
    WHERE {{ column_name }} IS NOT NULL
        AND {{ column_name }} <= 0
{% endtest %}
```

Apply in schema.yml:
```yaml
columns:
  - name: price
    tests:
      - positive_price
```

## Troubleshooting

### Connection Issues

```bash
# Verify Snowflake connection
dbt debug --profiles-dir .

# Check credentials file exists
cat profiles.yml
cat config/local_credentials.json

# Test raw Python connection
python -c "
import snowflake.connector
import json
with open('config/local_credentials.json') as f:
    c = json.load(f)
conn = snowflake.connector.connect(**c)
print('Connected:', conn.is_connected())
conn.close()
"
```

### Stage Upload Failures

```sql
-- Check stage contents
LIST @INSIDE_AIRBNB_STAGE;

-- Remove file from stage
REMOVE @INSIDE_AIRBNB_STAGE/listings.csv.gz;

-- Re-upload with Python
cursor.execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE OVERWRITE=TRUE")
```

### Incremental Model Issues

```bash
# Full refresh a single model
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Check incremental logic
dbt compile --select fct_listing_calendar --profiles-dir .
# View compiled SQL in target/compiled/
```

### Test Failures

```bash
# Run tests with verbose output
dbt test --select stg_airbnb__listings --profiles-dir .

# Store test failures in database
dbt test --store-failures --profiles-dir .

# Query failed test results
# Results stored in ANALYTICS_TESTS schema
```

### Dashboard Not Loading Data

```python
# Add debug logging
import logging
logging.basicConfig(level=logging.DEBUG)

# Check connection
conn = get_connection()
print("Connection alive:", conn.is_connected())

# Test query manually
cursor = conn.cursor()
cursor.execute("SELECT COUNT(*) FROM agg_neighbourhood_monthly_performance")
print("Row count:", cursor.fetchone()[0])
cursor.close()
```

### Missing Dependencies

```bash
# Reinstall requirements
pip install -r requirements.txt --upgrade

# Key packages
pip install dbt-snowflake snowflake-connector-python streamlit pandas
```

## Advanced Patterns

### Using dbt Snapshots

```sql
-- snapshots/listings_snapshot.sql
{% snapshot listings_snapshot %}

{{
    config(
      target_schema='snapshots',
      unique_key='listing_id',
      strategy='timestamp',
      updated_at='last_scraped',
    )
}}

SELECT * FROM {{ source('raw_airbnb', 'LISTINGS') }}

{% endsnapshot %}
```

Run snapshot:
```bash
dbt snapshot --profiles-dir .
```

### Environment-Specific Targets

```yaml
# profiles.yml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      schema: DEV_ANALYTICS
      # ... other config
    
    prod:
      type: snowflake
      schema: ANALYTICS
      # ... other config
```

Run against production:
```bash
dbt run --target prod --profiles-dir .
```

This skill provides complete coverage of building modern analytics pipelines with Snowflake, dbt, and Streamlit, enabling AI agents to assist developers in creating production-ready data warehouses and dashboards.
