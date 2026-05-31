---
name: snowflake-dbt-airbnb-analytics
description: Build end-to-end analytics pipelines with Snowflake, dbt, and Streamlit using Inside Airbnb data
triggers:
  - how do I set up the Snowflake dbt Airbnb project
  - load Inside Airbnb data into Snowflake
  - create dbt models for Airbnb analytics
  - build incremental fact tables with dbt and Snowflake
  - run dbt tests on Airbnb data
  - create a Streamlit dashboard for Snowflake marts
  - transform Inside Airbnb data with dbt
  - set up analytics engineering pipeline with Snowflake
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill helps you build a complete analytics engineering pipeline using Snowflake as the data warehouse, dbt for transformations, and Streamlit for visualization, using Inside Airbnb open data as the source.

## What This Project Does

The Snowflake_DBT_Project is an end-to-end analytics engineering reference implementation that:

- Loads Inside Airbnb CSV/GZIP files into Snowflake internal stages
- Creates raw, staging, intermediate, and mart layers using dbt
- Implements incremental fact table patterns with Snowflake merge strategy
- Includes comprehensive data quality tests (generic and singular)
- Powers a Streamlit dashboard for neighbourhood and listing analytics
- Demonstrates public-safe credential management (no secrets in git)

**Key architectural layers:**
1. **RAW schema** — text-preserving raw tables from CSV files
2. **Staging** — cleaning, type casting, standardization
3. **Intermediate** — joins, enrichment, revenue proxy calculations
4. **Marts** — dimensions (`dim_listings`, `dim_hosts`) and facts (`fct_listing_calendar`, `fct_reviews`)
5. **Aggregates** — monthly performance rollups by listing and neighbourhood

## Installation

### Prerequisites

- Python 3.8+
- Snowflake account with create database/schema/table privileges
- Inside Airbnb data files (recommended: New York City)

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

### Configuration (Credential Files)

This project uses **local credential files** (not environment variables) that are git-ignored.

**Step 1: Create profiles.yml**

```bash
cp profiles.yml.example profiles.yml
```

Edit `profiles.yml`:

```yaml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT  # e.g., xy12345.us-east-1
      user: YOUR_USERNAME
      password: YOUR_PASSWORD
      role: YOUR_ROLE  # e.g., ACCOUNTADMIN
      database: INSIDE_AIRBNB
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

**Step 2: Create local_credentials.json**

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
  "database": "INSIDE_AIRBNB",
  "schema": "ANALYTICS",
  "role": "YOUR_ROLE"
}
```

**These files are git-ignored and never committed.**

## Data Acquisition

Download Inside Airbnb data for your chosen city (e.g., New York):

```bash
# Create data directory
mkdir -p data/raw

# Download files (example URLs for NYC)
# Visit http://insideairbnb.com/get-the-data/ for latest links
curl -o data/raw/listings.csv.gz "http://data.insideairbnb.com/united-states/ny/new-york-city/2024-03-06/data/listings.csv.gz"
curl -o data/raw/calendar.csv.gz "http://data.insideairbnb.com/united-states/ny/new-york-city/2024-03-06/data/calendar.csv.gz"
curl -o data/raw/reviews.csv.gz "http://data.insideairbnb.com/united-states/ny/new-york-city/2024-03-06/data/reviews.csv.gz"
curl -o data/raw/neighbourhoods.csv "http://data.insideairbnb.com/united-states/ny/new-york-city/2024-03-06/visualisations/neighbourhoods.csv"
```

Expected files:
- `data/raw/listings.csv.gz`
- `data/raw/calendar.csv.gz`
- `data/raw/reviews.csv.gz`
- `data/raw/neighbourhoods.csv`

## Loading Data into Snowflake

The Python loader script handles the entire raw data pipeline:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What this script does:**

1. Reads credentials from `config/local_credentials.json`
2. Executes Snowflake setup SQL (`setup/snowflake_setup.sql`)
3. Creates database, schemas (RAW, ANALYTICS), and internal stage
4. Uploads local files to `INSIDE_AIRBNB_STAGE`
5. Creates raw tables with text columns preserving CSV structure
6. Copies staged files into RAW tables

