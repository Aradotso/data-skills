---
name: snowflake-dbt-airbnb-analytics
description: Analytics engineering project loading Inside Airbnb data into Snowflake, transforming with dbt, and visualizing with Streamlit
triggers:
  - set up snowflake dbt airbnb project
  - load inside airbnb data to snowflake
  - run dbt models for airbnb analytics
  - create streamlit dashboard for airbnb data
  - configure snowflake dbt profiles
  - build airbnb data warehouse with dbt
  - troubleshoot dbt snowflake connection
  - incremental dbt models in snowflake
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to work with the Snowflake_DBT_Project, a complete analytics engineering portfolio project that demonstrates:

- Loading Inside Airbnb open dataset into Snowflake
- dbt transformations with staging, intermediate, and mart layers
- Incremental fact modeling with Snowflake merge strategy
- Data quality testing with dbt tests
- Streamlit dashboard for analytics visualization

## Project Overview

The project implements a full ELT pipeline:

1. **Extract & Load**: Python script uploads CSV/GZIP files to Snowflake internal stage
2. **Transform**: dbt models create staging, intermediate, and mart layers
3. **Visualize**: Streamlit dashboard queries analytics marts

**Key Technologies**: Snowflake, dbt-core, dbt-snowflake, Streamlit, Python

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

**Requirements typically include:**
- `dbt-snowflake>=1.5.0`
- `snowflake-connector-python>=3.0.0`
- `streamlit>=1.20.0`
- `pandas>=1.5.0`

## Configuration

### 1. Set Up Snowflake Credentials

Create `profiles.yml` from the example:

```bash
cp profiles.yml.example profiles.yml
```

Edit `profiles.yml` with your Snowflake credentials:

```yaml
snowflake_dbt_airbnb:
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT  # e.g., abc12345.us-east-1
      user: YOUR_USERNAME
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"  # Use env var for security
      role: YOUR_ROLE
      database: AIRBNB_DB
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
  target: dev
```

**Security Best Practice**: Use environment variables:

```bash
export SNOWFLAKE_PASSWORD='your_password_here'
```

### 2. Set Up Dashboard Credentials

Create `config/local_credentials.json` from the example:

```bash
cp config/local_credentials.example.json config/local_credentials.json
```

Edit `config/local_credentials.json`:

```json
{
  "account": "YOUR_ACCOUNT",
  "user": "YOUR_USERNAME",
  "password": "YOUR_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DB",
  "schema": "ANALYTICS",
  "role": "YOUR_ROLE"
}
```

**Note**: This file is `.gitignore`d for security.

### 3. Download Inside Airbnb Data

Download raw data files from [Inside Airbnb](https://insideairbnb.com/get-the-data/) (recommended: New York City):

```text
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

## Loading Data to Snowflake

The Python loader script handles the complete data ingestion process:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What this script does:**

1. Connects to Snowflake using credentials from `config/local_credentials.json`
2. Executes `setup/snowflake_setup.sql` to create database objects
3. Uploads local CSV/GZIP files to Snowflake internal stage `INSIDE_AIRBNB_STAGE`
4. Creates raw tables with `VARCHAR` columns to preserve all data
5. Copies staged data into `RAW` schema tables

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
    schema='RAW'
)

cursor = conn.cursor()

# Upload file to stage
cursor.execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")

# Copy into raw table
cursor.execute("""
    COPY INTO RAW.LISTINGS
    FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
    FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
    ON_ERROR = 'CONTINUE'
""")

conn.close()
```

## dbt Commands

### Core Workflow

```bash
# Test dbt connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select dim_listings+ --profiles-dir .

# Run specific model and upstream dependencies
dbt run --select +fct_listing_calendar --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Test specific model
dbt test --select dim_listings --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .

# Serve documentation
dbt docs serve --profiles-dir .
```

### Incremental Model Refresh

When loading new Inside Airbnb snapshot:

```bash
# Load new data
python scripts/load_inside_airbnb_to_snowflake.py

# Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Run tests
dbt test --profiles-dir .
```

## dbt Model Layers

### Staging Models (`models/staging/`)

Clean and standardize raw data:

**Example: `stg_airbnb__listings.sql`**

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
        TRY_CAST(last_review AS DATE) AS last_review_date,
        TRY_CAST(availability_365 AS INTEGER) AS availability_365,
        CURRENT_TIMESTAMP() AS dbt_loaded_at
    FROM source
)

SELECT * FROM cleaned
WHERE listing_id IS NOT NULL
```

### Intermediate Models (`models/intermediate/`)

Join and enrich data:

**Example: `int_airbnb__calendar_enriched.sql`**

```sql
WITH calendar AS (
    SELECT * FROM {{ ref('stg_airbnb__calendar') }}
),

