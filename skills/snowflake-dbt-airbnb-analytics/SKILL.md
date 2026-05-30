---
name: snowflake-dbt-airbnb-analytics
description: Analytics engineering project with Snowflake, dbt, Inside Airbnb data, incremental models, data quality tests, and Streamlit dashboard
triggers:
  - set up snowflake dbt project with inside airbnb data
  - build dbt models for airbnb analytics in snowflake
  - create incremental fact tables with dbt and snowflake
  - load inside airbnb data into snowflake with python
  - build streamlit dashboard from snowflake dbt marts
  - configure dbt staging intermediate and mart layers
  - implement dbt data quality tests for airbnb dataset
  - create snowflake analytics project with dbt transformations
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill helps you build and work with a Snowflake + dbt analytics engineering project using the Inside Airbnb open dataset. The project demonstrates modern data warehouse patterns: raw data ingestion, multi-layer dbt transformations (staging → intermediate → marts), incremental fact modeling, data quality testing, and a Streamlit reporting dashboard.

## What This Project Does

- **Data Ingestion**: Loads Inside Airbnb CSV/GZIP files (listings, calendar, reviews, neighbourhoods) into Snowflake internal stages
- **dbt Transformations**: Three-layer architecture (staging, intermediate, marts) with views, tables, and incremental models
- **Incremental Facts**: Uses Snowflake merge strategy for efficient daily calendar fact updates
- **Data Quality**: Generic and singular dbt tests for primary keys, accepted values, relationships, and business logic
- **Dashboard**: Streamlit app querying Snowflake marts for neighbourhood, host, and listing analytics

## Installation

### Prerequisites

- Snowflake account with database creation permissions
- Python 3.8+
- Inside Airbnb data files downloaded to `data/raw/`:
  - `listings.csv.gz`
  - `calendar.csv.gz`
  - `reviews.csv.gz`
  - `neighbourhoods.csv`

### Setup Steps

```bash
# Clone or navigate to project
cd Snowflake_DBT_Project

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Configure local credentials (not committed to git)
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

Edit `profiles.yml` with your Snowflake connection:

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
```

Edit `config/local_credentials.json` for the Streamlit dashboard:

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

**Security Note**: Never commit these files. They are `.gitignore`d by default.

## Loading Raw Data into Snowflake

The Python loader script automates Snowflake setup and data ingestion:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

### What the Loader Does

1. Executes `setup/snowflake_setup.sql` to create database, schemas, warehouse, and internal stage
2. Uploads local CSV/GZIP files to `INSIDE_AIRBNB_STAGE`
3. Infers columns from CSV headers
4. Creates raw tables in the `RAW` schema with all VARCHAR columns (preserves source exactly)
5. Copies data from stage into tables

### Loader Code Pattern

```python
import snowflake.connector
import json

# Load credentials (agent: read from ENV or config file)
with open('config/local_credentials.json') as f:
    creds = json.load(f)

conn = snowflake.connector.connect(
    account=creds['account'],
    user=creds['user'],
    password=creds['password'],
    warehouse=creds['warehouse'],
    database=creds['database']
)

cursor = conn.cursor()

# Example: Upload file to stage
cursor.execute("USE SCHEMA RAW")
cursor.execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")

# Example: Create raw table from CSV header
cursor.execute("""
CREATE OR REPLACE TABLE RAW.LISTINGS (
  ID VARCHAR,
  NAME VARCHAR,
  HOST_ID VARCHAR,
  NEIGHBOURHOOD VARCHAR,
  ROOM_TYPE VARCHAR,
  PRICE VARCHAR,
  MINIMUM_NIGHTS VARCHAR,
  NUMBER_OF_REVIEWS VARCHAR,
  -- ... other columns as VARCHAR
)
""")

# Example: Copy from stage
cursor.execute("""
COPY INTO RAW.LISTINGS
FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
ON_ERROR = 'CONTINUE'
""")

conn.close()
```

## dbt Model Architecture

### Layer 1: Staging (`models/staging/`)

Clean, cast, and standardize raw data.

**Example: `stg_airbnb__listings.sql`**

```sql
WITH source AS (
    SELECT * FROM {{ source('airbnb_raw', 'listings') }}
),

cleaned AS (
    SELECT
        TRY_TO_NUMBER(id) AS listing_id,
        name AS listing_name,
        TRY_TO_NUMBER(host_id) AS host_id,
        host_name,
        neighbourhood,
        neighbourhood_group,
        latitude::FLOAT AS latitude,
        longitude::FLOAT AS longitude,
        room_type,
        TRY_TO_NUMBER(REPLACE(price, '$', '')) AS price,
        TRY_TO_NUMBER(minimum_nights) AS minimum_nights,
        TRY_TO_NUMBER(number_of_reviews) AS number_of_reviews,
        last_review::DATE AS last_review_date,
        TRY_TO_NUMBER(reviews_per_month) AS reviews_per_month,
        TRY_TO_NUMBER(calculated_host_listings_count) AS host_listings_count,
        TRY_TO_NUMBER(availability_365) AS availability_365
    FROM source
)

SELECT * FROM cleaned
WHERE listing_id IS NOT NULL
```

