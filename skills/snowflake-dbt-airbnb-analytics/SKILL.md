---
name: snowflake-dbt-airbnb-analytics
description: Build Snowflake data warehouses with dbt using the Inside Airbnb dataset, including staging, intermediate, and mart layers with Streamlit dashboards.
triggers:
  - set up snowflake dbt project with airbnb data
  - create dbt models for inside airbnb dataset
  - load inside airbnb data into snowflake
  - build dbt marts for airbnb analytics
  - configure snowflake dbt project with streamlit
  - run dbt transformations on airbnb listings
  - create incremental dbt models in snowflake
  - build analytics dashboard with streamlit and dbt
---

# Snowflake dbt Airbnb Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates analytics engineering with Snowflake and dbt using the Inside Airbnb open dataset. It covers raw data ingestion, multi-layer dbt transformations (staging → intermediate → marts), incremental models, data quality testing, and a Streamlit dashboard for reporting.

## What This Project Does

- Loads Inside Airbnb CSV/GZIP files into Snowflake internal stages
- Creates raw-zone tables with text-preserving schema
- Transforms data through dbt staging, intermediate, and mart layers
- Builds incremental fact tables using Snowflake merge strategy
- Validates data quality with dbt generic and singular tests
- Powers a Streamlit dashboard with analytics-ready marts

## Installation

```bash
# Clone the repository
git clone https://github.com/analyticsdurgesh/Snowflake_DBT_Project.git
cd Snowflake_DBT_Project

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

## Configuration

### Snowflake Credentials

Create local credential files (these are git-ignored):

```bash
# Copy example files
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

Edit `profiles.yml` with your Snowflake connection:

```yaml
snowflake_dbt_project:
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
  target: dev
```

Edit `config/local_credentials.json` for Streamlit:

```json
{
  "account": "YOUR_ACCOUNT",
  "user": "YOUR_USER",
  "password": "YOUR_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DB",
  "schema": "ANALYTICS",
  "role": "YOUR_ROLE"
}
```

**Never commit these files.** Use environment variables in production:

```python
import os
credentials = {
    "account": os.getenv("SNOWFLAKE_ACCOUNT"),
    "user": os.getenv("SNOWFLAKE_USER"),
    "password": os.getenv("SNOWFLAKE_PASSWORD"),
}
```

## Dataset Setup

