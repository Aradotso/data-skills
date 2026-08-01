---
name: zomato-ai-data-engineering-pipeline
description: End-to-end data pipeline with S3, Snowflake, dbt, Airflow, and OpenAI for batch ETL and AI-powered analytics
triggers:
  - build a zomato data pipeline
  - set up snowflake dbt airflow pipeline
  - create batch data engineering workflow
  - integrate openai with data warehouse
  - build medallion architecture pipeline
  - set up incremental dbt models
  - create RAG chat with snowflake
  - implement text to sql with llm
---

# Zomato AI Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A production-ready batch data pipeline demonstrating modern data engineering patterns: S3 data lake → Snowflake warehouse → dbt transformations (medallion architecture) → Airflow orchestration → OpenAI-powered analytics (LLM enrichment, RAG, text-to-SQL).

## What It Does

- **Ingests** 10M+ orders, 23M+ order items, 300K reviews from CSV → S3 → Snowflake
- **Transforms** data through Bronze (RAW) → Silver (STAGING) → Gold (MARTS) layers using dbt
- **Orchestrates** daily batch pipeline with Apache Airflow 3
- **Enriches** reviews with GPT-4 (sentiment, topics)
- **Enables** natural language queries via RAG and text-to-SQL

## Architecture Overview

```
CSV files → S3 bucket → Snowflake (storage integration)
                            ↓
                    RAW tables (Bronze)
                            ↓
                    dbt STAGING (Silver) — views
                            ↓
                    dbt MARTS (Gold) — dims, facts, aggregates
                            ↓
                    AI layer (OpenAI) — enriched, RAG, text-to-SQL
                            ↓
                    Streamlit dashboards
```

## Prerequisites

- **AWS**: S3 bucket, IAM role with trust to Snowflake
- **Snowflake**: Account, warehouse, database
- **Python**: 3.9+
- **Docker**: For Airflow
- **OpenAI**: API key for AI features

## Installation

```bash
# Clone the repository
git clone https://github.com/darshilparmar/zomato-ai-data-engineering-end-to-end-project.git
cd zomato-ai-data-engineering-end-to-end-project

# Download dataset from Google Drive (link in README)
# Place CSVs in data/ directory

# Install dbt dependencies
cd zomato
pip install dbt-snowflake
```

## Snowflake Setup

### 1. Create Core Objects

```sql
-- Warehouse
CREATE WAREHOUSE ZOMATO_WH 
  WITH WAREHOUSE_SIZE = 'MEDIUM' 
  AUTO_SUSPEND = 300 
  AUTO_RESUME = TRUE;

-- Database and schemas
CREATE DATABASE ZOMATO;
CREATE SCHEMA ZOMATO.RAW;
CREATE SCHEMA ZOMATO.STAGING;
CREATE SCHEMA ZOMATO.MARTS;
CREATE SCHEMA ZOMATO.SNAPSHOTS;
CREATE SCHEMA ZOMATO.AI;

-- Role for dbt
CREATE ROLE DBT_ROLE;
GRANT USAGE ON WAREHOUSE ZOMATO_WH TO ROLE DBT_ROLE;
GRANT ALL ON DATABASE ZOMATO TO ROLE DBT_ROLE;
GRANT ROLE DBT_ROLE TO USER <YOUR_USER>;
```

### 2. Set Up S3 Storage Integration

First, create AWS IAM policy (`aws/iam/s3-read-policy.json`):

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
        "arn:aws:s3:::your-bucket-name/*",
        "arn:aws:s3:::your-bucket-name"
      ]
    }
  ]
}
```

Create IAM role with initial trust policy, then create Snowflake integration:

```sql
-- In Snowflake
CREATE STORAGE INTEGRATION zomato_s3_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = S3
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake-s3-role'
  STORAGE_ALLOWED_LOCATIONS = ('s3://your-bucket-name/raw/');

-- Get Snowflake's IAM user and external ID
DESC STORAGE INTEGRATION zomato_s3_integration;
-- Note STORAGE_AWS_IAM_USER_ARN and STORAGE_AWS_EXTERNAL_ID
```

Update IAM role trust policy with values from DESC output (`aws/iam/snowflake-role-trust-policy-final.json`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/abc-123"
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

### 3. Create Stage and RAW Tables

```sql
-- Stage pointing to S3
CREATE STAGE ZOMATO.RAW.S3_STAGE
  STORAGE_INTEGRATION = zomato_s3_integration
  URL = 's3://your-bucket-name/raw/'
  FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"');

