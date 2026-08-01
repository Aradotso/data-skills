---
name: zomato-ai-data-engineering-pipeline
description: End-to-end batch data pipeline with Snowflake, dbt, Airflow, and OpenAI for food delivery analytics
triggers:
  - build zomato data pipeline
  - setup snowflake dbt airflow pipeline
  - orchestrate batch data with airflow
  - transform data with dbt medallion architecture
  - enrich data with openai llm
  - create rag chatbot for reviews
  - implement text to sql with openai
  - deploy end to end data engineering project
---

# Zomato AI Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A complete batch data pipeline that processes food delivery data through a medallion architecture: **CSV → S3 → Snowflake → dbt (Bronze/Silver/Gold) → Airflow → OpenAI AI Layer**. Includes LLM enrichment, RAG chat, and text-to-SQL capabilities.

## Architecture Overview

**Data Flow:**
1. Raw CSVs (10M orders, 23M order items, 300K reviews) → Amazon S3
2. Snowflake loads via `COPY INTO` (storage integration, no keys)
3. dbt transforms through medallion layers (RAW → STAGING → MARTS)
4. Airflow orchestrates daily batch runs
5. OpenAI enriches reviews and powers analytics

**Stack:** Python, Pandas, S3, Snowflake, dbt-snowflake, Airflow 3 (Docker), OpenAI, Streamlit

## Project Structure

```
zomato-ai-data-engineering/
├── airflow/
│   ├── Dockerfile              # Airflow + Snowflake/OpenAI providers
│   ├── docker-compose.yaml     # Postgres + API + scheduler
│   ├── example.env             # Credential template
│   └── dags/zomato_batch.py    # Main pipeline DAG
├── zomato/                     # dbt project
│   ├── dbt_project.yml
│   ├── profiles.yml
│   ├── models/
│   │   ├── sources.yml
│   │   ├── staging/            # Silver layer views
│   │   └── marts/              # Gold layer tables
│   └── macros/
├── ai/
│   ├── enrich_reviews.py       # LLM enrichment
│   ├── rag_chat.py             # RAG chatbot (Streamlit)
│   └── text_to_sql.py          # Natural language SQL (Streamlit)
├── aws/iam/                    # IAM policies for S3-Snowflake
└── data/                       # Local CSVs (not in repo)
```

## Installation & Setup

### Prerequisites

```bash
# Required accounts/services
- AWS account (S3 bucket)
- Snowflake account
- OpenAI API key
- Docker & Docker Compose
- Python 3.9+
```

### 1. Snowflake Setup

```sql
-- Create warehouse, database, schemas
CREATE WAREHOUSE ZOMATO_WH WITH WAREHOUSE_SIZE = 'MEDIUM';
CREATE DATABASE ZOMATO;
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
GRANT ROLE DBT_ROLE TO USER <YOUR_USER>;
```

### 2. AWS S3 & IAM Setup

**Create S3 bucket:**
```bash
aws s3 mb s3://zomato-data-lake
```

**Create IAM policy** (`aws/iam/s3-read-policy.json`):
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:GetObjectVersion", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::zomato-data-lake",
      "arn:aws:s3:::zomato-data-lake/*"
    ]
  }]
}
```

**Create IAM role** with trust policy for Snowflake:
```bash
# Initial trust (placeholder)
aws iam create-role --role-name snowflake-s3-role \
  --assume-role-policy-document file://aws/iam/snowflake-role-trust-policy-initial.json

# Attach read policy
aws iam attach-role-policy --role-name snowflake-s3-role \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/zomato-s3-read
```

### 3. Snowflake Storage Integration

```sql
-- Create storage integration
CREATE STORAGE INTEGRATION s3_zomato_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'S3'
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::<ACCOUNT_ID>:role/snowflake-s3-role'
  STORAGE_ALLOWED_LOCATIONS = ('s3://zomato-data-lake/raw/');

-- Get Snowflake's IAM user for trust policy
DESC STORAGE INTEGRATION s3_zomato_integration;
-- Copy STORAGE_AWS_IAM_USER_ARN and STORAGE_AWS_EXTERNAL_ID
```

**Update IAM role trust** (`aws/iam/snowflake-role-trust-policy-final.json`):
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "<STORAGE_AWS_IAM_USER_ARN>"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": "<STORAGE_AWS_EXTERNAL_ID>"
      }
    }
  }]
}
```

