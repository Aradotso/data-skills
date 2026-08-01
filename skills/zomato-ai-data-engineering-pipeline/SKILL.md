---
name: zomato-ai-data-engineering-pipeline
description: End-to-end batch data pipeline with S3, Snowflake, dbt, Airflow, and AI (LLM enrichment, RAG, text-to-SQL) for food delivery analytics
triggers:
  - build a zomato data pipeline
  - set up snowflake dbt airflow pipeline
  - create food delivery analytics with AI
  - implement text-to-sql on snowflake
  - build RAG chat with reviews
  - orchestrate dbt with airflow
  - enrich data with openai llm
  - set up medallion architecture in snowflake
---

# Zomato AI Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A complete batch data engineering pipeline that processes food delivery data through a medallion architecture (Bronze → Silver → Gold) using S3, Snowflake, dbt, Airflow orchestration, and AI capabilities (LLM enrichment, RAG, text-to-SQL).

## What This Project Does

Takes Zomato-style food delivery data (restaurants, users, orders, reviews) through:

1. **Data Lake** — CSVs uploaded to S3 (`s3://bucket/raw/table/`)
2. **Bronze Layer** — Raw tables in Snowflake (`ZOMATO.RAW`) loaded via `COPY INTO`
3. **Silver Layer** — Cleaned staging views (`ZOMATO.STAGING`) via dbt
4. **Gold Layer** — Business-ready dimensions, incremental facts, and marts (`ZOMATO.MARTS`)
5. **AI Layer** — LLM enrichment, RAG chat, text-to-SQL (`ZOMATO.AI`)
6. **Orchestration** — Airflow DAG runs the entire pipeline daily

**Key Features:**
- Keyless S3 → Snowflake integration via IAM role
- Incremental fact table processing (MERGE strategy)
- AI-powered review sentiment/topic extraction
- Chat with reviews (RAG) and warehouse (text-to-SQL)
- 10M+ orders, 23M+ order items, 300K reviews

## Installation & Setup

### Prerequisites

```bash
# Required accounts/services
- AWS account (S3 bucket)
- Snowflake account
- OpenAI API key
- Docker & Docker Compose
- Python 3.8+
```

### 1. Clone and Get Dataset

```bash
git clone https://github.com/darshilparmar/zomato-ai-data-engineering-end-to-end-project.git
cd zomato-ai-data-engineering-end-to-end-project

# Download CSVs from Google Drive (link in README) and place in data/
# Expected files:
# data/restaurants.csv
# data/users.csv
# data/food.csv
# data/menu.csv
# data/orders.csv
# data/order_items.csv
# data/reviews.csv
```

### 2. AWS Setup (S3 + IAM)

```bash
# Create S3 bucket
aws s3 mb s3://your-zomato-bucket

# Upload data
aws s3 cp data/ s3://your-zomato-bucket/raw/ --recursive

# Create IAM policy (s3-read-policy.json)
aws iam create-policy \
  --policy-name zomato-s3-read \
  --policy-document file://aws/iam/s3-read-policy.json

# Create IAM role (initial trust)
aws iam create-role \
  --role-name snowflake-s3-role \
  --assume-role-policy-document file://aws/iam/snowflake-role-trust-policy-initial.json

# Attach policy to role
aws iam attach-role-policy \
  --role-name snowflake-s3-role \
  --policy-arn arn:aws:iam::YOUR_ACCOUNT:policy/zomato-s3-read
```

### 3. Snowflake Setup

