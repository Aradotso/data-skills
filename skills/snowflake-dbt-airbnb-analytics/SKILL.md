---
name: snowflake-dbt-airbnb-analytics
description: Build end-to-end analytics pipelines with Snowflake, dbt, and Streamlit using Inside Airbnb data
triggers:
  - "set up snowflake dbt analytics project"
  - "load airbnb data into snowflake"
  - "build dbt models for airbnb"
  - "create incremental fact tables in dbt"
  - "run dbt tests for data quality"
  - "build streamlit dashboard with snowflake"
  - "configure dbt profiles for snowflake"
  - "aggregate airbnb listing performance"
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build complete analytics engineering pipelines using Snowflake as the data warehouse, dbt for transformations, and Streamlit for dashboards. The project demonstrates staging → intermediate → mart layering, incremental models, data quality tests, and public-safe credential management.

## What This Project Does

The Snowflake_DBT_Project loads Inside Airbnb open data (listings, calendar, reviews, neighbourhoods) into Snowflake, transforms it through dbt layers (staging, intermediate, marts), validates data quality with tests, and powers a Streamlit dashboard. It demonstrates:

- Raw data ingestion from CSV/GZIP into Snowflake internal stages
- dbt staging models for cleaning and type casting
- Intermediate enrichment layers with joins and business logic
- Dimension and fact tables (incremental merge strategy)
- Monthly aggregation models for reporting
- Generic and singular dbt tests
- Streamlit dashboard connected to analytics marts

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

Create local credential files from examples (these are gitignored):

```bash
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

Edit `profiles.yml`:

```yaml
snowflake_dbt_project:
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

**Security**: Never commit real credentials. Use environment variables in production:

```python
import os
credentials = {
    "user": os.getenv("SNOWFLAKE_USER"),
    "password": os.getenv("SNOWFLAKE_PASSWORD"),
    # ...
}
```

### 3. Download Inside Airbnb Data

Download these files from [Inside Airbnb](https://insideairbnb.com/get-the-data/) (NYC recommended):

```text
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

## Key Commands

### Load Raw Data to Snowflake

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

This script:
1. Reads credentials from `config/local_credentials.json`
2. Executes `setup/snowflake_setup.sql` to create database/schemas/stage
3. Uploads CSV files to Snowflake internal stage
4. Creates raw tables with text columns
5. Copies data from stage to `RAW` schema

### dbt Workflow

```bash
# Verify dbt connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select stg_airbnb__listings+ --profiles-dir .

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

## Project Structure

```text
models/
├── staging/
│   ├── _stg_airbnb__sources.yml        # Source definitions
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
    ├── fct_listing_calendar.sql        # Incremental
    ├── fct_reviews.sql
    ├── agg_listing_monthly_performance.sql
    └── agg_neighbourhood_monthly_performance.sql
```

## dbt Model Examples

### Staging Model: Type Casting and Cleaning

`models/staging/stg_airbnb__listings.sql`:

```sql
with source as (
    select * from {{ source('airbnb_raw', 'listings') }}
),

cleaned as (
    select
        try_cast(id as integer) as listing_id,
        name as listing_name,
        host_id::integer as host_id,
        host_name,
        neighbourhood_cleansed as neighbourhood,
        latitude::float as latitude,
        longitude::float as longitude,
        room_type,
        try_cast(replace(price, '$', '') as number(10,2)) as price,
        minimum_nights::integer as minimum_nights,
        number_of_reviews::integer as number_of_reviews,
        try_to_date(last_review) as last_review_date,
        reviews_per_month::float as reviews_per_month,
        calculated_host_listings_count::integer as host_listings_count,
        availability_365::integer as availability_365
    from source
)

select * from cleaned
```

### Intermediate Model: Enrichment with Joins

`models/intermediate/int_airbnb__listing_enriched.sql`:

```sql
with listings as (
    select * from {{ ref('stg_airbnb__listings') }}
),

neighbourhoods as (
    select * from {{ ref('stg_airbnb__neighbourhoods') }}
),

enriched as (
    select
        l.*,
        n.neighbourhood_group,
        case
            when l.number_of_reviews = 0 then 'No Reviews'
            when l.number_of_reviews between 1 and 10 then 'Low Activity'
            when l.number_of_reviews between 11 and 50 then 'Medium Activity'
            else 'High Activity'
        end as review_activity_segment
    from listings l
    left join neighbourhoods n
        on l.neighbourhood = n.neighbourhood
)

select * from enriched
```

### Incremental Fact Table

`models/marts/fct_listing_calendar.sql`:

```sql
{{
    config(
        materialized='incremental',
        unique_key='calendar_id',
        merge_update_columns=['available', 'price', 'minimum_nights', 'maximum_nights']
    )
}}

with calendar as (
    select * from {{ ref('int_airbnb__calendar_enriched') }}
),

final as (
    select
        {{ dbt_utils.generate_surrogate_key(['listing_id', 'date']) }} as calendar_id,
        listing_id,
        date,
        available,
        price,
        minimum_nights,
        maximum_nights,
        listing_name,
        neighbourhood,
        room_type
    from calendar
    {% if is_incremental() %}
    where date > (select max(date) from {{ this }})
    {% endif %}
)

select * from final
```

### Aggregate Model: Monthly Performance

`models/marts/agg_listing_monthly_performance.sql`:

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
        date_trunc('month', date) as month,
        count(*) as total_days,
        sum(case when available = false then 1 else 0 end) as unavailable_days,
        avg(price) as avg_daily_price,
        sum(case when available = false then price else 0 end) as estimated_revenue
    from calendar_facts
    group by 1, 2, 3, 4, 5
)

