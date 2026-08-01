---
name: zomato-ai-data-engineering-pipeline
description: End-to-end Zomato food delivery data pipeline with S3, Snowflake, dbt, Airflow, and OpenAI for AI-powered analytics
triggers:
  - "build a zomato data pipeline"
  - "set up snowflake with dbt and airflow"
  - "create an AI-powered data warehouse"
  - "implement LLM enrichment in a data pipeline"
  - "build RAG for business analytics"
  - "set up text-to-SQL with OpenAI"
  - "orchestrate dbt with airflow"
  - "create a medallion architecture pipeline"
---

# Zomato AI Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A complete batch data pipeline for food delivery analytics using S3 → Snowflake → dbt → Airflow → OpenAI. Implements medallion architecture (Bronze/Silver/Gold), incremental fact tables, LLM enrichment, RAG chat, and text-to-SQL capabilities.

## What It Does

- **Data Lake**: Uploads CSV data to S3 organized by table
- **Bronze Layer**: Loads raw data into Snowflake via `COPY INTO` using keyless storage integration
- **Silver Layer**: dbt staging views for data cleaning and typing
- **Gold Layer**: Dimensional models, incremental fact tables (MERGE), business marts, SCD2 snapshots
- **AI Layer**: LLM enrichment for sentiment/topic extraction, RAG for review chat, text-to-SQL
- **Orchestration**: Airflow DAG running the full pipeline daily

## Architecture

```
CSVs → S3 → Snowflake RAW → dbt STAGING → dbt MARTS → AI enrichment → BI/Apps
                ↑                              ↓
              Airflow orchestrates everything
```

## Installation

### Prerequisites

```bash
# Required services
- AWS account with S3 bucket
- Snowflake account
- OpenAI API key
- Docker & Docker Compose (for Airflow)
```

### 1. Clone and Setup Data

```bash
git clone https://github.com/darshilparmar/zomato-ai-data-engineering-end-to-end-project.git
cd zomato-ai-data-engineering-end-to-end-project

# Download dataset from Google Drive (link in README) to data/ directory
# Expected files: restaurants.csv, users.csv, food.csv, menu.csv, 
#                 orders.csv, order_items.csv, reviews.csv
```

### 2. AWS S3 Setup

```bash
# Create S3 bucket
aws s3 mb s3://your-zomato-bucket

# Upload data to organized structure
aws s3 cp data/restaurants.csv s3://your-zomato-bucket/raw/restaurants/
aws s3 cp data/users.csv s3://your-zomato-bucket/raw/users/
aws s3 cp data/food.csv s3://your-zomato-bucket/raw/food/
aws s3 cp data/menu.csv s3://your-zomato-bucket/raw/menu/
aws s3 cp data/orders.csv s3://your-zomato-bucket/raw/orders/
aws s3 cp data/order_items.csv s3://your-zomato-bucket/raw/order_items/
aws s3 cp data/reviews.csv s3://your-zomato-bucket/raw/reviews/
```

### 3. AWS IAM Configuration

Create IAM policy for S3 read access (use `aws/iam/s3-read-policy.json`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::your-zomato-bucket",
        "arn:aws:s3:::your-zomato-bucket/*"
      ]
    }
  ]
}
```

Create IAM role `snowflake-s3-role` with initial trust policy:

```bash
# Attach the s3-read-policy to the role
# Initial trust policy uses placeholder - will update after Snowflake integration
```

### 4. Snowflake Setup

```sql
-- Create warehouse and database
CREATE WAREHOUSE ZOMATO_WH WITH WAREHOUSE_SIZE = 'MEDIUM' AUTO_SUSPEND = 60;
CREATE DATABASE ZOMATO;

-- Create schemas for medallion architecture
CREATE SCHEMA ZOMATO.RAW;
CREATE SCHEMA ZOMATO.STAGING;
CREATE SCHEMA ZOMATO.MARTS;
CREATE SCHEMA ZOMATO.SNAPSHOTS;
CREATE SCHEMA ZOMATO.AI;

-- Create role for dbt
CREATE ROLE DBT_ROLE;
GRANT USAGE ON WAREHOUSE ZOMATO_WH TO ROLE DBT_ROLE;
GRANT ALL ON DATABASE ZOMATO TO ROLE DBT_ROLE;
GRANT ALL ON ALL SCHEMAS IN DATABASE ZOMATO TO ROLE DBT_ROLE;
GRANT ROLE DBT_ROLE TO USER your_user;