-- RAW tables (example for orders)
CREATE TABLE ZOMATO.RAW.ORDERS (
  order_id VARCHAR,
  user_id VARCHAR,
  restaurant_id VARCHAR,
  order_date VARCHAR,
  order_time VARCHAR,
  order_value VARCHAR,
  delivery_time VARCHAR,
  order_status VARCHAR,
  payment_method VARCHAR,
  discount_applied VARCHAR,
  delivery_fee VARCHAR
);

-- Load data
COPY INTO ZOMATO.RAW.ORDERS
FROM @ZOMATO.RAW.S3_STAGE/orders/
FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"')
ON_ERROR = 'CONTINUE';
```

## dbt Configuration

### profiles.yml

Create `~/.dbt/profiles.yml`:

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

Set environment variables:

```bash
export SNOWFLAKE_ACCOUNT=xy12345.us-east-1
export SNOWFLAKE_USER=your_username
export SNOWFLAKE_PASSWORD=your_password
```

### Key dbt Commands

```bash
cd zomato

# Test connection
dbt debug

# Compile models without running
dbt compile

# Run staging models only
dbt run --select staging

# Run marts with incremental refresh
dbt run --select marts

# Run models and tests
dbt build

# Run excluding AI models
dbt build --exclude tag:ai

# Test only
dbt test

# Generate documentation
dbt docs generate
dbt docs serve
```

## dbt Model Structure

### Staging (Silver Layer)

Transform RAW → clean views with type casting and business logic:

```sql
-- models/staging/stg_orders.sql
{{ config(materialized='view') }}

WITH source AS (
  SELECT * FROM {{ source('raw', 'orders') }}
),

cleaned AS (
  SELECT
    order_id::VARCHAR AS order_id,
    user_id::VARCHAR AS user_id,
    restaurant_id::VARCHAR AS restaurant_id,
    TO_DATE(order_date, 'DD-MM-YYYY') AS order_date,
    order_time::TIME AS order_time,
    TRY_CAST(order_value AS DECIMAL(10,2)) AS order_value,
    TRY_CAST(delivery_time AS INTEGER) AS delivery_time_minutes,
    LOWER(TRIM(order_status)) AS order_status,
    LOWER(TRIM(payment_method)) AS payment_method,
    TRY_CAST(discount_applied AS DECIMAL(10,2)) AS discount_applied,
    TRY_CAST(delivery_fee AS DECIMAL(10,2)) AS delivery_fee,
    -- Derived fields
    CASE 
      WHEN LOWER(order_status) = 'delivered' THEN TRUE 
      ELSE FALSE 
    END AS is_delivered
  FROM source
)

SELECT * FROM cleaned
```

### Marts (Gold Layer) - Incremental Facts

Use incremental materialization for large fact tables:

```sql
-- models/marts/fct_orders.sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    on_schema_change='sync_all_columns'
) }}

SELECT
  o.order_id,
  o.user_id,
  o.restaurant_id,
  o.order_date,
  o.order_time,
  o.order_value,
  o.delivery_time_minutes,
  o.order_status,
  o.payment_method,
  o.discount_applied,
  o.delivery_fee,
  o.is_delivered,
  -- Metrics
  o.order_value - COALESCE(o.discount_applied, 0) AS net_order_value,
  o.order_value + COALESCE(o.delivery_fee, 0) AS total_order_value
FROM {{ ref('stg_orders') }} o

{% if is_incremental() %}
  WHERE o.order_date > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

### Business Marts

Aggregate marts for analytics:

