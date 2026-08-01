---
name: zomato-ai-data-engineering-pipeline
description: End-to-end batch data pipeline with Snowflake, dbt, Airflow, and AI-powered analytics for food delivery data
triggers:
  - build a zomato data pipeline
  - set up medallion architecture with dbt
  - configure snowflake s3 storage integration
  - orchestrate data pipeline with airflow
  - implement llm enrichment in data pipeline
  - create rag chat with warehouse data
  - build text to sql with openai
  - deploy incremental dbt models
---

# zomato-ai-data-engineering-pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

Complete batch data engineering pipeline that processes food delivery data through a medallion architecture (Bronze → Silver → Gold) using Snowflake, dbt, Airflow orchestration, and AI capabilities (LLM enrichment, RAG, text-to-SQL).

## What This Project Does

- **Data Lake**: Stores raw CSVs in Amazon S3 (restaurants, users, orders, reviews)
- **Bronze Layer**: Loads raw data from S3 to Snowflake via `COPY INTO` with keyless storage integration
- **Silver Layer**: dbt staging views that clean and standardize data
- **Gold Layer**: Business-ready dimensions, incremental facts (10M+ orders), and analytical marts
- **Orchestration**: Airflow DAG that runs the complete pipeline daily
- **AI Capabilities**: LLM enrichment of reviews, RAG chat, and text-to-SQL query interface

## Architecture Overview

```
CSV Files → S3 → Snowflake RAW → dbt STAGING → dbt MARTS → AI Layer
                                                              ↓
                                                    Streamlit Dashboards
```

**Key Components:**
- 7 source tables (4 dimensions, 3 facts: 10M orders, 23M order items, 300K reviews)
- Keyless S3→Snowflake integration via IAM role trust
- Incremental dbt models with MERGE strategy
- Daily Airflow orchestration
- OpenAI-powered analytics (gpt-4o-mini, text-embedding-3-small)

## Installation & Setup

### Prerequisites

```bash
# Required tools
- Python 3.8+
- Docker & Docker Compose (for Airflow)
- dbt-core with dbt-snowflake adapter
- AWS account (for S3)
- Snowflake account
- OpenAI API key
```

### Clone and Prepare Data

```bash
git clone https://github.com/darshilparmar/zomato-ai-data-engineering-end-to-end-project.git
cd zomato-ai-data-engineering-end-to-end-project

# Download dataset from Google Drive (link in README)
# Place CSVs in data/ directory:
# data/restaurants.csv, data/users.csv, data/food.csv, data/menu.csv
# data/orders.csv, data/order_items.csv, data/reviews.csv
```

### AWS S3 Setup

1. **Create S3 bucket** and upload CSVs to folder structure:
```
s3://your-bucket/raw/restaurants/
s3://your-bucket/raw/users/
s3://your-bucket/raw/food/
s3://your-bucket/raw/menu/
s3://your-bucket/raw/orders/
s3://your-bucket/raw/order_items/
s3://your-bucket/raw/reviews/
```

2. **Create IAM policy** (`zomato-s3-read`) using `aws/iam/s3-read-policy.json`:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:GetObjectVersion"],
      "Resource": "arn:aws:s3:::your-bucket/raw/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::your-bucket"
    }
  ]
}
```

3. **Create IAM role** (`snowflake-s3-role`) with initial trust policy, attach the policy

### Snowflake Setup

```sql
-- Create warehouse, database, schemas
CREATE WAREHOUSE ZOMATO_WH WITH WAREHOUSE_SIZE = 'LARGE';
CREATE DATABASE ZOMATO;
CREATE SCHEMA ZOMATO.RAW;
CREATE SCHEMA ZOMATO.STAGING;
CREATE SCHEMA ZOMATO.MARTS;
CREATE SCHEMA ZOMATO.SNAPSHOTS;
CREATE SCHEMA ZOMATO.AI;

-- Create dbt role with permissions
CREATE ROLE DBT_ROLE;
GRANT USAGE ON WAREHOUSE ZOMATO_WH TO ROLE DBT_ROLE;
GRANT ALL ON DATABASE ZOMATO TO ROLE DBT_ROLE;
GRANT ALL ON ALL SCHEMAS IN DATABASE ZOMATO TO ROLE DBT_ROLE;
GRANT ALL ON FUTURE SCHEMAS IN DATABASE ZOMATO TO ROLE DBT_ROLE;

