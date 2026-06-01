---
name: snowflake-dbt-airbnb-analytics
description: End-to-end analytics engineering with Snowflake, dbt, and Streamlit using Inside Airbnb data
triggers:
  - set up dbt with Snowflake for Airbnb analytics
  - load Inside Airbnb data into Snowflake
  - build dbt staging and mart models for Airbnb
  - create incremental fact tables in dbt
  - run dbt tests for data quality
  - build a Streamlit dashboard for Snowflake data
  - configure dbt profiles for Snowflake connection
  - transform Airbnb calendar data with dbt
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build a complete analytics engineering pipeline using Snowflake, dbt, and Streamlit with the Inside Airbnb dataset. The project demonstrates raw data ingestion, multi-layer dbt transformations, incremental modeling, data quality testing, and dashboard visualization.

## What This Project Does

The Snowflake_DBT_Project provides a full analytics engineering workflow:

1. **Data Loading**: Python script uploads Inside Airbnb CSV/GZIP files to Snowflake internal stages
2. **Raw Layer**: Creates text-preserving raw tables in Snowflake
3. **dbt Transformations**: Builds staging, intermediate, and mart layers with proper typing and business logic
4. **Incremental Modeling**: Implements incremental fact tables with Snowflake merge strategy
5. **Data Quality**: Generic and singular dbt tests validate data integrity
6. **Dashboard**: Streamlit app queries marts for neighbourhood, host, and listing analytics

## Installation

### Prerequisites

- Python 3.8+
- Snowflake account with warehouse, database, and schema permissions
- Inside Airbnb raw data files (listings.csv.gz, calendar.csv.gz, reviews.csv.gz, neighbourhoods.csv)

### Setup Steps

```bash
# Clone and navigate to project
cd /path/to/Snowflake_DBT_Project

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configuration

**Create local credentials files (never commit these):**

```bash
# Copy example files
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

**Edit `profiles.yml`:**

```yaml
airbnb_analytics:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT.region
      user: YOUR_USERNAME
      password: YOUR_PASSWORD
      role: YOUR_ROLE
      database: AIRBNB_ANALYTICS
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

**Edit `config/local_credentials.json`:**

```json
{
  "account": "YOUR_ACCOUNT.region",
  "user": "YOUR_USERNAME",
  "password": "YOUR_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_ANALYTICS",
  "schema": "ANALYTICS",
  "role": "YOUR_ROLE"
}
```

**For environment variable approach (recommended for production):**

```bash
export SNOWFLAKE_ACCOUNT="your_account.region"
export SNOWFLAKE_USER="your_username"
export SNOWFLAKE_PASSWORD="your_password"
export SNOWFLAKE_WAREHOUSE="COMPUTE_WH"
export SNOWFLAKE_DATABASE="AIRBNB_ANALYTICS"
export SNOWFLAKE_ROLE="YOUR_ROLE"
```

Then modify `profiles.yml` to use env vars:

```yaml
airbnb_analytics:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: "{{ env_var('SNOWFLAKE_ROLE') }}"
      database: "{{ env_var('SNOWFLAKE_DATABASE') }}"
      warehouse: "{{ env_var('SNOWFLAKE_WAREHOUSE') }}"
      schema: ANALYTICS
      threads: 4
```

## Loading Raw Data

Place Inside Airbnb files in `data/raw/`:

```text
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

Run the loader script:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What the loader does:**

1. Reads credentials from `config/local_credentials.json`
2. Executes `setup/snowflake_setup.sql` to create database, schemas, and stage
3. Uploads local files to Snowflake internal stage `INSIDE_AIRBNB_STAGE`
4. Creates raw tables with inferred schema from CSV headers
5. Copies data from stage into `RAW` schema tables

**Example loader code pattern:**

```python
import snowflake.connector
import json
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
    role=creds['role']
)

cursor = conn.cursor()

# Upload to stage
cursor.execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE")

# Create raw table
cursor.execute("""
CREATE OR REPLACE TABLE RAW.LISTINGS (
    id VARCHAR,
    name VARCHAR,
    host_id VARCHAR,
    -- ... all columns as VARCHAR for raw layer
)
""")

# Copy into table
cursor.execute("""
COPY INTO RAW.LISTINGS
FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
FILE_FORMAT = (TYPE = CSV FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
ON_ERROR = 'CONTINUE'
""")
```

## dbt Commands

### Basic Workflow