select
    *,
    round(100.0 * unavailable_days / nullif(total_days, 0), 2) as occupancy_rate_pct
from monthly_agg
```

## Data Quality Tests

### Generic Tests in Schema YAML

`models/staging/_stg_airbnb__models.yml`:

```yaml
version: 2

models:
  - name: stg_airbnb__listings
    description: Cleaned and typed listing data
    columns:
      - name: listing_id
        description: Primary key
        tests:
          - unique
          - not_null
      - name: price
        description: Nightly price in USD
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
      - name: room_type
        description: Type of accommodation
        tests:
          - accepted_values:
              values: ['Entire home/apt', 'Private room', 'Shared room', 'Hotel room']
```

### Singular Test

`tests/no_duplicate_listing_dates.sql`:

```sql
-- Ensure no listing has duplicate calendar entries for the same date
select
    listing_id,
    date,
    count(*) as occurrences
from {{ ref('fct_listing_calendar') }}
group by listing_id, date
having count(*) > 1
```

## Streamlit Dashboard Pattern

`dashboard/streamlit_app.py`:

```python
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
        user=creds['user'],
        password=creds['password'],
        account=creds['account'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema'],
        role=creds['role']
    )

conn = get_connection()

st.title("Airbnb Analytics Dashboard")

# Query aggregated data
@st.cache_data
def load_neighbourhood_performance():
    query = """
        SELECT
            neighbourhood,
            month,
            sum(estimated_revenue) as total_revenue,
            avg(occupancy_rate_pct) as avg_occupancy
        FROM agg_neighbourhood_monthly_performance
        GROUP BY neighbourhood, month
        ORDER BY total_revenue DESC
    """
    return pd.read_sql(query, conn)

df = load_neighbourhood_performance()

# Display top neighbourhoods
st.subheader("Top Neighbourhoods by Revenue")
top_n = st.slider("Number of neighbourhoods", 5, 20, 10)
st.bar_chart(df.groupby('neighbourhood')['total_revenue'].sum().nlargest(top_n))

# Filter by neighbourhood
neighbourhood = st.selectbox("Select Neighbourhood", df['neighbourhood'].unique())
filtered = df[df['neighbourhood'] == neighbourhood]
st.line_chart(filtered.set_index('month')['avg_occupancy'])
```

## Common Patterns

### Full Pipeline Refresh (New Data Snapshot)

```bash
# 1. Load new raw data
python scripts/load_inside_airbnb_to_snowflake.py

# 2. Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 3. Run all other models
dbt run --exclude fct_listing_calendar --profiles-dir .

# 4. Validate
dbt test --profiles-dir .
```

### Selective Model Execution

```bash
# Run only staging layer
dbt run --select staging.* --profiles-dir .

# Run specific model and all downstream
dbt run --select stg_airbnb__calendar+ --profiles-dir .

