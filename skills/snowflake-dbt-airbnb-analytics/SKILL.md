---
name: snowflake-dbt-airbnb-analytics
description: Build Snowflake data warehouses with dbt using Inside Airbnb data for analytics engineering, including staging, marts, incremental models, tests, and Streamlit dashboards.
triggers:
  - set up snowflake dbt project with airbnb data
  - create dbt models for inside airbnb dataset
  - build incremental fact tables in snowflake with dbt
  - configure dbt profiles for snowflake warehouse
  - load inside airbnb data into snowflake
  - add dbt tests for data quality validation
  - create streamlit dashboard for snowflake analytics
  - implement dbt staging and mart layers
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build complete analytics engineering pipelines using Snowflake, dbt, and the Inside Airbnb dataset. The project demonstrates modern data warehouse patterns: raw data ingestion, dbt staging/intermediate/mart layers, incremental fact modeling, data quality tests, and Streamlit reporting dashboards.

## What This Project Does

The `Snowflake_DBT_Project` is a portfolio-grade analytics engineering pipeline that:

- Loads Inside Airbnb CSV/GZIP files into Snowflake raw tables via internal stages
- Transforms data through dbt staging → intermediate → mart layers
- Builds incremental fact tables using Snowflake merge strategy
- Validates data quality with generic and singular dbt tests
- Powers a Streamlit dashboard with analytics-ready marts
- Demonstrates public-safe credential management (no committed secrets)

**Key Components:**
- Python loader script for Snowflake data ingestion
- dbt models: staging, intermediate, dimensions, facts, aggregates
- Incremental `fct_listing_calendar` with merge strategy
- Generic and singular tests for data quality
- Streamlit dashboard reading from Snowflake marts

## Installation

### Prerequisites

- Python 3.8+
- Snowflake account with warehouse, database, schema creation privileges
- Inside Airbnb dataset files (listings.csv.gz, calendar.csv.gz, reviews.csv.gz, neighbourhoods.csv)

### Setup Steps

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

### Credential Configuration

**Never commit real credentials.** Use local-only configuration files:

```bash
# Create local profiles.yml for dbt
cp profiles.yml.example profiles.yml

# Create local Snowflake credentials for Python scripts
cp config/local_credentials.example.json config/local_credentials.json
```

Edit `profiles.yml`:

```yaml
snowflake_dbt_airbnb:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_SNOWFLAKE_ACCOUNT  # e.g., abc12345.us-east-1
      user: YOUR_USERNAME
      password: YOUR_PASSWORD
      role: YOUR_ROLE
      warehouse: YOUR_WAREHOUSE
      database: AIRBNB_ANALYTICS
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

Edit `config/local_credentials.json`:

```json
{
  "account": "YOUR_SNOWFLAKE_ACCOUNT",
  "user": "YOUR_USERNAME",
  "password": "YOUR_PASSWORD",
  "warehouse": "YOUR_WAREHOUSE",
  "database": "AIRBNB_ANALYTICS",
  "role": "YOUR_ROLE"
}
```

**Git-ignored files:**
- `profiles.yml`
- `config/local_credentials.json`
- `.user.yml`
- `target/`
- `logs/`
- `.venv/`

## Loading Raw Data into Snowflake

Place Inside Airbnb files in `data/raw/`:

```text
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

Download from: [Inside Airbnb NYC](https://insideairbnb.com/get-the-data/)

Run the Python loader:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What the loader does:**
1. Executes `setup/snowflake_setup.sql` to create database, schemas, stage
2. Uploads local files to Snowflake internal stage `INSIDE_AIRBNB_STAGE`
3. Infers CSV schemas and creates raw tables in `RAW` schema
4. Copies data from stage to raw tables using `COPY INTO`

**Loader script pattern (simplified):**

```python
import snowflake.connector
import json

# Load credentials from local config (never commit this file)
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

# Upload file to internal stage
cursor.execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")

# Create raw table and copy data
cursor.execute("""
    CREATE OR REPLACE TABLE RAW.LISTINGS (
        id TEXT,
        name TEXT,
        host_id TEXT,
        ...
    )
""")

cursor.execute("""
    COPY INTO RAW.LISTINGS
    FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
    FILE_FORMAT = (TYPE=CSV SKIP_HEADER=1 FIELD_OPTIONALLY_ENCLOSED_BY='"')
""")

cursor.close()
conn.close()
```

## dbt Project Structure

```text
models/
├── staging/
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

### Key dbt Commands

```bash
# Verify connection to Snowflake
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select dim_listings+ --profiles-dir .

# Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Run specific test
dbt test --select stg_airbnb__listings --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .

# Serve documentation locally
dbt docs serve --profiles-dir .
```

## dbt Model Examples

### Staging Model: `stg_airbnb__listings.sql`

Staging models clean, cast, and standardize raw data.

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
        TRY_CAST(price AS FLOAT) AS price,
        TRY_CAST(minimum_nights AS INTEGER) AS minimum_nights,
        TRY_CAST(number_of_reviews AS INTEGER) AS number_of_reviews,
        TRY_TO_DATE(last_review) AS last_review_date,
        TRY_CAST(reviews_per_month AS FLOAT) AS reviews_per_month,
        TRY_CAST(availability_365 AS INTEGER) AS availability_365,
        CURRENT_TIMESTAMP() AS _loaded_at
    FROM source
)

SELECT * FROM cleaned
```

### Intermediate Model: `int_airbnb__listing_enriched.sql`

Intermediate models perform joins and business logic.

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
        l.last_review_date,
        l.reviews_per_month,
        l.availability_365,
        CASE
            WHEN l.number_of_reviews >= 10 THEN 'High Activity'
            WHEN l.number_of_reviews BETWEEN 1 AND 9 THEN 'Medium Activity'
            ELSE 'Low Activity'
        END AS activity_tier,
        l._loaded_at
    FROM listings l
    LEFT JOIN neighbourhoods n
        ON l.neighbourhood = n.neighbourhood
)