-- Create storage integration (replace YOUR_IAM_ROLE_ARN)
CREATE STORAGE INTEGRATION ZOMATO_S3_INT
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'S3'
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake-s3-role'
  STORAGE_ALLOWED_LOCATIONS = ('s3://your-bucket/raw/');

-- Get Snowflake's IAM user and external ID
DESC STORAGE INTEGRATION ZOMATO_S3_INT;
-- Note STORAGE_AWS_IAM_USER_ARN and STORAGE_AWS_EXTERNAL_ID
```

4. **Update IAM role trust policy** with values from `DESC INTEGRATION`:
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
          "sts:ExternalId": "ABC123_SFCRole=1_xyz"
        }
      }
    }
  ]
}
```

### Create Snowflake Stages and Tables

```sql
-- Create stages for each table
USE SCHEMA ZOMATO.RAW;

CREATE STAGE restaurants_stage
  STORAGE_INTEGRATION = ZOMATO_S3_INT
  URL = 's3://your-bucket/raw/restaurants/'
  FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1);

-- Repeat for users, food, menu, orders, order_items, reviews

-- Create raw tables
CREATE TABLE restaurants (
  restaurant_id INT,
  restaurant_name STRING,
  city STRING,
  address STRING,
  rating FLOAT,
  rating_count INT,
  cost_for_two STRING,
  cuisine_type STRING,
  lic_no STRING,
  link STRING,
  menu STRING
);

-- Create other raw tables (users, food, menu, orders, order_items, reviews)
-- See Snowflake schema definitions in the project
```

## dbt Configuration

### dbt Profile Setup

Create/edit `~/.dbt/profiles.yml`:

```yaml
zomato:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: DBT_ROLE
      database: ZOMATO
      warehouse: ZOMATO_WH
      schema: STAGING
      threads: 4
```

### Environment Variables

```bash
export SNOWFLAKE_ACCOUNT=your_account.region
export SNOWFLAKE_USER=your_user
export SNOWFLAKE_PASSWORD=your_password
```

### dbt Project Structure

```
zomato/
├── dbt_project.yml
├── models/
│   ├── sources.yml          # Source definitions
│   ├── staging/
│   │   ├── stg_restaurants.sql
│   │   ├── stg_users.sql
│   │   ├── stg_food.sql
│   │   ├── stg_menu.sql
│   │   ├── stg_orders.sql
│   │   ├── stg_order_items.sql
│   │   └── stg_reviews.sql
│   └── marts/
│       ├── dimensions/
│       │   ├── dim_restaurants.sql
│       │   ├── dim_customer.sql
│       │   ├── dim_food.sql
│       │   └── dim_date.sql
│       ├── facts/
│       │   ├── fct_orders.sql
│       │   └── fct_order_items.sql
│       └── business/
│           ├── mart_daily_city_metrics.sql
│           ├── mart_restaurant_performance.sql
│           ├── mart_delivery_sla.sql
│           └── mart_review_insights.sql
└── macros/
    └── generate_schema_name.sql
```

### Key dbt Commands

```bash
cd zomato

# Test connection
dbt debug

# Run staging models only
dbt run --select staging

# Build everything (run + test)
dbt build

# Build excluding AI models
dbt build --exclude tag:ai

# Run incremental models with full refresh
dbt run --select fct_orders --full-refresh

# Test data quality
dbt test

# Generate and serve documentation
dbt docs generate
dbt docs serve
```

### Example Staging Model

```sql
-- models/staging/stg_restaurants.sql
WITH source AS (
    SELECT * FROM {{ source('zomato_raw', 'restaurants') }}
),

cleaned AS (
    SELECT
        restaurant_id,
        TRIM(restaurant_name) AS restaurant_name,
        LOWER(TRIM(city)) AS city,
        TRIM(address) AS address,
        rating,
        rating_count,
        -- Parse "₹ 200" to 200
        CAST(
            NULLIF(
                REGEXP_REPLACE(cost_for_two, '[^0-9]', ''),
                ''
            ) AS INT
        ) AS cost_for_two,
        TRIM(cuisine_type) AS cuisine_type,
        NULLIF(TRIM(lic_no), '--') AS lic_no,
        link,
        menu
    FROM source
)

SELECT * FROM cleaned
```

### Example Incremental Fact Model