```sql
-- models/marts/mart_daily_city_metrics.sql
{{ config(materialized='table') }}

SELECT
  r.city,
  o.order_date,
  COUNT(DISTINCT o.order_id) AS total_orders,
  COUNT(DISTINCT CASE WHEN o.is_delivered THEN o.order_id END) AS delivered_orders,
  SUM(o.net_order_value) AS gmv,
  AVG(o.net_order_value) AS avg_order_value,
  SUM(o.discount_applied) AS total_discounts,
  -- Cancel rate
  1.0 - (COUNT(DISTINCT CASE WHEN o.is_delivered THEN o.order_id END)::FLOAT 
         / NULLIF(COUNT(DISTINCT o.order_id), 0)) AS cancel_rate
FROM {{ ref('fct_orders') }} o
JOIN {{ ref('dim_restaurants') }} r ON o.restaurant_id = r.restaurant_id
GROUP BY 1, 2
```

## Airflow Orchestration

### Setup

```bash
cd airflow

# Copy and fill environment variables
cp example.env .env
# Edit .env with your credentials:
# SNOWFLAKE_ACCOUNT=xy12345.us-east-1
# SNOWFLAKE_USER=your_user
# SNOWFLAKE_PASSWORD=your_password
# SNOWFLAKE_WAREHOUSE=ZOMATO_WH
# SNOWFLAKE_DATABASE=ZOMATO
# SNOWFLAKE_ROLE=DBT_ROLE
# OPENAI_API_KEY=sk-...
# SAMPLE_N=1000

# Build and start Airflow
docker compose build
docker compose up -d

# Access UI at http://localhost:8080
# Default credentials: airflow / airflow
```

### Pipeline DAG

The `zomato_batch` DAG runs daily with four tasks:

```python
# airflow/dags/zomato_batch.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from datetime import datetime, timedelta
import subprocess

default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

def run_dbt_build(exclude_tag=None):
    """Run dbt build with optional tag exclusion"""
    cmd = ['dbt', 'build', '--project-dir', '/opt/airflow/zomato']
    if exclude_tag:
        cmd.extend(['--exclude', f'tag:{exclude_tag}'])
    subprocess.run(cmd, check=True)

def run_ai_enrichment():
    """Run OpenAI review enrichment"""
    subprocess.run([
        'python', '/opt/airflow/ai/enrich_reviews.py'
    ], check=True)

with DAG(
    'zomato_batch',
    default_args=default_args,
    description='Zomato batch pipeline: S3 → Snowflake → dbt → AI',
    schedule_interval='0 2 * * *',  # Daily at 2 AM
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=['zomato', 'batch'],
) as dag:

    # Task 1: Reload RAW tables from S3
    reload_raw = SnowflakeOperator(
        task_id='reload_raw',
        snowflake_conn_id='snowflake_default',
        sql="""
        COPY INTO ZOMATO.RAW.ORDERS FROM @ZOMATO.RAW.S3_STAGE/orders/
        FILE_FORMAT = (TYPE=CSV SKIP_HEADER=1) ON_ERROR='CONTINUE';
        
        COPY INTO ZOMATO.RAW.ORDER_ITEMS FROM @ZOMATO.RAW.S3_STAGE/order_items/
        FILE_FORMAT = (TYPE=CSV SKIP_HEADER=1) ON_ERROR='CONTINUE';
        
        COPY INTO ZOMATO.RAW.REVIEWS FROM @ZOMATO.RAW.S3_STAGE/reviews/
        FILE_FORMAT = (TYPE=CSV SKIP_HEADER=1) ON_ERROR='CONTINUE';
        """
    )

    # Task 2: dbt build (staging + marts, exclude AI)
    dbt_build_core = PythonOperator(
        task_id='dbt_build_core',
        python_callable=run_dbt_build,
        op_kwargs={'exclude_tag': 'ai'}
    )

    # Task 3: AI enrichment
    enrich_reviews = PythonOperator(
        task_id='enrich_reviews',
        python_callable=run_ai_enrichment
    )

    # Task 4: Build AI marts
    dbt_build_ai = PythonOperator(
        task_id='dbt_build_ai',
        python_callable=run_dbt_build,
        op_kwargs={'exclude_tag': None}
    )

    reload_raw >> dbt_build_core >> enrich_reviews >> dbt_build_ai
```

## AI Layer

### 1. LLM Enrichment

Enrich reviews with sentiment and topics using GPT-4:

```python
# ai/enrich_reviews.py
import os
import snowflake.connector
from openai import OpenAI
import json

# Configuration
SAMPLE_N = int(os.getenv('SAMPLE_N', 1000))
client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))

def get_snowflake_connection():
    return snowflake.connector.connect(
        account=os.getenv('SNOWFLAKE_ACCOUNT'),
        user=os.getenv('SNOWFLAKE_USER'),
        password=os.getenv('SNOWFLAKE_PASSWORD'),
        warehouse=os.getenv('SNOWFLAKE_WAREHOUSE'),
        database=os.getenv('SNOWFLAKE_DATABASE'),
        role=os.getenv('SNOWFLAKE_ROLE')
    )

def enrich_review(review_text):
    """Call OpenAI to extract sentiment and topics"""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{
            "role": "system",
            "content": "Extract sentiment (positive/negative/neutral) and topics from food delivery reviews. Return only valid JSON."
        }, {
            "role": "user",
            "content": f"Review: {review_text}\n\nReturn JSON: {{\"sentiment\": \"...\", \"topics\": [\"...\", \"...\"]}}"
        }],
        temperature=0.3
    )
    
    result = json.loads(response.choices[0].message.content)
    return result['sentiment'], json.dumps(result['topics'])

def main():
    conn = get_snowflake_connection()
    cur = conn.cursor()
    
    # Create enriched table if not exists
    cur.execute("""
        CREATE TABLE IF NOT EXISTS ZOMATO.AI.REVIEW_ENRICHED (
            review_id VARCHAR,
            review_text VARCHAR,
            sentiment VARCHAR,
            topics VARCHAR,
            enriched_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
        )
    """)
    
    # Get reviews not yet enriched
    cur.execute(f"""
        SELECT r.review_id, r.review_text
        FROM ZOMATO.RAW.REVIEWS r
        LEFT JOIN ZOMATO.AI.REVIEW_ENRICHED e ON r.review_id = e.review_id
        WHERE e.review_id IS NULL
        LIMIT {SAMPLE_N}
    """)
    
    reviews = cur.fetchall()
    print(f"Enriching {len(reviews)} reviews...")
    
    for review_id, review_text in reviews:
        try:
            sentiment, topics = enrich_review(review_text)
            cur.execute("""
                INSERT INTO ZOMATO.AI.REVIEW_ENRICHED (review_id, review_text, sentiment, topics)
                VALUES (%s, %s, %s, %s)
            """, (review_id, review_text, sentiment, topics))
            print(f"✓ {review_id}: {sentiment}")
        except Exception as e:
            print(f"✗ {review_id}: {e}")
    
    conn.commit()
    cur.close()
    conn.close()
    print("Done!")

if __name__ == '__main__':
    main()
```

Run manually:

```bash
export OPENAI_API_KEY=sk-...
export SNOWFLAKE_ACCOUNT=xy12345.us-east-1
export SNOWFLAKE_USER=your_user
export SNOWFLAKE_PASSWORD=your_password
export SNOWFLAKE_WAREHOUSE=ZOMATO_WH
export SNOWFLAKE_DATABASE=ZOMATO
export SNOWFLAKE_ROLE=DBT_ROLE
export SAMPLE_N=1000

python ai/enrich_reviews.py
```

### 2. RAG Chat with Reviews

Chat interface using embeddings and retrieval:

```python
# ai/rag_chat.py
import streamlit as st
import snowflake.connector
from openai import OpenAI
import numpy as np

client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))

def get_embedding(text):
    """Get OpenAI embedding"""
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return response.data[0].embedding

def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

def search_reviews(query, top_k=5):
    """Search for most relevant reviews"""
    query_embedding = get_embedding(query)
    
    conn = get_snowflake_connection()
    cur = conn.cursor()
    
    # Get all reviews with embeddings (in production, use vector DB)
    cur.execute("""
        SELECT review_id, review_text, sentiment
        FROM ZOMATO.AI.REVIEW_ENRICHED
        LIMIT 1000
    """)
    
    reviews = cur.fetchall()
    
    # Calculate similarity (simplified - use vector DB for scale)
    results = []
    for review_id, text, sentiment in reviews:
        review_embedding = get_embedding(text)
        similarity = cosine_similarity(query_embedding, review_embedding)
        results.append((similarity, review_id, text, sentiment))
    
    results.sort(reverse=True, key=lambda x: x[0])
    return results[:top_k]

def generate_answer(query, context_reviews):
    """Generate answer using retrieved reviews"""
    context = "\n\n".join([
        f"Review {i+1} ({sentiment}): {text}"
        for i, (_, _, text, sentiment) in enumerate(context_reviews)
    ])
    
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{
            "role": "system",
            "content": "Answer questions about food delivery reviews using only the provided context."
        }, {
            "role": "user",
            "content": f"Context:\n{context}\n\nQuestion: {query}"
        }]
    )
    
    return response.choices[0].message.content

# Streamlit UI
st.title("🍕 Chat with Zomato Reviews")

query = st.text_input("Ask about customer reviews:")

if query:
    with st.spinner("Searching reviews..."):
        relevant_reviews = search_reviews(query)
    
    with st.spinner("Generating answer..."):
        answer = generate_answer(query, relevant_reviews)
    
    st.write("### Answer")
    st.write(answer)
    
    st.write("### Sources")
    for similarity, review_id, text, sentiment in relevant_reviews:
        st.write(f"**{review_id}** ({sentiment}) - Similarity: {similarity:.3f}")
        st.write(text)
        st.write("---")
```

Run:

```bash
streamlit run ai/rag_chat.py
```

### 3. Text-to-SQL

Natural language queries to SQL:

```python
# ai/text_to_sql.py
import streamlit as st
import snowflake.connector
from openai import OpenAI
import os

client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))

# Schema context for the LLM
SCHEMA_CONTEXT = """
Available tables in ZOMATO.MARTS:

1. DIM_RESTAURANTS: restaurant_id, name, city, cuisine, rating, cost_for_two
2. DIM_CUSTOMER: user_id, name, email, city, age, age_segment
3. DIM_FOOD: food_id, name, category, is_veg
4. FCT_ORDERS: order_id, user_id, restaurant_id, order_date, order_value, is_delivered
5. MART_DAILY_CITY_METRICS: city, order_date, total_orders, gmv, avg_order_value, cancel_rate
6. MART_RESTAURANT_PERFORMANCE: restaurant_id, total_orders, total_revenue, avg_rating
"""

def generate_sql(question):
    """Convert natural language to SQL"""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{
            "role": "system",
            "content": f"You are a Snowflake SQL expert. Generate valid SELECT queries.\n\n{SCHEMA_CONTEXT}"
        }, {
            "role": "user",
            "content": f"Write a SQL query for: {question}\n\nReturn ONLY the SQL, no explanation."
        }],
        temperature=0
    )
    
    return response.choices[0].message.content.strip()

def validate_sql(sql):
    """Basic validation - only allow SELECT"""
    sql_upper = sql.upper().strip()
    dangerous = ['DROP', 'DELETE', 'UPDATE', 'INSERT', 'TRUNCATE', 'ALTER', 'CREATE']
    
    if not sql_upper.startswith('SELECT'):
        return False, "Only SELECT queries are allowed"
    
    for keyword in dangerous:
        if keyword in sql_upper:
            return False, f"Keyword {keyword} is not allowed"
    
    return True, "OK"

def execute_sql(sql):
    """Execute SQL and return results"""
    conn = snowflake.connector.connect(
        account=os.getenv('SNOWFLAKE_ACCOUNT'),
        user=os.getenv('SNOWFLAKE_USER'),
        password=os.getenv('SNOWFLAKE_PASSWORD'),
        warehouse=os.getenv('SNOWFLAKE_WAREHOUSE'),
        database=os.getenv('SNOWFLAKE_DATABASE'),
        role=os.getenv('SNOWFLAKE_ROLE')
    )
    
    cur = conn.cursor()
    cur.execute(sql)
    results = cur.fetchall()
    columns = [desc[0] for desc in cur.description]
    
    cur.close()
    conn.close()
    
    return columns, results

# Streamlit UI
st.title("💬 Chat with Your Warehouse")
st.write("Ask questions in plain English, get SQL and results!")

question = st.text_input("What would you like to know?", 
                         placeholder="Which cities have the highest cancel rate?")

if question:
    with st.spinner("Generating SQL..."):
        sql = generate_sql(question)
    
    st.write("### Generated SQL")
    st.code(sql, language='sql')
    
    is_valid, msg = validate_sql(sql)
    
    if not is_valid:
        st.error(f"❌ Validation failed: {msg}")
    else:
        if st.button("Execute Query"):
            try:
                with st.spinner("Running query..."):
                    columns, results = execute_sql(sql)
                
                st.write("### Results")
                st.dataframe(
                    data=results,
                    column_config={col: col for col in columns}
                )
                st.success(f"✓ {len(results)} rows returned")
                
            except Exception as e:
                st.error(f"❌ Query failed: {str(e)}")
```