**Key script logic:**

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
    role=creds.get('role', 'ACCOUNTADMIN')
)

# Execute setup SQL
with open('setup/snowflake_setup.sql') as f:
    setup_sql = f.read()
    for stmt in setup_sql.split(';'):
        if stmt.strip():
            conn.cursor().execute(stmt)

# Upload files to stage
conn.cursor().execute("USE DATABASE INSIDE_AIRBNB")
conn.cursor().execute("USE SCHEMA RAW")

for file_path in Path('data/raw').glob('*'):
    conn.cursor().execute(f"PUT file://{file_path} @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")

# Copy into raw tables
conn.cursor().execute("""
    COPY INTO RAW.LISTINGS
    FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
    FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
""")
```

## dbt Workflow

### Project Structure

```
models/
├── sources.yml              # Source definitions for RAW tables
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
    ├── fct_listing_calendar.sql          # Incremental
    ├── fct_reviews.sql
    ├── agg_listing_monthly_performance.sql
    └── agg_neighbourhood_monthly_performance.sql
```

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

# Test specific model
dbt test --select dim_listings --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .

# Run only changed models
dbt run --select state:modified+ --profiles-dir .
```

### Staging Layer Example

**stg_airbnb__listings.sql**

```sql
with source as (
    select * from {{ source('inside_airbnb', 'listings') }}
),

cleaned as (
    select
        -- Primary key
        cast(id as integer) as listing_id,
        
        -- Host attributes
        cast(host_id as integer) as host_id,
        host_name,
        cast(host_since as date) as host_since,
        cast(host_listings_count as integer) as host_listings_count,
        
        -- Listing attributes
        name as listing_name,
        neighbourhood_cleansed as neighbourhood,
        cast(latitude as float) as latitude,
        cast(longitude as float) as longitude,
        room_type,
        cast(accommodates as integer) as accommodates,
        cast(bedrooms as integer) as bedrooms,
        cast(beds as integer) as beds,
        cast(price as float) as price,
        cast(minimum_nights as integer) as minimum_nights,
        cast(maximum_nights as integer) as maximum_nights,
        cast(availability_365 as integer) as availability_365,
        cast(number_of_reviews as integer) as number_of_reviews,
        cast(review_scores_rating as float) as review_scores_rating,
        instant_bookable,
        
        -- Metadata
        current_timestamp() as dbt_loaded_at
        
    from source
)

select * from cleaned
```

### Intermediate Layer Example

**int_airbnb__calendar_enriched.sql**

```sql
with calendar as (
    select * from {{ ref('stg_airbnb__calendar') }}
),

listings as (
    select * from {{ ref('int_airbnb__listing_enriched') }}
),

enriched as (
    select
        c.calendar_date,
        c.listing_id,
        c.available,
        c.price,
        c.adjusted_price,
        c.minimum_nights,
        c.maximum_nights,
        
        -- Add listing context
        l.host_id,
        l.neighbourhood,
        l.neighbourhood_group,
        l.room_type,
        l.accommodates,
        
        -- Calculate revenue proxy
        case 
            when c.available = 'f' then coalesce(c.adjusted_price, c.price, 0.0)
            else 0.0
        end as estimated_revenue,
        
        current_timestamp() as dbt_loaded_at
        
    from calendar c
    left join listings l on c.listing_id = l.listing_id
)

select * from enriched
```

### Incremental Fact Table

**fct_listing_calendar.sql**

```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        merge_update_columns=['available', 'price', 'adjusted_price', 'estimated_revenue']
    )
}}

with calendar_enriched as (
    select * from {{ ref('int_airbnb__calendar_enriched') }}
)

select
    listing_id,
    calendar_date,
    available,
    price,
    adjusted_price,
    minimum_nights,
    maximum_nights,
    host_id,
    neighbourhood,
    neighbourhood_group,
    room_type,
    accommodates,
    estimated_revenue,
    dbt_loaded_at

from calendar_enriched

{% if is_incremental() %}
    -- Only process new or updated dates
    where calendar_date > (select max(calendar_date) from {{ this }})
{% endif %}
```

