---
name: snowflake-dbt-airbnb-analytics
description: Analytics engineering project loading Inside Airbnb data into Snowflake, transforming with dbt multi-layer models, and visualizing with Streamlit
triggers:
  - set up airbnb analytics with snowflake and dbt
  - build dbt models for inside airbnb data
  - create snowflake data warehouse for airbnb
  - configure dbt staging intermediate and mart layers
  - load inside airbnb into snowflake
  - run dbt incremental models in snowflake
  - build streamlit dashboard for airbnb analytics
  - troubleshoot dbt snowflake connection
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to work with a complete analytics engineering stack: loading Inside Airbnb open data into Snowflake, transforming it through dbt staging/intermediate/mart layers, validating with tests, and building Streamlit dashboards.

## What This Project Does

- Loads Inside Airbnb CSV/GZIP files into Snowflake RAW schema
- Transforms data through dbt staging → intermediate → marts pipeline
- Implements incremental fact tables with Snowflake merge strategy
- Validates data quality with dbt generic and singular tests
- Powers Streamlit dashboards from analytics marts
- Demonstrates public-safe credential management (no committed secrets)

## Architecture Layers

```
Inside Airbnb files → Python loader → Snowflake stage → RAW schema
→ dbt staging (clean/cast) → dbt intermediate (joins/enrichment)
→ dbt marts (dimensions/facts/aggregates) → Streamlit dashboard
```

## Installation

```bash
# Clone and setup environment
git clone https://github.com/analyticsdurgesh/Snowflake_DBT_Project.git
cd Snowflake_DBT_Project
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Configuration

### Snowflake Credentials (Local, Not Committed)

Create local credential files from examples:

```bash
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

**profiles.yml** (dbt connection):
```yaml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT
      user: YOUR_USER
      password: YOUR_PASSWORD  # Or use authenticator: externalbrowser
      role: YOUR_ROLE
      database: AIRBNB_DB
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
```

**config/local_credentials.json** (Streamlit connection):
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

Both files are gitignored. Never commit real credentials.

## Data Source

