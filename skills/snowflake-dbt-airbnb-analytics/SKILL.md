---
name: snowflake-dbt-airbnb-analytics
description: Snowflake and dbt analytics engineering project for Inside Airbnb data with staging, marts, tests, and Streamlit dashboard
triggers:
  - set up snowflake dbt airbnb project
  - load inside airbnb data to snowflake
  - run dbt models for airbnb analytics
  - create dbt staging and mart layers
  - build incremental dbt fact tables
  - configure snowflake credentials for dbt
  - deploy streamlit dashboard for airbnb data
  - test dbt data quality with airbnb dataset
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to work with the Snowflake_DBT_Project, a complete analytics engineering portfolio that ingests Inside Airbnb open data into Snowflake, transforms it with dbt across staging/intermediate/mart layers, validates quality with tests, and powers a Streamlit dashboard.

## What This Project Does

- **Raw data ingestion**: Python loader uploads CSV/GZIP files to Snowflake internal stage
- **dbt transformations**: Staging views → intermediate enrichment → dimensional/fact marts
- **Incremental modeling**: `fct_listing_calendar` uses Snowflake merge strategy
- **Data quality**: Generic and singular dbt tests for nulls, uniqueness, relationships, accepted values
- **Analytics dashboard**: Streamlit app for neighbourhood, host, and listing metrics
- **Public-safe config**: Credentials in git-ignored local files, no environment variables required

## Repository Structure

```
.
├── config/
│   └── local_credentials.example.json
├── dashboard/
│   └── streamlit_app.py
├── data/raw/
│   ├── listings.csv.gz
│   ├── calendar.csv.gz
│   ├── reviews.csv.gz
│   └── neighbourhoods.csv
├── models/
│   ├── staging/
│   │   ├── stg_airbnb__listings.sql
│   │   ├── stg_airbnb__calendar.sql
│   │   ├── stg_airbnb__reviews.sql
│   │   └── stg_airbnb__neighbourhoods.sql
│   ├── intermediate/
│   │   ├── int_airbnb__listing_enriched.sql
│   │   ├── int_airbnb__calendar_enriched.sql
│   │   └── int_airbnb__reviews_enriched.sql
│   └── marts/
│       ├── dim_listings.sql
│       ├── dim_hosts.sql
│       ├── fct_listing_calendar.sql
│       ├── fct_reviews.sql
│       ├── agg_listing_monthly_performance.sql
│       └── agg_neighbourhood_monthly_performance.sql
├── scripts/
│   └── load_inside_airbnb_to_snowflake.py
├── setup/
│   └── snowflake_setup.sql
├── tests/
├── dbt_project.yml
├── profiles.yml.example
└── requirements.txt
```

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

Create local credential files from examples:

```bash
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

Edit `profiles.yml`:

```yaml
snowflake_dbt_airbnb:
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

Edit `config/local_credentials.json`:

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

**Note**: These files are git-ignored. Never commit real credentials.

### 3. Download Inside Airbnb Data