```bash
# Test connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select dim_listings+ --profiles-dir .

# Run specific model and upstream dependencies
dbt run --select +fct_reviews --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Test specific model
dbt test --select dim_hosts --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

### Incremental Model Refresh

When new Inside Airbnb data is loaded:

```bash
# Reload raw data
python scripts/load_inside_airbnb_to_snowflake.py

# Full refresh incremental models and downstream
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Run tests
dbt test --profiles-dir .
```

### Model Selection Patterns

```bash
# Run only staging models
dbt run --select staging.* --profiles-dir .

# Run intermediate and marts
dbt run --select intermediate.*,marts.* --profiles-dir .

# Run models with specific tag
dbt run --select tag:daily --profiles-dir .

# Exclude specific models
dbt run --exclude fct_listing_calendar --profiles-dir .
```

## dbt Model Architecture

### Project Structure

```text
models/
├── staging/
│   ├── _staging.yml
│   ├── stg_airbnb__listings.sql
│   ├── stg_airbnb__calendar.sql
│   ├── stg_airbnb__reviews.sql
│   └── stg_airbnb__neighbourhoods.sql
├── intermediate/
│   ├── _intermediate.yml
│   ├── int_airbnb__listing_enriched.sql
│   ├── int_airbnb__calendar_enriched.sql
│   └── int_airbnb__reviews_enriched.sql
└── marts/
    ├── _marts.yml
    ├── dim_listings.sql
    ├── dim_hosts.sql
    ├── fct_listing_calendar.sql
    ├── fct_reviews.sql
    ├── agg_listing_monthly_performance.sql
    └── agg_neighbourhood_monthly_performance.sql
```

### Staging Layer Example

**models/staging/stg_airbnb__listings.sql:**

```sql
WITH source AS (
    SELECT * FROM {{ source('airbnb_raw', 'listings') }}
),

cleaned AS (
    SELECT
        TRY_CAST(id AS INTEGER) AS listing_id,
        TRIM(name) AS listing_name,
        TRY_CAST(host_id AS INTEGER) AS host_id,
        TRIM(host_name) AS host_name,
        TRIM(neighbourhood_cleansed) AS neighbourhood,
        TRIM(room_type) AS room_type,
        TRY_CAST(price AS DECIMAL(10,2)) AS price,
        TRY_CAST(minimum_nights AS INTEGER) AS minimum_nights,
        TRY_CAST(number_of_reviews AS INTEGER) AS number_of_reviews,
        TRY_CAST(reviews_per_month AS DECIMAL(10,2)) AS reviews_per_month,
        TRY_CAST(availability_365 AS INTEGER) AS availability_365,
        CURRENT_TIMESTAMP() AS loaded_at
    FROM source
    WHERE id IS NOT NULL
)

SELECT * FROM cleaned
```

### Intermediate Layer Example

**models/intermediate/int_airbnb__listing_enriched.sql:**

```sql
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
        l.room_type,
        l.price,
        l.minimum_nights,
        l.number_of_reviews,
        l.reviews_per_month,
        l.availability_365,
        CASE
            WHEN l.availability_365 = 0 THEN 'Never Available'
            WHEN l.availability_365 < 90 THEN 'Low Availability'
            WHEN l.availability_365 < 180 THEN 'Medium Availability'
            ELSE 'High Availability'
        END AS availability_category,
        l.loaded_at
    FROM listings l
    LEFT JOIN neighbourhoods n
        ON l.neighbourhood = n.neighbourhood
)

SELECT * FROM enriched
```

### Incremental Fact Model Example

**models/marts/fct_listing_calendar.sql:**

```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        merge_update_columns=['available', 'price', 'adjusted_price', 'minimum_nights', 'maximum_nights']
    )
}}

WITH calendar AS (
    SELECT * FROM {{ ref('int_airbnb__calendar_enriched') }}
    {% if is_incremental() %}
    WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
    {% endif %}
),

final AS (
    SELECT
        listing_id,
        calendar_date,
        available,
        price,
        adjusted_price,
        minimum_nights,
        maximum_nights,
        CASE
            WHEN available = FALSE THEN adjusted_price
            ELSE 0
        END AS estimated_revenue,
        CURRENT_TIMESTAMP() AS dbt_updated_at
    FROM calendar
)

SELECT * FROM final
```

### Aggregate Mart Example

**models/marts/agg_listing_monthly_performance.sql:**

```sql
WITH calendar_facts AS (
    SELECT * FROM {{ ref('fct_listing_calendar') }}
),

listings AS (
    SELECT * FROM {{ ref('dim_listings') }}
),