**Example: `stg_airbnb__calendar.sql`**

```sql
WITH source AS (
    SELECT * FROM {{ source('airbnb_raw', 'calendar') }}
),

cleaned AS (
    SELECT
        TRY_TO_NUMBER(listing_id) AS listing_id,
        date::DATE AS calendar_date,
        CASE 
            WHEN LOWER(available) = 't' THEN TRUE
            WHEN LOWER(available) = 'f' THEN FALSE
            ELSE NULL
        END AS is_available,
        TRY_TO_NUMBER(REPLACE(REPLACE(price, '$', ''), ',', '')) AS price,
        TRY_TO_NUMBER(minimum_nights) AS minimum_nights,
        TRY_TO_NUMBER(maximum_nights) AS maximum_nights
    FROM source
)

SELECT * FROM cleaned
WHERE listing_id IS NOT NULL
  AND calendar_date IS NOT NULL
```

### Layer 2: Intermediate (`models/intermediate/`)

Joins, enrichment, and business logic.

**Example: `int_airbnb__calendar_enriched.sql`**

```sql
WITH calendar AS (
    SELECT * FROM {{ ref('stg_airbnb__calendar') }}
),

listings AS (
    SELECT
        listing_id,
        neighbourhood,
        neighbourhood_group,
        room_type
    FROM {{ ref('int_airbnb__listing_enriched') }}
)

SELECT
    c.listing_id,
    c.calendar_date,
    c.is_available,
    c.price,
    c.minimum_nights,
    c.maximum_nights,
    l.neighbourhood,
    l.neighbourhood_group,
    l.room_type,
    -- Revenue proxy: unavailable nights treated as booked
    CASE 
        WHEN c.is_available = FALSE AND c.price > 0 
        THEN c.price 
        ELSE 0 
    END AS estimated_revenue,
    DATE_TRUNC('month', c.calendar_date) AS calendar_month
FROM calendar c
LEFT JOIN listings l ON c.listing_id = l.listing_id
```

### Layer 3: Marts (`models/marts/`)

Analytics-ready dimensions and facts.

**Example: `dim_listings.sql`**

```sql
SELECT
    listing_id,
    listing_name,
    host_id,
    neighbourhood,
    neighbourhood_group,
    latitude,
    longitude,
    room_type,
    price,
    minimum_nights,
    availability_365,
    last_review_date
FROM {{ ref('int_airbnb__listing_enriched') }}
```

**Example: Incremental Fact `fct_listing_calendar.sql`**

```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        merge_update_columns=['is_available', 'price', 'estimated_revenue']
    )
}}

SELECT
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
    calendar_month
FROM {{ ref('int_airbnb__calendar_enriched') }}

{% if is_incremental() %}
WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
{% endif %}
```

**Example: Aggregate `agg_listing_monthly_performance.sql`**

```sql
SELECT
    listing_id,
    calendar_month,
    neighbourhood,
    neighbourhood_group,
    room_type,
    COUNT(*) AS total_days,
    SUM(CASE WHEN is_available = FALSE THEN 1 ELSE 0 END) AS unavailable_days,
    SUM(estimated_revenue) AS total_estimated_revenue,
    AVG(price) AS avg_price,
    ROUND(SUM(CASE WHEN is_available = FALSE THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS occupancy_rate
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, calendar_month, neighbourhood, neighbourhood_group, room_type
```

## dbt Sources Configuration

**`models/staging/sources.yml`**

```yaml
version: 2

sources:
  - name: airbnb_raw
    database: AIRBNB_DB
    schema: RAW
    tables:
      - name: listings
        description: Raw listings data from Inside Airbnb
        columns:
          - name: id
            description: Unique listing identifier
            tests:
              - not_null
      - name: calendar
        description: Daily availability and pricing
        columns:
          - name: listing_id
            tests:
              - not_null
          - name: date
            tests:
              - not_null
      - name: reviews
        description: Guest reviews
      - name: neighbourhoods
        description: Geographic neighbourhood reference
```

## dbt Tests

### Generic Tests in Schema Files

**`models/marts/schema.yml`**

```yaml
version: 2

models:
  - name: dim_listings
    description: Listing dimension
    columns:
      - name: listing_id
        description: Primary key
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
    description: Daily calendar fact
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

**`tests/assert_no_duplicate_listing_dates.sql`**

```sql
-- Ensure no duplicate listing-date combinations in calendar fact
SELECT
    listing_id,
    calendar_date,
    COUNT(*) AS row_count
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, calendar_date
HAVING COUNT(*) > 1
```

## Running dbt Commands

```bash
# Validate connection and credentials
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select fct_listing_calendar+ --profiles-dir .

# Full refresh incremental model (rebuild from scratch)
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Run tests for specific model
dbt test --select dim_listings --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .

# Serve documentation site
dbt docs serve --profiles-dir . --port 8080
```

## Streamlit Dashboard

The dashboard queries Snowflake marts directly using `snowflake-connector-python`.

**Run the dashboard:**

```bash
streamlit run dashboard/streamlit_app.py
```

**Dashboard Code Pattern (`dashboard/streamlit_app.py`):**

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

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
        schema=creds['schema'],
        role=creds['role']
    )

conn = get_connection()

# Query aggregate mart
@st.cache_data
def load_neighbourhood_performance():
    query = """
    SELECT
        neighbourhood,
        calendar_month,
        SUM(total_estimated_revenue) AS revenue,
        AVG(occupancy_rate) AS avg_occupancy
    FROM agg_neighbourhood_monthly_performance
    GROUP BY neighbourhood, calendar_month
    ORDER BY revenue DESC
    """
    return pd.read_sql(query, conn)

df = load_neighbourhood_performance()

st.title("Airbnb Neighbourhood Performance")
st.dataframe(df)

# Filter by neighbourhood
neighbourhood = st.selectbox("Select Neighbourhood", df['NEIGHBOURHOOD'].unique())
filtered = df[df['NEIGHBOURHOOD'] == neighbourhood]
st.line_chart(filtered.set_index('CALENDAR_MONTH')['REVENUE'])
```

## Common Workflows

### Initial Setup and First Run

```bash
# 1. Set up credentials
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
# Edit both files with your Snowflake values

# 2. Install dependencies
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 3. Load raw data
python scripts/load_inside_airbnb_to_snowflake.py

# 4. Run dbt pipeline
dbt run --profiles-dir .
dbt test --profiles-dir .

# 5. Launch dashboard
streamlit run dashboard/streamlit_app.py
```

### Incremental Data Refresh

When new Inside Airbnb data is available:

```bash
# 1. Download new data files to data/raw/

# 2. Reload raw tables
python scripts/load_inside_airbnb_to_snowflake.py

# 3. Full refresh incremental fact and downstream models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 4. Validate data quality
dbt test --profiles-dir .
```

### Adding a New dbt Model

1. Create model file in appropriate layer:

```sql
-- models/intermediate/int_airbnb__new_model.sql
SELECT
    listing_id,
    -- transformation logic
FROM {{ ref('stg_airbnb__listings') }}
```

2. Add to schema file:

```yaml
# models/intermediate/schema.yml
models:
  - name: int_airbnb__new_model
    description: Purpose of this model
    columns:
      - name: listing_id
        tests:
          - not_null
```

3. Run and test:

```bash
dbt run --select int_airbnb__new_model --profiles-dir .
dbt test --select int_airbnb__new_model --profiles-dir .
```

## Troubleshooting

### dbt Connection Errors

```bash
# Verify profiles.yml location and structure
dbt debug --profiles-dir .

# Common issues:
# - Wrong account identifier (check format: account.region.cloud)
# - Role lacks permissions on AIRBNB_DB
# - Warehouse suspended or invalid name
```

### Snowflake Loader Fails

```python
# Check file paths exist
import os
print(os.path.exists('data/raw/listings.csv.gz'))  # Should be True

# Verify stage created
cursor.execute("SHOW STAGES IN SCHEMA RAW")
print(cursor.fetchall())

# Check file upload status
cursor.execute("LIST @INSIDE_AIRBNB_STAGE")
print(cursor.fetchall())
```

### Incremental Model Not Updating

```bash
# Force full rebuild
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Check incremental logic
dbt compile --select fct_listing_calendar --profiles-dir .
# Review compiled SQL in target/compiled/
```

### Test Failures

```bash
# Run specific test with verbose output
dbt test --select dim_listings --profiles-dir . --store-failures

# Query failed test results (stored in Snowflake)
# SELECT * FROM AIRBNB_DB.ANALYTICS.unique_dim_listings_listing_id
```

### Dashboard Credential Issues

```python
# Verify config file exists and is valid JSON
import json
with open('config/local_credentials.json') as f:
    print(json.load(f))

# Test connection separately
import snowflake.connector
conn = snowflake.connector.connect(**creds)
cursor = conn.cursor()
cursor.execute("SELECT CURRENT_DATABASE(), CURRENT_SCHEMA()")
print(cursor.fetchone())
```

## Project Configuration

### dbt Project File (`dbt_project.yml`)

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
      +schema: analytics
```

### Dependencies (`requirements.txt`)

```
dbt-snowflake==1.5.0
snowflake-connector-python==3.0.0
streamlit==1.25.0
pandas==2.0.3
```

## Key Patterns to Remember

1. **Raw schema preserves source**: All VARCHAR columns, no transformations
2. **Staging layer cleans**: Use `TRY_TO_NUMBER()`, `::DATE`, `REPLACE()` for safe casting
3. **Intermediate layer enriches**: Joins, derived columns, business logic
4. **Marts are simple**: Clean SELECT from intermediate, optimized for queries
5. **Incremental facts use unique_key**: Enables merge strategy for updates
6. **Tests belong in schema files**: Co-locate with model definitions
7. **Dashboard reads marts**: Never query staging or intermediate layers
8. **Credentials never committed**: Always use local-only config files

This skill enables AI agents to help developers build, extend, and troubleshoot the Snowflake dbt Airbnb Analytics project following analytics engineering best practices.