-- Create storage integration (replace with your IAM role ARN)
CREATE STORAGE INTEGRATION s3_zomato_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'S3'
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::YOUR_ACCOUNT:role/snowflake-s3-role'
  STORAGE_ALLOWED_LOCATIONS = ('s3://your-zomato-bucket/raw/');

-- Get Snowflake IAM user and external ID
DESC STORAGE INTEGRATION s3_zomato_integration;
-- Copy STORAGE_AWS_IAM_USER_ARN and STORAGE_AWS_EXTERNAL_ID
```

Update IAM role trust policy with values from `DESC INTEGRATION`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/abc-snowflake"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "ABC123_SFCRole=1_xYz="
        }
      }
    }
  ]
}
```

Create external stages:

```sql
USE SCHEMA ZOMATO.RAW;

CREATE STAGE s3_restaurants_stage
  URL = 's3://your-zomato-bucket/raw/restaurants/'
  STORAGE_INTEGRATION = s3_zomato_integration
  FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1);

CREATE STAGE s3_users_stage
  URL = 's3://your-zomato-bucket/raw/users/'
  STORAGE_INTEGRATION = s3_zomato_integration
  FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1);

-- Repeat for food, menu, orders, order_items, reviews
```

Create RAW tables:

```sql
-- Example for restaurants
CREATE TABLE ZOMATO.RAW.restaurants (
    restaurant_id INTEGER,
    restaurant_name VARCHAR,
    city VARCHAR,
    address VARCHAR,
    locality VARCHAR,
    latitude FLOAT,
    longitude FLOAT,
    cuisines VARCHAR,
    average_cost_for_two VARCHAR,
    currency VARCHAR,
    has_table_booking VARCHAR,
    has_online_delivery VARCHAR,
    is_delivering_now VARCHAR,
    price_range INTEGER,
    aggregate_rating FLOAT,
    rating_text VARCHAR,
    votes INTEGER
);

-- Similar CREATE TABLE for users, food, menu, orders, order_items, reviews
```

### 5. dbt Setup

```bash
cd zomato

# Install dbt
pip install dbt-snowflake

# Configure profiles.yml (or use env vars)
export SNOWFLAKE_ACCOUNT=your-account
export SNOWFLAKE_USER=your-user
export SNOWFLAKE_PASSWORD=your-password
export SNOWFLAKE_ROLE=DBT_ROLE
export SNOWFLAKE_WAREHOUSE=ZOMATO_WH
export SNOWFLAKE_DATABASE=ZOMATO

# Test connection
dbt debug

# Run models
dbt build --exclude tag:ai
```

Example `profiles.yml` using env vars:

```yaml
zomato:
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
      schema: staging
      threads: 4
```

### 6. Airflow Setup

```bash
cd airflow

# Copy and configure environment
cp example.env .env

# Edit .env with your credentials:
# SNOWFLAKE_ACCOUNT=xyz12345
# SNOWFLAKE_USER=your_user
# SNOWFLAKE_PASSWORD=your_password
# SNOWFLAKE_ROLE=DBT_ROLE
# SNOWFLAKE_WAREHOUSE=ZOMATO_WH
# SNOWFLAKE_DATABASE=ZOMATO
# OPENAI_API_KEY=sk-...
# SAMPLE_N=1000

# Build and start Airflow
docker compose build
docker compose up -d

# Access UI at http://localhost:8080
# Default credentials: airflow/airflow
```

## Key Commands

### dbt Commands

```bash
# Test connection
dbt debug

# Compile models without running
dbt compile

# Run all models
dbt run

# Run tests only
dbt test

# Run models and tests in dependency order
dbt build

# Run specific models
dbt run --select staging
dbt run --select marts
dbt run --select fct_orders

# Run incremental models with full-refresh
dbt run --select fct_orders --full-refresh

# Generate and serve documentation
dbt docs generate
dbt docs serve

# Exclude AI models
dbt build --exclude tag:ai
```

### Airflow DAG

The main DAG `zomato_batch` has 4 tasks:

1. **reload_raw**: `COPY INTO` from S3 stages
2. **dbt_build_core**: Runs `dbt build --exclude tag:ai`
3. **enrich_reviews**: OpenAI LLM enrichment
4. **dbt_build_ai**: Builds AI marts

