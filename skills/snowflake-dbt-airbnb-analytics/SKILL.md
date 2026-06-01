---
name: snowflake-dbt-airbnb-analytics
description: Analytics engineering with Snowflake, dbt, and Inside Airbnb data including staging, marts, incremental models, tests, and Streamlit dashboards
triggers:
  - set up Snowflake dbt project with Inside Airbnb data
  - load Inside Airbnb data into Snowflake
  - build dbt staging and mart models for Airbnb analytics
  - create incremental fact tables in dbt with Snowflake
  - run dbt tests for data quality validation
  - build Streamlit dashboard connected to Snowflake marts
  - configure dbt profiles for Snowflake connection
  - troubleshoot dbt incremental merge strategy
---

# Snowflake dbt Airbnb Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill covers the **Snowflake_DBT_Project** — a complete analytics engineering pipeline that loads Inside Airbnb open data into Snowflake, transforms it through dbt staging/intermediate/mart layers, validates with dbt tests, and powers a Streamlit dashboard.

## What This Project Does

- **Loads raw CSV data** from Inside Airbnb into Snowflake internal stages
- **Transforms data** through dbt layers: staging → intermediate → marts (dimensions, facts, aggregates)
- **Incremental fact modeling** using Snowflake merge strategy for calendar availability
- **Data quality tests** with dbt generic and singular tests
- **Streamlit dashboard** for neighbourhood and host analytics
- **Public-safe credentials** via local ignored config files (no env vars committed)

## Installation & Setup

### 1. Clone and Install Dependencies

```bash
git clone https://github.com/analyticsdurgesh/Snowflake_DBT_Project.git
cd Snowflake_DBT_Project
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

**Key dependencies:**
- `dbt-snowflake`
- `snowflake-connector-python`
- `streamlit`
- `pandas`

### 2. Configure Local Credentials

Create local config files from examples:

```bash
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

**Edit `profiles.yml`** with your Snowflake credentials:

```yaml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT  # e.g., xy12345.us-east-1
      user: YOUR_USERNAME
      password: YOUR_PASSWORD
      role: YOUR_ROLE
      warehouse: YOUR_WAREHOUSE
      database: AIRBNB_DB
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

**Edit `config/local_credentials.json`** for Streamlit:

```json
{
  "account": "YOUR_ACCOUNT",
  "user": "YOUR_USERNAME",
  "password": "YOUR_PASSWORD",
  "warehouse": "YOUR_WAREHOUSE",
  "database": "AIRBNB_DB",
  "schema": "ANALYTICS",
  "role": "YOUR_ROLE"
}
```

⚠️ **Security**: These files are `.gitignore`d. Never commit real credentials.

### 3. Download Inside Airbnb Data

Download from [Inside Airbnb](https://insideairbnb.com/get-the-data/) (recommended: New York City):

- `listings.csv.gz`
- `calendar.csv.gz`
- `reviews.csv.gz`
- `neighbourhoods.csv`

Place them in `data/raw/`:

```bash
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

## Loading Data into Snowflake

The Python loader script automates raw data ingestion:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What it does:**

1. Executes `setup/snowflake_setup.sql` to create database, schemas, stage
2. Uploads local files to Snowflake internal stage `INSIDE_AIRBNB_STAGE`
3. Infers CSV headers and creates raw tables in `RAW` schema
4. Copies stage data into raw tables

**Key loader logic** (from `load_inside_airbnb_to_snowflake.py`):

```python
import snowflake.connector
import json

# Load credentials from local config
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
    for statement in f.read().split(';'):
        if statement.strip():
            cursor.execute(statement)

# Upload file to stage
cursor.execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")

# Create raw table from CSV structure
cursor.execute("""
CREATE OR REPLACE TABLE RAW.LISTINGS (
    id TEXT,
    name TEXT,
    host_id TEXT,
    host_name TEXT,
    neighbourhood TEXT,
    latitude TEXT,
    longitude TEXT,
    room_type TEXT,
    price TEXT,
    minimum_nights TEXT,
    availability_365 TEXT
    -- ... more columns
)
""")

# Copy stage data into table
cursor.execute("""
COPY INTO RAW.LISTINGS
FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"')
""")

cursor.close()
conn.close()
```

## dbt Model Architecture

### Layer Structure

