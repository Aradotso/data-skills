---
name: snowflake-dbt-airbnb-analytics
description: End-to-end analytics engineering with Snowflake, dbt, and Streamlit for Inside Airbnb data
triggers:
  - set up airbnb analytics with dbt and snowflake
  - build a dbt project with snowflake data warehouse
  - create analytics marts for airbnb listings
  - load inside airbnb data into snowflake
  - run dbt models with staging intermediate and marts
  - build streamlit dashboard for snowflake analytics
  - configure dbt profiles for snowflake connection
  - implement incremental fact tables in dbt
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to work with a complete analytics engineering project that ingests Inside Airbnb data into Snowflake, transforms it with dbt using staging/intermediate/mart layers, validates data quality with tests, and serves analytics through a Streamlit dashboard.

## What This Project Does

**Snowflake_DBT_Project** is a portfolio analytics engineering project demonstrating:

- **Data ingestion**: Python script loads Inside Airbnb CSV/GZIP files into Snowflake internal stages
- **Raw layer**: Text-preserving raw tables in Snowflake RAW schema
- **dbt transformation**: Staged models → intermediate enrichment → dimensional marts → monthly aggregates
- **Incremental modeling**: `fct_listing_calendar` uses Snowflake merge strategy for efficient updates
- **Data quality**: Generic and singular dbt tests for uniqueness, not-null, relationships, and accepted values
- **Documentation**: dbt docs with model descriptions and lineage
- **Reporting**: Streamlit dashboard connected to analytics marts

The project uses Inside Airbnb open data (listings, calendar, reviews, neighbourhoods) and creates a revenue proxy from calendar availability and pricing.

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

**Dependencies include:**
- `dbt-snowflake` - dbt adapter for Snowflake
- `snowflake-connector-python` - Python connector for Snowflake
- `streamlit` - Dashboard framework
- `pandas` - Data manipulation

## Configuration

### 1. Snowflake Credentials Setup

The project uses **local credential files** (not environment variables) that are git-ignored.

**Create `profiles.yml` from example:**

```bash
cp profiles.yml.example profiles.yml
```

Edit `profiles.yml` with your Snowflake credentials:

```yaml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT  # e.g., abc12345.us-east-1
      user: YOUR_USERNAME
      password: YOUR_PASSWORD
      role: YOUR_ROLE  # e.g., ACCOUNTADMIN or custom role
      database: AIRBNB_DB
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

**Create `config/local_credentials.json` from example:**

```bash
cp config/local_credentials.example.json config/local_credentials.json
```

Edit `config/local_credentials.json`:

```json
{
  "user": "YOUR_USERNAME",
  "password": "YOUR_PASSWORD",
  "account": "YOUR_ACCOUNT",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DB",
  "schema": "ANALYTICS",
  "role": "YOUR_ROLE"
}
```

**Security:** Both files are in `.gitignore` and will never be committed.

### 2. Data Acquisition

Download Inside Airbnb data for New York City from [insideairbnb.com/get-the-data](http://insideairbnb.com/get-the-data/).

Place files in `data/raw/`:

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

**What this does:**
1. Reads credentials from `config/local_credentials.json`
2. Executes `setup/snowflake_setup.sql` to create database, schemas, stage
3. Uploads local CSV/GZIP files to Snowflake internal stage `INSIDE_AIRBNB_STAGE`
4. Creates raw tables by inferring schema from CSV headers
5. Copies staged data into `RAW.LISTINGS`, `RAW.CALENDAR`, `RAW.REVIEWS`, `RAW.NEIGHBOURHOODS`

### dbt Workflow

```bash
# Verify connection
dbt debug --profiles-dir .

# Run all models (staging → intermediate → marts → aggregates)
dbt run --profiles-dir .

# Run specific model and its downstream dependencies
dbt run --select stg_airbnb__listings+ --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Test specific model
dbt test --select dim_listings --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .

# Serve documentation locally
dbt docs serve --profiles-dir .
```

### Full Refresh for Incremental Models

When new Inside Airbnb data is loaded:

```bash
# Load new raw data
python scripts/load_inside_airbnb_to_snowflake.py

# Full refresh the incremental fact table and downstream models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Run tests
dbt test --profiles-dir .
```

### Run Streamlit Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

Dashboard connects using `config/local_credentials.json` and queries analytics marts.

## dbt Model Layers

### Source Tables (RAW Schema)

Defined in `models/staging/sources.yml`:

```yaml
version: 2