```sql
-- Run in Snowsight as ACCOUNTADMIN

-- Create warehouse
CREATE WAREHOUSE ZOMATO_WH WITH WAREHOUSE_SIZE = 'MEDIUM';

-- Create database and schemas
CREATE DATABASE ZOMATO;
CREATE SCHEMA ZOMATO.RAW;
CREATE SCHEMA ZOMATO.STAGING;
CREATE SCHEMA ZOMATO.MARTS;
CREATE SCHEMA ZOMATO.SNAPSHOTS;
CREATE SCHEMA ZOMATO.AI;

-- Create role
CREATE ROLE DBT_ROLE;
GRANT USAGE ON WAREHOUSE ZOMATO_WH TO ROLE DBT_ROLE;
GRANT ALL ON DATABASE ZOMATO TO ROLE DBT_ROLE;
GRANT ALL ON ALL SCHEMAS IN DATABASE ZOMATO TO ROLE DBT_ROLE;
GRANT CREATE SCHEMA ON DATABASE ZOMATO TO ROLE DBT_ROLE;
GRANT ROLE DBT_ROLE TO USER YOUR_USER;

-- Create storage integration (replace ARN)
CREATE STORAGE INTEGRATION s3_zomato_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = S3
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::YOUR_ACCOUNT:role/snowflake-s3-role'
  STORAGE_ALLOWED_LOCATIONS = ('s3://your-zomato-bucket/raw/');

-- Get Snowflake's IAM user ARN and external ID
DESC STORAGE INTEGRATION s3_zomato_integration;
-- Note STORAGE_AWS_IAM_USER_ARN and STORAGE_AWS_EXTERNAL_ID
```

### 4. Update IAM Trust Policy

```bash
# Edit aws/iam/snowflake-role-trust-policy-final.json with:
# - STORAGE_AWS_IAM_USER_ARN as Principal.AWS
# - STORAGE_AWS_EXTERNAL_ID as Condition.StringEquals.sts:ExternalId

# Update the role
aws iam update-assume-role-policy \
  --role-name snowflake-s3-role \
  --policy-document file://aws/iam/snowflake-role-trust-policy-final.json
```

### 5. Create Snowflake Stages and Tables

```sql
-- Create stages for each table
CREATE OR REPLACE STAGE ZOMATO.RAW.restaurants_stage
  URL = 's3://your-zomato-bucket/raw/restaurants/'
  STORAGE_INTEGRATION = s3_zomato_integration
  FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"');

-- Repeat for: users, food, menu, orders, order_items, reviews

-- Create raw tables
CREATE OR REPLACE TABLE ZOMATO.RAW.restaurants (
  restaurant_id INTEGER,
  restaurant_name VARCHAR,
  city VARCHAR,
  address VARCHAR,
  locality VARCHAR,
  latitude FLOAT,
  longitude FLOAT,
  cuisine_type VARCHAR,
  average_cost_for_two VARCHAR,
  has_online_delivery VARCHAR,
  aggregate_rating FLOAT,
  votes INTEGER
);

-- Create tables for: users, food, menu, orders, order_items, reviews
-- (See project for full schemas)
```

## dbt Configuration

### profiles.yml

```yaml
# ~/.dbt/profiles.yml or zomato/profiles.yml
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

### Key dbt Commands

```bash
cd zomato

# Set environment variables
export SNOWFLAKE_ACCOUNT=your_account.region
export SNOWFLAKE_USER=your_user
export SNOWFLAKE_PASSWORD=your_password

# Test connection
dbt debug

# Run models only (exclude AI)
dbt run --exclude tag:ai

# Run models + tests
dbt build --exclude tag:ai

# Run specific model
dbt run --select dim_restaurants

# Run incremental facts only
dbt run --select fct_orders fct_order_items

# Full refresh (rebuild incremental models)
dbt run --full-refresh --select fct_orders

# Test data quality
dbt test

# Generate documentation
dbt docs generate
dbt docs serve
```

## dbt Project Structure

### Staging Models (Silver)

```sql
-- models/staging/stg_restaurants.sql
{{
    config(
        materialized='view'
    )
}}