| Layer | Schema | Purpose | Example Models |
|-------|--------|---------|----------------|
| **Sources** | `RAW` | Raw Snowflake tables | `LISTINGS`, `CALENDAR`, `REVIEWS`, `NEIGHBOURHOODS` |
| **Staging** | `ANALYTICS` | Type casting, cleaning | `stg_airbnb__listings`, `stg_airbnb__calendar` |
| **Intermediate** | `ANALYTICS` | Joins, enrichment | `int_airbnb__listing_enriched`, `int_airbnb__calendar_enriched` |
| **Marts** | `ANALYTICS` | Analytics-ready | `dim_listings`, `dim_hosts`, `fct_listing_calendar` |
| **Aggregates** | `ANALYTICS` | Monthly rollups | `agg_listing_monthly_performance` |

### Key dbt Commands

```bash
# Verify connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select dim_listings+ --profiles-dir .

# Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

## dbt Model Examples

### Staging Model: `stg_airbnb__listings.sql`

```sql
-- models/staging/stg_airbnb__listings.sql
{{ config(materialized='view') }}

WITH source AS (
    SELECT * FROM {{ source('airbnb_raw', 'LISTINGS') }}
),

cleaned AS (
    SELECT
        TRY_CAST(id AS INTEGER) AS listing_id,
        TRIM(name) AS listing_name,
        TRY_CAST(host_id AS INTEGER) AS host_id,
        TRIM(host_name) AS host_name,
        TRIM(neighbourhood) AS neighbourhood,
        TRY_CAST(latitude AS FLOAT) AS latitude,
        TRY_CAST(longitude AS FLOAT) AS longitude,
        TRIM(room_type) AS room_type,
        TRY_CAST(REPLACE(REPLACE(price, '$', ''), ',', '') AS DECIMAL(10,2)) AS price,
        TRY_CAST(minimum_nights AS INTEGER) AS minimum_nights,
        TRY_CAST(availability_365 AS INTEGER) AS availability_365
    FROM source
    WHERE id IS NOT NULL
)

SELECT * FROM cleaned
```

### Intermediate Model: `int_airbnb__calendar_enriched.sql`

```sql
-- models/intermediate/int_airbnb__calendar_enriched.sql
{{ config(materialized='view') }}

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
        c.price,
        l.listing_name,
        l.neighbourhood,
        l.room_type,
        l.host_id,
        l.host_name,
        -- Revenue proxy: if not available, assume it was booked
        CASE
            WHEN c.available = 'f' THEN c.price
            ELSE 0
        END AS estimated_revenue
    FROM calendar c
    LEFT JOIN listings l ON c.listing_id = l.listing_id
)

SELECT * FROM enriched
```

### Incremental Fact Model: `fct_listing_calendar.sql`

```sql
-- models/marts/fct_listing_calendar.sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        merge_update_columns=['available', 'price', 'estimated_revenue']
    )
}}

WITH calendar_enriched AS (
    SELECT * FROM {{ ref('int_airbnb__calendar_enriched') }}
)

SELECT
    listing_id,
    calendar_date,
    available,
    price,
    estimated_revenue,
    neighbourhood,
    room_type,
    host_id,
    CURRENT_TIMESTAMP() AS _loaded_at
FROM calendar_enriched

{% if is_incremental() %}
    WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
{% endif %}
```

**Incremental strategy**: Uses Snowflake `MERGE` with composite unique key `[listing_id, calendar_date]`.

### Aggregate Model: `agg_listing_monthly_performance.sql`

```sql
-- models/marts/agg_listing_monthly_performance.sql
{{ config(materialized='table') }}

WITH calendar_facts AS (
    SELECT * FROM {{ ref('fct_listing_calendar') }}
),

monthly AS (
    SELECT
        listing_id,
        DATE_TRUNC('MONTH', calendar_date) AS month,
        neighbourhood,
        room_type,
        host_id,
        COUNT(*) AS total_days,
        SUM(CASE WHEN available = 'f' THEN 1 ELSE 0 END) AS unavailable_days,
        SUM(estimated_revenue) AS total_estimated_revenue,
        AVG(price) AS avg_price
    FROM calendar_facts
    GROUP BY 1, 2, 3, 4, 5
)

SELECT
    *,
    ROUND(unavailable_days::FLOAT / NULLIF(total_days, 0), 3) AS unavailability_rate
FROM monthly
```

## dbt Sources Configuration

```yaml
# models/staging/sources.yml
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
          - name: date
            tests:
              - not_null
      - name: REVIEWS
      - name: NEIGHBOURHOODS