SELECT * FROM enriched
```

### Incremental Fact Model: `fct_listing_calendar.sql`

Incremental models use merge strategy to handle large datasets efficiently.

```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        merge_update_columns=['available', 'price', 'adjusted_price', 'minimum_nights', 'maximum_nights', '_loaded_at']
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
    adjusted_price,
    minimum_nights,
    maximum_nights,
    neighbourhood,
    neighbourhood_group,
    room_type,
    -- Revenue proxy: if unavailable, assume booked
    CASE
        WHEN available = FALSE AND price > 0 THEN price
        ELSE 0
    END AS estimated_revenue,
    _loaded_at
FROM calendar_enriched

{% if is_incremental() %}
    WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
{% endif %}
```

### Aggregate Model: `agg_listing_monthly_performance.sql`

Aggregates build reporting-ready datasets.

```sql
WITH calendar_facts AS (
    SELECT * FROM {{ ref('fct_listing_calendar') }}
),

monthly AS (
    SELECT
        listing_id,
        DATE_TRUNC('month', calendar_date) AS month_start,
        neighbourhood,
        neighbourhood_group,
        room_type,
        COUNT(*) AS total_days,
        SUM(CASE WHEN available = FALSE THEN 1 ELSE 0 END) AS unavailable_days,
        SUM(CASE WHEN available = TRUE THEN 1 ELSE 0 END) AS available_days,
        ROUND(AVG(price), 2) AS avg_price,
        ROUND(SUM(estimated_revenue), 2) AS total_estimated_revenue,
        ROUND(SUM(CASE WHEN available = FALSE THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS occupancy_rate_pct
    FROM calendar_facts
    GROUP BY 1, 2, 3, 4, 5
)

SELECT * FROM monthly
```

## dbt Testing

### Generic Tests (schema.yml)

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

models:
  - name: stg_airbnb__listings
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

  - name: fct_listing_calendar
    tests:
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns:
            - listing_id
            - calendar_date
    columns:
      - name: price
        tests:
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
```

### Singular Test (tests/no_duplicate_listing_dates.sql)

```sql
-- Test: Ensure no duplicate listing-date combinations in incremental fact
SELECT
    listing_id,
    calendar_date,
    COUNT(*) AS record_count
FROM {{ ref('fct_listing_calendar') }}
GROUP BY 1, 2
HAVING COUNT(*) > 1
```

## Streamlit Dashboard

The dashboard reads from Snowflake marts using credentials from `config/local_credentials.json`.

### Running the Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

### Dashboard Code Pattern

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load Snowflake credentials (never commit this file)
with open('config/local_credentials.json') as f:
    creds = json.load(f)

@st.cache_resource
def get_snowflake_connection():
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        database='AIRBNB_ANALYTICS',
        schema='ANALYTICS',
        role=creds['role']
    )

def run_query(query):
    conn = get_snowflake_connection()
    df = pd.read_sql(query, conn)
    return df

st.title("Airbnb Analytics Dashboard")

# Example: Top neighbourhoods by revenue
st.header("Top Neighbourhoods by Estimated Revenue")

query = """
    SELECT
        neighbourhood_group,
        neighbourhood,
        SUM(total_estimated_revenue) AS total_revenue
    FROM ANALYTICS.agg_neighbourhood_monthly_performance
    GROUP BY 1, 2
    ORDER BY 3 DESC
    LIMIT 10
"""

df_revenue = run_query(query)
st.dataframe(df_revenue)
st.bar_chart(df_revenue.set_index('neighbourhood')['total_revenue'])

# Example: Occupancy rate by room type
st.header("Occupancy Rate by Room Type")

query_occupancy = """
    SELECT
        room_type,
        ROUND(AVG(occupancy_rate_pct), 2) AS avg_occupancy_rate
    FROM ANALYTICS.agg_listing_monthly_performance
    GROUP BY 1
    ORDER BY 2 DESC
"""

df_occupancy = run_query(query_occupancy)
st.dataframe(df_occupancy)
```

## Common Patterns

### Adding a New dbt Model

1. Create SQL file in appropriate layer:
   - `models/staging/` for source cleaning
   - `models/intermediate/` for joins/enrichment
   - `models/marts/` for analytics-ready tables

2. Reference upstream models with `{{ ref('model_name') }}`

3. Add to `schema.yml` for tests and documentation:

```yaml
models:
  - name: my_new_model
    description: "Business description of the model"
    columns:
      - name: key_column
        tests:
          - not_null
          - unique
```

4. Run the new model:

```bash
dbt run --select my_new_model --profiles-dir .
dbt test --select my_new_model --profiles-dir .
```

### Incremental Model Pattern

Use for large fact tables to avoid full rebuilds:

```sql
{{
    config(
        materialized='incremental',
        unique_key='unique_id',
        merge_update_columns=['col1', 'col2', '_loaded_at']
    )
}}

SELECT * FROM {{ ref('source_model') }}

{% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

Full refresh when needed:

```bash
dbt run --full-refresh --select fct_my_table --profiles-dir .
```

### Using dbt Macros

Example macro for standardized timestamp:

```sql
-- macros/get_current_timestamp.sql
{% macro get_current_timestamp() %}
    CURRENT_TIMESTAMP()
{% endmacro %}
```

Use in models:

```sql
SELECT
    id,
    name,
    {{ get_current_timestamp() }} AS _loaded_at
FROM source
```

### dbt Source Freshness

Check data freshness:

```yaml
sources:
  - name: airbnb_raw
    tables:
      - name: listings
        loaded_at_field: _loaded_at
        freshness:
          warn_after: {count: 24, period: hour}
          error_after: {count: 48, period: hour}
```

Run freshness check:

```bash
dbt source freshness --profiles-dir .
```

## Troubleshooting

### Connection Issues

**Problem:** `dbt debug` fails with authentication error

**Solution:**
- Verify `profiles.yml` credentials match your Snowflake account
- Check account identifier format: `<account_locator>.<region>` (e.g., `abc12345.us-east-1`)
- Ensure user has correct role and warehouse access
- Test connection directly with Python:

```python
import snowflake.connector
conn = snowflake.connector.connect(
    account='YOUR_ACCOUNT',
    user='YOUR_USER',
    password='YOUR_PASSWORD',
    warehouse='YOUR_WAREHOUSE'
)
print(conn.cursor().execute("SELECT CURRENT_USER()").fetchone())
conn.close()
```

### Loader Script Fails

**Problem:** `load_inside_airbnb_to_snowflake.py` can't find files

**Solution:**
- Ensure files are in `data/raw/` with exact names:
  - `listings.csv.gz`
  - `calendar.csv.gz`
  - `reviews.csv.gz`
  - `neighbourhoods.csv`
- Check file permissions (readable)
- Verify `config/local_credentials.json` exists and has valid JSON

**Problem:** Stage upload fails with "File not found"

**Solution:**
- Use absolute paths or ensure working directory is project root
- Check Snowflake warehouse is running (resume if suspended)

### dbt Model Failures

**Problem:** `fct_listing_calendar` incremental merge fails

**Solution:**
- Full refresh to rebuild from scratch:

```bash
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

- Check unique_key columns have no nulls
- Verify source data has no duplicates

**Problem:** Tests fail with schema mismatch

**Solution:**
- Re-run staging models to refresh schemas:

```bash
dbt run --select staging.* --profiles-dir .
```

- Check source data types in Snowflake match model expectations
- Use `TRY_CAST()` for safe type conversion

### Streamlit Dashboard Issues

**Problem:** Dashboard shows "connection refused"

**Solution:**
- Verify `config/local_credentials.json` exists
- Check Snowflake warehouse is running
- Test connection with Python script before running Streamlit
- Ensure schema `ANALYTICS` exists in `AIRBNB_ANALYTICS` database

**Problem:** Queries timeout

**Solution:**
- Use larger Snowflake warehouse
- Add query result caching with `@st.cache_data`
- Pre-aggregate data in dbt marts instead of querying facts directly

### Data Quality Issues

**Problem:** Price values are null or negative

**Solution:**
- Add tests to catch issues:

```yaml
- name: price
  tests:
    - not_null
    - dbt_utils.expression_is_true:
        expression: ">= 0"
```

- Clean in staging layer:

```sql
SELECT
    CASE
        WHEN price < 0 THEN NULL
        ELSE price
    END AS price
FROM source
```

**Problem:** Duplicate listing-date combinations in incremental fact

**Solution:**
- Add singular test (already included in project)
- Use `ROW_NUMBER()` to deduplicate in intermediate layer:

```sql
WITH deduplicated AS (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY listing_id, calendar_date ORDER BY _loaded_at DESC) AS rn
    FROM source
)
SELECT * FROM deduplicated WHERE rn = 1
```

## Advanced Usage

### Custom dbt Packages

Install dbt packages for advanced testing:

```yaml
# packages.yml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1
  - package: calogica/dbt_expectations
    version: 0.10.1
```

Install:

```bash
dbt deps --profiles-dir .
```

### Snapshot Tables (SCD Type 2)

Track slowly changing dimensions:

```sql
-- snapshots/listings_snapshot.sql
{% snapshot listings_snapshot %}

{{
    config(
      target_schema='snapshots',
      unique_key='listing_id',
      strategy='timestamp',
      updated_at='_loaded_at',
    )
}}

SELECT * FROM {{ ref('stg_airbnb__listings') }}

{% endsnapshot %}
```

Run snapshots:

```bash
dbt snapshot --profiles-dir .
```

### Exposure Definitions

Document downstream dashboards:

```yaml
# models/exposures.yml
version: 2

exposures:
  - name: airbnb_streamlit_dashboard
    type: dashboard
    maturity: medium
    owner:
      name: Durgesh Yadav
      email: durgesh@example.com
    description: "Streamlit dashboard showing neighbourhood and host analytics"
    depends_on:
      - ref('dim_listings')
      - ref('dim_hosts')
      - ref('agg_listing_monthly_performance')
      - ref('agg_neighbourhood_monthly_performance')
    url: http://localhost:8501
```

## Environment Variables (Alternative to Local Config)

Instead of `config/local_credentials.json`, use environment variables:

```bash
export SNOWFLAKE_ACCOUNT="abc12345.us-east-1"
export SNOWFLAKE_USER="myuser"
export SNOWFLAKE_PASSWORD="mypassword"
export SNOWFLAKE_WAREHOUSE="COMPUTE_WH"
export SNOWFLAKE_DATABASE="AIRBNB_ANALYTICS"
export SNOWFLAKE_ROLE="ACCOUNTADMIN"
```

Update `profiles.yml`:

```yaml
snowflake_dbt_airbnb:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: "{{ env_var('SNOWFLAKE_ROLE') }}"
      warehouse: "{{ env_var('SNOWFLAKE_WAREHOUSE') }}"
      database: "{{ env_var('SNOWFLAKE_DATABASE') }}"
      schema: ANALYTICS
      threads: 4
```

Update loader script:

```python
import os
import snowflake.connector

conn = snowflake.connector.connect(
    account=os.getenv('SNOWFLAKE_ACCOUNT'),
    user=os.getenv('SNOWFLAKE_USER'),
    password=os.getenv('SNOWFLAKE_PASSWORD'),
    warehouse=os.getenv('SNOWFLAKE_WAREHOUSE'),
    database=os.getenv('SNOWFLAKE_DATABASE'),
    role=os.getenv('SNOWFLAKE_ROLE')
)
```

Update Streamlit app:

```python
import os
import snowflake.connector

conn = snowflake.connector.connect(
    account=os.getenv('SNOWFLAKE_ACCOUNT'),
    user=os.getenv('SNOWFLAKE_USER'),
    password=os.getenv('SNOWFLAKE_PASSWORD'),
    warehouse=os.getenv('SNOWFLAKE_WAREHOUSE'),
    database=os.getenv('SNOWFLAKE_DATABASE'),
    schema='ANALYTICS',
    role=os.getenv('SNOWFLAKE_ROLE')
)
```

## Best Practices for AI Agents

1. **Always use local credential files or env vars** — never commit real Snowflake credentials
2. **Run `dbt debug` first** to validate connection before building models
3. **Use `--profiles-dir .`** to reference local `profiles.yml` in project root
4. **Full refresh incremental models** after reloading raw data
5. **Add tests when creating new models** — data quality is critical
6. **Document models in schema.yml** — explain business logic for maintainability
7. **Use dbt ref() for dependencies** — never hardcode table names
8. **Test Streamlit queries separately** before adding to dashboard
9. **Keep staging layer simple** — clean, cast, rename only
10. **Put business logic in intermediate layer** — staging should be reusable

This skill enables AI agents to guide developers through complete analytics engineering workflows using Snowflake, dbt, and Streamlit with production-ready patterns.