### Aggregate Marts Example

**agg_neighbourhood_monthly_performance.sql**

```sql
with listing_monthly as (
    select * from {{ ref('agg_listing_monthly_performance') }}
)

select
    year_month,
    neighbourhood,
    neighbourhood_group,
    
    -- Aggregates
    count(distinct listing_id) as total_listings,
    sum(days_in_month) as total_days,
    sum(days_available) as total_days_available,
    sum(days_booked) as total_days_booked,
    sum(total_revenue) as total_revenue,
    
    -- Calculated metrics
    round(avg(availability_rate), 4) as avg_availability_rate,
    round(avg(occupancy_rate), 4) as avg_occupancy_rate,
    round(avg(avg_daily_price), 2) as avg_daily_price,
    round(sum(total_revenue) / nullif(sum(days_booked), 0), 2) as revenue_per_booked_night,
    
    current_timestamp() as dbt_loaded_at

from listing_monthly
group by 1, 2, 3
```

## Testing

### Generic Tests (schema.yml)

```yaml
models:
  - name: dim_listings
    description: Dimension table for Airbnb listings
    columns:
      - name: listing_id
        description: Primary key
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

**tests/assert_no_duplicate_listing_dates.sql**

```sql
with calendar_facts as (
    select
        listing_id,
        calendar_date,
        count(*) as occurrence_count
    from {{ ref('fct_listing_calendar') }}
    group by 1, 2
    having count(*) > 1
)

select *
from calendar_facts
```

Run tests:

```bash
dbt test --profiles-dir .
```

## Streamlit Dashboard

The dashboard reads from the analytics marts using the same credential file.

**dashboard/streamlit_app.py** (key sections):

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load credentials
@st.cache_resource
def get_connection():
    with open('config/local_credentials.json') as f:
        creds = json.load(f)
    
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema'],
        role=creds.get('role', 'ACCOUNTADMIN')
    )

# Query neighbourhood performance
@st.cache_data
def get_neighbourhood_performance():
    conn = get_connection()
    query = """
        SELECT 
            neighbourhood,
            total_listings,
            total_revenue,
            avg_occupancy_rate,
            avg_daily_price
        FROM agg_neighbourhood_monthly_performance
        WHERE year_month = (SELECT MAX(year_month) FROM agg_neighbourhood_monthly_performance)
        ORDER BY total_revenue DESC
        LIMIT 20
    """
    df = pd.read_sql(query, conn)
    return df

# Streamlit app
st.title('Inside Airbnb Analytics Dashboard')

st.header('Top Neighbourhoods by Revenue')
df_neighbourhoods = get_neighbourhood_performance()
st.dataframe(df_neighbourhoods)

st.bar_chart(df_neighbourhoods.set_index('NEIGHBOURHOOD')['TOTAL_REVENUE'])
```

Run the dashboard:

```bash
streamlit run dashboard/streamlit_app.py
```

## Common Patterns

### Full Data Refresh Workflow

When you have new Inside Airbnb data:

```bash
# 1. Download new data files to data/raw/

# 2. Reload Snowflake raw tables
python scripts/load_inside_airbnb_to_snowflake.py

# 3. Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 4. Run tests
dbt test --profiles-dir .

# 5. Regenerate docs
dbt docs generate --profiles-dir .
```

### Working with Subsets

```bash
# Run only staging models
dbt run --select staging --profiles-dir .

# Run only marts and their dependencies
dbt run --select marts --profiles-dir .

# Run a specific model and everything downstream
dbt run --select int_airbnb__listing_enriched+ --profiles-dir .

# Run everything upstream of a model
dbt run --select +dim_listings --profiles-dir .
```

### Adding a New Metric

1. **Create intermediate model** if needed:

```sql
-- models/intermediate/int_airbnb__listing_reviews_summary.sql
with reviews as (
    select * from {{ ref('stg_airbnb__reviews') }}
)

select
    listing_id,
    count(*) as review_count,
    count(distinct reviewer_id) as unique_reviewers,
    min(review_date) as first_review_date,
    max(review_date) as last_review_date
from reviews
group by 1
```