```

## dbt Tests

### Generic Tests in Model Config

```yaml
# models/marts/dim_listings.yml
version: 2

models:
  - name: dim_listings
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
```

### Singular Test

```sql
-- tests/no_duplicate_listing_dates.sql
-- Check for duplicate listing_id + calendar_date combinations in fact table

SELECT
    listing_id,
    calendar_date,
    COUNT(*) AS occurrences
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, calendar_date
HAVING COUNT(*) > 1
```

Run tests:

```bash
dbt test --profiles-dir .
```

## Streamlit Dashboard

The dashboard queries Snowflake marts directly using `snowflake-connector-python`.

**Run the dashboard:**

```bash
streamlit run dashboard/streamlit_app.py
```

**Key dashboard code** (`dashboard/streamlit_app.py`):

```python
import streamlit as st
import pandas as pd
import snowflake.connector
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

st.title("Airbnb Analytics Dashboard")

# Neighbourhood performance
st.header("Top Neighbourhoods by Estimated Revenue")
query = """
SELECT
    neighbourhood,
    SUM(total_estimated_revenue) AS total_revenue,
    AVG(unavailability_rate) AS avg_unavailability_rate,
    COUNT(DISTINCT listing_id) AS num_listings
FROM ANALYTICS.AGG_NEIGHBOURHOOD_MONTHLY_PERFORMANCE
GROUP BY neighbourhood
ORDER BY total_revenue DESC
LIMIT 10
"""
df = pd.read_sql(query, conn)
st.bar_chart(df.set_index('NEIGHBOURHOOD')['TOTAL_REVENUE'])
st.dataframe(df)

# Host leaderboard
st.header("Top Hosts by Listing Count")
query = """
SELECT
    host_id,
    host_name,
    COUNT(*) AS listing_count,
    AVG(price) AS avg_price
FROM ANALYTICS.DIM_LISTINGS
GROUP BY host_id, host_name
ORDER BY listing_count DESC
LIMIT 10
"""
df_hosts = pd.read_sql(query, conn)
st.dataframe(df_hosts)

# Room type breakdown
st.header("Average Price by Room Type")
query = """
SELECT
    room_type,
    AVG(avg_price) AS avg_price
FROM ANALYTICS.AGG_LISTING_MONTHLY_PERFORMANCE
GROUP BY room_type
ORDER BY avg_price DESC
"""
df_room = pd.read_sql(query, conn)
st.bar_chart(df_room.set_index('ROOM_TYPE'))

conn.close()
```

## Common Patterns & Workflows

### Initial Setup

```bash
# 1. Install and configure
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
# Edit both files with real credentials

# 2. Load raw data
python scripts/load_inside_airbnb_to_snowflake.py

# 3. Run dbt pipeline
dbt run --profiles-dir .
dbt test --profiles-dir .

# 4. Launch dashboard
streamlit run dashboard/streamlit_app.py
```

### Refresh with New Data Snapshot

```bash
# Download new Inside Airbnb files into data/raw/
python scripts/load_inside_airbnb_to_snowflake.py

# Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .
dbt test --profiles-dir .
```

### Developing a New Mart Model

```bash
# 1. Create model file
# models/marts/dim_new_model.sql

# 2. Run just that model
dbt run --select dim_new_model --profiles-dir .

# 3. Add tests in schema.yml
# models/marts/dim_new_model.yml

# 4. Run tests
dbt test --select dim_new_model --profiles-dir .

# 5. Rebuild downstream dependencies
dbt run --select dim_new_model+ --profiles-dir .
```

### Adding a dbt Macro

```python
# macros/cents_to_dollars.sql
{% macro cents_to_dollars(column_name) %}
    ({{ column_name }} / 100.0)::DECIMAL(10,2)
{% endmacro %}
```

Usage in model:

```sql
SELECT
    listing_id,
    {{ cents_to_dollars('price_in_cents') }} AS price