Download Inside Airbnb data for New York City from [insideairbnb.com/get-the-data](http://insideairbnb.com/get-the-data/).

Place files in `data/raw/`:
```
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

## Loading Raw Data into Snowflake

The Python loader creates Snowflake objects, uploads files to internal stage, and copies into RAW schema:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What it does:**
1. Reads `config/local_credentials.json`
2. Executes `setup/snowflake_setup.sql` (creates database, schemas, stage, file formats)
3. Uploads local CSV/GZIP files to Snowflake internal stage `@INSIDE_AIRBNB_STAGE`
4. Creates RAW tables with text columns (preserves raw values)
5. Copies staged files into `RAW.LISTINGS`, `RAW.CALENDAR`, `RAW.REVIEWS`, `RAW.NEIGHBOURHOODS`

**Key loader code pattern** (`scripts/load_inside_airbnb_to_snowflake.py`):

```python
import snowflake.connector
import json

# Load credentials from local file
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

# Upload files to Snowflake stage
cursor.execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")
cursor.execute("PUT file://data/raw/calendar.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")
cursor.execute("PUT file://data/raw/reviews.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")
cursor.execute("PUT file://data/raw/neighbourhoods.csv @INSIDE_AIRBNB_STAGE")

# Copy into RAW tables
cursor.execute("""
COPY INTO RAW.LISTINGS
FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
FILE_FORMAT = (FORMAT_NAME = 'CSV_GZIP_FORMAT')
ON_ERROR = 'CONTINUE'
""")

conn.close()
```

## dbt Model Layers

### Staging Layer (`models/staging/`)

Clean, cast, and standardize raw data:

**stg_airbnb__listings.sql**:
```sql
WITH source AS (
    SELECT * FROM {{ source('raw_airbnb', 'listings') }}
)

SELECT
    id::BIGINT AS listing_id,
    name AS listing_name,
    host_id::BIGINT AS host_id,
    host_name,
    neighbourhood_cleansed AS neighbourhood,
    latitude::FLOAT AS latitude,
    longitude::FLOAT AS longitude,
    room_type,
    price,
    minimum_nights::INT AS minimum_nights,
    number_of_reviews::INT AS number_of_reviews,
    last_review::DATE AS last_review,
    reviews_per_month::FLOAT AS reviews_per_month,
    calculated_host_listings_count::INT AS host_listings_count,
    availability_365::INT AS availability_365
FROM source
WHERE id IS NOT NULL
```

**stg_airbnb__calendar.sql**:
```sql
WITH source AS (
    SELECT * FROM {{ source('raw_airbnb', 'calendar') }}
)

SELECT
    listing_id::BIGINT AS listing_id,
    date::DATE AS calendar_date,
    available,
    CASE
        WHEN available = 't' THEN TRUE
        WHEN available = 'f' THEN FALSE
        ELSE NULL
    END AS is_available,
    price,
    CAST(
        REPLACE(REPLACE(price, '$', ''), ',', '') AS FLOAT
    ) AS price_usd,
    adjusted_price,
    minimum_nights::INT AS minimum_nights,
    maximum_nights::INT AS maximum_nights
FROM source
WHERE listing_id IS NOT NULL
  AND date IS NOT NULL
```

### Intermediate Layer (`models/intermediate/`)

Joins, enrichment, and business logic:

**int_airbnb__listing_enriched.sql**:
```sql
WITH listings AS (
    SELECT * FROM {{ ref('stg_airbnb__listings') }}
),

neighbourhoods AS (
    SELECT * FROM {{ ref('stg_airbnb__neighbourhoods') }}
)

SELECT
    l.*,
    n.neighbourhood_group,
    CASE
        WHEN l.room_type = 'Entire home/apt' THEN 'Entire Place'
        WHEN l.room_type = 'Private room' THEN 'Private Room'
        WHEN l.room_type = 'Shared room' THEN 'Shared Room'
        ELSE 'Other'
    END AS room_type_category
FROM listings l
LEFT JOIN neighbourhoods n
    ON l.neighbourhood = n.neighbourhood
```

**int_airbnb__calendar_enriched.sql** (revenue proxy logic):
```sql
WITH calendar AS (
    SELECT * FROM {{ ref('stg_airbnb__calendar') }}
),

listings AS (
    SELECT * FROM {{ ref('int_airbnb__listing_enriched') }}
)

SELECT
    c.listing_id,
    c.calendar_date,
    c.is_available,
    c.price_usd,
    l.neighbourhood,
    l.neighbourhood_group,
    l.room_type,
    -- Revenue proxy: if unavailable, assume booked and revenue = price
    CASE
        WHEN c.is_available = FALSE AND c.price_usd > 0
        THEN c.price_usd
        ELSE 0
    END AS estimated_revenue
FROM calendar c
INNER JOIN listings l
    ON c.listing_id = l.listing_id
```

### Marts Layer (`models/marts/`)

Analytics-ready dimensions and facts:

**dim_listings.sql**:
```sql
SELECT
    listing_id,
    listing_name,
    host_id,
    host_name,
    neighbourhood,
    neighbourhood_group,
    latitude,
    longitude,
    room_type,
    room_type_category,
    minimum_nights,
    number_of_reviews,
    last_review,
    reviews_per_month,
    host_listings_count,
    availability_365
FROM {{ ref('int_airbnb__listing_enriched') }}
```

**fct_listing_calendar.sql** (incremental merge):
```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        incremental_strategy='merge',
        merge_update_columns=['is_available', 'price_usd', 'estimated_revenue']
    )
}}

SELECT
    listing_id,
    calendar_date,
    is_available,
    price_usd,
    neighbourhood,
    neighbourhood_group,
    room_type,
    estimated_revenue
FROM {{ ref('int_airbnb__calendar_enriched') }}

{% if is_incremental() %}
WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
{% endif %}
```

**agg_listing_monthly_performance.sql** (aggregate mart):
```sql
SELECT
    listing_id,
    DATE_TRUNC('month', calendar_date) AS month,
    COUNT(*) AS total_days,
    SUM(CASE WHEN is_available = FALSE THEN 1 ELSE 0 END) AS unavailable_days,
    ROUND(
        SUM(CASE WHEN is_available = FALSE THEN 1 ELSE 0 END) * 100.0 / COUNT(*),
        2
    ) AS occupancy_rate,
    AVG(price_usd) AS avg_daily_price,
    SUM(estimated_revenue) AS total_estimated_revenue
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, DATE_TRUNC('month', calendar_date)
```

## dbt Commands

```bash
# Verify connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream
dbt run --select fct_listing_calendar+ --profiles-dir .

# Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Test specific model
dbt test --select dim_listings --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

## dbt Tests

**Generic tests** (`models/schema.yml`):

```yaml
version: 2

sources:
  - name: raw_airbnb
    schema: RAW
    tables:
      - name: listings
        columns:
          - name: id
            tests:
              - unique
              - not_null

models:
  - name: dim_listings
    columns:
      - name: listing_id
        tests:
          - unique
          - not_null
      - name: room_type
        tests:
          - accepted_values:
              values: ['Entire home/apt', 'Private room', 'Shared room', 'Hotel room']

  - name: fct_listing_calendar
    tests:
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns:
            - listing_id
            - calendar_date
    columns:
      - name: price_usd
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: '>= 0'
```

**Singular test** (`tests/assert_calendar_has_valid_listing.sql`):

```sql
-- Calendar entries must reference existing listings
SELECT
    c.listing_id,
    c.calendar_date
FROM {{ ref('fct_listing_calendar') }} c
LEFT JOIN {{ ref('dim_listings') }} l
    ON c.listing_id = l.listing_id
WHERE l.listing_id IS NULL
```

## Streamlit Dashboard

**dashboard/streamlit_app.py**:

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load credentials from local file (not committed)
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

st.title("Airbnb Analytics Dashboard")

# Top neighbourhoods by revenue
st.header("Top 10 Neighbourhoods by Estimated Revenue")
query = """
SELECT
    neighbourhood_group,
    SUM(total_estimated_revenue) AS total_revenue,
    AVG(occupancy_rate) AS avg_occupancy
FROM analytics.agg_neighbourhood_monthly_performance
GROUP BY neighbourhood_group
ORDER BY total_revenue DESC
LIMIT 10
"""
df = pd.read_sql(query, conn)
st.dataframe(df)

# Top hosts by listing count
st.header("Top 10 Hosts by Number of Listings")
query = """
SELECT
    host_name,
    COUNT(DISTINCT listing_id) AS listing_count,
    AVG(number_of_reviews) AS avg_reviews
FROM analytics.dim_listings
GROUP BY host_name
ORDER BY listing_count DESC
LIMIT 10
"""
df_hosts = pd.read_sql(query, conn)
st.bar_chart(df_hosts.set_index('host_name')['listing_count'])

# Average price by room type
st.header("Average Price by Room Type")
query = """
SELECT
    room_type,
    AVG(price_usd) AS avg_price
FROM analytics.fct_listing_calendar
WHERE price_usd > 0
GROUP BY room_type
ORDER BY avg_price DESC
"""
df_price = pd.read_sql(query, conn)
st.bar_chart(df_price.set_index('room_type'))

conn.close()
```

Run dashboard:
```bash
streamlit run dashboard/streamlit_app.py
```

## Common Workflows

### Initial Setup

```bash
# 1. Download Inside Airbnb data to data/raw/
# 2. Create local credentials
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
# 3. Edit both files with real Snowflake credentials
# 4. Install dependencies
pip install -r requirements.txt
# 5. Load raw data
python scripts/load_inside_airbnb_to_snowflake.py
# 6. Run dbt
dbt run --profiles-dir .
dbt test --profiles-dir .
# 7. Launch dashboard
streamlit run dashboard/streamlit_app.py
```

### Refresh with New Snapshot

```bash
# 1. Download new Inside Airbnb files to data/raw/ (overwrite old)
# 2. Reload raw data
python scripts/load_inside_airbnb_to_snowflake.py
# 3. Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .
# 4. Test
dbt test --profiles-dir .
```

### Develop New dbt Model

```bash
# 1. Create model file in models/marts/
# models/marts/dim_hosts.sql
cat > models/marts/dim_hosts.sql << 'EOF'
SELECT DISTINCT
    host_id,
    host_name,
    host_listings_count,
    COUNT(DISTINCT listing_id) AS total_listings
FROM {{ ref('dim_listings') }}
GROUP BY host_id, host_name, host_listings_count
EOF

# 2. Run specific model
dbt run --select dim_hosts --profiles-dir .