listings AS (
    SELECT * FROM {{ ref('int_airbnb__listing_enriched') }}
),

enriched AS (
    SELECT
        c.calendar_date,
        c.listing_id,
        c.available,
        c.price AS calendar_price,
        l.listing_name,
        l.neighbourhood,
        l.room_type,
        l.host_id,
        CASE 
            WHEN c.available = 'f' AND c.price > 0 
            THEN c.price 
            ELSE 0 
        END AS estimated_revenue,
        CURRENT_TIMESTAMP() AS dbt_updated_at
    FROM calendar c
    INNER JOIN listings l ON c.listing_id = l.listing_id
)

SELECT * FROM enriched
```

### Mart Models (`models/marts/`)

Analytics-ready dimensions and facts:

**Example: `dim_listings.sql`**

```sql
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
    availability_365,
    dbt_loaded_at
FROM {{ ref('int_airbnb__listing_enriched') }}
```

**Example: Incremental Fact Table `fct_listing_calendar.sql`**

```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        merge_update_columns=['available', 'calendar_price', 'estimated_revenue', 'dbt_updated_at']
    )
}}

SELECT
    calendar_date,
    listing_id,
    available,
    calendar_price,
    estimated_revenue,
    dbt_updated_at
FROM {{ ref('int_airbnb__calendar_enriched') }}

{% if is_incremental() %}
    WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
{% endif %}
```

### Aggregate Models (`models/marts/`)

**Example: `agg_listing_monthly_performance.sql`**

```sql
WITH calendar_facts AS (
    SELECT * FROM {{ ref('fct_listing_calendar') }}
),

monthly_agg AS (
    SELECT
        listing_id,
        DATE_TRUNC('month', calendar_date) AS month,
        COUNT(*) AS total_days,
        SUM(CASE WHEN available = 'f' THEN 1 ELSE 0 END) AS unavailable_days,
        SUM(estimated_revenue) AS total_estimated_revenue,
        AVG(calendar_price) AS avg_price
    FROM calendar_facts
    GROUP BY listing_id, DATE_TRUNC('month', calendar_date)
)

SELECT * FROM monthly_agg
```

## dbt Testing

### Schema Tests

Define tests in `models/schema.yml`:

```yaml
version: 2

models:
  - name: dim_listings
    description: Dimension table for Airbnb listings
    columns:
      - name: listing_id
        description: Primary key for listings
        tests:
          - unique
          - not_null
      - name: room_type
        description: Type of room
        tests:
          - accepted_values:
              values: ['Entire home/apt', 'Private room', 'Shared room', 'Hotel room']
      - name: price
        description: Listing price per night
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"

  - name: fct_listing_calendar
    description: Daily calendar facts for listings
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

Create custom SQL tests in `tests/`:

**Example: `tests/duplicate_listing_dates.sql`**

```sql
-- Check for duplicate listing-date combinations
SELECT
    listing_id,
    calendar_date,
    COUNT(*) AS duplicate_count
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, calendar_date
HAVING COUNT(*) > 1
```

**Example: `tests/negative_revenue.sql`**

```sql
-- Check for negative estimated revenue
SELECT *
FROM {{ ref('fct_listing_calendar') }}
WHERE estimated_revenue < 0
```

## Streamlit Dashboard

Run the dashboard:

```bash
streamlit run dashboard/streamlit_app.py
```

**Example dashboard code pattern:**

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

@st.cache_data
def run_query(query):
    conn = get_connection()
    df = pd.read_sql(query, conn)
    return df

st.title('Airbnb Analytics Dashboard')

# Neighbourhood performance
st.header('Top Neighbourhoods by Revenue')
query = """
SELECT 
    neighbourhood,
    SUM(total_estimated_revenue) AS total_revenue,
    AVG(avg_price) AS avg_price
FROM agg_neighbourhood_monthly_performance
GROUP BY neighbourhood
ORDER BY total_revenue DESC
LIMIT 10
"""
df = run_query(query)
st.bar_chart(df.set_index('neighbourhood')['total_revenue'])

# Host performance
st.header('Top Hosts by Listing Count')
query = """
SELECT 
    host_id,
    host_name,
    COUNT(*) AS listing_count
FROM dim_listings
GROUP BY host_id, host_name
ORDER BY listing_count DESC
LIMIT 10
"""
df = run_query(query)
st.dataframe(df)
```

## Common Patterns

### Adding a New Source

1. Add to `models/staging/_sources.yml`:

```yaml
sources:
  - name: airbnb_raw
    database: AIRBNB_DB
    schema: RAW
    tables:
      - name: new_table
        description: Description of new table