```sql
-- models/marts/facts/fct_orders.sql
{{
    config(
        materialized='incremental',
        unique_key='order_id',
        on_schema_change='fail'
    )
}}

WITH orders AS (
    SELECT
        order_id,
        customer_id,
        restaurant_id,
        order_date,
        order_time,
        order_status,
        total_amount,
        delivery_time_minutes,
        delivery_rating,
        -- Derive business flags
        CASE 
            WHEN order_status = 'Delivered' THEN TRUE 
            ELSE FALSE 
        END AS is_delivered,
        CASE 
            WHEN order_status = 'Cancelled' THEN TRUE 
            ELSE FALSE 
        END AS is_cancelled
    FROM {{ ref('stg_orders') }}
    {% if is_incremental() %}
        WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
    {% endif %}
)

SELECT * FROM orders
```

### Example Business Mart

```sql
-- models/marts/business/mart_daily_city_metrics.sql
WITH daily_orders AS (
    SELECT
        order_date,
        city,
        COUNT(DISTINCT order_id) AS total_orders,
        COUNT(DISTINCT CASE WHEN is_delivered THEN order_id END) AS delivered_orders,
        COUNT(DISTINCT CASE WHEN is_cancelled THEN order_id END) AS cancelled_orders,
        SUM(total_amount) AS gmv,
        AVG(total_amount) AS aov,
        AVG(delivery_time_minutes) AS avg_delivery_time,
        PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY delivery_time_minutes) AS p50_delivery_time,
        PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY delivery_time_minutes) AS p90_delivery_time
    FROM {{ ref('fct_orders') }}
    JOIN {{ ref('dim_restaurants') }} USING (restaurant_id)
    GROUP BY 1, 2
)

SELECT
    *,
    ROUND(100.0 * cancelled_orders / NULLIF(total_orders, 0), 2) AS cancel_rate_pct,
    ROUND(gmv / NULLIF(delivered_orders, 0), 2) AS gmv_per_order
FROM daily_orders
```

## Airflow Setup

### Configure Environment

```bash
cd airflow
cp example.env .env
```

Edit `.env`:

```bash
# Snowflake
SNOWFLAKE_ACCOUNT=your_account.region
SNOWFLAKE_USER=your_user
SNOWFLAKE_PASSWORD=your_password
SNOWFLAKE_DATABASE=ZOMATO
SNOWFLAKE_WAREHOUSE=ZOMATO_WH
SNOWFLAKE_ROLE=DBT_ROLE

# OpenAI
OPENAI_API_KEY=sk-your-key-here

# AI enrichment sample size (to limit API costs)
SAMPLE_N=1000

# Airflow connection URI (auto-generated from above)
AIRFLOW_CONN_SNOWFLAKE_DEFAULT=snowflake://${SNOWFLAKE_USER}:${SNOWFLAKE_PASSWORD}@${SNOWFLAKE_ACCOUNT}/${SNOWFLAKE_DATABASE}?warehouse=${SNOWFLAKE_WAREHOUSE}&role=${SNOWFLAKE_ROLE}
```

### Start Airflow

```bash
cd airflow

# Build custom image with Snowflake provider
docker compose build

# Start services (postgres, scheduler, webserver)
docker compose up -d

# Check logs
docker compose logs -f

# Access UI at http://localhost:8080
# Default credentials: admin/admin (configure in docker-compose.yaml)
```

### Airflow DAG Structure

```python
# airflow/dags/zomato_batch.py
from airflow import DAG
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from airflow.operators.bash import BashOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-eng',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

with DAG(
    'zomato_batch',
    default_args=default_args,
    description='Daily Zomato data pipeline',
    schedule_interval='@daily',
    catchup=False,
    tags=['data-engineering', 'batch'],
) as dag:

    # Task 1: Reload raw tables from S3
    reload_raw = SnowflakeOperator(
        task_id='reload_raw',
        snowflake_conn_id='snowflake_default',
        sql="""
            COPY INTO ZOMATO.RAW.RESTAURANTS 
            FROM @ZOMATO.RAW.restaurants_stage
            FILE_FORMAT = (TYPE='CSV' SKIP_HEADER=1)
            PURGE = TRUE;
            
            -- Repeat for all 7 tables
        """,
    )

    # Task 2: Run dbt core models (staging + marts)
    dbt_build_core = BashOperator(
        task_id='dbt_build_core',
        bash_command='cd /opt/airflow/dbt/zomato && source /opt/dbt-venv/bin/activate && dbt build --exclude tag:ai',
        env={
            'SNOWFLAKE_ACCOUNT': '{{ var.value.SNOWFLAKE_ACCOUNT }}',
            'SNOWFLAKE_USER': '{{ var.value.SNOWFLAKE_USER }}',
            'SNOWFLAKE_PASSWORD': '{{ var.value.SNOWFLAKE_PASSWORD }}',
        },
    )

    # Task 3: LLM enrichment of reviews
    enrich_reviews = BashOperator(
        task_id='enrich_reviews',
        bash_command='python /opt/airflow/ai/enrich_reviews.py',
        env={
            'OPENAI_API_KEY': '{{ var.value.OPENAI_API_KEY }}',
            'SAMPLE_N': '{{ var.value.SAMPLE_N }}',
        },
    )

    # Task 4: Build AI marts
    dbt_build_ai = BashOperator(
        task_id='dbt_build_ai',
        bash_command='cd /opt/airflow/dbt/zomato && source /opt/dbt-venv/bin/activate && dbt build --select tag:ai',
    )

    # Define pipeline
    reload_raw >> dbt_build_core >> enrich_reviews >> dbt_build_ai
```

