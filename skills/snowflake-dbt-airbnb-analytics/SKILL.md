```markdown
---
name: snowflake-dbt-airbnb-analytics
description: Analytics engineering with dbt, Snowflake, and Inside Airbnb data for dimensional modeling and incremental fact tables
triggers:
  - load inside airbnb data into snowflake
  - build dbt models for airbnb analytics
  - create incremental fact tables with dbt
  - set up snowflake dbt project
  - run airbnb analytics pipeline
  - configure dbt staging and marts
  - build streamlit dashboard for snowflake
  - test dbt data quality
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables agents to work with a complete analytics engineering project that loads Inside Airbnb data into Snowflake, transforms it using dbt (staging → intermediate → marts), builds incremental fact tables, validates data quality with tests, and powers a Streamlit dashboard.

## What This Project Does

- **Data ingestion**: Loads Inside Airbnb CSV/GZIP files (listings, calendar, reviews, neighbourhoods) into Snowflake via internal stages
- **dbt transformation**: Multi-layer transformation (staging → intermediate → marts) with dimensional and fact tables
- **Incremental modeling**: Uses Snowflake merge strategy for efficient fact table updates
- **Data quality**: Generic and singular dbt tests for validation
- **Reporting**: Streamlit dashboard connected to analytics marts

**Key models:**
- Staging: `stg_airbnb__listings`, `stg_airbnb__calendar`, `stg_airbnb__reviews`, `stg_airbnb__neighbourhoods`
- Intermediate: `int_airbnb__listing_enriched`, `int_airbnb__calendar_enriched`, `int_airbnb__reviews_enriched`
- Marts: `dim_listings`, `dim_hosts`, `fct_listing_calendar` (incremental), `fct_reviews`
- Aggregates: `agg_listing_monthly_performance`, `agg_neighbourhood_monthly_performance`

## Installation

```bash
# Clone repository
git clone https://github.com/analyticsdurgesh/Snowflake_DBT_Project.git
cd Snowflake_DBT_Project

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

**Expected dependencies:**
- `snowflake-connector-python`
- `dbt-snowflake`
- `streamlit`
- `pandas`

## Configuration

### 1. Snowflake Credentials (dbt)

Create `profiles.yml` from the example (this file is git-ignored):

```bash
cp profiles.yml.example profiles.yml
```

Edit `profiles.yml` with your Snowflake credentials:

```yaml
snowflake_dbt_airbnb:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT
      user: YOUR_USERNAME
      password: YOUR_PASSWORD  # Or use authenticator: externalbrowser
      role: YOUR_ROLE
      database: AIRBNB_DB
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

**Environment variable alternative:**

```yaml
password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
```

### 2. Loader Script Credentials

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
  "role": "YOUR_ROLE",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DB",
  "schema": "RAW"
}
```

### 3. Raw Data Files

Place Inside Airbnb files in `data/raw/`:

```text
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

Download from [Inside Airbnb](https://insideairbnb.com/get-the-data/) (recommended: New York City dataset).

## Key Commands

### Load Raw Data into Snowflake

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What this does:**
1. Reads credentials from `config/local_credentials.json`
2. Executes setup SQL from `setup/snowflake_setup.sql` (creates database, schemas, stage)
3. Uploads local files to Snowflake internal stage `INSIDE_AIRBNB_STAGE`
4. Creates raw tables from CSV headers
5. Copies data into `RAW` schema tables

### dbt Commands

```bash
# Verify connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select fct_listing_calendar+ --profiles-dir .

# Full refresh (rebuild incremental models from scratch)
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Test specific model
dbt test --select dim_listings --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .

# Serve documentation
dbt docs serve --profiles-dir .
```

### Streamlit Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

The dashboard reads Snowflake credentials from `config/local_credentials.json`.

## Code Examples

### Python Loader Script Pattern

```python
import json
import snowflake.connector
from pathlib import Path

# Load credentials
with open('config/local_credentials.json', 'r') as f:
    creds = json.load(f)

# Connect to Snowflake
conn = snowflake.connector.connect(
    account=creds['account'],
    user=creds['user'],
    password=creds['password'],
    role=creds['role'],
    warehouse=creds['warehouse'],
    database=creds['database'],
    schema=creds['schema']
)

cursor = conn.cursor()

# Upload file to internal stage
cursor.execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")

# Create raw table
cursor.execute("""
    CREATE OR REPLACE TABLE RAW.LISTINGS (
        id VARCHAR,
        name VARCHAR,
        host_id VARCHAR,
        host_name VARCHAR,
        neighbourhood_group VARCHAR,
        neighbourhood VARCHAR,
        latitude VARCHAR,
        longitude VARCHAR,
        room_type VARCHAR,
        price VARCHAR,
        minimum_nights VARCHAR,
        number_of_reviews VARCHAR,
        last_review VARCHAR,
        reviews_per_month VARCHAR,
        calculated_host_listings_count VARCHAR,
        availability_365 VARCHAR
    )
""")