```bash
# Update the role
aws iam update-assume-role-policy --role-name snowflake-s3-role \
  --policy-document file://aws/iam/snowflake-role-trust-policy-final.json
```

### 4. Create Snowflake Stage

```sql
CREATE STAGE ZOMATO.RAW.s3_stage
  STORAGE_INTEGRATION = s3_zomato_integration
  URL = 's3://zomato-data-lake/raw/'
  FILE_FORMAT = (TYPE = 'CSV' FIELD_DELIMITER = ',' SKIP_HEADER = 1);

-- Test access
LIST @ZOMATO.RAW.s3_stage;
```

### 5. dbt Setup

```bash
cd zomato
pip install dbt-snowflake

# Configure profiles.yml (uses env vars)
export SNOWFLAKE_ACCOUNT=<account_identifier>
export SNOWFLAKE_USER=<your_user>
export SNOWFLAKE_PASSWORD=<your_password>

# Test connection
dbt debug

# Run transformations
dbt build --exclude tag:ai
```

**dbt profiles.yml** (in `~/.dbt/` or `zomato/`):
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

### 6. Airflow Setup

```bash
cd airflow

# Create .env from template
cp example.env .env

# Edit .env with credentials
cat .env
```

**.env file:**
```bash
SNOWFLAKE_ACCOUNT=abc12345.us-east-1
SNOWFLAKE_USER=dbt_user
SNOWFLAKE_PASSWORD=your_password
SNOWFLAKE_DATABASE=ZOMATO
SNOWFLAKE_WAREHOUSE=ZOMATO_WH
SNOWFLAKE_ROLE=DBT_ROLE
OPENAI_API_KEY=sk-...
SAMPLE_N=1000  # Limit for LLM enrichment
```

**Start Airflow:**
```bash
docker compose build
docker compose up -d

# Access UI: http://localhost:8080
# Default credentials: airflow / airflow
```

## Key dbt Models

### Staging Layer (Silver)

**`models/staging/stg_restaurants.sql`:**
```sql
{{ config(materialized='view') }}

SELECT
    restaurant_id,
    restaurant_name,
    NULLIF(TRIM(cuisine_type), '--') AS cuisine_type,
    city,
    CAST(REPLACE(REPLACE(average_cost_for_two, '₹', ''), ' ', '') AS INTEGER) AS average_cost_for_two,
    rating::FLOAT AS rating,
    votes::INTEGER AS votes,
    delivery_time::INTEGER AS delivery_time_minutes
FROM {{ source('zomato_raw', 'restaurants') }}
WHERE restaurant_id IS NOT NULL
```

**`models/staging/stg_orders.sql`:**
```sql
{{ config(materialized='view') }}

SELECT
    order_id,
    user_id,
    restaurant_id,
    order_date::DATE AS order_date,
    order_time::TIME AS order_time,
    order_value::FLOAT AS order_value,
    delivery_fee::FLOAT AS delivery_fee,
    LOWER(payment_method) AS payment_method,
    discount_applied::FLOAT AS discount_applied,
    LOWER(order_status) AS order_status,
    CASE WHEN LOWER(order_status) = 'delivered' THEN TRUE ELSE FALSE END AS is_delivered,
    rating::FLOAT AS rating
FROM {{ source('zomato_raw', 'orders') }}
WHERE order_id IS NOT NULL
```

### Marts Layer (Gold)

**`models/marts/dim_restaurants.sql`:**
```sql
{{ config(materialized='table') }}

SELECT
    restaurant_id,
    restaurant_name,
    cuisine_type,
    city,
    average_cost_for_two,
    rating,
    votes,
    delivery_time_minutes,
    CURRENT_TIMESTAMP() AS updated_at
FROM {{ ref('stg_restaurants') }}
```

**Incremental fact table** (`models/marts/fct_orders.sql`):
```sql
{{
  config(
    materialized='incremental',
    unique_key='order_id',
    on_schema_change='fail'
  )
}}

SELECT
    o.order_id,
    o.user_id,
    o.restaurant_id,
    o.order_date,
    o.order_time,
    o.order_value,
    o.delivery_fee,
    o.payment_method,
    o.discount_applied,
    o.order_status,
    o.is_delivered,
    o.rating AS order_rating
FROM {{ ref('stg_orders') }} o

{% if is_incremental() %}
WHERE o.order_date > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

**Business mart** (`models/marts/mart_daily_city_revenue.sql`):
```sql
{{ config(materialized='table') }}