monthly_agg AS (
    SELECT
        cf.listing_id,
        DATE_TRUNC('month', cf.calendar_date) AS month,
        COUNT(*) AS total_days,
        SUM(CASE WHEN cf.available = FALSE THEN 1 ELSE 0 END) AS unavailable_days,
        AVG(cf.price) AS avg_price,
        SUM(cf.estimated_revenue) AS total_estimated_revenue
    FROM calendar_facts cf
    GROUP BY cf.listing_id, DATE_TRUNC('month', cf.calendar_date)
),

final AS (
    SELECT
        ma.listing_id,
        l.listing_name,
        l.neighbourhood,
        l.room_type,
        ma.month,
        ma.total_days,
        ma.unavailable_days,
        ma.total_days - ma.unavailable_days AS available_days,
        ROUND(ma.unavailable_days / ma.total_days * 100, 2) AS unavailable_rate,
        ma.avg_price,
        ma.total_estimated_revenue
    FROM monthly_agg ma
    INNER JOIN listings l ON ma.listing_id = l.listing_id
)

SELECT * FROM final
```

## dbt Testing

### Source Tests

**models/staging/_staging.yml:**

```yaml
version: 2

sources:
  - name: airbnb_raw
    database: AIRBNB_ANALYTICS
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
        columns:
          - name: id
            tests:
              - not_null
              - unique
```

### Model Tests

**models/marts/_marts.yml:**

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
    description: Incremental fact table for listing calendar availability and pricing
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
    tests:
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns:
            - listing_id
            - calendar_date
```

### Singular Tests

**tests/assert_no_negative_revenue.sql:**

```sql
SELECT
    listing_id,
    calendar_date,
    estimated_revenue
FROM {{ ref('fct_listing_calendar') }}
WHERE estimated_revenue < 0
```

### Running Tests

```bash
# Run all tests
dbt test --profiles-dir .

# Run tests for specific model
dbt test --select dim_listings --profiles-dir .

# Run specific test type
dbt test --select test_type:generic --profiles-dir .
dbt test --select test_type:singular --profiles-dir .

# Run tests with verbose output
dbt test --profiles-dir . --store-failures
```

## Streamlit Dashboard

### Running the Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

### Dashboard Connection Pattern

**dashboard/streamlit_app.py:**

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json
from pathlib import Path

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
        schema=creds['schema'],
        role=creds['role']
    )

# Query helper
@st.cache_data(ttl=600)
def run_query(query):
    conn = get_snowflake_connection()
    return pd.read_sql(query, conn)

# Dashboard layout
st.title("Airbnb Analytics Dashboard")

# Neighbourhood performance
st.header("Top Neighbourhoods by Revenue")
neighbourhood_query = """
SELECT
    neighbourhood,
    SUM(total_estimated_revenue) AS total_revenue,
    AVG(unavailable_rate) AS avg_occupancy_rate
FROM ANALYTICS.agg_neighbourhood_monthly_performance
GROUP BY neighbourhood
ORDER BY total_revenue DESC
LIMIT 10
"""
df_neighbourhoods = run_query(neighbourhood_query)
st.bar_chart(df_neighbourhoods.set_index('neighbourhood')['total_revenue'])

# Host analysis
st.header("Top Hosts by Listing Count")
host_query = """
SELECT
    host_id,
    host_name,
    listing_count,
    avg_price,
    total_reviews
FROM ANALYTICS.dim_hosts
ORDER BY listing_count DESC
LIMIT 10
"""
df_hosts = run_query(host_query)
st.dataframe(df_hosts)

# Room type distribution
st.header("Room Type Pricing")
room_query = """
SELECT
    room_type,
    COUNT(*) AS listing_count,
    AVG(price) AS avg_price
FROM ANALYTICS.dim_listings
GROUP BY room_type
ORDER BY avg_price DESC
"""
df_rooms = run_query(room_query)
st.bar_chart(df_rooms.set_index('room_type'))
```

## Common Patterns and Workflows

### Full Pipeline Execution

```bash
# 1. Load new data
python scripts/load_inside_airbnb_to_snowflake.py

# 2. Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 3. Run all other models
dbt run --exclude fct_listing_calendar --profiles-dir .

# 4. Run tests
dbt test --profiles-dir .

# 5. Generate docs
dbt docs generate --profiles-dir .

# 6. Launch dashboard
streamlit run dashboard/streamlit_app.py
```

### Daily Incremental Update

```bash
# Assume raw data is refreshed daily via external process
# Run incremental models only
dbt run --select fct_listing_calendar+ --profiles-dir .