# Copy staged data into table
cursor.execute("""
    COPY INTO RAW.LISTINGS
    FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
    FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
    ON_ERROR = 'CONTINUE'
""")

cursor.close()
conn.close()
```

### dbt Staging Model (stg_airbnb__listings.sql)

```sql
with source as (
    select * from {{ source('raw_airbnb', 'listings') }}
),

renamed as (
    select
        id::bigint as listing_id,
        name as listing_name,
        host_id::bigint as host_id,
        host_name,
        neighbourhood_group,
        neighbourhood,
        latitude::float as latitude,
        longitude::float as longitude,
        room_type,
        -- Clean price: remove $ and commas
        replace(replace(price, '$', ''), ',', '')::float as price,
        minimum_nights::int as minimum_nights,
        number_of_reviews::int as number_of_reviews,
        try_to_date(last_review, 'YYYY-MM-DD') as last_review_date,
        reviews_per_month::float as reviews_per_month,
        calculated_host_listings_count::int as host_listing_count,
        availability_365::int as availability_365
    from source
)

select * from renamed
```

### dbt Incremental Fact Model (fct_listing_calendar.sql)

```sql
{{
    config(
        materialized='incremental',
        unique_key='calendar_key',
        on_schema_change='fail'
    )
}}

with calendar_enriched as (
    select * from {{ ref('int_airbnb__calendar_enriched') }}
),

final as (
    select
        {{ dbt_utils.generate_surrogate_key(['listing_id', 'calendar_date']) }} as calendar_key,
        listing_id,
        calendar_date,
        available,
        price,
        adjusted_price,
        minimum_nights,
        maximum_nights,
        -- Revenue proxy: if not available, assume booked
        case
            when available = false then adjusted_price
            else 0
        end as estimated_revenue,
        listing_name,
        neighbourhood,
        room_type,
        host_id,
        host_name
    from calendar_enriched
)

select * from final

{% if is_incremental() %}
    -- Only load new or updated records
    where calendar_date > (select max(calendar_date) from {{ this }})
{% endif %}
```

### dbt Aggregate Model (agg_listing_monthly_performance.sql)

```sql
with calendar_facts as (
    select * from {{ ref('fct_listing_calendar') }}
),

monthly_agg as (
    select
        listing_id,
        listing_name,
        neighbourhood,
        room_type,
        host_id,
        date_trunc('month', calendar_date) as month_date,
        
        count(*) as days_in_month,
        sum(case when available = false then 1 else 0 end) as unavailable_days,
        sum(case when available = true then 1 else 0 end) as available_days,
        
        avg(price) as avg_price,
        sum(estimated_revenue) as total_estimated_revenue,
        
        round(
            sum(case when available = false then 1 else 0 end) * 100.0 / count(*),
            2
        ) as occupancy_rate_pct
        
    from calendar_facts
    group by 1, 2, 3, 4, 5, 6
)

select * from monthly_agg
```

### dbt Schema Test (schema.yml)

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

models:
  - name: dim_listings
    description: "Dimension table for Airbnb listings"
    columns:
      - name: listing_id
        description: "Unique listing identifier"
        tests:
          - not_null
          - unique
      - name: room_type
        description: "Type of room"
        tests:
          - accepted_values:
              values: ['Entire home/apt', 'Private room', 'Shared room', 'Hotel room']
      - name: price
        description: "Daily price"
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
```

### Streamlit Dashboard Pattern

```python
import streamlit as st
import pandas as pd
import snowflake.connector
import json

# Load credentials
with open('config/local_credentials.json', 'r') as f:
    creds = json.load(f)

# Connect to Snowflake
@st.cache_resource
def get_snowflake_connection():
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        role=creds['role'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema='ANALYTICS'
    )

conn = get_snowflake_connection()

# Query mart
@st.cache_data
def load_neighbourhood_performance():
    query = """
        SELECT 
            neighbourhood,
            month_date,
            total_estimated_revenue,
            avg_occupancy_rate_pct,
            total_listings
        FROM agg_neighbourhood_monthly_performance
        ORDER BY month_date DESC, total_estimated_revenue DESC
    """
    return pd.read_sql(query, conn)

df = load_neighbourhood_performance()

st.title("Inside Airbnb Analytics")
st.subheader("Neighbourhood Performance")
st.dataframe(df)

# Top neighbourhoods by revenue
top_n = st.slider("Top N neighbourhoods", 5, 20, 10)
top_neighbourhoods = df.groupby('neighbourhood')['total_estimated_revenue'].sum() \
    .nlargest(top_n).reset_index()

st.bar_chart(top_neighbourhoods.set_index('neighbourhood'))
```

## Common Patterns

### Full Pipeline Refresh

When you have new Inside Airbnb data:

```bash
# 1. Load new raw data
python scripts/load_inside_airbnb_to_snowflake.py

# 2. Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 3. Run tests
dbt test --profiles-dir .

# 4. Update docs
dbt docs generate --profiles-dir .
```