Trigger manually or runs daily at scheduled time.

### AI Scripts

```bash
# LLM enrichment (creates ZOMATO.AI.REVIEW_ENRICHED)
export OPENAI_API_KEY=sk-...
export SNOWFLAKE_ACCOUNT=...
export SNOWFLAKE_USER=...
export SNOWFLAKE_PASSWORD=...
python ai/enrich_reviews.py

# RAG chat with reviews
streamlit run ai/rag_chat.py

# Text-to-SQL warehouse chat
streamlit run ai/text_to_sql.py
```

## dbt Model Structure

### Sources (`models/staging/sources.yml`)

```yaml
version: 2

sources:
  - name: raw
    database: ZOMATO
    schema: RAW
    tables:
      - name: restaurants
      - name: users
      - name: food
      - name: menu
      - name: orders
      - name: order_items
      - name: reviews
```

### Staging Models (Silver)

Example: `models/staging/stg_restaurants.sql`

```sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'restaurants') }}
),

cleaned AS (
    SELECT
        restaurant_id,
        restaurant_name,
        city,
        address,
        locality,
        latitude,
        longitude,
        cuisines,
        -- Clean average_cost_for_two: '₹ 200' -> 200, '--' -> null
        CASE 
            WHEN average_cost_for_two = '--' THEN NULL
            ELSE TRY_CAST(REGEXP_REPLACE(average_cost_for_two, '[^0-9]', '') AS INTEGER)
        END AS average_cost_for_two,
        currency,
        -- Convert yes/no to boolean
        CASE WHEN LOWER(has_table_booking) = 'yes' THEN TRUE ELSE FALSE END AS has_table_booking,
        CASE WHEN LOWER(has_online_delivery) = 'yes' THEN TRUE ELSE FALSE END AS has_online_delivery,
        CASE WHEN LOWER(is_delivering_now) = 'yes' THEN TRUE ELSE FALSE END AS is_delivering_now,
        price_range,
        aggregate_rating,
        rating_text,
        votes
    FROM source
)

SELECT * FROM cleaned
```

### Dimensional Models (Gold)

Example: `models/marts/dim_customer.sql`

```sql
WITH customers AS (
    SELECT * FROM {{ ref('stg_users') }}
),

final AS (
    SELECT
        user_id AS customer_key,
        user_name AS customer_name,
        LOWER(email) AS email,
        phone,
        address,
        city,
        state,
        country,
        zipcode,
        age,
        gender,
        -- Age segmentation
        CASE
            WHEN age < 18 THEN 'Under 18'
            WHEN age BETWEEN 18 AND 24 THEN '18-24'
            WHEN age BETWEEN 25 AND 34 THEN '25-34'
            WHEN age BETWEEN 35 AND 44 THEN '35-44'
            WHEN age BETWEEN 45 AND 54 THEN '45-54'
            WHEN age >= 55 THEN '55+'
        END AS age_segment,
        registration_date
    FROM customers
)

SELECT * FROM final
```

### Incremental Fact Models

Example: `models/marts/fct_orders.sql`

```sql
{{
    config(
        materialized='incremental',
        unique_key='order_id',
        on_schema_change='append_new_columns'
    )
}}

WITH orders AS (
    SELECT * FROM {{ ref('stg_orders') }}
    {% if is_incremental() %}
    WHERE order_date >= (SELECT MAX(order_date) FROM {{ this }})
    {% endif %}
),

final AS (
    SELECT
        order_id,
        user_id AS customer_key,
        restaurant_id AS restaurant_key,
        order_date,
        order_time,
        delivery_date,
        delivery_time,
        order_status,
        -- Derive is_delivered flag
        CASE WHEN LOWER(order_status) = 'delivered' THEN TRUE ELSE FALSE END AS is_delivered,
        total_amount,
        discount_amount,
        final_amount,
        payment_method,
        rating,
        delivery_rating
    FROM orders
)

SELECT * FROM final
```

### Business Marts

Example: `models/marts/mart_daily_city_revenue.sql`