sources:
  - name: raw_airbnb
    schema: RAW
    tables:
      - name: LISTINGS
        description: "Raw listings data from Inside Airbnb"
      - name: CALENDAR
        description: "Raw calendar availability and pricing"
      - name: REVIEWS
        description: "Raw guest reviews"
      - name: NEIGHBOURHOODS
        description: "Neighbourhood geojson boundaries"
```

### Staging Models (`models/staging/`)

Clean, cast, and standardize raw data:

**Example: `stg_airbnb__listings.sql`**

```sql
WITH source AS (
    SELECT * FROM {{ source('raw_airbnb', 'LISTINGS') }}
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
        TRY_TO_DATE(last_review, 'YYYY-MM-DD') AS last_review_date,
        TRY_CAST(reviews_per_month AS FLOAT) AS reviews_per_month,
        TRY_CAST(calculated_host_listings_count AS INTEGER) AS host_listing_count,
        TRY_CAST(availability_365 AS INTEGER) AS availability_365
    FROM source
)

SELECT * FROM cleaned
WHERE listing_id IS NOT NULL
```

**Example: `stg_airbnb__calendar.sql`**

```sql
WITH source AS (
    SELECT * FROM {{ source('raw_airbnb', 'CALENDAR') }}
),

cleaned AS (
    SELECT
        TRY_CAST(listing_id AS INTEGER) AS listing_id,
        TRY_TO_DATE(date, 'YYYY-MM-DD') AS calendar_date,
        CASE 
            WHEN LOWER(TRIM(available)) = 't' THEN TRUE
            WHEN LOWER(TRIM(available)) = 'f' THEN FALSE
            ELSE NULL
        END AS is_available,
        TRY_CAST(REPLACE(REPLACE(price, '$', ''), ',', '') AS FLOAT) AS price
    FROM source
)

SELECT * FROM cleaned
WHERE listing_id IS NOT NULL AND calendar_date IS NOT NULL
```

### Intermediate Models (`models/intermediate/`)

Join and enrich staging models:

**Example: `int_airbnb__listing_enriched.sql`**

```sql
WITH listings AS (
    SELECT * FROM {{ ref('stg_airbnb__listings') }}
),

neighbourhoods AS (
    SELECT * FROM {{ ref('stg_airbnb__neighbourhoods') }}
)

SELECT
    l.*,
    n.neighbourhood_group
FROM listings l
LEFT JOIN neighbourhoods n
    ON l.neighbourhood = n.neighbourhood
```

**Example: `int_airbnb__calendar_enriched.sql`**

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
    c.price AS calendar_price,
    l.listing_name,
    l.neighbourhood,
    l.neighbourhood_group,
    l.room_type,
    -- Revenue proxy: price when unavailable (booked)
    CASE 
        WHEN c.is_available = FALSE THEN c.price 
        ELSE 0 
    END AS estimated_revenue
FROM calendar c
INNER JOIN listings l
    ON c.listing_id = l.listing_id
```

### Mart Models (`models/marts/`)

Analytics-ready dimensions and facts:

**Dimension: `dim_listings.sql`**

```sql
SELECT
    listing_id,
    listing_name,
    host_id,
    host_name,
    neighbourhood,
    neighbourhood_group,
    room_type,
    price,
    minimum_nights,
    number_of_reviews,
    last_review_date,
    reviews_per_month,
    host_listing_count,
    availability_365
FROM {{ ref('int_airbnb__listing_enriched') }}
```

**Fact (Incremental): `fct_listing_calendar.sql`**

```sql
{{
    config(
        materialized='incremental',
        unique_key='listing_calendar_key',
        merge_update_columns=['is_available', 'calendar_price', 'estimated_revenue']
    )
}}

WITH calendar_enriched AS (
    SELECT * FROM {{ ref('int_airbnb__calendar_enriched') }}
)

SELECT
    {{ dbt_utils.generate_surrogate_key(['listing_id', 'calendar_date']) }} AS listing_calendar_key,
    listing_id,
    calendar_date,
    is_available,
    calendar_price,
    neighbourhood,
    neighbourhood_group,
    room_type,
    estimated_revenue
FROM calendar_enriched

{% if is_incremental() %}
WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
{% endif %}
```

**Aggregate: `agg_listing_monthly_performance.sql`**