SELECT
    restaurant_id,
    restaurant_name,
    city,
    address,
    locality,
    latitude,
    longitude,
    cuisine_type,
    -- Clean cost field: '₹ 200' -> 200, '--' -> NULL
    CASE
        WHEN average_cost_for_two = '--' THEN NULL
        ELSE TRY_CAST(REPLACE(REPLACE(average_cost_for_two, '₹', ''), ' ', '') AS NUMBER)
    END AS average_cost_for_two,
    -- Convert boolean-like strings
    CASE WHEN LOWER(has_online_delivery) = 'yes' THEN TRUE ELSE FALSE END AS has_online_delivery,
    aggregate_rating,
    votes
FROM {{ source('raw', 'restaurants') }}
```

### Dimension Models (Gold)

```sql
-- models/marts/dim_customer.sql
{{
    config(
        materialized='table'
    )
}}

SELECT
    user_id AS customer_key,
    name AS customer_name,
    LOWER(email) AS email,
    phone,
    address,
    city,
    DATE(created_at) AS registration_date,
    age,
    CASE
        WHEN age < 25 THEN '18-24'
        WHEN age BETWEEN 25 AND 34 THEN '25-34'
        WHEN age BETWEEN 35 AND 44 THEN '35-44'
        WHEN age BETWEEN 45 AND 54 THEN '45-54'
        ELSE '55+'
    END AS age_segment
FROM {{ ref('stg_users') }}
```

### Incremental Fact Models (Gold)

```sql
-- models/marts/fct_orders.sql
{{
    config(
        materialized='incremental',
        unique_key='order_key',
        merge_update_columns=['status', 'delivery_time', 'rating']
    )
}}

SELECT
    o.order_id AS order_key,
    o.user_id AS customer_key,
    o.restaurant_id AS restaurant_key,
    DATE(o.order_date) AS order_date_key,
    o.order_date AS order_timestamp,
    o.status,
    o.total_amount,
    o.delivery_time,
    o.payment_method,
    o.rating,
    CASE WHEN o.status = 'Delivered' THEN TRUE ELSE FALSE END AS is_delivered,
    CASE WHEN o.status = 'Cancelled' THEN TRUE ELSE FALSE END AS is_cancelled
FROM {{ ref('stg_orders') }} o

{% if is_incremental() %}
    WHERE o.order_date > (SELECT MAX(order_timestamp) FROM {{ this }})
{% endif %}
```

### Business Marts (Gold)

```sql
-- models/marts/mart_daily_city_revenue.sql
{{
    config(
        materialized='table'
    )
}}

SELECT
    order_date_key,
    r.city,
    COUNT(DISTINCT f.order_key) AS total_orders,
    COUNT(DISTINCT CASE WHEN f.is_delivered THEN f.order_key END) AS delivered_orders,
    COUNT(DISTINCT CASE WHEN f.is_cancelled THEN f.order_key END) AS cancelled_orders,
    SUM(f.total_amount) AS gmv,
    AVG(f.total_amount) AS aov,
    ROUND(cancelled_orders::FLOAT / NULLIF(total_orders, 0) * 100, 2) AS cancel_rate_pct
FROM {{ ref('fct_orders') }} f
JOIN {{ ref('dim_restaurants') }} r ON f.restaurant_key = r.restaurant_key
GROUP BY 1, 2
```

## Airflow Setup

### Configuration

```bash
cd airflow

# Copy and edit environment file
cp example.env .env

# Edit .env with:
# SNOWFLAKE_ACCOUNT=your_account.region
# SNOWFLAKE_USER=your_user
# SNOWFLAKE_PASSWORD=your_password
# SNOWFLAKE_ROLE=DBT_ROLE
# SNOWFLAKE_WAREHOUSE=ZOMATO_WH
# SNOWFLAKE_DATABASE=ZOMATO
# OPENAI_API_KEY=sk-...
# SAMPLE_N=1000  # Max reviews to enrich per run
```

### Start Airflow

```bash
# Build and start containers
docker compose build
docker compose up -d

# Check logs
docker compose logs -f