# Run tests on updated models
dbt test --select fct_listing_calendar+ --profiles-dir .
```

### Adding a New Mart Model

1. Create SQL file in `models/marts/`:

```sql
-- models/marts/dim_property_types.sql
WITH listings AS (
    SELECT * FROM {{ ref('dim_listings') }}
),

property_agg AS (
    SELECT
        room_type,
        COUNT(*) AS listing_count,
        AVG(price) AS avg_price,
        AVG(number_of_reviews) AS avg_reviews,
        AVG(availability_365) AS avg_availability
    FROM listings
    GROUP BY room_type
)

SELECT * FROM property_agg
```

2. Add documentation in `models/marts/_marts.yml`:

```yaml
models:
  - name: dim_property_types
    description: Aggregated metrics by property type
    columns:
      - name: room_type
        description: Type of room (Entire home/apt, Private room, etc.)
        tests:
          - not_null
          - unique
```

3. Run the model:

```bash
dbt run --select dim_property_types --profiles-dir .
```

## Troubleshooting

### Connection Issues

**Problem:** `dbt debug` fails with authentication error

**Solution:**

```bash
# Verify credentials in profiles.yml
cat profiles.yml

# Test Snowflake connection with Python
python -c "
import snowflake.connector
import json
with open('config/local_credentials.json') as f:
    creds = json.load(f)
conn = snowflake.connector.connect(**creds)
print('Connection successful')
"
```

### Incremental Model Not Updating

**Problem:** `fct_listing_calendar` not picking up new dates

**Solution:**

```bash
# Force full refresh
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Check max date in target table
dbt run-operation query_table --args '{sql: "SELECT MAX(calendar_date) FROM ANALYTICS.fct_listing_calendar"}'
```

### Test Failures

**Problem:** Relationship test fails between fact and dimension

**Solution:**

```sql
-- Check for orphaned records
SELECT DISTINCT fc.listing_id
FROM ANALYTICS.fct_listing_calendar fc
LEFT JOIN ANALYTICS.dim_listings dl ON fc.listing_id = dl.listing_id
WHERE dl.listing_id IS NULL
LIMIT 10;

-- Rebuild staging and intermediate models
dbt run --select +dim_listings --profiles-dir .
```

### Raw Data Load Errors

**Problem:** `COPY INTO` fails with format errors

**Solution:**

```sql
-- Check stage files
LIST @INSIDE_AIRBNB_STAGE;

-- Preview data with error handling
SELECT $1, $2, $3
FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
(FILE_FORMAT => 'CSV', ON_ERROR => 'CONTINUE')
LIMIT 10;

-- Use more permissive COPY options
COPY INTO RAW.LISTINGS
FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
FILE_FORMAT = (
    TYPE = CSV
    FIELD_OPTIONALLY_ENCLOSED_BY = '"'
    SKIP_HEADER = 1
    ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE
)
ON_ERROR = 'CONTINUE';
```

### Streamlit Dashboard Errors

**Problem:** Dashboard can't connect to Snowflake

**Solution:**

```python
# Add debug connection test
import streamlit as st
import snowflake.connector
import json

try:
    with open('config/local_credentials.json') as f:
        creds = json.load(f)
    conn = snowflake.connector.connect(**creds)
    st.success("Connected to Snowflake!")
    cursor = conn.cursor()
    cursor.execute("SELECT CURRENT_USER(), CURRENT_ROLE(), CURRENT_DATABASE()")
    result = cursor.fetchone()
    st.write(f"User: {result[0]}, Role: {result[1]}, Database: {result[2]}")
except Exception as e:
    st.error(f"Connection failed: {str(e)}")
```

## Advanced Configuration

### Using dbt Variables

**dbt_project.yml:**

```yaml
vars:
  start_date: '2024-01-01'
  min_price_threshold: 10
```

**In models:**

```sql
WHERE calendar_date >= '{{ var("start_date") }}'
  AND price >= {{ var("min_price_threshold") }}
```

**Override at runtime:**

```bash
dbt run --vars '{"start_date": "2025-01-01"}' --profiles-dir .
```

### Custom Schema Names

**dbt_project.yml:**

```yaml
models:
  airbnb_analytics:
    staging:
      +schema: staging
    intermediate:
      +schema: intermediate
    marts:
      +schema: marts
```

### Post-Hooks

**models/marts/_marts.yml:**

```yaml
models:
  - name: dim_listings
    config:
      post-hook:
        - "GRANT SELECT ON {{ this }} TO ROLE ANALYST_ROLE"
```

This skill provides complete guidance for building, testing, and deploying an analytics engineering pipeline with Snowflake, dbt, and Streamlit using real-world open data.