### Manual DAG Trigger

```bash
# From Airflow UI: enable DAG, click "Trigger DAG"

# Or via CLI inside container:
docker compose exec airflow-scheduler airflow dags trigger zomato_batch
```

## AI Layer

### LLM Enrichment

Extracts sentiment and topic from free-text reviews using OpenAI:

```python
# ai/enrich_reviews.py
import os
import json
import snowflake.connector
from openai import OpenAI

client = OpenAI(api_key=os.environ['OPENAI_API_KEY'])

def get_snowflake_connection():
    return snowflake.connector.connect(
        account=os.environ['SNOWFLAKE_ACCOUNT'],
        user=os.environ['SNOWFLAKE_USER'],
        password=os.environ['SNOWFLAKE_PASSWORD'],
        warehouse=os.environ['SNOWFLAKE_WAREHOUSE'],
        database=os.environ['SNOWFLAKE_DATABASE'],
        role=os.environ['SNOWFLAKE_ROLE'],
    )

def enrich_review(review_text):
    """Call GPT-4o-mini to extract sentiment and topic."""
    prompt = f"""Analyze this restaurant review and return JSON with:
    - sentiment: "positive", "negative", or "neutral"
    - topic: one of ["food_quality", "service", "delivery", "value", "ambiance", "other"]
    
    Review: "{review_text}"
    
    Return ONLY valid JSON, no markdown."""
    
    response = client.chat.completions.create(
        model='gpt-4o-mini',
        messages=[
            {'role': 'system', 'content': 'You are a review analysis assistant.'},
            {'role': 'user', 'content': prompt}
        ],
        temperature=0.3,
    )
    
    result = json.loads(response.choices[0].message.content)
    return result['sentiment'], result['topic']

def main():
    conn = get_snowflake_connection()
    cursor = conn.cursor()
    
    # Create enriched table if not exists
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS ZOMATO.AI.REVIEW_ENRICHED (
            review_id INT,
            review_text STRING,
            sentiment STRING,
            topic STRING,
            enriched_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
        )
    """)
    
    # Get unenriched reviews (idempotent)
    sample_n = int(os.environ.get('SAMPLE_N', 1000))
    cursor.execute(f"""
        SELECT r.review_id, r.review_text
        FROM ZOMATO.RAW.REVIEWS r
        LEFT JOIN ZOMATO.AI.REVIEW_ENRICHED e ON r.review_id = e.review_id
        WHERE e.review_id IS NULL
        LIMIT {sample_n}
    """)
    
    reviews = cursor.fetchall()
    print(f"Enriching {len(reviews)} reviews...")
    
    for review_id, review_text in reviews:
        try:
            sentiment, topic = enrich_review(review_text)
            cursor.execute("""
                INSERT INTO ZOMATO.AI.REVIEW_ENRICHED 
                (review_id, review_text, sentiment, topic)
                VALUES (%s, %s, %s, %s)
            """, (review_id, review_text, sentiment, topic))
            print(f"✓ Review {review_id}: {sentiment}/{topic}")
        except Exception as e:
            print(f"✗ Review {review_id}: {e}")
    
    conn.commit()
    cursor.close()
    conn.close()

if __name__ == '__main__':
    main()
```

Run standalone:

```bash
export OPENAI_API_KEY=sk-your-key
export SNOWFLAKE_ACCOUNT=your_account
export SNOWFLAKE_USER=your_user
export SNOWFLAKE_PASSWORD=your_password
export SNOWFLAKE_WAREHOUSE=ZOMATO_WH
export SNOWFLAKE_DATABASE=ZOMATO
export SNOWFLAKE_ROLE=DBT_ROLE
export SAMPLE_N=100

python ai/enrich_reviews.py
```