# Access UI at http://localhost:8080
# Default user: airflow / airflow
```

### DAG Structure

```python
# airflow/dags/zomato_batch.py
from airflow import DAG
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from airflow.operators.bash import BashOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'airflow',
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

with DAG(
    'zomato_batch',
    default_args=default_args,
    description='Zomato data pipeline',
    schedule_interval='@daily',
    start_date=datetime(2024, 1, 1),
    catchup=False,
) as dag:

    # Task 1: Load raw data from S3
    reload_raw = SnowflakeOperator(
        task_id='reload_raw',
        snowflake_conn_id='snowflake_default',
        sql="""
            COPY INTO ZOMATO.RAW.restaurants
            FROM @ZOMATO.RAW.restaurants_stage
            FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1)
            FORCE = TRUE;
            
            -- Repeat for all tables
        """
    )

    # Task 2: Run dbt (core models + tests)
    dbt_build_core = BashOperator(
        task_id='dbt_build_core',
        bash_command='cd /opt/airflow/zomato && dbt build --exclude tag:ai'
    )

    # Task 3: AI enrichment
    enrich_reviews = BashOperator(
        task_id='enrich_reviews',
        bash_command='python /opt/airflow/ai/enrich_reviews.py'
    )

    # Task 4: Build AI marts
    dbt_build_ai = BashOperator(
        task_id='dbt_build_ai',
        bash_command='cd /opt/airflow/zomato && dbt build --select tag:ai'
    )

    reload_raw >> dbt_build_core >> enrich_reviews >> dbt_build_ai
```

## AI Layer

### 1. LLM Enrichment (Sentiment & Topic)

```python
# ai/enrich_reviews.py
import os
import json
import snowflake.connector
from openai import OpenAI

client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))

# Connect to Snowflake
conn = snowflake.connector.connect(
    account=os.getenv('SNOWFLAKE_ACCOUNT'),
    user=os.getenv('SNOWFLAKE_USER'),
    password=os.getenv('SNOWFLAKE_PASSWORD'),
    warehouse=os.getenv('SNOWFLAKE_WAREHOUSE'),
    database=os.getenv('SNOWFLAKE_DATABASE'),
    role=os.getenv('SNOWFLAKE_ROLE')
)

cursor = conn.cursor()

# Get unenriched reviews (limit by SAMPLE_N)
SAMPLE_N = int(os.getenv('SAMPLE_N', 1000))

cursor.execute(f"""
    SELECT r.review_id, r.review_text
    FROM ZOMATO.RAW.reviews r
    LEFT JOIN ZOMATO.AI.REVIEW_ENRICHED e ON r.review_id = e.review_id
    WHERE e.review_id IS NULL
    LIMIT {SAMPLE_N}
""")

reviews = cursor.fetchall()

enriched = []
for review_id, review_text in reviews:
    try:
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": """You are a sentiment analyzer. 
                Return JSON with: sentiment (positive/negative/neutral) and topic (food/service/delivery/price)."""},
                {"role": "user", "content": f"Review: {review_text}"}
            ],
            response_format={"type": "json_object"}
        )
        
        result = json.loads(response.choices[0].message.content)
        enriched.append((
            review_id,
            review_text,
            result.get('sentiment'),
            result.get('topic')
        ))
    except Exception as e:
        print(f"Error processing review {review_id}: {e}")

# Insert enriched data
if enriched:
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS ZOMATO.AI.REVIEW_ENRICHED (
            review_id INTEGER PRIMARY KEY,
            review_text VARCHAR,
            sentiment VARCHAR,
            topic VARCHAR,
            enriched_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
        )
    """)
    
    cursor.executemany(
        "INSERT INTO ZOMATO.AI.REVIEW_ENRICHED (review_id, review_text, sentiment, topic) VALUES (%s, %s, %s, %s)",
        enriched
    )
    conn.commit()