Download New York City data from [Inside Airbnb](https://insideairbnb.com/get-the-data/) and place in `data/raw/`:

- `listings.csv.gz`
- `calendar.csv.gz`
- `reviews.csv.gz`
- `neighbourhoods.csv`

## Loading Raw Data to Snowflake

The Python loader script creates Snowflake objects, uploads files to internal stage, and copies data into RAW schema:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What it does**:
1. Executes `setup/snowflake_setup.sql` to create database, schemas, stage, warehouse
2. Uploads local files to `INSIDE_AIRBNB_STAGE`
3. Creates raw tables with `COPY INTO` from stage

### Key Script Logic

```python
import snowflake.connector
import json

# Load credentials
with open('config/local_credentials.json') as f:
    creds = json.load(f)

# Connect
conn = snowflake.connector.connect(
    account=creds['account'],
    user=creds['user'],
    password=creds['password'],
    warehouse=creds['warehouse'],
    database=creds['database']
)

# Execute setup
with open('setup/snowflake_setup.sql') as f:
    for statement in f.read().split(';'):
        if statement.strip():
            conn.cursor().execute(statement)

# Upload files
conn.cursor().execute("USE SCHEMA RAW")
conn.cursor().execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")
conn.cursor().execute("PUT file://data/raw/calendar.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")
conn.cursor().execute("PUT file://data/raw/reviews.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")
conn.cursor().execute("PUT file://data/raw/neighbourhoods.csv @INSIDE_AIRBNB_STAGE")

# Copy into tables
conn.cursor().execute("""
COPY INTO RAW.LISTINGS
FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
""")
```

## dbt Model Layers

### Staging Layer (`models/staging/`)

Clean, cast, and standardize raw data:

```sql
-- stg_airbnb__listings.sql
WITH source AS (
    SELECT * FROM {{ source('raw_airbnb', 'listings') }}
)

SELECT
    id::INTEGER AS listing_id,
    name::VARCHAR AS listing_name,
    host_id::INTEGER AS host_id,
    host_name::VARCHAR AS host_name,
    neighbourhood_cleansed::VARCHAR AS neighbourhood,
    room_type::VARCHAR AS room_type,
    price::VARCHAR AS price_raw,
    CAST(REPLACE(REPLACE(price, '$', ''), ',', '') AS DECIMAL(10,2)) AS price,
    minimum_nights::INTEGER AS minimum_nights,
    number_of_reviews::INTEGER AS number_of_reviews,
    last_review::DATE AS last_review_date,
    reviews_per_month::DECIMAL(10,2) AS reviews_per_month,
    availability_365::INTEGER AS availability_365
FROM source
```

### Intermediate Layer (`models/intermediate/`)

Join, enrich, and add business logic:

```sql
-- int_airbnb__calendar_enriched.sql
WITH calendar AS (
    SELECT * FROM {{ ref('stg_airbnb__calendar') }}
),

listings AS (
    SELECT * FROM {{ ref('int_airbnb__listing_enriched') }}
)

SELECT
    c.listing_id,
    c.calendar_date,
    c.available,
    c.price,
    l.neighbourhood,
    l.room_type,
    l.host_id,
    CASE 
        WHEN c.available = 'f' THEN c.price
        ELSE 0
    END AS estimated_revenue,
    DATE_TRUNC('month', c.calendar_date) AS month_date
FROM calendar c
LEFT JOIN listings l ON c.listing_id = l.listing_id
```

### Marts Layer (`models/marts/`)

Analytics-ready dimensions and facts:

**Dimension Example**:

```sql
-- dim_listings.sql
{{ config(materialized='table') }}

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
    reviews_per_month,
    availability_365
FROM {{ ref('int_airbnb__listing_enriched') }}
```

**Incremental Fact Example**:

```sql
-- fct_listing_calendar.sql
{{ config(
    materialized='incremental',
    unique_key='calendar_key',
    merge_update_columns=['available', 'price', 'estimated_revenue']
) }}

SELECT
    {{ dbt_utils.generate_surrogate_key(['listing_id', 'calendar_date']) }} AS calendar_key,
    listing_id,
    calendar_date,
    available,
    price,
    neighbourhood,
    room_type,
    host_id,
    estimated_revenue,
    month_date
FROM {{ ref('int_airbnb__calendar_enriched') }}

{% if is_incremental() %}
WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
{% endif %}
```

**Aggregate Example**:

```sql
-- agg_neighbourhood_monthly_performance.sql
SELECT
    neighbourhood,
    month_date,
    COUNT(DISTINCT listing_id) AS total_listings,
    SUM(estimated_revenue) AS total_estimated_revenue,
    AVG(price) AS avg_price,
    SUM(CASE WHEN available = 'f' THEN 1 ELSE 0 END) AS unavailable_nights,
    COUNT(*) AS total_nights,
    ROUND(unavailable_nights / total_nights * 100, 2) AS occupancy_rate
FROM {{ ref('fct_listing_calendar') }}
GROUP BY neighbourhood, month_date
```

## Running dbt Commands

### Standard Workflow

```bash
# Verify connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select dim_listings+ --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .

# Serve documentation
dbt docs serve --profiles-dir .
```

### Incremental Model Refresh

When reloading new Inside Airbnb snapshot:

```bash
# Reload raw data
python scripts/load_inside_airbnb_to_snowflake.py

# Full refresh incremental fact table and downstream
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Test data quality
dbt test --profiles-dir .
```

### Useful Selectors

```bash
# Run only staging models
dbt run --select staging.* --profiles-dir .

# Run only marts
dbt run --select marts.* --profiles-dir .

# Run one model and all parents
dbt run --select +agg_neighbourhood_monthly_performance --profiles-dir .

# Test one model
dbt test --select dim_listings --profiles-dir .
```

## Data Quality Tests

### Generic Tests in Schema Files

```yaml
# models/staging/schema.yml
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
  - name: stg_airbnb__listings
    columns:
      - name: listing_id
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
```

### Singular Tests

```sql
-- tests/assert_no_duplicate_listing_dates.sql
SELECT
    listing_id,
    calendar_date,
    COUNT(*) AS duplicate_count
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, calendar_date
HAVING COUNT(*) > 1
```

### Relationship Tests

```yaml
# models/marts/schema.yml
models:
  - name: fct_listing_calendar
    columns:
      - name: listing_id
        tests:
          - relationships:
              to: ref('dim_listings')
              field: listing_id
```

## Streamlit Dashboard

The dashboard connects to Snowflake marts for interactive analytics:

```python
# dashboard/streamlit_app.py
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load credentials
with open('config/local_credentials.json') as f:
    creds = json.load(f)

# Connect to Snowflake
@st.cache_resource
def get_connection():
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema']
    )

conn = get_connection()

# Query neighbourhood performance
@st.cache_data
def load_neighbourhood_performance():
    query = """
    SELECT
        neighbourhood,
        SUM(total_estimated_revenue) AS revenue,
        AVG(avg_price) AS avg_price,
        AVG(occupancy_rate) AS avg_occupancy
    FROM agg_neighbourhood_monthly_performance
    GROUP BY neighbourhood
    ORDER BY revenue DESC
    LIMIT 10
    """
    return pd.read_sql(query, conn)

st.title("Airbnb Analytics Dashboard")
st.subheader("Top 10 Neighbourhoods by Revenue")

df = load_neighbourhood_performance()
st.bar_chart(df.set_index('neighbourhood')['revenue'])

# Display metrics
col1, col2, col3 = st.columns(3)
col1.metric("Total Neighbourhoods", len(df))
col2.metric("Avg Price", f"${df['avg_price'].mean():.2f}")
col3.metric("Avg Occupancy", f"{df['avg_occupancy'].mean():.1f}%")
```

### Run Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

## Common Patterns

### Adding a New Staging Model

1. Create SQL file in `models/staging/`:

```sql
-- models/staging/stg_airbnb__new_source.sql
WITH source AS (
    SELECT * FROM {{ source('raw_airbnb', 'new_table') }}
)

SELECT
    id::INTEGER AS new_id,
    name::VARCHAR AS new_name
FROM source
```

2. Add to `models/staging/schema.yml`:

```yaml
models:
  - name: stg_airbnb__new_source
    columns:
      - name: new_id
        tests:
          - unique
          - not_null
```

3. Run:

```bash
dbt run --select stg_airbnb__new_source --profiles-dir .
```

### Creating a New Mart

```sql
-- models/marts/dim_new_dimension.sql
{{ config(materialized='table') }}

SELECT
    new_id,
    new_name,
    derived_field
FROM {{ ref('int_airbnb__new_enriched') }}
```

### Adding Custom Macro

```sql
-- macros/generate_date_spine.sql
{% macro generate_date_spine(start_date, end_date) %}
SELECT
    DATEADD(day, SEQ4(), '{{ start_date }}') AS date_day
FROM TABLE(GENERATOR(ROWCOUNT => DATEDIFF(day, '{{ start_date }}', '{{ end_date }}')))
{% endmacro %}
```

Use in model:

```sql
WITH date_spine AS (
    {{ generate_date_spine('2020-01-01', '2024-12-31') }}
)
SELECT * FROM date_spine
```

## Troubleshooting

### dbt Connection Issues

**Problem**: `dbt debug` fails with authentication error

**Solution**: Verify `profiles.yml` credentials match Snowflake account. Ensure role has access to database/warehouse.

```bash
# Test Snowflake connection directly
python -c "
import snowflake.connector
conn = snowflake.connector.connect(
    account='YOUR_ACCOUNT',
    user='YOUR_USER',
    password='YOUR_PASSWORD'
)
print('Connected:', conn.cursor().execute('SELECT CURRENT_VERSION()').fetchone())
"
```

### Incremental Model Not Updating

**Problem**: `fct_listing_calendar` doesn't pick up new data

**Solution**: Force full refresh:

```bash
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

Or check incremental logic:

```sql
{% if is_incremental() %}
WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
{% endif %}
```

### Stage File Upload Fails

**Problem**: `PUT` command times out

**Solution**: Check file paths and Snowflake stage permissions:

```sql
-- Verify stage exists
SHOW STAGES LIKE 'INSIDE_AIRBNB_STAGE';

-- List files in stage
LIST @INSIDE_AIRBNB_STAGE;

-- Grant permissions
GRANT USAGE ON STAGE INSIDE_AIRBNB_STAGE TO ROLE YOUR_ROLE;
```

### Test Failures

**Problem**: `dbt test` reports relationship test failures

**Solution**: Check referential integrity:

```sql
-- Find orphaned records
SELECT DISTINCT listing_id
FROM {{ ref('fct_listing_calendar') }}
WHERE listing_id NOT IN (SELECT listing_id FROM {{ ref('dim_listings') }})
```

Fix by adding inner join in intermediate model or filtering nulls.

### Streamlit Dashboard Errors

**Problem**: `snowflake.connector.errors.ProgrammingError: Object does not exist`

**Solution**: Verify mart tables exist and credentials have access:

```bash
# Run dbt models first
dbt run --profiles-dir .

# Check table exists
python -c "
import snowflake.connector
import json
with open('config/local_credentials.json') as f:
    creds = json.load(f)
conn = snowflake.connector.connect(**creds)
result = conn.cursor().execute('SHOW TABLES IN ANALYTICS').fetchall()
print([r[1] for r in result])
"
```

## Advanced Usage

### Custom dbt Variables

```bash
# Pass variable at runtime
dbt run --vars '{"min_price": 100}' --profiles-dir .
```

Use in model:

```sql
WHERE price >= {{ var('min_price', 0) }}
```

### dbt Cloud Alternative

For production, configure `profiles.yml` to use OAuth or key-pair authentication:

```yaml
snowflake_dbt_airbnb:
  target: prod
  outputs:
    prod:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      authenticator: externalbrowser  # Or use private_key_path
      database: AIRBNB_DB
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      role: ANALYTICS_ROLE
```

### Scheduling with Airflow

```python
# Example DAG (not in repo)
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime

with DAG('airbnb_dbt_pipeline', start_date=datetime(2024, 1, 1), schedule_interval='@daily') as dag:
    load_raw = BashOperator(
        task_id='load_raw_data',
        bash_command='python /path/to/scripts/load_inside_airbnb_to_snowflake.py'
    )
    
    run_dbt = BashOperator(
        task_id='run_dbt_models',
        bash_command='dbt run --profiles-dir /path/to/project'
    )
    
    test_dbt = BashOperator(
        task_id='test_dbt_models',
        bash_command='dbt test --profiles-dir /path/to/project'
    )
    
    load_raw >> run_dbt >> test_dbt
```

## Key Files Reference

| File | Purpose |
|------|---------|
| `dbt_project.yml` | dbt project configuration, model paths, materializations |
| `profiles.yml` | Snowflake connection settings (git-ignored) |
| `config/local_credentials.json` | Streamlit/Python Snowflake credentials (git-ignored) |
| `setup/snowflake_setup.sql` | Database, schema, stage, warehouse DDL |
| `scripts/load_inside_airbnb_to_snowflake.py` | Raw data loader to Snowflake |
| `models/staging/schema.yml` | Source and staging model tests/docs |
| `models/marts/schema.yml` | Mart model tests/docs |

## Example Business Queries

### Top 10 Hosts by Listings

```sql
SELECT
    host_id,
    host_name,
    COUNT(*) AS listing_count,
    AVG(price) AS avg_price
FROM dim_listings
GROUP BY host_id, host_name
ORDER BY listing_count DESC
LIMIT 10
```

### Neighbourhood Monthly Revenue Trend

```sql
SELECT
    neighbourhood,
    month_date,
    total_estimated_revenue,
    occupancy_rate
FROM agg_neighbourhood_monthly_performance
WHERE neighbourhood = 'Manhattan'
ORDER BY month_date
```

### Room Type Performance

```sql
SELECT
    room_type,
    COUNT(DISTINCT listing_id) AS listings,
    AVG(price) AS avg_price,
    SUM(estimated_revenue) AS total_revenue
FROM fct_listing_calendar
GROUP BY room_type
ORDER BY total_revenue DESC
```

This skill provides comprehensive guidance for AI agents to help developers set up, configure, transform, test, and visualize Inside Airbnb data using Snowflake, dbt, and Streamlit in a production-quality analytics engineering workflow.