SELECT
    o.order_date,
    r.city,
    COUNT(DISTINCT o.order_id) AS total_orders,
    SUM(o.order_value) AS gross_revenue,
    AVG(o.order_value) AS avg_order_value,
    SUM(CASE WHEN o.is_delivered THEN 0 ELSE 1 END) AS cancelled_orders,
    ROUND(100.0 * SUM(CASE WHEN o.is_delivered THEN 0 ELSE 1 END) / COUNT(*), 2) AS cancellation_rate_pct
FROM {{ ref('fct_orders') }} o
JOIN {{ ref('dim_restaurants') }} r ON o.restaurant_id = r.restaurant_id
GROUP BY 1, 2
ORDER BY 1 DESC, 3 DESC
```

## Airflow DAG

**`airflow/dags/zomato_batch.py`:**
```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from datetime import datetime, timedelta
import os

default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'email_on_failure': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    'zomato_batch',
    default_args=default_args,
    description='Zomato end-to-end data pipeline',
    schedule_interval='@daily',
    catchup=False,
)

# Task 1: Reload raw tables from S3
reload_raw = SnowflakeOperator(
    task_id='reload_raw',
    snowflake_conn_id='snowflake_default',
    sql="""
        USE SCHEMA ZOMATO.RAW;
        
        -- Truncate and reload orders
        TRUNCATE TABLE orders;
        COPY INTO orders FROM @s3_stage/orders/
        FILE_FORMAT = (TYPE = 'CSV' FIELD_DELIMITER = ',' SKIP_HEADER = 1)
        ON_ERROR = 'CONTINUE';
        
        -- Truncate and reload order_items
        TRUNCATE TABLE order_items;
        COPY INTO order_items FROM @s3_stage/order_items/
        FILE_FORMAT = (TYPE = 'CSV' FIELD_DELIMITER = ',' SKIP_HEADER = 1)
        ON_ERROR = 'CONTINUE';
        
        -- Reviews
        TRUNCATE TABLE reviews;
        COPY INTO reviews FROM @s3_stage/reviews/
        FILE_FORMAT = (TYPE = 'CSV' FIELD_DELIMITER = ',' SKIP_HEADER = 1)
        ON_ERROR = 'CONTINUE';
    """,
    dag=dag,
)

# Task 2: dbt build (core models)
dbt_build_core = BashOperator(
    task_id='dbt_build_core',
    bash_command="""
        cd /opt/airflow/dbt/zomato && \
        source /opt/airflow/dbt_venv/bin/activate && \
        dbt build --exclude tag:ai
    """,
    env={
        'SNOWFLAKE_ACCOUNT': os.getenv('SNOWFLAKE_ACCOUNT'),
        'SNOWFLAKE_USER': os.getenv('SNOWFLAKE_USER'),
        'SNOWFLAKE_PASSWORD': os.getenv('SNOWFLAKE_PASSWORD'),
    },
    dag=dag,
)

# Task 3: Enrich reviews with OpenAI
enrich_reviews = BashOperator(
    task_id='enrich_reviews',
    bash_command="""
        cd /opt/airflow/ai && \
        python enrich_reviews.py
    """,
    env={
        'SNOWFLAKE_ACCOUNT': os.getenv('SNOWFLAKE_ACCOUNT'),
        'SNOWFLAKE_USER': os.getenv('SNOWFLAKE_USER'),
        'SNOWFLAKE_PASSWORD': os.getenv('SNOWFLAKE_PASSWORD'),
        'OPENAI_API_KEY': os.getenv('OPENAI_API_KEY'),
        'SAMPLE_N': os.getenv('SAMPLE_N', '1000'),
    },
    dag=dag,
)