### RAG Chat (Chat with Reviews)

Retrieval-Augmented Generation to answer questions grounded in reviews:

```python
# ai/rag_chat.py
import os
import streamlit as st
import snowflake.connector
from openai import OpenAI
import numpy as np

client = OpenAI(api_key=os.environ['OPENAI_API_KEY'])

def get_embedding(text):
    """Generate embedding for search query."""
    response = client.embeddings.create(
        model='text-embedding-3-small',
        input=text
    )
    return response.data[0].embedding

def retrieve_reviews(question, top_k=5):
    """Retrieve most relevant reviews using vector similarity."""
    conn = snowflake.connector.connect(
        account=os.environ['SNOWFLAKE_ACCOUNT'],
        user=os.environ['SNOWFLAKE_USER'],
        password=os.environ['SNOWFLAKE_PASSWORD'],
        warehouse=os.environ['SNOWFLAKE_WAREHOUSE'],
        database=os.environ['SNOWFLAKE_DATABASE'],
        role=os.environ['SNOWFLAKE_ROLE'],
    )
    cursor = conn.cursor()
    
    # Simple keyword search (or use Snowflake vector search if available)
    query = f"""
        SELECT review_text, sentiment, topic
        FROM ZOMATO.AI.REVIEW_ENRICHED
        WHERE review_text ILIKE '%{question}%'
        LIMIT {top_k}
    """
    cursor.execute(query)
    reviews = cursor.fetchall()
    cursor.close()
    conn.close()
    
    return reviews

def generate_answer(question, reviews):
    """Generate answer from retrieved reviews."""
    context = "\n\n".join([
        f"Review ({sentiment}, {topic}): {text}"
        for text, sentiment, topic in reviews
    ])
    
    prompt = f"""Based on these customer reviews, answer the question:

Reviews:
{context}

Question: {question}

Provide a concise answer grounded in the reviews above."""
    
    response = client.chat.completions.create(
        model='gpt-4o-mini',
        messages=[
            {'role': 'system', 'content': 'You answer questions using only the provided reviews.'},
            {'role': 'user', 'content': prompt}
        ],
        temperature=0.5,
    )
    
    return response.choices[0].message.content

# Streamlit UI
st.title("🍕 Chat with Zomato Reviews")

question = st.text_input("Ask a question about reviews:", 
                         placeholder="What do customers say about delivery time?")

if st.button("Search") and question:
    with st.spinner("Retrieving reviews..."):
        reviews = retrieve_reviews(question, top_k=5)
    
    if reviews:
        with st.spinner("Generating answer..."):
            answer = generate_answer(question, reviews)
        
        st.success(answer)
        
        with st.expander("📄 Source Reviews"):
            for text, sentiment, topic in reviews:
                st.markdown(f"**{sentiment}** ({topic}): {text}")
    else:
        st.warning("No relevant reviews found.")
```

Run:

```bash
streamlit run ai/rag_chat.py
```

### Text-to-SQL (Chat with Warehouse)

Query the data warehouse using natural language:

```python
# ai/text_to_sql.py
import os
import re
import streamlit as st
import snowflake.connector
from openai import OpenAI

client = OpenAI(api_key=os.environ['OPENAI_API_KEY'])

def get_schema():
    """Get marts schema for context."""
    return """
Available tables in ZOMATO.MARTS:

1. MART_DAILY_CITY_METRICS
   - order_date DATE
   - city STRING
   - total_orders INT
   - delivered_orders INT
   - cancelled_orders INT
   - gmv DECIMAL
   - aov DECIMAL
   - cancel_rate_pct DECIMAL

2. MART_RESTAURANT_PERFORMANCE
   - restaurant_id INT
   - restaurant_name STRING
   - city STRING
   - total_orders INT
   - avg_rating DECIMAL
   - total_revenue DECIMAL

3. MART_DELIVERY_SLA
   - city STRING
   - hour_of_day INT
   - p50_delivery_time INT (minutes)
   - p90_delivery_time INT (minutes)

4. MART_REVIEW_INSIGHTS
   - sentiment STRING (positive/negative/neutral)
   - topic STRING
   - review_count INT
   - avg_rating DECIMAL
"""

def generate_sql(question):
    """Generate Snowflake SQL from natural language."""
    schema = get_schema()
    
    prompt = f"""You are a Snowflake SQL expert. Generate a SELECT query to answer this question.

Schema:
{schema}

Rules:
- Use only SELECT statements
- Use proper Snowflake SQL syntax
- Use ZOMATO.MARTS schema prefix
- Return ONLY the SQL query, no explanation

Question: {question}

SQL:"""
    
    response = client.chat.completions.create(
        model='gpt-4o-mini',
        messages=[
            {'role': 'system', 'content': 'You generate Snowflake SQL queries.'},
            {'role': 'user', 'content': prompt}
        ],
        temperature=0.1,
    )
    
    sql = response.choices[0].message.content.strip()
    # Clean markdown code blocks
    sql = re.sub(r'^```sql\n?', '', sql)
    sql = re.sub(r'\n?```$', '', sql)
    return sql

