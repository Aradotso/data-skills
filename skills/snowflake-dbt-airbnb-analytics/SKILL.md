---
name: snowflake-dbt-airbnb-analytics
description: Analytics engineering project with Snowflake, dbt, and Inside Airbnb data including staging, marts, tests, and Streamlit dashboard
triggers:
  - how do I set up the Airbnb dbt project with Snowflake
  - help me load Inside Airbnb data into Snowflake
  - show me how to run the dbt models for this project
  - configure Snowflake credentials for this dbt project
  - how do I run the Streamlit dashboard for Airbnb analytics
  - troubleshoot dbt incremental models in this project
  - build the Airbnb analytics marts with dbt
  - run dbt tests for the Inside Airbnb dataset
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the **Snowflake_DBT_Project**, a complete analytics engineering project that loads Inside Airbnb open data into Snowflake, transforms it through dbt staging/intermediate/mart layers, validates data quality with tests, and powers a Streamlit dashboard.

## What This Project Does

- **Ingests** Inside Airbnb CSV/GZIP files (listings, calendar, reviews, neighbourhoods) into Snowflake raw tables
- **Transforms** raw data through dbt staging → intermediate → marts (dimensions, facts, aggregates)
- **Implements** incremental fact modeling with Snowflake merge strategy
- **Validates** data quality with dbt generic and singular tests
- **Documents** models with dbt docs
- **Visualizes** analytics through a Streamlit dashboard connected to marts

The project uses a proxy revenue model based on calendar unavailability and price (Inside Airbnb does not provide actual booking transactions).

## Installation

### 1. Clone and Set Up Python Environment

```bash
git clone https://github.com/analyticsdurgesh/Snowflake_DBT_Project.git
cd Snowflake_DBT_Project
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Configure Snowflake Credentials

Create local credential files (not committed to git):

```bash
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

**Edit `profiles.yml`:**

```yaml
dbt_airbnb:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT  # e.g., abc12345.us-east-1
      user: YOUR_USERNAME
      password: YOUR_PASSWORD
      role: YOUR_ROLE
      database: AIRBNB_DB
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

**Edit `config/local_credentials.json`:**

```json
{
  "account": "YOUR_ACCOUNT.us-east-1",
  "user": "YOUR_USERNAME",
  "password": "YOUR_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DB",
  "schema": "ANALYTICS",
  "role": "YOUR_ROLE"
}
```

### 3. Download Inside Airbnb Data

Download NYC dataset files from [Inside Airbnb](https://insideairbnb.com/get-the-data/) and place them in:

```
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

## Key Commands

### Load Raw Data into Snowflake

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

This script:
1. Connects to Snowflake using `config/local_credentials.json`
2. Executes `setup/snowflake_setup.sql` to create database, schemas, and stage
3. Uploads local files to Snowflake internal stage `INSIDE_AIRBNB_STAGE`
4. Creates raw tables from CSV headers
5. Copies data from stage to `RAW` schema tables

### Run dbt Models

```bash
# Test connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select dim_listings+ --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

### Run Streamlit Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

The dashboard reads credentials from `config/local_credentials.json` and connects to the mart models.

## Project Structure

### dbt Model Layers

```
models/
├── staging/
│   ├── stg_airbnb__calendar.sql      # Clean calendar with DATE cast
│   ├── stg_airbnb__listings.sql      # Clean listings with type casting
│   ├── stg_airbnb__neighbourhoods.sql
│   └── stg_airbnb__reviews.sql
├── intermediate/
│   ├── int_airbnb__calendar_enriched.sql   # Calendar + listing join
│   ├── int_airbnb__listing_enriched.sql    # Listings + neighbourhoods
│   └── int_airbnb__reviews_enriched.sql
└── marts/
    ├── dim_hosts.sql                   # Host dimension
    ├── dim_listings.sql                # Listing dimension
    ├── fct_listing_calendar.sql        # Incremental fact table
    ├── fct_reviews.sql
    ├── agg_listing_monthly_performance.sql
    └── agg_neighbourhood_monthly_performance.sql
```

### Data Flow

```
RAW tables (Snowflake)
  ↓
Staging views (clean + cast)
  ↓
Intermediate views (joins + enrichment)
  ↓
Marts (dimensions + facts + aggregates)
  ↓
Streamlit Dashboard
```

## Code Examples

### Staging Model: stg_airbnb__listings.sql

```sql
WITH source AS (
    SELECT * FROM {{ source('airbnb_raw', 'LISTINGS') }}
)

SELECT
    id::INTEGER AS listing_id,
    name AS listing_name,
    host_id::INTEGER AS host_id,
    host_name,
    neighbourhood_cleansed AS neighbourhood,
    latitude::FLOAT AS latitude,
    longitude::FLOAT AS longitude,
    room_type,
    price::FLOAT AS price,
    minimum_nights::INTEGER AS minimum_nights,
    number_of_reviews::INTEGER AS number_of_reviews,
    last_review::DATE AS last_review_date,
    reviews_per_month::FLOAT AS reviews_per_month,
    calculated_host_listings_count::INTEGER AS host_listing_count,
    availability_365::INTEGER AS availability_365