Run:

```bash
streamlit run ai/text_to_sql.py
```

## Common Patterns

### Incremental Processing Pattern

For large tables, use incremental materialization:

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    on_schema_change='sync_all_columns'
) }}

SELECT ... FROM {{ ref('source_model') }}

{% if is_incremental() %}
  WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

### Idempotent AI Enrichment

Never pay twice for the same LLM call:

```python
# Check if already enriched
cur.execute("""
    SELECT r.id, r.text
    FROM raw_table r
    LEFT JOIN enriched_table e ON r.id = e.id
    WHERE e.id IS NULL  -- Only new records
    LIMIT {BATCH_SIZE}
""")
```

### Medallion Layer Testing

Test data quality at each layer:

```yaml
# models/staging/schema.yml
version: 2

models:
  - name: stg_orders
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: order_value
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
```

### Airflow Task Dependencies

Chain tasks with clear dependencies:

```python
# Sequential
task1 >> task2 >> task3

# Parallel then merge
[task1, task2] >> task3

# Conditional
task1 >> branch_task
branch_task >> [taskA, taskB]
```

## Troubleshooting

### Snowflake Storage Integration Fails

**Error**: `Access Denied` when copying from S3

**Solution**: Verify trust policy has correct IAM user ARN from `DESC INTEGRATION`:

```sql
DESC STORAGE INTEGRATION zomato_s3_integration;
-- Copy STORAGE_AWS_IAM_USER_ARN and STORAGE_AWS_EXTERNAL_ID
-- Update IAM role trust policy in AWS
```

Never use `CREATE OR REPLACE` on integration after setup—it regenerates external ID.

### dbt Incremental Not Working

**Error**: Full refresh happening every run

**Solution**: Check `unique_key` exists and matches:

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id'  -- Must be in SELECT
) }}
```

Force full refresh when needed:

```bash
dbt run --full-refresh --select fct_orders
```

### Airflow Task Fails Silently

**Error**: Task marked success but didn't run

**Solution**: Check logs in container:

```bash
docker compose logs airflow-scheduler
docker compose exec airflow-scheduler cat /opt/airflow/logs/dag_id/task_id/...
```

Ensure `check=True` in subprocess calls:

```python
subprocess.run(cmd, check=True)  # Raises on non-zero exit
```

### OpenAI Rate Limits

**Error**: `RateLimitError` during enrichment

**Solution**: Add retry logic with exponential backoff:

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=1, max=60))
def enrich_review(text):
    return client.chat.completions.create(...)
```

Or batch and throttle:

```python
import time

for batch in chunks(reviews, 100):
    process_batch(batch)
    time.sleep(1)  # Rate limit to 100 req/sec
```

### dbt Tests Fail on NULL

**Error**: `unique` test fails on NULL values

**Solution**: Add `not_null` test first:

```yaml
- name: order_id
  tests:
    - not_null      # Run first
    - unique        # Then check uniqueness
```

### S3 Files Not Loading

**Error**: `COPY INTO` finds no files

**Solution**: Check stage path:

```sql
LIST @ZOMATO.RAW.S3_STAGE/orders/;

-- Verify file format
SELECT $1, $2 FROM @ZOMATO.RAW.S3_STAGE/orders/ LIMIT 10;
```

Ensure S3 path matches:

```sql
CREATE STAGE ... URL = 's3://bucket/raw/'  -- No trailing folder in URL
COPY INTO ... FROM @stage/orders/          -- Folder in COPY
```

## Best Practices

1. **Never commit credentials** — use env vars or secrets managers
2. **Test incrementally** — `dbt build` runs models + tests in dependency order