# Task 4: Build AI mart
dbt_build_ai = BashOperator(
    task_id='dbt_build_ai',
    bash_command="""
        cd /opt/airflow/dbt/zomato && \
        source /opt/airflow/dbt_venv/bin/activate && \
        dbt run --select tag:ai
    """,
    env={
        'SNOWFLAKE_ACCOUNT': os.getenv('SNOWFLAKE_ACCOUNT'),
        'SNOWFLAKE_USER': os.getenv('SNOWFLAKE_USER'),
        'SNOWFLAKE_PASSWORD': os.getenv('SNOWFLAKE_PASSWORD'),
    },
    dag=dag,
)

# Pipeline flow
reload_raw >> dbt_build_core >> enrich_reviews >> dbt_build_ai
```

## AI Layer

### 1. LLM Enrichment

**`ai/enrich_reviews.py`:**
```python
import os
import json
from openai import OpenAI
import snowflake.connector

# Initialize
client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))
SAMPLE_N = int(os.getenv('SAMPLE_N', 1000))

# Connect to Snowflake
conn = snowflake.connector.connect(
    account=os.getenv('SNOWFLAKE_ACCOUNT'),
    user=os.getenv('SNOWFLAKE_USER'),
    password=os.getenv('SNOWFLAKE_PASSWORD'),
    database='ZOMATO',
    warehouse='ZOMATO_WH',
    role='DBT_ROLE'
)
cursor = conn.cursor()

# Create enriched table if not exists
cursor.execute("""
    CREATE TABLE IF NOT EXISTS ZOMATO.AI.REVIEW_ENRICHED (
        review_id INTEGER,
        review_text STRING,
        sentiment STRING,
        topic STRING,
        processed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
        PRIMARY KEY (review_id)
    )
""")

# Fetch unenriched reviews
cursor.execute(f"""
    SELECT r.review_id, r.review_text
    FROM ZOMATO.RAW.reviews r
    LEFT JOIN ZOMATO.AI.REVIEW_ENRICHED e ON r.review_id = e.review_id
    WHERE e.review_id IS NULL
    LIMIT {SAMPLE_N}
""")
reviews = cursor.fetchall()

print(f"Enriching {len(reviews)} reviews...")

for review_id, review_text in reviews:
    try:
        response = client.chat.completions.create(
            model='gpt-4o-mini',
            messages=[{
                'role': 'system',
                'content': 'You are a sentiment analyzer. Return JSON with "sentiment" (positive/negative/neutral) and "topic" (food/delivery/service/price).'
            }, {
                'role': 'user',
                'content': f'Analyze: {review_text}'
            }],
            response_format={'type': 'json_object'}
        )
        
        result = json.loads(response.choices[0].message.content)
        sentiment = result.get('sentiment', 'neutral')
        topic = result.get('topic', 'general')
        
        # Insert enriched review
        cursor.execute("""
            INSERT INTO ZOMATO.AI.REVIEW_ENRICHED (review_id, review_text, sentiment, topic)
            VALUES (%s, %s, %s, %s)
        """, (review_id, review_text, sentiment, topic))
        
        if review_id % 100 == 0:
            print(f"Processed {review_id}")
    
    except Exception as e:
        print(f"Error on review {review_id}: {e}")
        continue

conn.commit()
cursor.close()
conn.close()
print("Enrichment complete!")
```

### 2. RAG Chat with Reviews

**`ai/rag_chat.py`:**
```python
import os
import streamlit as st
from openai import OpenAI
import snowflake.connector
import numpy as np

client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))

# Connect to Snowflake
@st.cache_resource
def get_connection():
    return snowflake.connector.connect(
        account=os.getenv('SNOWFLAKE_ACCOUNT'),
        user=os.getenv('SNOWFLAKE_USER'),
        password=os.getenv('SNOWFLAKE_PASSWORD'),
        database='ZOMATO',
        warehouse='ZOMATO_WH'
    )

def embed_text(text):
    response = client.embeddings.create(
        model='text-embedding-3-small',
        input=text
    )
    return response.data[0].embedding

def retrieve_reviews(question_embedding, top_k=5):
    conn = get_connection()
    cursor = conn.cursor()
    
    # Fetch all reviews (in production, use vector search)
    cursor.execute("""
        SELECT review_id, review_text, sentiment, topic
        FROM ZOMATO.AI.REVIEW_ENRICHED
        LIMIT 10000
    """)
    reviews = cursor.fetchall()
    cursor.close()
    
    # Compute similarity (simplified - use vector DB in production)
    similarities = []
    for review_id, text, sentiment, topic in reviews:
        review_embedding = embed_text(text)
        similarity = np.dot(question_embedding, review_embedding)
        similarities.append((similarity, review_id, text, sentiment, topic))
    
    # Return top K
    similarities.sort(reverse=True)
    return similarities[:top_k]