```

2. Create staging model `models/staging/stg_airbnb__new_table.sql`:

```sql
WITH source AS (
    SELECT * FROM {{ source('airbnb_raw', 'new_table') }}
),
cleaned AS (
    SELECT
        -- cleaned columns
    FROM source
)
SELECT * FROM cleaned
```

### Creating a Metric

Use dbt metrics in `models/metrics.yml`:

```yaml
version: 2

metrics:
  - name: total_estimated_revenue
    label: Total Estimated Revenue
    model: ref('fct_listing_calendar')
    description: Sum of estimated revenue from unavailable nights
    calculation_method: sum
    expression: estimated_revenue
    timestamp: calendar_date
    time_grains: [day, week, month, year]
    dimensions:
      - neighbourhood
      - room_type
```

### Using dbt Macros

Create reusable SQL in `macros/`:

**Example: `macros/get_revenue.sql`**

```sql
{% macro get_revenue(available_col, price_col) %}
    CASE 
        WHEN {{ available_col }} = 'f' AND {{ price_col }} > 0 
        THEN {{ price_col }} 
        ELSE 0 
    END
{% endmacro %}
```

Use in models:

```sql
SELECT
    listing_id,
    calendar_date,
    {{ get_revenue('available', 'price') }} AS estimated_revenue
FROM {{ ref('stg_airbnb__calendar') }}
```

## Troubleshooting

### dbt Connection Issues

**Error: "Could not connect to Snowflake"**

```bash
# Test connection
dbt debug --profiles-dir .

# Check profile structure
cat profiles.yml

# Verify environment variables
echo $SNOWFLAKE_PASSWORD

# Test Snowflake connection directly
python -c "import snowflake.connector; print('OK')"
```

**Fix**: Ensure `profiles.yml` is in project root and contains correct account identifier format.

### Incremental Model Issues

**Error: "Incremental model missing unique_key"**

```sql
-- Add config block to model
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date']
    )
}}
```

**Full refresh when schema changes:**

```bash
dbt run --full-refresh --select fct_listing_calendar
```

### Data Loading Issues

**Error: "File format error during COPY INTO"**

Check file encoding and delimiters:

```sql
-- Test file format manually in Snowflake
SELECT $1, $2, $3 
FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
(FILE_FORMAT => 'CSV') 
LIMIT 10;
```

**Fix**: Adjust FILE_FORMAT parameters in loader script:

```python
cursor.execute("""
    COPY INTO RAW.LISTINGS
    FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
    FILE_FORMAT = (
        TYPE = 'CSV' 
        FIELD_OPTIONALLY_ENCLOSED_BY = '"' 
        SKIP_HEADER = 1
        ENCODING = 'UTF8'
        ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE
    )
    ON_ERROR = 'CONTINUE'
""")
```

### Streamlit Connection Issues

**Error: "Failed to connect to Snowflake from Streamlit"**

```python
# Add error handling
try:
    conn = snowflake.connector.connect(**creds)
    st.success("Connected to Snowflake")
except Exception as e:
    st.error(f"Connection failed: {e}")
```

**Fix**: Verify `config/local_credentials.json` exists and contains valid credentials.

### Test Failures

**Error: "Test failed: duplicate_listing_dates"**

Investigate duplicate data:

```sql
-- Find duplicates
SELECT listing_id, calendar_date, COUNT(*)
FROM fct_listing_calendar
GROUP BY listing_id, calendar_date
HAVING COUNT(*) > 1;

-- Remove duplicates (if needed)
CREATE OR REPLACE TABLE fct_listing_calendar AS
SELECT DISTINCT *
FROM fct_listing_calendar;
```

## Best Practices

1. **Never commit credentials**: Use `.gitignore` for `profiles.yml` and `config/local_credentials.json`
2. **Use environment variables**: Reference `{{ env_var('SNOWFLAKE_PASSWORD') }}` in `profiles.yml`
3. **Test incrementally**: Run `dbt test` after each model change
4. **Document models**: Add descriptions in `schema.yml` files
5. **Version control dbt**: Commit all dbt models, tests, and docs
6. **Use full-refresh sparingly**: Only when schema changes require it
7. **Monitor warehouse usage**: Snowflake costs scale with compute time

## Project Structure Reference

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

This skill provides comprehensive coverage for AI agents to assist developers in setting up, configuring, running, and troubleshooting the Snowflake dbt Airbnb analytics project.