FROM {{ ref('stg_airbnb__listings') }}
```

## Troubleshooting

### `dbt debug` fails with authentication error

**Symptom:**
```
Connection test failed: 250001: Could not connect to Snowflake backend
```

**Solution:**
- Verify `profiles.yml` credentials (account, user, password, role, warehouse)
- Ensure Snowflake account format is correct: `xy12345.region` or `orgname-accountname`
- Check network access / IP whitelisting in Snowflake

### Incremental model not updating

**Symptom:**
```
dbt run completes but fct_listing_calendar has stale data
```

**Solution:**
- Run with `--full-refresh`:
  ```bash
  dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
  ```
- Check `is_incremental()` logic in model
- Verify `unique_key` matches expected composite key

### Test failures on price columns

**Symptom:**
```
dbt test fails: "expression_is_true" on price >= 0
```

**Solution:**
- Inspect source data for invalid price formats (e.g., `$1,234.56`)
- Update staging model to strip `$` and `,`:
  ```sql
  TRY_CAST(REPLACE(REPLACE(price, '$', ''), ',', '') AS DECIMAL(10,2)) AS price
  ```
- Re-run staging and downstream:
  ```bash
  dbt run --select stg_airbnb__listings+ --profiles-dir .
  ```

### Streamlit dashboard connection error

**Symptom:**
```
snowflake.connector.errors.DatabaseError: 250001
```

**Solution:**
- Verify `config/local_credentials.json` has correct values
- Ensure Snowflake user has `USAGE` on database and schema
- Grant `SELECT` on mart tables:
  ```sql
  GRANT SELECT ON ALL TABLES IN SCHEMA ANALYTICS TO ROLE YOUR_ROLE;
  ```

### dbt docs not showing lineage

**Symptom:**
```
dbt docs generate runs but lineage graph is incomplete
```

**Solution:**
- Ensure all `ref()` and `source()` calls are correct
- Regenerate:
  ```bash
  dbt clean --profiles-dir .
  dbt compile --profiles-dir .
  dbt docs generate --profiles-dir .
  dbt docs serve --profiles-dir .
  ```
- Check browser console for JavaScript errors

### Python loader script fails on file upload

**Symptom:**
```
FileNotFoundError: [Errno 2] No such file or directory: 'data/raw/listings.csv.gz'
```

**Solution:**
- Verify files exist in `data/raw/`
- Check file names match exactly (case-sensitive on Linux/Mac)
- Ensure `.gz` files are downloaded as-is (not auto-decompressed by browser)

## Key Configuration Files

### `dbt_project.yml`

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
  - "logs"

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
      +schema: marts
```

### `profiles.yml` Structure

```yaml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"  # or hardcode for local dev
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: "{{ env_var('SNOWFLAKE_ROLE') }}"
      warehouse: "{{ env_var('SNOWFLAKE_WAREHOUSE') }}"
      database: AIRBNB_DB
      schema: ANALYTICS
      threads: 4
```

For local dev, replace `{{ env_var(...) }}` with actual values since this project uses local credential files, not environment variables.

## Example Business Queries

**Top neighbourhoods by revenue:**

```sql
SELECT
    neighbourhood,
    SUM(total_estimated_revenue) AS total_revenue
FROM ANALYTICS.AGG_NEIGHBOURHOOD_MONTHLY_PERFORMANCE
GROUP BY neighbourhood
ORDER BY total_revenue DESC
LIMIT 10;
```

**Hosts with most listings:**

```sql
SELECT
    host_id,
    host_name,
    COUNT(*) AS listing_count
FROM ANALYTICS.DIM_LISTINGS
GROUP BY host_id, host_name
ORDER BY listing_count DESC
LIMIT 10;
```

**Average price by room type:**

```sql
SELECT
    room_type,
    AVG(price) AS avg_price,
    COUNT(*) AS listing_count
FROM ANALYTICS.DIM_LISTINGS
GROUP BY room_type
ORDER BY avg_price DESC;
```

**Monthly availability trends:**

```sql
SELECT
    month,
    AVG(unavailability_rate) AS avg_unavailability,
    SUM(total_estimated_revenue) AS total_revenue
FROM ANALYTICS.AGG_LISTING_MONTHLY_PERFORMANCE
GROUP BY month
ORDER BY month;
```

## Additional Resources

- [Inside Airbnb data source](https://insideairbnb.com/get-the-data/)
- [dbt Snowflake adapter docs](https://docs.getdbt.com/reference/warehouse-setups/snowflake-setup)
- [Snowflake MERGE (incremental) strategy](https://docs.getdbt.com/reference/resource-configs/snowflake-configs#merge-behavior-incremental-models)
- [dbt testing guide](https://docs.getdbt.com/docs/build/tests)
- [Streamlit + Snowflake connector](https://docs.streamlit.io/knowledge-base/tutorials/databases/snowflake)

---

This skill enables AI coding agents to guide developers through the complete Snowflake + dbt + Streamlit analytics pipeline, from raw data loading to production dashboards.