FROM source
WHERE id IS NOT NULL
```

### Intermediate Model: int_airbnb__listing_enriched.sql

```sql
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
    l.latitude,
    l.longitude,
    l.room_type,
    l.price,
    l.minimum_nights,
    l.number_of_reviews,
    l.last_review_date,
    l.reviews_per_month,
    l.host_listing_count,
    l.availability_365
FROM listings l
LEFT JOIN neighbourhoods n
    ON l.neighbourhood = n.neighbourhood
```

### Incremental Fact Model: fct_listing_calendar.sql

```sql
{{
    config(
        materialized='incremental',
        unique_key='calendar_key'
    )
}}

WITH calendar_enriched AS (
    SELECT * FROM {{ ref('int_airbnb__calendar_enriched') }}
)

SELECT
    {{ dbt_utils.generate_surrogate_key(['listing_id', 'calendar_date']) }} AS calendar_key,
    listing_id,
    calendar_date,
    available,
    price,
    minimum_nights,
    maximum_nights,
    listing_name,
    neighbourhood,
    neighbourhood_group,
    room_type,
    CASE 
        WHEN available = 'f' THEN price 
        ELSE 0 
    END AS estimated_revenue
FROM calendar_enriched

{% if is_incremental() %}
    WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
{% endif %}
```

### Aggregate Mart: agg_neighbourhood_monthly_performance.sql

```sql
WITH listing_monthly AS (
    SELECT * FROM {{ ref('agg_listing_monthly_performance') }}
)

SELECT
    year_month,
    neighbourhood,
    neighbourhood_group,
    COUNT(DISTINCT listing_id) AS total_listings,
    SUM(total_days) AS total_calendar_days,
    SUM(available_days) AS total_available_days,
    SUM(unavailable_days) AS total_unavailable_days,
    AVG(availability_rate) AS avg_availability_rate,
    AVG(avg_price) AS avg_listing_price,
    SUM(estimated_revenue) AS total_estimated_revenue
FROM listing_monthly
GROUP BY 
    year_month,
    neighbourhood,
    neighbourhood_group
```

## Configuration Patterns

### dbt_project.yml Key Settings

```yaml
name: 'dbt_airbnb'
version: '1.0.0'
config-version: 2

profile: 'dbt_airbnb'

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
  - "logs"

models:
  dbt_airbnb:
    staging:
      +materialized: view
      +schema: staging
    intermediate:
      +materialized: view
      +schema: intermediate
    marts:
      +materialized: table
      +schema: marts
```

### Sources Configuration: models/staging/sources.yml

```yaml
version: 2

sources:
  - name: airbnb_raw
    database: AIRBNB_DB
    schema: RAW
    tables:
      - name: LISTINGS
        columns:
          - name: id
            tests:
              - not_null
              - unique
      - name: CALENDAR
        columns:
          - name: listing_id
            tests:
              - not_null
      - name: REVIEWS
        columns:
          - name: id
            tests:
              - not_null
              - unique
      - name: NEIGHBOURHOODS
```

### Python Loader Script Pattern

```python
import snowflake.connector
import json
import os
from pathlib import Path

# Load credentials
with open('config/local_credentials.json', 'r') as f:
    creds = json.load(f)

conn = snowflake.connector.connect(
    user=creds['user'],
    password=creds['password'],
    account=creds['account'],
    warehouse=creds['warehouse'],
    database=creds['database'],
    schema=creds['schema'],
    role=creds['role']
)

cursor = conn.cursor()

# Execute setup SQL
with open('setup/snowflake_setup.sql', 'r') as f:
    setup_sql = f.read()
    for statement in setup_sql.split(';'):
        if statement.strip():
            cursor.execute(statement)

# Upload file to stage
cursor.execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE")

# Copy into table
cursor.execute("""
COPY INTO RAW.LISTINGS
FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
ON_ERROR = 'CONTINUE'
""")

cursor.close()
conn.close()
```

### Streamlit Dashboard Connection

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

@st.cache_resource
def get_connection():
    with open('config/local_credentials.json', 'r') as f:
        creds = json.load(f)
    
    return snowflake.connector.connect(
        user=creds['user'],
        password=creds['password'],
        account=creds['account'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema='marts',
        role=creds['role']
    )

conn = get_connection()

@st.cache_data
def load_neighbourhood_performance():
    query = """
    SELECT * 
    FROM agg_neighbourhood_monthly_performance
    ORDER BY year_month DESC, total_estimated_revenue DESC
    """
    return pd.read_sql(query, conn)

df = load_neighbourhood_performance()
st.dataframe(df)
```

## Common Workflows

### Initial Setup Workflow

```bash
# 1. Set up environment
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 2. Configure credentials
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
# Edit both files with your Snowflake credentials

# 3. Download Inside Airbnb data to data/raw/

# 4. Load raw data
python scripts/load_inside_airbnb_to_snowflake.py

# 5. Run dbt
dbt debug --profiles-dir .
dbt run --profiles-dir .
dbt test --profiles-dir .

# 6. Launch dashboard
streamlit run dashboard/streamlit_app.py
```