Download Inside Airbnb data for New York City from [insideairbnb.com/get-the-data](https://insideairbnb.com/get-the-data/):

```text
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

## Loading Raw Data

The Python loader script creates Snowflake objects and uploads data:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

This script:
1. Reads credentials from `config/local_credentials.json`
2. Executes `setup/snowflake_setup.sql` to create database, schemas, and stage
3. Uploads raw files to Snowflake internal stage `INSIDE_AIRBNB_STAGE`
4. Creates raw tables from CSV headers
5. Copies data into `RAW` schema tables

### Example Loader Code Pattern

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
    role=creds['role']
)

cursor = conn.cursor()

# Create internal stage
cursor.execute("""
CREATE STAGE IF NOT EXISTS RAW.INSIDE_AIRBNB_STAGE
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1)
""")

# Upload local file
cursor.execute("""
PUT file://data/raw/listings.csv.gz @RAW.INSIDE_AIRBNB_STAGE
AUTO_COMPRESS=FALSE
""")

# Copy into table
cursor.execute("""
COPY INTO RAW.LISTINGS
FROM @RAW.INSIDE_AIRBNB_STAGE/listings.csv.gz
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"')
ON_ERROR = CONTINUE
""")

cursor.close()
conn.close()
```

## dbt Model Layers

### Staging Layer

Staging models clean and standardize raw data with consistent naming:

```sql
-- models/staging/stg_airbnb__listings.sql
WITH source AS (
    SELECT * FROM {{ source('airbnb_raw', 'listings') }}
)

SELECT
    id::NUMBER AS listing_id,
    name AS listing_name,
    host_id::NUMBER AS host_id,
    host_name,
    neighbourhood_cleansed AS neighbourhood,
    room_type,
    price::VARCHAR AS price_text,
    CAST(REPLACE(REPLACE(price, '$', ''), ',', '') AS FLOAT) AS price,
    minimum_nights::NUMBER AS minimum_nights,
    number_of_reviews::NUMBER AS number_of_reviews,
    last_review::DATE AS last_review_date,
    reviews_per_month::FLOAT AS reviews_per_month,
    availability_365::NUMBER AS availability_365
FROM source
```

### Intermediate Layer

Intermediate models perform reusable joins and enrichment:

```sql
-- models/intermediate/int_airbnb__listing_enriched.sql
WITH listings AS (
    SELECT * FROM {{ ref('stg_airbnb__listings') }}
),

neighbourhoods AS (
    SELECT * FROM {{ ref('stg_airbnb__neighbourhoods') }}
)

SELECT
    l.listing_id,
    l.listing_name,
    l.host_id,
    l.host_name,
    l.neighbourhood,
    n.neighbourhood_group,
    l.room_type,
    l.price,
    l.minimum_nights,
    l.number_of_reviews,
    l.last_review_date,
    l.reviews_per_month,
    l.availability_365,
    CASE
        WHEN l.number_of_reviews >= 10 THEN 'High Activity'
        WHEN l.number_of_reviews >= 1 THEN 'Medium Activity'
        ELSE 'Low Activity'
    END AS activity_level
FROM listings l
LEFT JOIN neighbourhoods n
    ON l.neighbourhood = n.neighbourhood
```

### Marts Layer

Marts are analytics-ready dimensions and facts:

```sql
-- models/marts/dim_listings.sql
{{ config(materialized='table') }}

SELECT
    listing_id,
    listing_name,
    host_id,
    host_name,
    neighbourhood,
    neighbourhood_group,
    room_type,
    price,
    minimum_nights,
    number_of_reviews,
    last_review_date,
    activity_level,
    CURRENT_TIMESTAMP() AS dbt_updated_at
FROM {{ ref('int_airbnb__listing_enriched') }}
```

### Incremental Fact Models

Use incremental materialization for large fact tables:

```sql
-- models/marts/fct_listing_calendar.sql
{{
    config(
        materialized='incremental',
        unique_key='calendar_key',
        merge_update_columns=['available', 'price', 'adjusted_price']
    )
}}

WITH calendar AS (
    SELECT * FROM {{ ref('int_airbnb__calendar_enriched') }}
)

SELECT
    {{ dbt_utils.generate_surrogate_key(['listing_id', 'calendar_date']) }} AS calendar_key,
    listing_id,
    calendar_date,
    available,
    price,
    adjusted_price,
    listing_name,
    neighbourhood,
    room_type
FROM calendar

{% if is_incremental() %}
WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
{% endif %}
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

# Run specific test
dbt test --select stg_airbnb__listings --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .

# Serve documentation
dbt docs serve --profiles-dir .
```

## Data Quality Testing

### Generic Tests (schema.yml)

```yaml
# models/staging/schema.yml
version: 2

sources:
  - name: airbnb_raw
    database: AIRBNB_DB
    schema: RAW
    tables:
      - name: listings
        columns:
          - name: id
            tests:
              - unique
              - not_null

models:
  - name: stg_airbnb__listings
    columns:
      - name: listing_id
        tests:
          - unique
          - not_null
      - name: price
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0
      - name: room_type
        tests:
          - accepted_values:
              values: ['Entire home/apt', 'Private room', 'Shared room', 'Hotel room']
```

### Singular Tests

```sql
-- tests/assert_no_duplicate_calendar_dates.sql
SELECT
    listing_id,
    calendar_date,
    COUNT(*) AS occurrences
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, calendar_date
HAVING COUNT(*) > 1
```

## Aggregation Models

Create monthly performance aggregates:

```sql
-- models/marts/agg_listing_monthly_performance.sql
{{ config(materialized='table') }}

WITH calendar_facts AS (
    SELECT * FROM {{ ref('fct_listing_calendar') }}
)

SELECT
    listing_id,
    listing_name,
    neighbourhood,
    room_type,
    DATE_TRUNC('month', calendar_date) AS month,
    COUNT(*) AS days_in_month,
    SUM(CASE WHEN available = 'f' THEN 1 ELSE 0 END) AS days_unavailable,
    SUM(CASE WHEN available = 't' THEN 1 ELSE 0 END) AS days_available,
    ROUND(days_unavailable * 100.0 / days_in_month, 2) AS occupancy_rate,
    SUM(CASE WHEN available = 'f' THEN price ELSE 0 END) AS estimated_revenue,
    AVG(price) AS avg_price
FROM calendar_facts
GROUP BY listing_id, listing_name, neighbourhood, room_type, month
```

## Streamlit Dashboard

Run the dashboard:

```bash
streamlit run dashboard/streamlit_app.py
```

### Dashboard Connection Pattern

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load credentials
with open('config/local_credentials.json') as f:
    creds = json.load(f)

@st.cache_resource
def get_snowflake_connection():
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema'],
        role=creds['role']
    )

def run_query(query):
    conn = get_snowflake_connection()
    return pd.read_sql(query, conn)

# Example query
st.title("Neighbourhood Performance")

neighbourhood_data = run_query("""
    SELECT
        neighbourhood,
        SUM(estimated_revenue) AS total_revenue,
        AVG(occupancy_rate) AS avg_occupancy
    FROM agg_neighbourhood_monthly_performance
    GROUP BY neighbourhood
    ORDER BY total_revenue DESC
    LIMIT 10
""")