2. **Join to existing mart**:

```sql
-- Add to models/marts/dim_listings.sql
left join {{ ref('int_airbnb__listing_reviews_summary') }} r
    on l.listing_id = r.listing_id
```

3. **Run and test**:

```bash
dbt run --select int_airbnb__listing_reviews_summary+ --profiles-dir .
dbt test --select dim_listings --profiles-dir .
```

## Troubleshooting

### Connection Issues

```bash
# Verify Snowflake connection
dbt debug --profiles-dir .

# Check credentials file exists
ls -la config/local_credentials.json
cat profiles.yml
```

### Stage Upload Errors

```python
# Manually check stage contents
import snowflake.connector
import json

with open('config/local_credentials.json') as f:
    creds = json.load(f)

conn = snowflake.connector.connect(**creds)
cursor = conn.cursor()

# List stage files
cursor.execute("USE DATABASE INSIDE_AIRBNB")
cursor.execute("USE SCHEMA RAW")
cursor.execute("LIST @INSIDE_AIRBNB_STAGE")
print(cursor.fetchall())

# Remove old files if needed
cursor.execute("REMOVE @INSIDE_AIRBNB_STAGE/listings.csv.gz")
```

### dbt Model Failures

```bash
# Run in debug mode
dbt --debug run --select failing_model --profiles-dir .

# Compile SQL without running
dbt compile --select failing_model --profiles-dir .

# Check compiled SQL in target/compiled/snowflake_dbt_project/
cat target/compiled/snowflake_dbt_project/models/marts/dim_listings.sql
```

### Incremental Model Issues

```bash
# Force full refresh
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Check for duplicates
dbt test --select assert_no_duplicate_listing_dates --profiles-dir .
```

### Data Quality Issues

```sql
-- Query raw data directly in Snowflake
SELECT COUNT(*), COUNT(DISTINCT id) 
FROM INSIDE_AIRBNB.RAW.LISTINGS;

-- Check for nulls in key fields
SELECT 
    COUNT(*) as total,
    COUNT(id) as id_count,
    COUNT(host_id) as host_id_count,
    COUNT(price) as price_count
FROM INSIDE_AIRBNB.RAW.LISTINGS;
```

### Streamlit Dashboard Errors

```python
# Add error handling to dashboard queries
try:
    df = get_neighbourhood_performance()
    st.dataframe(df)
except Exception as e:
    st.error(f"Error loading data: {str(e)}")
    st.info("Check Snowflake connection and schema permissions")
```

## Business Questions You Can Answer

With this pipeline, you can query:

```sql
-- Top revenue neighbourhoods
SELECT neighbourhood, SUM(total_revenue) as revenue
FROM agg_neighbourhood_monthly_performance
GROUP BY 1
ORDER BY 2 DESC
LIMIT 10;

-- Average price by room type
SELECT room_type, AVG(price) as avg_price
FROM dim_listings
WHERE price > 0
GROUP BY 1;

-- Superhosts (hosts with most listings)
SELECT host_name, COUNT(DISTINCT listing_id) as listing_count
FROM dim_listings
GROUP BY 1
ORDER BY 2 DESC
LIMIT 20;

-- Availability trends over time
SELECT 
    DATE_TRUNC('month', calendar_date) as month,
    AVG(CASE WHEN available = 't' THEN 1.0 ELSE 0.0 END) as availability_rate
FROM fct_listing_calendar
GROUP BY 1
ORDER BY 1;
```

## Best Practices

1. **Never commit credentials** — always use git-ignored config files
2. **Use incremental models** for large fact tables (calendar has ~365 rows per listing)
3. **Test early and often** — run `dbt test` after every model change
4. **Document your models** — add descriptions in schema.yml files
5. **Use intermediate layers** — don't repeat complex joins in multiple marts
6. **Version your data** — keep track of Inside Airbnb snapshot dates in source names or metadata

This skill provides everything needed to build, test, and deploy a production-quality analytics pipeline using Snowflake, dbt, and Streamlit.