### Incremental Data Refresh Workflow

```bash
# 1. Download new Inside Airbnb snapshot to data/raw/

# 2. Reload raw tables
python scripts/load_inside_airbnb_to_snowflake.py

# 3. Full refresh incremental model and downstream
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 4. Run tests
dbt test --profiles-dir .

# 5. Regenerate docs
dbt docs generate --profiles-dir .
```

### Development Workflow

```bash
# Run specific model
dbt run --select stg_airbnb__listings --profiles-dir .

# Run model and all downstream dependencies
dbt run --select stg_airbnb__listings+ --profiles-dir .

# Run only modified models and downstream
dbt run --select state:modified+ --profiles-dir .

# Test specific model
dbt test --select dim_listings --profiles-dir .

# Compile without running
dbt compile --select fct_listing_calendar --profiles-dir .
```

## Testing Patterns

### Generic Tests in Schema YAML

```yaml
models:
  - name: dim_listings
    columns:
      - name: listing_id
        tests:
          - not_null
          - unique
      - name: price
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
      - name: room_type
        tests:
          - accepted_values:
              values: ['Entire home/apt', 'Private room', 'Shared room', 'Hotel room']
```

### Singular Test Example: tests/assert_no_duplicate_calendar_dates.sql

```sql
SELECT
    listing_id,
    calendar_date,
    COUNT(*) AS occurrences
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, calendar_date
HAVING COUNT(*) > 1
```

## Troubleshooting

### "Credentials not found" Error

**Problem:** dbt cannot find `profiles.yml` or Snowflake connection fails.

**Solution:**
1. Ensure `profiles.yml` exists in project root (not committed to git)
2. Run dbt with `--profiles-dir .` flag to use local profiles
3. Verify credentials in `profiles.yml` match your Snowflake account

```bash
dbt debug --profiles-dir .
```

### "Object does not exist" Error in dbt

**Problem:** Raw tables not found in Snowflake.

**Solution:** Run the Python loader first to create raw tables:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

### Incremental Model Not Updating

**Problem:** `fct_listing_calendar` not picking up new dates.

**Solution:** Full refresh the incremental model:

```bash
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

### Streamlit Connection Error

**Problem:** Dashboard cannot connect to Snowflake.

**Solution:** 
1. Verify `config/local_credentials.json` exists and has correct values
2. Ensure the schema in credentials matches the mart schema (usually `marts`)
3. Check Snowflake user has SELECT permissions on mart tables

```python
# Test connection manually
import snowflake.connector
import json

with open('config/local_credentials.json', 'r') as f:
    creds = json.load(f)

conn = snowflake.connector.connect(**creds)
cursor = conn.cursor()
cursor.execute("SELECT CURRENT_USER(), CURRENT_ROLE(), CURRENT_DATABASE()")
print(cursor.fetchone())
```

### dbt Test Failures

**Problem:** Tests fail on data quality issues.

**Solution:** Investigate failed tests:

```bash
# Run tests with verbose output
dbt test --profiles-dir . --store-failures

# Query stored failures
# Failures are stored in dbt_test__audit schema
```

### Performance Issues with Large Calendar Table

**Problem:** `fct_listing_calendar` runs slowly.

**Solution:**
1. Ensure incremental strategy is working (not full refresh every time)
2. Add cluster keys in model config:

```sql
{{
    config(
        materialized='incremental',
        unique_key='calendar_key',
        cluster_by=['calendar_date', 'listing_id']
    )
}}
```

3. Use `dbt run --select fct_listing_calendar` (without `+`) to run only that model

### File Upload Errors

**Problem:** Python loader fails to upload files to Snowflake stage.

**Solution:**
1. Verify files exist in `data/raw/` directory
2. Check file permissions
3. Ensure Snowflake user has USAGE on warehouse and CREATE STAGE privilege
4. Manually test stage creation:

```sql
USE SCHEMA RAW;
CREATE OR REPLACE STAGE INSIDE_AIRBNB_STAGE;
PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE;
LIST @INSIDE_AIRBNB_STAGE;
```

## Best Practices

1. **Never commit credentials** — Always use ignored local files (`profiles.yml`, `config/local_credentials.json`)
2. **Use `--profiles-dir .`** — Keep profiles local to project, not in `~/.dbt/`
3. **Run tests after every dbt run** — Catch data quality issues early
4. **Full refresh incrementals carefully** — Only when underlying data changes or model logic changes
5. **Document models** — Add descriptions in schema YAML files for dbt docs
6. **Use refs, not direct table names** — `{{ ref('stg_airbnb__listings') }}` not `RAW.LISTINGS`
7. **Layer models properly** — Staging (clean) → Intermediate (join) → Marts (business logic)
8. **Cache Streamlit connections** — Use `@st.cache_resource` for Snowflake connections