st.title("🍔 Chat with Zomato Reviews")
st.write("Ask questions about customer reviews!")

question = st.text_input("Your question:", "What do customers say about delivery times?")

if st.button("Search"):
    with st.spinner("Searching reviews..."):
        # Embed question
        question_emb = embed_text(question)
        
        # Retrieve similar reviews
        results = retrieve_reviews(question_emb, top_k=5)
        
        # Build context
        context = "\n\n".join([
            f"Review {i+1} ({sentiment}, {topic}): {text}"
            for i, (_, _, text, sentiment, topic) in enumerate(results)
        ])
        
        # Generate answer
        response = client.chat.completions.create(
            model='gpt-4o-mini',
            messages=[{
                'role': 'system',
                'content': 'Answer based on the provided reviews. Cite review numbers.'
            }, {
                'role': 'user',
                'content': f"Context:\n{context}\n\nQuestion: {question}"
            }]
        )
        
        st.success(response.choices[0].message.content)
        
        st.subheader("Source Reviews:")
        for i, (_, review_id, text, sentiment, topic) in enumerate(results):
            st.markdown(f"**Review {i+1}** ({sentiment}, {topic}): {text}")
```

### 3. Text-to-SQL

**`ai/text_to_sql.py`:**
```python
import os
import streamlit as st
from openai import OpenAI
import snowflake.connector
import re

client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))

@st.cache_resource
def get_connection():
    return snowflake.connector.connect(
        account=os.getenv('SNOWFLAKE_ACCOUNT'),
        user=os.getenv('SNOWFLAKE_USER'),
        password=os.getenv('SNOWFLAKE_PASSWORD'),
        database='ZOMATO',
        warehouse='ZOMATO_WH',
        role='DBT_ROLE'
    )

# Schema context for LLM
SCHEMA_CONTEXT = """
Available tables in ZOMATO.MARTS:
- dim_restaurants (restaurant_id, restaurant_name, cuisine_type, city, rating, delivery_time_minutes)
- fct_orders (order_id, user_id, restaurant_id, order_date, order_value, is_delivered, order_rating)
- mart_daily_city_revenue (order_date, city, total_orders, gross_revenue, avg_order_value, cancellation_rate_pct)
- mart_restaurant_performance (restaurant_id, restaurant_name, total_orders, total_revenue, avg_rating)
"""

def generate_sql(question):
    response = client.chat.completions.create(
        model='gpt-4o-mini',
        messages=[{
            'role': 'system',
            'content': f'You are a Snowflake SQL expert. Generate valid SQL queries.\n\n{SCHEMA_CONTEXT}\n\nReturn ONLY the SQL query, no explanations.'
        }, {
            'role': 'user',
            'content': question
        }]
    )
    return response.choices[0].message.content.strip().strip('```sql').strip('```').strip()

def is_safe_sql(sql):
    """Validate SQL is SELECT-only"""
    sql_lower = sql.lower()
    forbidden = ['insert', 'update', 'delete', 'drop', 'truncate', 'alter', 'create']
    return not any(keyword in sql_lower for keyword in forbidden)

st.title("💬 Chat with Zomato Data Warehouse")
st.write("Ask questions in plain English - I'll write the SQL!")

question = st.text_input("Your question:", "Which city has the highest average order value?")

if st.button("Run Query"):
    with st.spinner("Generating SQL..."):
        sql = generate_sql(question)
        
        st.code(sql, language='sql')
        
        if not is_safe_sql(sql):
            st.error("⚠️ Query contains unsafe operations. Only SELECT queries are allowed.")
        else:
            try:
                conn = get_connection()
                cursor = conn.cursor()
                cursor.execute(sql)
                results = cursor.fetchall()
                columns = [desc[0] for desc in cursor.description]
                cursor.close()
                
                st.success(f"✅ Returned {len(results)} rows")
                st.dataframe(results, column_config={col: col for col in columns})
            
            except Exception as e:
                st.error(f"Query error: {e}")