print(f"Enriched {len(enriched)} reviews")
cursor.close()
conn.close()
```

### 2. RAG Chat (Chat with Reviews)

```python
# ai/rag_chat.py
import os
import streamlit as st
from openai import OpenAI
import snowflake.connector
import numpy as np

client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))

st.title("🍕 Chat with Zomato Reviews")

# Connect to Snowflake
@st.cache_resource
def get_snowflake_connection():
    return snowflake.connector.connect(
        account=os.getenv('SNOWFLAKE_ACCOUNT'),
        user=os.getenv('SNOWFLAKE_USER'),
        password=os.getenv('SNOWFLAKE_PASSWORD'),
        warehouse=os.getenv('SNOWFLAKE_WAREHOUSE'),
        database=os.getenv('SNOWFLAKE_DATABASE'),
        role=os.getenv('SNOWFLAKE_ROLE')
    )

# Load and embed reviews (cache)
@st.cache_data
def load_review_embeddings():
    conn = get_snowflake_connection()
    cursor = conn.cursor()
    
    cursor.execute("""
        SELECT review_id, review_text
        FROM ZOMATO.AI.REVIEW_ENRICHED
        LIMIT 1000
    """)
    
    reviews = []
    for review_id, review_text in cursor.fetchall():
        embedding = client.embeddings.create(
            model="text-embedding-3-small",
            input=review_text
        ).data[0].embedding
        
        reviews.append({
            'review_id': review_id,
            'text': review_text,
            'embedding': np.array(embedding)
        })
    
    return reviews

reviews = load_review_embeddings()

# User query
query = st.text_input("Ask about customer reviews:", placeholder="What do customers say about delivery speed?")

if query and st.button("Search"):
    # Embed query
    query_embedding = np.array(
        client.embeddings.create(
            model="text-embedding-3-small",
            input=query
        ).data[0].embedding
    )
    
    # Find similar reviews (cosine similarity)
    similarities = []
    for review in reviews:
        similarity = np.dot(query_embedding, review['embedding']) / (
            np.linalg.norm(query_embedding) * np.linalg.norm(review['embedding'])
        )
        similarities.append((review, similarity))
    
    # Top 5
    top_reviews = sorted(similarities, key=lambda x: x[1], reverse=True)[:5]
    context = "\n\n".join([f"Review: {r[0]['text']}" for r in top_reviews])
    
    # Generate answer
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "You are a helpful assistant that answers questions based on customer reviews. Cite specific reviews."},
            {"role": "user", "content": f"Question: {query}\n\nRelevant reviews:\n{context}"}
        ]
    )
    
    st.write("### Answer")
    st.write(response.choices[0].message.content)
    
    st.write("### Source Reviews")
    for review, score in top_reviews:
        st.write(f"**Similarity: {score:.2f}**")
        st.write(review['text'])
        st.write("---")
```

### 3. Text-to-SQL (Chat with Warehouse)

```python
# ai/text_to_sql.py
import os
import streamlit as st
from openai import OpenAI
import snowflake.connector

client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))

st.title("💬 Chat with Zomato Warehouse")

# Get schema
@st.cache_data
def get_schema():
    conn = snowflake.connector.connect(
        account=os.getenv('SNOWFLAKE_ACCOUNT'),
        user=os.getenv('SNOWFLAKE_USER'),
        password=os.getenv('SNOWFLAKE_PASSWORD'),
        warehouse=os.getenv('SNOWFLAKE_WAREHOUSE'),
        database=os.getenv('SNOWFLAKE_DATABASE'),
        role=os.getenv('SNOWFLAKE_ROLE')
    )
    cursor = conn.cursor()
    
    cursor.execute("""
        SELECT table_name, column_name, data_type
        FROM ZOMATO.INFORMATION_SCHEMA.COLUMNS
        WHERE table_schema = 'MARTS'
        ORDER BY table_name, ordinal_position
    """)
    
    schema = {}
    for table, column, dtype in cursor.fetchall():
        if table not in schema:
            schema[table] = []
        schema[table].append(f"{column} ({dtype})")
    
    return "\n\n".join([f"Table: ZOMATO.MARTS.{t}\nColumns: {', '.join(cols)}" for t, cols in schema.items()])