st.dataframe(neighbourhood_data)
st.bar_chart(neighbourhood_data.set_index('neighbourhood')['total_revenue'])
```

## Common Patterns

### Adding a New Staging Model

1. Create the model file:

```sql
-- models/staging/stg_airbnb__new_source.sql
WITH source AS (
    SELECT * FROM {{ source('airbnb_raw', 'new_source') }}
)

SELECT
    id::NUMBER AS record_id,
    created_at::TIMESTAMP AS created_at,
    -- add more columns
FROM source
```

2. Add to `schema.yml`:

```yaml
models:
  - name: stg_airbnb__new_source
    columns:
      - name: record_id
        tests:
          - unique
          - not_null
```

3. Run the model:

```bash
dbt run --select stg_airbnb__new_source --profiles-dir .
dbt test --select stg_airbnb__new_source --profiles-dir .
```

### Referencing Models

Always use `ref()` for model dependencies:

```sql
SELECT
    l.listing_id,
    r.review_count
FROM {{ ref('dim_listings') }} l
LEFT JOIN {{ ref('fct_reviews') }} r
    ON l.listing_id = r.listing_id
```

### Using Sources

Define sources in `schema.yml` and reference with `source()`:

```yaml
sources:
  - name: airbnb_raw
    tables:
      - name: listings
```

```sql
SELECT * FROM {{ source('airbnb_raw', 'listings') }}
```

## Troubleshooting

### dbt Connection Issues

```bash
# Verify profiles.yml is in project root
ls -la profiles.yml

# Test connection
dbt debug --profiles-dir .

# Check Snowflake permissions
# Ensure role has CREATE TABLE, CREATE VIEW on target schema
```

### Incremental Model Full Refresh

If incremental logic breaks or you reload raw data:

```bash
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

### Raw Data Loading Errors

```python
# Check stage contents
cursor.execute("LIST @RAW.INSIDE_AIRBNB_STAGE")
print(cursor.fetchall())

# Manual copy with error details
cursor.execute("""
COPY INTO RAW.LISTINGS
FROM @RAW.INSIDE_AIRBNB_STAGE/listings.csv.gz
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"')
ON_ERROR = CONTINUE
VALIDATION_MODE = RETURN_ERRORS
""")
```

### Test Failures

```bash
# Run single test
dbt test --select stg_airbnb__listings --profiles-dir .

# View test results in target/run_results.json
cat target/run_results.json | jq '.results[] | select(.status != "pass")'
```

### Streamlit Connection Timeout

Increase connection timeout in credentials:

```python
conn = snowflake.connector.connect(
    **creds,
    login_timeout=30,
    network_timeout=60
)
```

## Project Structure Reference

```
models/
├── staging/           # Clean, standardize raw data
│   ├── stg_airbnb__listings.sql
│   ├── stg_airbnb__calendar.sql
│   ├── stg_airbnb__reviews.sql
│   └── schema.yml
├── intermediate/      # Reusable joins and enrichment
│   ├── int_airbnb__listing_enriched.sql
│   ├── int_airbnb__calendar_enriched.sql
│   └── int_airbnb__reviews_enriched.sql
└── marts/            # Analytics-ready outputs
    ├── dim_listings.sql
    ├── dim_hosts.sql
    ├── fct_listing_calendar.sql
    ├── fct_reviews.sql
    ├── agg_listing_monthly_performance.sql
    └── agg_neighbourhood_monthly_performance.sql

tests/
└── assert_no_duplicate_calendar_dates.sql

scripts/
└── load_inside_airbnb_to_snowflake.py

dashboard/
└── streamlit_app.py
```

## Business Questions Answered

Query patterns for common analytics:

```sql
-- Top revenue neighbourhoods
SELECT
    neighbourhood,
    SUM(estimated_revenue) AS total_revenue
FROM agg_neighbourhood_monthly_performance
GROUP BY neighbourhood
ORDER BY total_revenue DESC
LIMIT 10;

-- Hosts with most listings
SELECT
    host_id,
    host_name,
    COUNT(*) AS listing_count
FROM dim_listings
GROUP BY host_id, host_name
ORDER BY listing_count DESC
LIMIT 10;

-- Room type pricing
SELECT
    room_type,
    AVG(price) AS avg_price,
    MIN(price) AS min_price,
    MAX(price) AS max_price
FROM dim_listings
GROUP BY room_type;

-- Monthly availability trends
SELECT
    month,
    AVG(occupancy_rate) AS avg_occupancy
FROM agg_listing_monthly_performance
GROUP BY month
ORDER BY month;
```