```

## Common dbt Commands

```bash
# Test connection
dbt debug

# Run all models
dbt run

# Run specific model
dbt run --select mart_daily_city_revenue

# Run model and downstream dependencies
dbt run --select stg_orders+

# Build (run + test)
dbt build

# Run only incremental models
dbt run --select config.materialized:incremental

# Test data quality
dbt test

# Generate documentation
dbt docs generate
dbt docs serve  # http://localhost:8080

# Snapshot (SCD Type 2)
dbt snapshot
```

## Configuration Files

**`zomato/dbt_project.yml`:**
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

target-path: "target"
clean-targets:
  - "target"
  - "dbt_packages"

models:
  zomato:
    staging:
      +materialized: view
      +schema: staging
    marts:
      +materialized: table
      +schema: marts
```

**`zomato/models/sources.yml`:**
```yaml
version: 2

sources:
  - name: zomato_raw
    database: ZOMATO
    schema: RAW
    tables:
      - name: restaurants
        columns:
          - name: restaurant_id
            tests:
              - unique
              - not_null
      - name: users
      - name: food
      - name: menu
      - name: orders
        columns:
          - name: order_id
            tests:
              - unique
              - not_null
      - name: order_items
      - name: reviews
```

## Troubleshooting

### Storage Integration Issues

**Error:** `SQL access control error: Insufficient privileges to operate on stage`

**Fix:**
```sql
-- Grant storage integration usage
GRANT USAGE ON INTEGRATION s3_zomato_integration TO ROLE DBT_ROLE;

-- Grant stage usage
GRANT USAGE ON STAGE ZOMATO.RAW.s3_stage TO ROLE DBT_ROLE;
```

### IAM Trust Policy Broken

**Error:** `S3_ACCESS_DENIED: Access Denied`

**Cause:** Re-running `CREATE OR REPLACE` on the storage integration regenerates the external ID.

**Fix:** Never use `CREATE OR REPLACE`. If broken:
1. `DESC STORAGE INTEGRATION s3_zomato_integration;`
2. Copy new `STORAGE_AWS_EXTERNAL_ID`
3. Update IAM role trust policy with new ID

### dbt Incremental Not Detecting New Rows

**Issue:** Incremental model rebuilds all rows.

**Debug:**
```sql
-- Check incremental logic
{% if is_incremental() %}
  -- This should filter to new rows only
  WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

**Fix:** Ensure `unique_key` is set and `this` table exists:
```sql
{{ config(
    materialized='incremental',
    unique_key='order_id'
) }}
```

### OpenAI Rate Limits

**Error:** `Rate limit exceeded`

**Fix:** Reduce `SAMPLE_N` in `.env`:
```bash
SAMPLE_N=100  # Process fewer reviews per run
```

Add retry logic:
```python
from tenacity import retry, wait_exponential, stop_after_attempt

@retry(wait=wait_exponential(min=1, max=60), stop=stop_after_attempt(3))
def call_openai(text):
    return client.chat.completions.create(...)
```

### Airflow Task Fails with dbt Error

**Debug:**
```bash
# View logs
docker logs airflow-scheduler-1

# Check dbt version in container
docker exec -it airflow-scheduler-1 bash
source /opt/airflow/dbt_venv/bin/activate
dbt --version
```

**Common fix:** Ensure dbt is in its own venv (not system Python):
```dockerfile
# In Dockerfile
RUN python -m venv /opt/airflow/dbt_venv && \
    /opt/airflow/dbt_venv/bin/pip install dbt-snowflake==1.7.0
```

### Snowflake Connection Timeout

**Error:** `250001: Could not connect to Snowflake backend`

**Fix:**
```python
# Add timeout and retry params
conn = snowflake.connector.connect(
    account=os.getenv('SNOWFLAKE_ACCOUNT'),
    user=os.getenv('SNOWFLAKE_USER'),
    password=os.getenv('SNOWFLAKE_PASSWORD'),
    login_timeout=30,
    network_timeout=30
)
```

## Best Practices

1. **Never commit credentials** — use env vars or secrets manager
2. **Use incremental models** for large fact tables (10M+ rows)
3. **Separate dbt