schema = get_schema()

# User question
question = st.text_input("Ask a business question:", placeholder="What are the top 5 cities by revenue?")

if question and st.button("Run Query"):
    # Generate SQL
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": f"""You are a Snowflake SQL expert. 
            Generate a SELECT query for the user's question. Only use tables/columns from this schema:
            
            {schema}
            
            Return ONLY the SQL query, no explanation."""},
            {"role": "user", "content": question}
        ]
    )
    
    sql = response.choices[0].message.content.strip().strip('```sql').strip('```').strip()
    
    st.code(sql, language='sql')
    
    # Validate (SELECT only)
    if not sql.upper().strip().startswith('SELECT'):
        st.error("Only SELECT queries are allowed")
    else:
        try:
            conn = snowflake.connector.connect(
                account=os.getenv('SNOWFLAKE_ACCOUNT'),
                user=os.getenv('SNOWFLAKE_USER'),
                password=os.getenv('SNOWFLAKE_PASSWORD'),
                warehouse=os.getenv('SNOWFLAKE_WAREHOUSE'),
                database=os.getenv('SNOWFLAKE_DATABASE'),
                role=os.getenv('SNOWFLAKE_ROLE')
            )
            cursor = conn.cursor()
            cursor.execute(sql)
            
            results = cursor.fetchall()
            columns = [desc[0] for desc in cursor.description]
            
            st.write("### Results")
            st.dataframe(results, columns=columns)
            
        except Exception as e:
            st.error(f"Query failed: {e}")
```

### Run AI Apps

```bash
# Set environment
export SNOWFLAKE_ACCOUNT=your_account.region
export SNOWFLAKE_USER=your_user
export SNOWFLAKE_PASSWORD=your_password
export SNOWFLAKE_ROLE=DBT_ROLE
export SNOWFLAKE_WAREHOUSE=ZOMATO_WH
export SNOWFLAKE_DATABASE=ZOMATO
export OPENAI_API_KEY=sk-...

# Run enrichment (once or in Airflow)
python ai/enrich_reviews.py

# Launch Streamlit apps
streamlit run ai/rag_chat.py       # Port 8501
streamlit run ai/text_to_sql.py    # Port 8502
```

## Common Patterns

### Load New Data to S3

```bash
# Upload new CSVs to S3
aws s3 cp data/orders_new.csv s3://your-zomato-bucket/raw/orders/

# Trigger Airflow DAG or run manually:
# The COPY INTO with FORCE=TRUE will reload
```

### Run Incremental Load

```bash
# Incremental facts will only process new rows
cd zomato
dbt run --select fct_orders fct_order_items

# Full refresh if needed
dbt run --full-refresh --select fct_orders
```

### Add New Mart

```sql
-- zomato/models/marts/mart_customer_lifetime_value.sql
{{
    config(
        materialized='table'
    )
}}

SELECT
    customer_key,
    COUNT(order_key) AS total_orders,
    SUM(total_amount) AS lifetime_value,
    AVG(total_amount) AS avg_order_value,
    MIN(order_date_key) AS first_order_date,
    MAX(order_date_key) AS last_order_date
FROM {{ ref('fct_orders') }}
WHERE is_delivered
GROUP BY 1
```

```bash
# Run the new model
dbt run --select mart_customer_lifetime_value
```

### Query Data

```sql
-- Top 5 cities by GMV
SELECT city, SUM(gmv) AS total_gmv
FROM ZOMATO.MARTS.MART_DAILY_CITY_REVENUE
GROUP BY city
ORDER BY total_gmv DESC
LIMIT 5;

-- Customer segments
SELECT age_segment, COUNT(*) AS customers
FROM ZOMATO.MARTS.DIM_CUSTOMER
GROUP BY age_segment;

-- Delivery SLA by city
SELECT city, PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY delivery_time) AS p50_delivery_time
FROM ZOMATO.MARTS.FCT_ORDERS
JOIN ZOMATO.MARTS.DIM_RESTAURANTS USING (restaurant_key)
WHERE is_delivered
GROUP BY city;

-- Review sentiment breakdown
SELECT sentiment, COUNT(*) AS review_count
FROM ZOMATO.AI.REVIEW_ENRICHED
GROUP BY sentiment;
```

## Troubleshooting

### S3 → Snowflake Access Denied

```sql
-- Verify storage integration
DESC STORAGE INTEGRATION s3_zomato_integration;

-- Test stage access
LIST @ZOMATO.RAW.restaurants_stage;

-- If fails, check:
-- 1. IAM role trust policy has correct STORAGE_AWS_IAM_USER_ARN
-- 2. External ID matches STORAGE_AWS_EXTERNAL_ID
-- 3. S3 bucket policy allows the role
```

### dbt Connection Issues

```bash
# Test connection
dbt debug

# Check profiles.yml path
export DBT_PROFILES_DIR=/path/to/profiles/dir

# Verify env vars are set
echo $SNOWFLAKE_ACCOUNT
echo $SNOWFLAKE_USER
echo $SNOWFLAKE_PASSWORD
```

### Incremental Models Not Updating

```bash
# Check incremental logic
dbt compile --select fct_orders

# View compiled SQL in target/compiled/

# Force full refresh
dbt run --full-refresh --select fct_orders
```

### Airflow DAG Not Appearing

```bash
# Check logs
docker compose logs airflow-scheduler

# Verify DAG file
docker compose exec airflow-scheduler ls /opt/airflow/dags

# Parse DAG manually
docker compose exec airflow-scheduler python /opt/airflow/dags/zomato_batch.py
```

### OpenAI Rate Limits

```python
# In enrich_reviews.py, add retry logic
from openai import RateLimitError
import time

try:
    response = client.chat.completions.create(...)
except RateLimitError:
    time.sleep(60)  # Wait 1 minute
    response = client.chat.completions.create(...)

# Or reduce SAMPLE_N in .env
SAMPLE_N=100
```

### Snowflake Query Performance

```sql
-- Enable query profiling
ALTER SESSION SET USE_CACHED_RESULT = FALSE;

-- Check query history
SELECT * FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE QUERY_TEXT LIKE '%fct_orders%'
ORDER BY START_TIME DESC
LIMIT 10;

-- Add clustering key for large tables
ALTER TABLE ZOMATO.MARTS.FCT_ORDERS
CLUSTER BY (order_date_key);
```

## Key Files Reference

| File | Purpose |
|------|---------|
| `zomato/dbt_project.yml` | dbt project config, model paths, schema generation |
| `zomato/models/sources.yml` | Source definitions for RAW tables |
| `zomato/models/staging/*.sql` | Silver layer staging views |
| `zomato/models/marts/*.sql` | Gold layer dimensions, facts, marts |
| `airflow/dags/zomato_batch.py` | Main orchestration DAG |
| `airflow/docker-compose.yaml` | Airflow services definition |
| `ai/enrich_reviews.py` | LLM sentiment/topic enrichment |
| `ai/rag_chat.py` | RAG chat interface (Streamlit) |
| `ai/text_to_sql.py` | Text-to-SQL interface (Streamlit) |
| `aws/iam/*.json` | IAM policies and trust documents |

## Architecture Summary

```
CSV Data → S3 (raw/) → Snowflake RAW (COPY INTO) → 
dbt Staging (views) → dbt Marts (tables/incremental) → 
AI Enrichment (OpenAI) → AI Marts → 
Streamlit Apps (RAG/Text-to-SQL)

Orchestrated by: Airflow DAG (daily)
```

This pipeline demonstrates modern