def validate_sql(sql):
    """Basic SQL injection protection."""
    dangerous_keywords = ['DROP', 'DELETE', 'INSERT', 'UPDATE', 'TRUNCATE', 'ALTER', 'CREATE']
    sql_upper = sql.upper()
    for keyword in dangerous_keywords:
        if keyword in sql_upper:
            raise ValueError(f"Query contains prohibited keyword: {keyword}")
    return True

def execute_query(sql):
    """Execute SQL against Snowflake."""
    conn = snowflake.connector.connect(
        account=os.environ['SNOWFLAKE_ACCOUNT'],
        user=os.environ['SNOWFLAKE_USER'],
        password=os.environ['SNOWFLAKE_PASSWORD'],
        warehouse=os.environ['SNOWFLAKE_WAREHOUSE'],
        database=os.environ['SNOWFLAKE_DATABASE'],
        role=os.environ['SNOWFLAKE_ROLE'],
    )
    cursor = conn.cursor()
    cursor.execute(sql)
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    cursor.close()
    conn.close()
    return columns, results

# Streamlit UI
st.title("💬 Chat with Zomato Data Warehouse")

st.info("Ask questions about orders, restaurants, delivery, or reviews in plain English.")

question = st.text_input(
    "Your question:",
    placeholder="Which city has the highest GMV this month?"
)

if st.button("Run Query") and question:
    with st.spinner("Generating SQL..."):
        sql = generate_sql(question)
    
    st.code(sql, language='sql')
    
    try:
        validate_sql(sql)
        
        with st.spinner("Executing query..."):
            columns, results = execute_query(sql)
        
        if results:
            import pandas as pd
            df = pd.DataFrame(results, columns=columns)
            st.dataframe(df, use_container_width=True)
            st.success(f"Returned {len(results)} rows")
        else:
            st.warning("Query returned no results.")
    
    except Exception as e:
        st.error(f"Error: {e}")
```

Run:

```bash
streamlit run ai/text_to_sql.py
```

## Common Patterns

### Incremental Model with Custom Logic

```sql
-- models/marts/facts/fct_order_items.sql
{{
    config(
        materialized='incremental',
        unique_key='order_item_id',
        incremental_strategy='merge',
        on_schema_change='fail'
    )
}}

WITH new_items AS (
    SELECT
        oi.order_item_id,
        oi.order_id,
        oi.food_id,
        oi.quantity,
        oi.item_price,
        o.order_date,
        -- Enrich with dimensions
        f.food_name,
        f.food_category,
        r.restaurant_name,
        r.city
    FROM {{ ref('stg_order_items') }} oi
    JOIN {{ ref('stg_orders') }} o USING (order_id)
    JOIN {{ ref('dim_food') }} f USING (food_id)
    JOIN {{ ref('dim_restaurants') }} r ON o.restaurant_id = r.restaurant_id
    
    {% if is_incremental() %}
        WHERE o.order_date > (SELECT MAX(order_date) FROM {{ this }})
    {% endif %}
)

SELECT * FROM new_items
```

### SCD Type 2 Snapshot

```sql
-- models/snapshots/restaurant_snapshot.sql
{% snapshot restaurant_snapshot %}

{{
    config(
        target_schema='snapshots',
        unique_key='restaurant_id',
        strategy='check',
        check_cols=['rating', 'rating_count', 'cost_for_two']
    )
}}

SELECT * FROM {{ ref('stg_restaurants') }}

{% endsnapshot %}
```

Run snapshot:

```bash
dbt snapshot
```

### Custom Schema Macro

```sql
-- macros/generate_schema_name.sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}
    
    {%- if custom_schema_name is none -%}
        {{ default_schema }}
    {%- else 