### Selective Model Runs

```bash
# Run only staging models
dbt run --select staging.* --profiles-dir .

# Run specific mart and upstream dependencies
dbt run --select +dim_listings --profiles-dir .

# Run downstream from a model
dbt run --select stg_airbnb__calendar+ --profiles-dir .

# Run modified models and downstream
dbt run --select state:modified+ --profiles-dir .
```

### Testing Strategy

```bash
# Test sources first
dbt test --select source:* --profiles-dir .

# Test staging layer
dbt test --select staging.* --profiles-dir .

# Test specific model lineage
dbt test --select fct_listing_calendar --profiles-dir .

# Run singular tests only
dbt test --select test_type:singular --profiles-dir .
```

### dbt Macro Usage

Create reusable SQL logic in `macros/`:

```sql
-- macros/clean_price.sql
{% macro clean_price(price_column) %}
    replace(replace({{ price_column }}, '$', ''), ',', '')::float
{% endmacro %}
```

Use in models:

```sql
select
    listing_id,
    {{ clean_price('price') }} as price_clean
from source
```

## Troubleshooting

### "SnowflakeOperationalError: 250001: Could not connect to Snowflake"

**Cause:** Incorrect credentials or network issue

**Fix:**
```bash
# Test connection
dbt debug --profiles-dir .

# Verify credentials in profiles.yml match your Snowflake account
# Check account identifier format: <account>.<region> or <account>
```

### "Database 'AIRBNB_DB' does not exist"

**Cause:** Snowflake objects not created

**Fix:**
```bash
# Run loader script to create database and schemas
python scripts/load_inside_airbnb_to_snowflake.py

# Or manually run setup SQL
# Execute contents of setup/snowflake_setup.sql in Snowflake
```

### "This is not a dbt project"

**Cause:** Running dbt from wrong directory or missing dbt_project.yml

**Fix:**
```bash
# Ensure you're in project root
cd Snowflake_DBT_Project

# Verify dbt_project.yml exists
ls -la dbt_project.yml

# Use --profiles-dir . to look for profiles.yml in current directory
dbt run --profiles-dir .
```

### Incremental model not updating

**Cause:** Incremental logic filtering out new data

**Fix:**
```bash
# Check filter condition in model
# Verify max date in existing table
dbt run-operation run_query --args "{'sql': 'select max(calendar_date) from fct_listing_calendar'}"

# Force full refresh
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

### "Compilation Error: depends on a node named 'X' which was not found"

**Cause:** Missing dependency or incorrect ref() call

**Fix:**
```bash
# Verify model exists in models/ directory
# Check ref() syntax: {{ ref('model_name') }} not {{ ref('schema.model_name') }}

# Parse project to check dependencies
dbt parse --profiles-dir .

# View lineage
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

### Streamlit "FileNotFoundError: config/local_credentials.json"

**Cause:** Running dashboard from wrong directory

**Fix:**
```bash
# Run from project root
cd Snowflake_DBT_Project
streamlit run dashboard/streamlit_app.py

# Or update dashboard script to use absolute path
```

### "SQL compilation error: Object does not exist"

**Cause:** Model not materialized or wrong schema reference

**Fix:**
```bash
# Check target schema in profiles.yml
# Verify models materialized
dbt run --profiles-dir .

# Check source configuration in schema.yml
# Verify database/schema names match Snowflake
```

## Project Structure Reference

```text
Snowflake_DBT_Project/
├── models/
│   ├── staging/
│   │   ├── _sources.yml           # Source definitions
│   │   ├── stg_airbnb__listings.sql
│   │   ├── stg_airbnb__calendar.sql
│   │   └── stg_airbnb__reviews.sql
│   ├── intermediate/
│   │   ├── int_airbnb__listing_enriched.sql
│   │   ├── int_airbnb__calendar_enriched.sql
│   │   └── int_airbnb__reviews_enriched.sql
│   └── marts/
│       ├── dim_listings.sql
│       ├── dim_hosts.sql
│       ├── fct_listing_calendar.sql    # Incremental
│       ├── fct_reviews.sql
│       ├── agg_listing_monthly_performance.sql
│       └── agg_neighbourhood_monthly_performance.sql
├── tests/
│   └── singular tests (.sql files)
├── dashboard/
│   └── streamlit_app.py
├── scripts/
│   └── load_inside_airbnb_to_snowflake.py
├── setup/
│   └── snowflake_setup.sql
├── dbt_project.yml
└── profiles.yml                    # Git-ignored, created from example
```

## Additional Resources

- Inside Airbnb data: https://insideairbnb.com/get-the-data/
- dbt Snowflake adapter: https://docs.getdbt.com/reference/warehouse-setups/snowflake-setup
- Incremental models: https://docs.getdbt.com/docs/build/incremental-models
- dbt tests: https://docs.getdbt.com/docs/build/tests

```