```sql
WITH calendar AS (
    SELECT * FROM {{ ref('fct_listing_calendar') }}
)

SELECT
    listing_id,
    DATE_TRUNC('month', calendar_date) AS month,
    neighbourhood,
    neighbourhood_group,
    room_type,
    COUNT(*) AS total_days,
    SUM(CASE WHEN is_available = TRUE THEN 1 ELSE 0 END) AS available_days,
    SUM(CASE WHEN is_available = FALSE THEN 1 ELSE 0 END) AS unavailable_days,
    ROUND(AVG(calendar_price), 2) AS avg_price,
    ROUND(SUM(estimated_revenue), 2) AS total_estimated_revenue,
    ROUND(
        SUM(CASE WHEN is_available = FALSE THEN 1 ELSE 0 END)::FLOAT / COUNT(*)::FLOAT * 100, 
        2
    ) AS occupancy_rate_pct
FROM calendar
GROUP BY 1, 2, 3, 4, 5
```

## Data Quality Tests

### Generic Tests in Schema YAML

**Example: `models/staging/schema.yml`**

```yaml
version: 2

models:
  - name: stg_airbnb__listings
    description: "Cleaned and typed listings"
    columns:
      - name: listing_id
        description: "Primary key"
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

  - name: stg_airbnb__calendar
    description: "Cleaned calendar availability"
    columns:
      - name: listing_id
        tests:
          - not_null
          - relationships:
              to: ref('stg_airbnb__listings')
              field: listing_id
      - name: calendar_date
        tests:
          - not_null
```

### Singular Tests

**Example: `tests/no_negative_prices.sql`**

```sql
SELECT
    listing_id,
    calendar_date,
    price
FROM {{ ref('fct_listing_calendar') }}
WHERE price < 0
```

**Example: `tests/no_duplicate_listing_dates.sql`**

```sql
SELECT
    listing_id,
    calendar_date,
    COUNT(*) AS occurrences
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, calendar_date
HAVING COUNT(*) > 1
```

## Python Loader Script Pattern

**Reading credentials:**

```python
import json
from pathlib import Path

def load_credentials():
    """Load Snowflake credentials from local config file."""
    creds_path = Path(__file__).parent.parent / 'config' / 'local_credentials.json'
    with open(creds_path, 'r') as f:
        return json.load(f)

creds = load_credentials()
```

**Connecting to Snowflake:**

```python
import snowflake.connector

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
```

**Uploading files to internal stage:**

```python
from pathlib import Path

data_dir = Path(__file__).parent.parent / 'data' / 'raw'

files = [
    'listings.csv.gz',
    'calendar.csv.gz',
    'reviews.csv.gz',
    'neighbourhoods.csv'
]

for file in files:
    file_path = data_dir / file
    cursor.execute(f"PUT file://{file_path} @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")
    print(f"Uploaded {file}")
```

**Creating table from CSV and copying data:**

```python
cursor.execute("""
    CREATE OR REPLACE TABLE RAW.LISTINGS
    USING TEMPLATE (
        SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*))
        FROM TABLE(
            INFER_SCHEMA(
                LOCATION => '@INSIDE_AIRBNB_STAGE/listings.csv.gz',
                FILE_FORMAT => 'CSV_FORMAT'
            )
        )
    )
""")

cursor.execute("""
    COPY INTO RAW.LISTINGS
    FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
    FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"')
    ON_ERROR = CONTINUE
""")
```

## Streamlit Dashboard Pattern

**Reading credentials:**

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json
from pathlib import Path

@st.cache_resource
def get_connection():
    creds_path = Path(__file__).parent.parent / 'config' / 'local_credentials.json'
    with open(creds_path, 'r') as f:
        creds = json.load(f)
    
    return snowflake.connector.connect(
        user=creds['user'],
        password=creds['password'],
        account=creds['account'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema'],
        role=creds['role']
    )
```

**Querying marts:**

```python
@st.cache_data
def load_neighbourhood_performance():
    conn = get_connection()
    query = """
        SELECT
            neighbourhood_group,
            neighbourhood,
            month,
            total_estimated_revenue,
            occupancy_rate_pct
        FROM agg_neighbourhood_monthly_performance
        ORDER BY total_estimated_revenue DESC
        LIMIT 100
    """
    return pd.read_sql(query, conn)

df = load_neighbourhood_performance()
st.dataframe(df)
```

## Common Workflows

### New Data Refresh Workflow

```bash
# 1. Download new Inside Airbnb data to data/raw/

# 2. Load into Snowflake
python scripts/load_inside_airbnb_to_snowflake.py

# 3. Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 4. Run all tests
dbt test --profiles-dir .

# 5. Regenerate documentation
dbt docs generate --profiles-dir .
```

### Development Workflow

```bash
# 1. Create new model in models/marts/
# 2. Add tests in schema.yml

# 3. Run only your new model
dbt run --select my_new_model --profiles-dir .

# 4. Test your model
dbt test --select my_new_model --profiles-dir .

# 5. Run downstream dependencies
dbt run --select my_new_model+ --profiles-dir .
```

### Selective Model Execution

```bash
# Run staging layer only
dbt run --select staging.* --profiles-dir .

# Run specific model and all upstream dependencies
dbt run --select +dim_listings --profiles-dir .

# Run specific model and all downstream dependencies
dbt run --select stg_airbnb__listings+ --profiles-dir .

# Run models by tag
dbt run --select tag:daily --profiles-dir .
```

## Troubleshooting

### dbt Connection Errors

**Problem:** `Credentials in profile "snowflake_dbt_project", target "dev" invalid`

**Solution:**
- Verify `profiles.yml` exists in project root (not `~/.dbt/`)
- Check credentials match your Snowflake account
- Ensure you're using `--profiles-dir .` flag
- Test with `dbt debug --profiles-dir .`

### Snowflake Loader Errors

**Problem:** `FileNotFoundError: config/local_credentials.json`

**Solution:**
```bash
cp config/local_credentials.example.json config/local_credentials.json
# Edit with your credentials
```

**Problem:** `snowflake.connector.errors.ProgrammingError: 002003: SQL compilation error: Database 'AIRBNB_DB' does not exist`

**Solution:**
The loader creates objects automatically. Check:
- Your role has `CREATE DATABASE` privilege
- The `setup/snowflake_setup.sql` executed successfully
- Review loader script output for errors

### Incremental Model Issues

**Problem:** Incremental model not picking up new records

**Solution:**
```bash
# Force full refresh
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

**Problem:** Duplicate records in incremental model

**Solution:**
- Check `unique_key` configuration in model config
- Verify source data doesn't have duplicates:
  ```bash
  dbt test --select no_duplicate_listing_dates --profiles-dir .
  ```

### Streamlit Dashboard Errors

**Problem:** `KeyError: 'user'` when launching dashboard

**Solution:**
Ensure `config/local_credentials.json` has all required keys:
```json
{
  "user": "...",
  "password": "...",
  "account": "...",
  "warehouse": "...",
  "database": "...",
  "schema": "...",
  "role": "..."
}
```

**Problem:** Dashboard shows no data

**Solution:**
- Verify dbt models have run successfully
- Check schema name in credentials matches dbt target schema
- Query Snowflake directly to verify data exists:
  ```sql
  SELECT COUNT(*) FROM AIRBNB_DB.ANALYTICS.AGG_NEIGHBOURHOOD_MONTHLY_PERFORMANCE;
  ```

## Best Practices

1. **Never commit credentials** - Use local-only config files in `.gitignore`
2. **Use `--profiles-dir .`** - Keep profiles.yml in project root for portability
3. **Test incrementally** - Run `dbt test --select model_name` during development
4. **Document models** - Add descriptions in schema.yml files
5. **Use intermediate models** - Separate reusable logic from final marts
6. **Tag models** - Use tags for selective execution (e.g., `tags: ['daily', 'core']`)
7. **Full refresh carefully** - Incremental models can be expensive to rebuild

## Example Business Queries

**Top neighbourhoods by revenue:**

```sql
SELECT
    neighbourhood,
    SUM(total_estimated_revenue) AS total_revenue
FROM AIRBNB_DB.ANALYTICS.AGG_NEIGHBOURHOOD_MONTHLY_PERFORMANCE
GROUP BY neighbourhood
ORDER BY total_revenue DESC
LIMIT 10;
```

**Hosts with most listings:**

```sql
SELECT
    host_id,
    host_name,
    COUNT(DISTINCT listing_id) AS listing_count
FROM AIRBNB_DB.ANALYTICS.DIM_LISTINGS
GROUP BY host_id, host_name
ORDER BY listing_count DESC
LIMIT 10;
```

**Average price by room type:**

```sql
SELECT
    room_type,
    ROUND(AVG(price), 2) AS avg_price
FROM AIRBNB_DB.ANALYTICS.DIM_LISTINGS
WHERE price > 0
GROUP BY room_type
ORDER BY avg_price DESC;
```
