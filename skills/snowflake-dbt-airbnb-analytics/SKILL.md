---
name: snowflake-dbt-airbnb-analytics
description: Build end-to-end analytics pipelines with Snowflake, dbt, and Streamlit using Inside Airbnb data
triggers:
  - set up airbnb analytics pipeline with dbt and snowflake
  - load inside airbnb data into snowflake
  - create dbt models for airbnb listings and calendar data
  - build incremental fact tables in snowflake with dbt
  - configure dbt staging and mart layers for analytics
  - run dbt tests for data quality validation
  - deploy streamlit dashboard with snowflake backend
  - troubleshoot dbt snowflake connection issues
---

# Snowflake dbt Airbnb Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill covers the **Snowflake_DBT_Project** by analyticsdurgesh, a complete analytics engineering pipeline that loads Inside Airbnb open data into Snowflake, transforms it through dbt staging/intermediate/mart layers, validates data quality with tests, and powers a Streamlit dashboard.

## What This Project Does

- **Raw data ingestion**: Loads Inside Airbnb CSV/GZIP files (listings, calendar, reviews, neighbourhoods) into Snowflake internal stages
- **dbt transformation pipeline**: Builds staging views → intermediate enrichment → dimension/fact marts → monthly aggregates
- **Incremental fact modeling**: Uses Snowflake merge strategy for `fct_listing_calendar` to efficiently update calendar facts
- **Data quality**: Generic and singular dbt tests validate uniqueness, relationships, accepted values, and business rules
- **Analytics dashboard**: Streamlit app queries marts for neighbourhood, host, and listing performance metrics
- **Public-safe credentials**: All secrets in local-only files ignored by git

## Installation

### 1. Clone and Set Up Environment

```bash
git clone https://github.com/analyticsdurgesh/Snowflake_DBT_Project.git
cd Snowflake_DBT_Project
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Download Inside Airbnb Data

Visit [Inside Airbnb](https://insideairbnb.com/get-the-data/) and download the New York City dataset files:

```bash
mkdir -p data/raw
# Download and place these files:
# data/raw/listings.csv.gz
# data/raw/calendar.csv.gz
# data/raw/reviews.csv.gz
# data/raw/neighbourhoods.csv
```

### 3. Configure Snowflake Credentials

Create local credential files from examples:

```bash
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

Edit `profiles.yml`:

```yaml
airbnb_dbt:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT  # e.g., abc12345.us-east-1
      user: YOUR_USERNAME
      password: YOUR_PASSWORD
      role: YOUR_ROLE
      warehouse: YOUR_WAREHOUSE
      database: AIRBNB_DB
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

Edit `config/local_credentials.json`:

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

**Security**: These files are gitignored. Never commit real credentials.

## Loading Raw Data into Snowflake

The Python loader script creates Snowflake objects, uploads files to internal stage, and copies data into raw tables:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

### What the Loader Does

1. Reads credentials from `config/local_credentials.json`
2. Executes `setup/snowflake_setup.sql` to create database, schemas, stage
3. Uploads `data/raw/*.csv.gz` and `*.csv` to `INSIDE_AIRBNB_STAGE`
4. Creates raw tables with `INFER_SCHEMA` from CSV headers
5. Copies stage data into `RAW.LISTINGS`, `RAW.CALENDAR`, `RAW.REVIEWS`, `RAW.NEIGHBOURHOODS`

### Snowflake Setup SQL

The loader executes this automatically:

```sql
CREATE DATABASE IF NOT EXISTS AIRBNB_DB;
USE DATABASE AIRBNB_DB;

CREATE SCHEMA IF NOT EXISTS RAW;
CREATE SCHEMA IF NOT EXISTS ANALYTICS;

CREATE OR REPLACE STAGE INSIDE_AIRBNB_STAGE
  FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"');
```

## dbt Model Layers

### Project Structure

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

### Staging Layer

Cleans, casts, and standardizes raw data:

**`models/staging/stg_airbnb__listings.sql`**:

```sql
WITH source AS (
    SELECT * FROM {{ source('raw_airbnb', 'listings') }}
),

cleaned AS (
    SELECT
        TRY_CAST(id AS INTEGER) AS listing_id,
        TRY_CAST(host_id AS INTEGER) AS host_id,
        name AS listing_name,
        neighbourhood_cleansed AS neighbourhood,
        room_type,
        TRY_CAST(price AS FLOAT) AS price,
        TRY_CAST(minimum_nights AS INTEGER) AS minimum_nights,
        TRY_CAST(number_of_reviews AS INTEGER) AS number_of_reviews,
        last_review::DATE AS last_review_date,
        TRY_CAST(reviews_per_month AS FLOAT) AS reviews_per_month,
        TRY_CAST(availability_365 AS INTEGER) AS availability_365
    FROM source
)

SELECT * FROM cleaned
WHERE listing_id IS NOT NULL
```

**`models/staging/_sources.yml`**:

```yaml
version: 2

sources:
  - name: raw_airbnb
    database: AIRBNB_DB
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
      - name: neighbourhoods
```

### Intermediate Layer

Joins and enriches staging models:

**`models/intermediate/int_airbnb__calendar_enriched.sql`**:

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
        c.available,
        c.price,
        c.adjusted_price,
        c.minimum_nights,
        c.maximum_nights,
        l.listing_name,
        l.neighbourhood,
        l.room_type,
        l.host_id,
        l.host_name,
        -- Revenue proxy: if unavailable, assume booked
        CASE 
            WHEN c.available = FALSE THEN COALESCE(c.adjusted_price, c.price, 0)
            ELSE 0
        END AS estimated_revenue
    FROM calendar c
    LEFT JOIN listings l
        ON c.listing_id = l.listing_id
)

SELECT * FROM enriched
```

### Marts Layer

Analytics-ready dimensions and facts:

**`models/marts/dim_listings.sql`**:

```sql
SELECT
    listing_id,
    listing_name,
    neighbourhood,
    room_type,
    host_id,
    host_name,
    price,
    minimum_nights,
    number_of_reviews,
    last_review_date,
    reviews_per_month,
    availability_365
FROM {{ ref('int_airbnb__listing_enriched') }}
```

**`models/marts/fct_listing_calendar.sql`** (incremental):

```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        merge_update_columns=['available', 'price', 'adjusted_price', 'estimated_revenue']
    )
}}

SELECT
    listing_id,
    calendar_date,
    available,
    price,
    adjusted_price,
    minimum_nights,
    maximum_nights,
    neighbourhood,
    room_type,
    estimated_revenue
FROM {{ ref('int_airbnb__calendar_enriched') }}

{% if is_incremental() %}
    WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
{% endif %}
```

**`models/marts/agg_neighbourhood_monthly_performance.sql`**:

```sql
WITH listing_monthly AS (
    SELECT * FROM {{ ref('agg_listing_monthly_performance') }}
),

neighbourhood_agg AS (
    SELECT
        neighbourhood,
        month_start,
        COUNT(DISTINCT listing_id) AS total_listings,
        SUM(total_days) AS total_days,
        SUM(available_days) AS total_available_days,
        SUM(unavailable_days) AS total_unavailable_days,
        SUM(estimated_revenue) AS total_estimated_revenue,
        AVG(avg_price) AS avg_price_per_listing,
        AVG(availability_rate) AS avg_availability_rate
    FROM listing_monthly
    GROUP BY neighbourhood, month_start
)

SELECT
    neighbourhood,
    month_start,
    total_listings,
    total_days,
    total_available_days,
    total_unavailable_days,
    total_estimated_revenue,
    ROUND(avg_price_per_listing, 2) AS avg_price_per_listing,
    ROUND(avg_availability_rate * 100, 2) AS avg_availability_rate_pct
FROM neighbourhood_agg
ORDER BY neighbourhood, month_start
```

## dbt Commands

### Initial Run

```bash
# Verify connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Generate docs
dbt docs generate --profiles-dir .

# Serve docs locally
dbt docs serve --profiles-dir .
```

### Run Specific Models

```bash
# Run only staging models
dbt run --select staging.* --profiles-dir .

# Run marts and downstream dependencies
dbt run --select marts.* --profiles-dir .

# Run specific model and its children
dbt run --select dim_listings+ --profiles-dir .

# Run incremental model with full refresh
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

### Test Specific Models

```bash
# Test only sources
dbt test --select source:* --profiles-dir .

# Test specific model
dbt test --select dim_listings --profiles-dir .
```

### Refresh Data Pipeline

When loading new Inside Airbnb snapshot:

```bash
# 1. Load new raw data
python scripts/load_inside_airbnb_to_snowflake.py

# 2. Full refresh incremental fact and downstream
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 3. Run all tests
dbt test --profiles-dir .
```

## Data Quality Tests

### Generic Tests in Schema Files

**`models/marts/_schema.yml`**:

```yaml
version: 2

models:
  - name: dim_listings
    description: Dimension table for Airbnb listings
    columns:
      - name: listing_id
        description: Unique listing identifier
        tests:
          - not_null
          - unique
      - name: room_type
        description: Type of room offered
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
    description: Fact table for listing calendar availability and revenue
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

**`tests/no_duplicate_listing_dates.sql`**:

```sql
SELECT
    listing_id,
    calendar_date,
    COUNT(*) AS duplicate_count
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, calendar_date
HAVING COUNT(*) > 1
```

**`tests/revenue_non_negative.sql`**:

```sql
SELECT *
FROM {{ ref('fct_listing_calendar') }}
WHERE estimated_revenue < 0
```

## Streamlit Dashboard

### Configuration

The dashboard reads Snowflake credentials from `config/local_credentials.json`:

**`dashboard/streamlit_app.py`**:

```python
import streamlit as st
import pandas as pd
import snowflake.connector
import json
from pathlib import Path

# Load credentials
creds_path = Path(__file__).parent.parent / 'config' / 'local_credentials.json'
with open(creds_path) as f:
    creds = json.load(f)

# Connect to Snowflake
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

conn = get_snowflake_connection()

# Query marts
@st.cache_data(ttl=600)
def load_neighbourhood_performance():
    query = """
    SELECT
        neighbourhood,
        month_start,
        total_listings,
        total_estimated_revenue,
        avg_price_per_listing,
        avg_availability_rate_pct
    FROM agg_neighbourhood_monthly_performance
    ORDER BY month_start DESC, total_estimated_revenue DESC
    """
    return pd.read_sql(query, conn)

st.title("🏠 Inside Airbnb Analytics Dashboard")

# Neighbourhood performance
st.header("Neighbourhood Monthly Performance")
df = load_neighbourhood_performance()
st.dataframe(df, use_container_width=True)

# Top neighbourhoods by revenue
st.subheader("Top Neighbourhoods by Estimated Revenue")
top_neighbourhoods = df.groupby('neighbourhood')['total_estimated_revenue'].sum().sort_values(ascending=False).head(10)
st.bar_chart(top_neighbourhoods)
```

### Run Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

Access at `http://localhost:8501`

## Common Patterns

### Adding a New Staging Model

1. Define source in `models/staging/_sources.yml`
2. Create staging SQL file with standardized column names and types
3. Add tests in schema YAML

**Example new staging model**:

```sql
-- models/staging/stg_airbnb__hosts.sql
WITH source AS (
    SELECT * FROM {{ source('raw_airbnb', 'hosts') }}
),

cleaned AS (
    SELECT
        TRY_CAST(host_id AS INTEGER) AS host_id,
        host_name,
        host_since::DATE AS host_since_date,
        TRY_CAST(host_listings_count AS INTEGER) AS host_listings_count,
        host_verifications
    FROM source
)

SELECT * FROM cleaned
WHERE host_id IS NOT NULL
```

### Creating a Custom dbt Test

**`tests/generic/test_positive_value.sql`**:

```sql
{% test positive_value(model, column_name) %}

SELECT *
FROM {{ model }}
WHERE {{ column_name }} IS NOT NULL
  AND {{ column_name }} <= 0

{% endtest %}
```

**Use in schema**:

```yaml
columns:
  - name: price
    tests:
      - positive_value
```

### Debugging dbt Models

```bash
# Compile model to see generated SQL
dbt compile --select dim_listings --profiles-dir .
cat target/compiled/airbnb_dbt/models/marts/dim_listings.sql

# Run single model with logging
dbt run --select dim_listings --profiles-dir . --log-level debug

# Show query plan
dbt show --select dim_listings --profiles-dir .
```

### Environment-Specific Targets

Modify `profiles.yml` for multiple environments:

```yaml
airbnb_dbt:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      warehouse: COMPUTE_WH
      database: AIRBNB_DEV_DB
      schema: ANALYTICS
    prod:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      warehouse: COMPUTE_WH_PROD
      database: AIRBNB_PROD_DB
      schema: ANALYTICS
```

Run against prod:

```bash
export SNOWFLAKE_ACCOUNT=your_account
export SNOWFLAKE_USER=your_user
export SNOWFLAKE_PASSWORD=your_password

dbt run --target prod --profiles-dir .
```

## Troubleshooting

### Connection Issues

**Problem**: `dbt debug` fails with authentication error

**Solution**:
- Verify credentials in `profiles.yml`
- Check Snowflake account identifier format: `<account_locator>.<region>` or `<org_name>-<account_name>`
- Test connection directly:

```python
import snowflake.connector
conn = snowflake.connector.connect(
    account='YOUR_ACCOUNT',
    user='YOUR_USER',
    password='YOUR_PASSWORD'
)
print(conn.cursor().execute("SELECT CURRENT_USER()").fetchone())
```

### Loader Script Errors

**Problem**: `load_inside_airbnb_to_snowflake.py` fails with "Stage not found"

**Solution**:
- Ensure `setup/snowflake_setup.sql` creates stage successfully
- Verify database and schema exist:

```sql
SHOW DATABASES LIKE 'AIRBNB_DB';
SHOW SCHEMAS IN DATABASE AIRBNB_DB;
SHOW STAGES IN SCHEMA RAW;
```

### Incremental Model Not Updating

**Problem**: `fct_listing_calendar` doesn't pick up new calendar dates

**Solution**:
- Check incremental logic condition
- Force full refresh:

```bash
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

- Verify `unique_key` matches grain:

```sql
SELECT listing_id, calendar_date, COUNT(*)
FROM {{ ref('int_airbnb__calendar_enriched') }}
GROUP BY 1, 2
HAVING COUNT(*) > 1
```

### Test Failures

**Problem**: `relationships` test fails on `fct_listing_calendar`

**Solution**:
- Check orphaned records:

```sql
SELECT DISTINCT fc.listing_id
FROM ANALYTICS.fct_listing_calendar fc
LEFT JOIN ANALYTICS.dim_listings dl ON fc.listing_id = dl.listing_id
WHERE dl.listing_id IS NULL
```

- Fix data in staging or add filter to intermediate model

### Dashboard Connection Timeout

**Problem**: Streamlit dashboard fails to connect to Snowflake

**Solution**:
- Verify `config/local_credentials.json` exists and has correct values
- Check network/firewall allows Snowflake connection
- Enable client session keep-alive in `profiles.yml`:

```yaml
client_session_keep_alive: True
```

### Missing Raw Files

**Problem**: Loader script can't find `data/raw/listings.csv.gz`

**Solution**:
- Download files from [Inside Airbnb](https://insideairbnb.com/get-the-data/)
- Ensure exact filenames:
  - `listings.csv.gz`
  - `calendar.csv.gz`
  - `reviews.csv.gz`
  - `neighbourhoods.csv`

### Warehouse Suspended

**Problem**: Queries timeout or fail with "warehouse not available"

**Solution**:
- Resume warehouse in Snowflake UI or SQL:

```sql
ALTER WAREHOUSE COMPUTE_WH RESUME;
```

- Set auto-resume in warehouse config

## Advanced Usage

### Custom Macros

**`macros/calculate_revenue_proxy.sql`**:

```sql
{% macro calculate_revenue_proxy(available_col, price_col, adjusted_price_col) %}
    CASE 
        WHEN {{ available_col }} = FALSE 
        THEN COALESCE({{ adjusted_price_col }}, {{ price_col }}, 0)
        ELSE 0
    END
{% endmacro %}
```

**Use in model**:

```sql
SELECT
    listing_id,
    calendar_date,
    {{ calculate_revenue_proxy('available', 'price', 'adjusted_price') }} AS estimated_revenue
FROM {{ ref('stg_airbnb__calendar') }}
```

### dbt Exposures

Document dashboard in dbt:

**`models/marts/_exposures.yml`**:

```yaml
version: 2

exposures:
  - name: streamlit_airbnb_dashboard
    type: dashboard
    maturity: medium
    owner:
      name: Analytics Team
      email: analytics@example.com
    description: Streamlit dashboard showing neighbourhood and listing performance
    depends_on:
      - ref('dim_listings')
      - ref('dim_hosts')
      - ref('agg_neighbourhood_monthly_performance')
    url: http://localhost:8501
```

### Pre-commit Hooks

**`.pre-commit-config.yaml`**:

```yaml
repos:
  - repo: https://github.com/dbt-checkpoint/dbt-checkpoint
    rev: v1.1.0
    hooks:
      - id: check-model-has-tests
      - id: check-model-has-description
      - id: dbt-compile
```

Install hooks:

```bash
pip install pre-commit
pre-commit install
```