# Run all marts
dbt run --select marts.* --profiles-dir .
```

### Adding a New Source Column

1. Update staging model SQL to include the column
2. Update downstream intermediate/mart models if needed
3. Add tests in schema YAML
4. Run:

```bash
dbt run --select stg_airbnb__listings+ --profiles-dir .
dbt test --select stg_airbnb__listings --profiles-dir .
```

## Troubleshooting

### "Object does not exist" Error

**Problem**: dbt can't find source tables.

**Solution**: Ensure raw data is loaded and source definitions match:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
dbt debug --profiles-dir .  # Verify connection
```

Check `models/staging/_stg_airbnb__sources.yml`:

```yaml
sources:
  - name: airbnb_raw
    database: AIRBNB_DB
    schema: RAW
    tables:
      - name: listings
      - name: calendar
      # ...
```

### Incremental Model Not Updating

**Problem**: `fct_listing_calendar` not picking up new dates.

**Solution**: Use `--full-refresh` or check incremental logic:

```sql
{% if is_incremental() %}
where date > (select max(date) from {{ this }})
{% endif %}
```

Force full refresh:

```bash
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

### Streamlit Can't Connect to Snowflake

**Problem**: Dashboard shows connection error.

**Solution**: Verify `config/local_credentials.json` exists and has correct values. Test connection:

```python
import snowflake.connector
import json

with open('config/local_credentials.json') as f:
    creds = json.load(f)

conn = snowflake.connector.connect(**creds)
print("Connected:", conn.is_closed() == False)
```

### dbt Test Failures

**Problem**: `accepted_values` test fails for `room_type`.

**Solution**: Check for unexpected values in raw data:

```sql
select distinct room_type
from {{ source('airbnb_raw', 'listings') }}
```

Update accepted values in schema YAML or clean in staging model.

### Price Parsing Errors

**Problem**: Prices with `$` or `,` symbols fail to cast.

**Solution**: Use `replace()` and `try_cast()`:

```sql
try_cast(
    replace(replace(price, '$', ''), ',', '') 
    as number(10,2)
) as price
```

## Advanced Techniques

### Custom dbt Macro for Surrogate Keys

If using `dbt_utils.generate_surrogate_key`, ensure dbt-utils is installed:

```yaml
# packages.yml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1
```

Install:

```bash
dbt deps --profiles-dir .
```

### Environment-Specific Targets

`profiles.yml`:

```yaml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      schema: DEV_ANALYTICS
      # ...
    prod:
      type: snowflake
      schema: PROD_ANALYTICS
      # ...
```

Run against production:

```bash
dbt run --target prod --profiles-dir .
```

### Revenue Proxy Logic

Inside Airbnb lacks transaction data. This project uses calendar unavailability as a proxy:

```sql
sum(case when available = false then price else 0 end) as estimated_revenue
```

Unavailable nights are assumed booked. Acknowledge limitations in documentation.

## Business Questions Answered

1. **Top neighbourhoods by revenue**: Query `agg_neighbourhood_monthly_performance`
2. **Host with most listings**: Query `dim_hosts` ordered by `host_listings_count`
3. **Average price by room type**: Aggregate `dim_listings` on `room_type`
4. **Occupancy trends over time**: Analyze `agg_listing_monthly_performance.occupancy_rate_pct`
5. **Listings with no reviews**: Filter `dim_listings` where `number_of_reviews = 0`

## Security Best Practices

- **Never commit** `profiles.yml` or `config/local_credentials.json`
- Use `.gitignore` for sensitive files (already configured)
- In production, use Snowflake key-pair authentication or OAuth
- Store secrets in environment variables or secret managers:

```python
import os
password = os.getenv("SNOWFLAKE_PASSWORD")
```

- Rotate credentials regularly
- Use Snowflake roles with least privilege

## Resources

- [dbt Documentation](https://docs.getdbt.com/)
- [Snowflake Python Connector](https://docs.snowflake.com/en/user-guide/python-connector.html)
- [Inside Airbnb Data](https://insideairbnb.com/get-the-data/)
- [Streamlit Docs](https://docs.streamlit.io/)

This skill covers the complete lifecycle of the Snowflake dbt Airbnb Analytics project: setup, data loading, transformation, testing, and visualization.