# 3. Add tests in models/schema.yml
# 4. Run tests
dbt test --select dim_hosts --profiles-dir .
```

## Troubleshooting

### "Object does not exist" in dbt

**Issue**: dbt can't find source tables.

**Solution**: Verify Snowflake objects exist and profiles.yml database/schema match:

```sql
-- In Snowflake UI
USE DATABASE AIRBNB_DB;
SHOW SCHEMAS;
SHOW TABLES IN SCHEMA RAW;
SELECT COUNT(*) FROM RAW.LISTINGS;
```

Check `profiles.yml`:
```yaml
database: AIRBNB_DB  # Must match
schema: ANALYTICS    # Target schema for dbt models
```

### "Connection error" in Streamlit

**Issue**: Dashboard can't connect to Snowflake.

**Solution**: Verify `config/local_credentials.json` exists and has correct values:

```python
# Test connection manually
import snowflake.connector
import json

with open('config/local_credentials.json') as f:
    creds = json.load(f)

conn = snowflake.connector.connect(**creds)
print(conn.cursor().execute("SELECT CURRENT_USER()").fetchone())
conn.close()
```

### dbt incremental model not updating

**Issue**: New data not appearing in `fct_listing_calendar`.

**Solution**: Full refresh the incremental model:

```bash
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

Or drop and rebuild:
```sql
-- In Snowflake
DROP TABLE IF EXISTS ANALYTICS.FCT_LISTING_CALENDAR;
```
```bash
dbt run --select fct_listing_calendar --profiles-dir .
```

### Python loader fails on stage upload

**Issue**: `PUT` command fails with authentication error.

**Solution**: Ensure Snowflake user has USAGE on warehouse and WRITE on stage:

```sql
GRANT USAGE ON WAREHOUSE COMPUTE_WH TO ROLE YOUR_ROLE;
GRANT USAGE ON DATABASE AIRBNB_DB TO ROLE YOUR_ROLE;
GRANT ALL ON SCHEMA AIRBNB_DB.RAW TO ROLE YOUR_ROLE;
GRANT READ, WRITE ON STAGE AIRBNB_DB.RAW.INSIDE_AIRBNB_STAGE TO ROLE YOUR_ROLE;
```

### dbt tests fail: "division by zero"

**Issue**: Occupancy rate calculation fails on empty calendar data.

**Solution**: Add null checks in aggregate model:

```sql
ROUND(
    NULLIF(COUNT(*), 0) / 
    SUM(CASE WHEN is_available = FALSE THEN 1 ELSE 0 END) * 100.0,
    2
) AS occupancy_rate
```

## Key Files Reference

- `dbt_project.yml`: dbt project config, model paths, target schema
- `profiles.yml`: Snowflake connection (local, gitignored)
- `config/local_credentials.json`: Streamlit Snowflake creds (local, gitignored)
- `models/sources.yml`: Raw table source definitions
- `models/schema.yml`: Model tests and documentation
- `scripts/load_inside_airbnb_to_snowflake.py`: Python loader
- `setup/snowflake_setup.sql`: DDL for database, schemas, stage, file formats

## Environment Variables (Alternative to JSON)

Instead of `config/local_credentials.json`, use environment variables:

```bash
export SNOWFLAKE_ACCOUNT=your_account
export SNOWFLAKE_USER=your_user
export SNOWFLAKE_PASSWORD=your_password
export SNOWFLAKE_WAREHOUSE=COMPUTE_WH
export SNOWFLAKE_DATABASE=AIRBNB_DB
export SNOWFLAKE_SCHEMA=ANALYTICS
export SNOWFLAKE_ROLE=YOUR_ROLE
```

Update loader and dashboard to read from env:

```python
import os

conn = snowflake.connector.connect(
    account=os.getenv('SNOWFLAKE_ACCOUNT'),
    user=os.getenv('SNOWFLAKE_USER'),
    password=os.getenv('SNOWFLAKE_PASSWORD'),
    warehouse=os.getenv('SNOWFLAKE_WAREHOUSE'),
    database=os.getenv('SNOWFLAKE_DATABASE'),
    role=os.getenv('SNOWFLAKE_ROLE')
)
```

Update `profiles.yml`:

```yaml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      # ... rest of config
```

This skill gives AI agents complete knowledge of the Snowflake dbt Airbnb analytics pipeline architecture, configuration, code patterns, and troubleshooting steps.