```sql
WITH orders AS (
    SELECT * FROM {{ ref('fct_orders') }}
),

restaurants AS (
    SELECT * FROM {{ ref('dim_restaurants') }}
),

daily_metrics AS (
    SELECT
        o.order_date,
        r.city,
        COUNT(DISTINCT o.order_id) AS total_orders,
        COUNT(DISTINCT CASE WHEN o.is_delivered THEN o.order_id END) AS delivered_orders,
        COUNT(DISTINCT CASE WHEN NOT o.is_delivered THEN o.order_id END) AS cancelled_orders,
        SUM(o.final_amount) AS gmv,
        AVG(o.final_amount) AS aov,
        -- Cancellation rate
        ROUND(
            COUNT(DISTINCT CASE WHEN NOT o.is_delivered THEN o.order_id END) * 100.0 / 
            NULLIF(COUNT(DISTINCT o.order_id), 0), 
            2
        ) AS cancellation_rate_pct
    FROM orders o
    JOIN restaurants r ON o.restaurant_key = r.restaurant_key
    GROUP BY 1, 2
)

SELECT * FROM daily_metrics
```

## AI Layer Implementation

### LLM Enrichment

`ai/enrich_reviews.py` - Extracts sentiment and topics from reviews:

```python
import os
import snowflake.connector
from openai import OpenAI
import json

# Configuration from env vars
SAMPLE_N = int(os.getenv('SAMPLE_N', 1000))

# Connect to Snowflake
conn = snowflake.connector.connect(
    account=os.getenv('SNOWFLAKE_ACCOUNT'),
    user=os.getenv('SNOWFLAKE_USER'),
    password=os.getenv('SNOWFLAKE_PASSWORD'),
    warehouse='ZOMATO_WH',
    database='ZOMATO',
    schema='RAW'
)

# OpenAI client
client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))

# Fetch reviews not yet enriched
query = f"""
SELECT review_id, review_text
FROM ZOMATO.RAW.reviews r
WHERE NOT EXISTS (
    SELECT 1 FROM ZOMATO.AI.review_enriched e
    WHERE e.review_id = r.review_id
)
LIMIT {SAMPLE_N}
"""

cursor = conn.cursor()
cursor.execute(query)
reviews = cursor.fetchall()

# Enrich each review
enriched = []
for review_id, review_text in reviews:
    try:
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "Extract sentiment (positive/negative/neutral) and main topic from food delivery review. Return JSON: {\"sentiment\": \"...\", \"topic\": \"...\"}"},
                {"role": "user", "content": review_text}
            ],
            temperature=0
        )
        
        result = json.loads(response.choices[0].message.content)
        enriched.append({
            'review_id': review_id,
            'sentiment': result['sentiment'],
            'topic': result['topic']
        })
        print(f"Enriched review {review_id}")
    except Exception as e:
        print(f"Error enriching {review_id}: {e}")

# Write back to Snowflake
if enriched:
    cursor.execute("CREATE SCHEMA IF NOT EXISTS ZOMATO.AI")
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS ZOMATO.AI.review_enriched (
            review_id INTEGER,
            sentiment VARCHAR,
            topic VARCHAR,
            enriched_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
        )
    """)
    
    for item in enriched:
        cursor.execute("""
            INSERT INTO ZOMATO.AI.review_enriched (review_id, sentiment, topic)
            VALUES (%s, %s, %s)
        """, (item['review_id'], item['sentiment'], item['topic']))
    
    conn.commit()
    print(f"Inserted {len(enriched)} enriched reviews")

cursor.close()
conn.close()
```

### RAG Chat

`ai/rag_chat.py` - Chat interface for reviews:

```python
import streamlit as st
import snowflake.connector
from openai import OpenAI
import numpy as np
import os

# Initialize
client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))
conn = snowflake.connector.connect(
    account=os.getenv('SNOWFLAKE_ACCOUNT'),
    user=os.getenv('SNOWFLAKE_USER'),
    password=os.getenv('SNOWFLAKE_PASSWORD'),
    warehouse='ZOMATO_WH',
    database='ZOMATO'
)

st.title("🍕 Chat with Zomato Reviews")

# Embed and store reviews (one-time setup)
if 'embeddings_loaded' not in st.session_state:
    cursor = conn.cursor()
    cursor.execute("""
        SELECT review_id, review_text, rating
        FROM ZOMATO.RAW.reviews
        LIMIT 5000
    """)
    reviews = cursor.fetchall()
    
    # Generate embeddings
    texts = [r[1] for r in reviews]
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=texts
    )
    
    st.session_state.reviews = reviews
    st.session_state.embeddings = np.array([e.embedding for e in response.data])
    st.session_state.embeddings_loaded = True
    cursor.close()

# Chat interface
question = st.text_input("Ask about the reviews:")

if question:
    # Embed question
    q_embedding = client.embeddings.create(
        model="text-embedding-3-small",
        input=[question]
    ).data[0].embedding
    
    # Find similar reviews (cosine similarity)
    similarities = np.dot(st.session_state.embeddings, q_embedding)
    top_5_idx = np.argsort(similarities)[-5:][::-1]
    
    context_reviews = [st.session_state.reviews[i] for i in top_5_idx]
    context_text = "\n\n".join([f"Review (rating {r[2]}): {r[1]}" for r in context_reviews])
    
    # Generate answer
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "Answer questions based on the provided Zomato reviews. Cite specific reviews."},
            {"role": "user", "content": f"Context:\n{context_text}\n\nQuestion: {question}"}
        ]
    )
    
    st.write("### Answer")
    st.write(response.choices[0].message.content)
    
    st.write("### Source Reviews")
    for rev in context_reviews:
        st.write(f"- Rating {rev[2]}: {rev[1][:200]}...")

conn.close()
```

### Text-to-SQL

`ai/text_to_sql.py` - Query warehouse in natural language:

```python
import streamlit as st
import snowflake.connector
from openai import OpenAI
import os
import re

client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))
conn = snowflake.connector.connect(
    account=os.getenv('SNOWFLAKE_ACCOUNT'),
    user=os.getenv('SNOWFLAKE_USER'),
    password=os.getenv('SNOWFLAKE_PASSWORD'),
    warehouse='ZOMATO_WH',
    database='ZOMATO',
    schema='MARTS',
    role='DBT_ROLE'
)

# Get schema context
cursor = conn.cursor()
cursor.execute("""
    SELECT table_name, column_name, data_type
    FROM ZOMATO.information_schema.columns
    WHERE table_schema = 'MARTS'
    ORDER BY table_name, ordinal_position
""")
schema_info = cursor.fetchall()

schema_context = "Available tables and columns:\n"
current_table = None
for table, column, dtype in schema_info:
    if table != current_table:
        schema_context += f"\n{table}:\n"
        current_table = table
    schema_context += f"  - {column} ({dtype})\n"

st.title("💬 Chat with Zomato Data Warehouse")

question = st.text_input("Ask a question about the data:")

if question:
    # Generate SQL
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": f"You are a Snowflake SQL expert. Generate SELECT-only queries for the ZOMATO.MARTS schema.\n\n{schema_context}"},
            {"role": "user", "content": f"Write a SQL query to answer: {question}"}
        ]
    )
    
    sql = response.choices[0].message.content
    # Extract SQL from markdown if present
    sql_match = re.search(r'```sql\n(.*?)\n```', sql, re.DOTALL)
    if sql_match:
        sql = sql_match.group(1)
    
    st.code(sql, language='sql')
    
    # Validate: only SELECT allowed
    if not sql.strip().upper().startswith('SELECT'):
        st.error("Only SELECT queries allowed")
    else:
        try:
            cursor.execute(sql)
            results = cursor.fetchall()
            columns = [desc[0] for desc in cursor.description]
            
            st.write(f"### Results ({len(results)} rows)")
            st.dataframe(results, columns=columns)
        except Exception as e:
            st.error(f"Query error: {e}")

cursor.close()
conn.close()
```

## Airflow DAG Structure

`airflow/dags/zomato_batch.py`:

```python
from airflow.decorators import dag, task
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from airflow.operators.bash import BashOperator
from datetime import datetime
import os

@dag(
    dag_id='zomato_batch',
    start_date=datetime(2024, 1, 1),
    schedule='@daily',
    catchup=False,
    default_args={'owner': 'data-eng'}
)
def zomato_pipeline():
    
    # Task 1: Reload raw tables from S3
    reload_raw = SnowflakeOperator(
        task_id='reload_raw',
        snowflake_conn_id='snowflake_default',
        sql="""
            USE SCHEMA ZOMATO.RAW;
            
            COPY INTO restaurants FROM @s3_restaurants_stage 
            FILE_FORMAT = (TYPE='CSV' SKIP_HEADER=1) 
            FORCE = TRUE;
            
            COPY INTO users FROM @s3_users_stage 
            FILE_FORMAT = (TYPE='CSV' SKIP_HEADER=1) 
            FORCE = TRUE;
            
            COPY INTO food FROM @s3_food_stage 
            FILE_FORMAT = (TYPE='CSV' SKIP_HEADER=1) 
            FORCE = TRUE;
            
            COPY INTO menu FROM @s3_menu_stage 
            FILE_FORMAT = (TYPE='CSV' SKIP_HEADER=1) 
            FORCE = TRUE;
            
            COPY INTO orders FROM @s3_orders_stage 
            FILE_FORMAT = (TYPE='CSV' SKIP_HEADER=1) 
            FORCE = TRUE;
            
            COPY INTO order_items FROM @s3_order_items_stage 
            FILE_FORMAT = (TYPE='CSV' SKIP_HEADER=1) 
            FORCE = TRUE;
            
            COPY INTO reviews FROM @s3_reviews_stage 
            FILE_FORMAT = (TYPE='CSV' SKIP_HEADER=1) 
            FORCE = TRUE;
        """
    )
    
    # Task 2: dbt build (core models, exclude AI)
    dbt_build_core = BashOperator(
        task_id='dbt_build_core',
        bash_command='cd /opt/airflow/zomato && dbt build --exclude tag:ai'
    )
    
    # Task 3: Enrich reviews with OpenAI
    enrich_reviews = BashOperator(
        task_id='enrich_reviews',
        bash_command='python /opt/airflow/ai/enrich_reviews.py',
        env={
            'OPENAI_API_KEY': os.getenv('OPENAI_API_KEY'),
            'SNOWFLAKE_ACCOUNT': os.getenv('SNOWFLAKE_ACCOUNT'),
            'SNOWFLAKE_USER': os.getenv('SNOWFLAKE_USER'),
            'SNOWFLAKE_PASSWORD': os.getenv('SNOWFLAKE_PASSWORD'),
            'SAMPLE_N': os.getenv('SAMPLE_N', '1000')
        }
    )
    
    # Task 4: dbt build AI marts
    dbt_build_ai = BashOperator(
        task_id='dbt_build_ai',
        bash_command='cd /opt/airflow/zomato && dbt build --select tag:ai'
    )
    
    # Dependencies
    reload_raw >> dbt_build_core >> enrich_reviews >> dbt_build_ai

zomato_pipeline()
```

## Configuration

### Environment Variables

All credentials via environment variables:

```bash
# Snowflake
SNOWFLAKE_ACCOUNT=abc12345.us-east-1
SNOWFLAKE_USER=your_user
SNOWFLAKE_PASSWORD=your_password
SNOWFLAKE_ROLE=DBT_ROLE
SNOWFLAKE_WAREHOUSE=ZOMATO_WH
SNOWFLAKE_DATABASE=ZOMATO

# OpenAI
OPENAI_API_KEY=sk-proj-...

# AI processing
SAMPLE_N=1000  # Number of reviews to enrich per run
```

### dbt Configuration

`zomato/dbt_project.yml`:

```yaml
name: 'zomato'
version: '1.0.0'
config-version: 2

profile: 'zomato'

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

clean-targets:
  - "target"
  - "dbt_packages"
  - "logs"

models:
  zomato:
    staging:
      +materialized: view
      +schema: staging
    marts:
      +materialized: table
      +schema: marts
      +tags: ['marts']
    ai:
      +materialized: table
      +schema: ai
      +tags: ['ai']
```

Custom schema macro (`macros/generate_schema_name.sql`):

```sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}
    {%- if custom_schema_name is none -%}
        {{ default_schema }}
    {%- else -%}
        {{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

## Common Patterns

### Incremental Loading Pattern

For large fact tables, use incremental materialization:

```sql
{{
    config(
        materialized='incremental',
        unique_key='order_id',
        on_schema_change='append_new_columns'
    )
}}

SELECT *
FROM {{ ref('stg_orders') }}
{% if is_incremental() %}
WHERE order_date >= (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

Full refresh when needed:

```bash
dbt run --select fct_orders --full-refresh
```

### Testing Pattern

Define tests in schema YAML:

```yaml
version: 2

models:
  - name: fct_orders
    columns:
      - name:
