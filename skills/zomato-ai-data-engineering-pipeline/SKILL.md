---
name: zomato-ai-data-engineering-pipeline
description: End-to-end batch data pipeline with Snowflake, dbt, Airflow, and OpenAI for food delivery analytics
triggers:
  - build a Zomato data pipeline with Snowflake and dbt
  - set up medallion architecture with bronze silver gold layers
  - create AI-powered analytics with OpenAI and RAG
  - orchestrate dbt with Airflow for batch processing
  - implement incremental models in dbt for large fact tables
  - add LLM enrichment to data pipeline
  - configure Snowflake storage integration with S3
  - build text-to-SQL and RAG applications
---

# Zomato AI Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates an end-to-end batch data pipeline that processes food delivery data (Zomato-style) through a medallion architecture: **S3 → Snowflake → dbt → Airflow → OpenAI**. It includes bronze/silver/gold layers, incremental fact tables, AI enrichment, RAG chat, and text-to-SQL capabilities.

## Architecture Overview

```
CSV Data → S3 Lake → Snowflake RAW (Bronze) → dbt STAGING (Silver) → dbt MARTS (Gold) → AI Layer
                                                                    ↑
                                                              Airflow Orchestration
```

**Key Components:**
- **Bronze (RAW)**: Direct `COPY INTO` from S3 via storage integration
- **Silver (STAGING)**: dbt views for cleaning and standardization
- **Gold (MARTS)**: Dimensions, incremental facts, business aggregates
- **AI Layer**: LLM enrichment, RAG, text-to-SQL powered by OpenAI

## Project Structure

```
airflow/
├── dags/zomato_batch.py        # Main orchestration DAG
├── docker-compose.yaml         # Airflow 3 setup
└── example.env                 # Environment template

zomato/                         # dbt project
├── models/
│   ├── staging/               # Silver layer (views)
│   └── marts/                 # Gold layer (tables)
├── dbt_project.yml
└── profiles.yml

ai/
├── enrich_reviews.py          # LLM sentiment/topic extraction
├── rag_chat.py                # Chat with reviews (Streamlit)
└── text_to_sql.py             # Natural language querying

aws/iam/                       # S3-Snowflake integration policies
```

## Installation & Setup

### 1. Prerequisites

```bash
# Python 3.10+
pip install dbt-snowflake apache-airflow openai streamlit pandas snowflake-connector-python

# Docker for Airflow
docker --version
docker-compose --version
```

### 2. Snowflake Setup

```sql
-- Create warehouse and database
CREATE WAREHOUSE ZOMATO_WH WITH WAREHOUSE_SIZE = 'MEDIUM';
CREATE DATABASE ZOMATO;

-- Create schemas
USE DATABASE ZOMATO;
CREATE SCHEMA RAW;
CREATE SCHEMA STAGING;
CREATE SCHEMA MARTS;
CREATE SCHEMA SNAPSHOTS;
CREATE SCHEMA AI;

-- Create role
CREATE ROLE DBT_ROLE;
GRANT USAGE ON WAREHOUSE ZOMATO_WH TO ROLE DBT_ROLE;
GRANT ALL ON DATABASE ZOMATO TO ROLE DBT_ROLE;
GRANT ALL ON ALL SCHEMAS IN DATABASE ZOMATO TO ROLE DBT_ROLE;
GRANT ROLE DBT_ROLE TO USER <your_user>;
```

### 3. S3 Storage Integration

Create IAM policy `zomato-s3-read` with read-only access to your bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:GetObjectVersion", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::your-bucket-name/*",
      "arn:aws:s3:::your-bucket-name"
    ]
  }]
}
```

Create IAM role `snowflake-s3-role`, attach policy, then create Snowflake integration:

```sql
CREATE STORAGE INTEGRATION zomato_s3_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'S3'
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake-s3-role'
  STORAGE_ALLOWED_LOCATIONS = ('s3://your-bucket-name/raw/');

-- Get the Snowflake IAM user and external ID
DESC STORAGE INTEGRATION zomato_s3_integration;
```

Update IAM role trust policy with `STORAGE_AWS_IAM_USER_ARN` and `STORAGE_AWS_EXTERNAL_ID` from above.

### 4. dbt Configuration

**`zomato/profiles.yml`:**
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
      schema: MARTS
      threads: 4
```

**Set environment variables:**
```bash
export SNOWFLAKE_ACCOUNT=xy12345.us-east-1
export SNOWFLAKE_USER=your_user
export SNOWFLAKE_PASSWORD=your_password
```

## dbt Models

### Staging Models (Silver)

**`models/staging/stg_orders.sql`:**
```sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'orders') }}
),

cleaned AS (
    SELECT
        order_id,
        customer_id,
        restaurant_id,
        TO_TIMESTAMP_NTZ(order_date) AS order_date,
        delivery_time_minutes,
        status,
        total_amount::NUMBER(10,2) AS total_amount,
        -- Derive business flags
        CASE 
            WHEN status = 'Delivered' THEN TRUE
            ELSE FALSE
        END AS is_delivered
    FROM source
)

SELECT * FROM cleaned
```

### Incremental Fact Models (Gold)

**`models/marts/fct_orders.sql`:**
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
),

enriched AS (
    SELECT
        order_id,
        customer_id,
        restaurant_id,
        order_date,
        DATE(order_date) AS order_date_key,
        HOUR(order_date) AS order_hour,
        delivery_time_minutes,
        status,
        total_amount,
        is_delivered,
        CURRENT_TIMESTAMP() AS dbt_loaded_at
    FROM orders
    {% if is_incremental() %}
    WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
    {% endif %}
)

SELECT * FROM enriched
```

**Key pattern**: `is_incremental()` macro ensures only new records are processed on subsequent runs.

### Business Marts

**`models/marts/mart_city_revenue_daily.sql`:**
```sql
WITH orders AS (
    SELECT * FROM {{ ref('fct_orders') }}
),

restaurants AS (
    SELECT * FROM {{ ref('dim_restaurants') }}
),

aggregated AS (
    SELECT
        r.city,
        DATE(o.order_date) AS order_date,
        COUNT(DISTINCT o.order_id) AS total_orders,
        SUM(o.total_amount) AS gmv,
        AVG(o.total_amount) AS aov,
        COUNT(DISTINCT CASE WHEN o.status = 'Cancelled' THEN o.order_id END) AS cancelled_orders,
        ROUND(cancelled_orders::FLOAT / NULLIF(total_orders, 0) * 100, 2) AS cancel_rate_pct
    FROM orders o
    JOIN restaurants r ON o.restaurant_id = r.restaurant_id
    GROUP BY 1, 2
)

SELECT * FROM aggregated
```

## Airflow Orchestration

### DAG Definition

**`airflow/dags/zomato_batch.py`:**
```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator

default_args = {
    'owner': 'data-eng',
    'depends_on_past': False,
    'email_on_failure': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

with DAG(
    'zomato_batch',
    default_args=default_args,
    description='Zomato end-to-end batch pipeline',
    schedule_interval='@daily',
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=['data-engineering', 'batch'],
) as dag:

    # Task 1: Reload raw tables from S3
    reload_raw = SnowflakeOperator(
        task_id='reload_raw',
        snowflake_conn_id='snowflake_default',
        sql="""
        USE DATABASE ZOMATO;
        USE SCHEMA RAW;
        
        COPY INTO orders FROM @zomato_stage/orders/
        FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1)
        ON_ERROR = 'CONTINUE';
        
        COPY INTO order_items FROM @zomato_stage/order_items/
        FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1)
        ON_ERROR = 'CONTINUE';
        
        COPY INTO reviews FROM @zomato_stage/reviews/
        FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1)
        ON_ERROR = 'CONTINUE';
        """
    )

    # Task 2: dbt build (staging + marts)
    dbt_build_core = BashOperator(
        task_id='dbt_build_core',
        bash_command='cd /opt/airflow/zomato && dbt build --exclude tag:ai',
        env={
            'SNOWFLAKE_ACCOUNT': '{{ var.value.SNOWFLAKE_ACCOUNT }}',
            'SNOWFLAKE_USER': '{{ var.value.SNOWFLAKE_USER }}',
            'SNOWFLAKE_PASSWORD': '{{ var.value.SNOWFLAKE_PASSWORD }}',
        }
    )

    # Task 3: AI enrichment
    def run_enrich_reviews():
        import sys
        sys.path.insert(0, '/opt/airflow/ai')
        from enrich_reviews import enrich_reviews
        enrich_reviews()

    enrich_reviews_task = PythonOperator(
        task_id='enrich_reviews',
        python_callable=run_enrich_reviews
    )

    # Task 4: Build AI marts
    dbt_build_ai = BashOperator(
        task_id='dbt_build_ai',
        bash_command='cd /opt/airflow/zomato && dbt build --select tag:ai',
        env={
            'SNOWFLAKE_ACCOUNT': '{{ var.value.SNOWFLAKE_ACCOUNT }}',
            'SNOWFLAKE_USER': '{{ var.value.SNOWFLAKE_USER }}',
            'SNOWFLAKE_PASSWORD': '{{ var.value.SNOWFLAKE_PASSWORD }}',
        }
    )

    reload_raw >> dbt_build_core >> enrich_reviews_task >> dbt_build_ai
```

### Airflow Setup with Docker

**`airflow/docker-compose.yaml`:**
```yaml
version: '3.8'
services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow

  airflow-init:
    build: .
    depends_on:
      - postgres
    environment:
      - AIRFLOW__CORE__SQL_ALCHEMY_CONN=postgresql+psycopg2://airflow:airflow@postgres/airflow
    command: >
      bash -c "airflow db migrate && 
               airflow users create --username admin --password admin --firstname Admin --lastname User --role Admin --email admin@example.com"

  airflow-webserver:
    build: .
    depends_on:
      - postgres
      - airflow-init
    ports:
      - "8080:8080"
    env_file:
      - .env
    command: airflow webserver

  airflow-scheduler:
    build: .
    depends_on:
      - postgres
      - airflow-init
    env_file:
      - .env
    command: airflow scheduler
```

**`airflow/.env`:**
```bash
AIRFLOW__CORE__SQL_ALCHEMY_CONN=postgresql+psycopg2://airflow:airflow@postgres/airflow
AIRFLOW__CORE__EXECUTOR=LocalExecutor
SNOWFLAKE_ACCOUNT=xy12345.us-east-1
SNOWFLAKE_USER=your_user
SNOWFLAKE_PASSWORD=your_password
OPENAI_API_KEY=sk-...
SAMPLE_N=1000
```

**Start Airflow:**
```bash
cd airflow
docker compose build
docker compose up -d
# Access at http://localhost:8080 (admin/admin)
```

## AI Layer

### 1. LLM Enrichment

**`ai/enrich_reviews.py`:**
```python
import os
import json
from openai import OpenAI
import snowflake.connector

client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))

def enrich_reviews():
    conn = snowflake.connector.connect(
        account=os.getenv('SNOWFLAKE_ACCOUNT'),
        user=os.getenv('SNOWFLAKE_USER'),
        password=os.getenv('SNOWFLAKE_PASSWORD'),
        warehouse='ZOMATO_WH',
        database='ZOMATO',
        schema='RAW'
    )
    
    cur = conn.cursor()
    
    # Get unenriched reviews
    sample_n = int(os.getenv('SAMPLE_N', 1000))
    cur.execute(f"""
        SELECT review_id, review_text 
        FROM ZOMATO.RAW.REVIEWS 
        WHERE review_id NOT IN (SELECT review_id FROM ZOMATO.AI.REVIEW_ENRICHED)
        LIMIT {sample_n}
    """)
    
    reviews = cur.fetchall()
    enriched_data = []
    
    for review_id, review_text in reviews:
        prompt = f"""Analyze this restaurant review and return JSON with:
        - sentiment: "positive", "negative", or "neutral"
        - topic: main topic (e.g., "food quality", "delivery time", "pricing")
        
        Review: {review_text}
        
        Return only valid JSON."""
        
        response = client.chat.completions.create(
            model='gpt-4o-mini',
            messages=[{'role': 'user', 'content': prompt}],
            temperature=0
        )
        
        result = json.loads(response.choices[0].message.content)
        enriched_data.append((
            review_id,
            result.get('sentiment'),
            result.get('topic')
        ))
    
    # Insert enriched data
    cur.execute("USE SCHEMA ZOMATO.AI")
    cur.executemany("""
        INSERT INTO REVIEW_ENRICHED (review_id, sentiment, topic)
        VALUES (%s, %s, %s)
    """, enriched_data)
    
    conn.commit()
    cur.close()
    conn.close()
    print(f"Enriched {len(enriched_data)} reviews")

if __name__ == '__main__':
    enrich_reviews()
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

def get_embedding(text):
    response = client.embeddings.create(
        model='text-embedding-3-small',
        input=text
    )
    return response.data[0].embedding

def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

def retrieve_reviews(question, top_k=5):
    conn = snowflake.connector.connect(
        account=os.getenv('SNOWFLAKE_ACCOUNT'),
        user=os.getenv('SNOWFLAKE_USER'),
        password=os.getenv('SNOWFLAKE_PASSWORD'),
        warehouse='ZOMATO_WH',
        database='ZOMATO'
    )
    
    cur = conn.cursor()
    cur.execute("""
        SELECT review_id, review_text 
        FROM ZOMATO.RAW.REVIEWS 
        LIMIT 1000
    """)
    reviews = cur.fetchall()
    cur.close()
    conn.close()
    
    # Embed question
    question_emb = get_embedding(question)
    
    # Compute similarity
    scored = []
    for review_id, review_text in reviews:
        review_emb = get_embedding(review_text)
        score = cosine_similarity(question_emb, review_emb)
        scored.append((score, review_id, review_text))
    
    scored.sort(reverse=True)
    return scored[:top_k]

def generate_answer(question, context_reviews):
    context = "\n\n".join([f"Review {i+1}: {rev[2]}" for i, rev in enumerate(context_reviews)])
    
    prompt = f"""Based on these restaurant reviews, answer the question.
    
Context:
{context}

Question: {question}

Provide a helpful answer grounded in the reviews above."""
    
    response = client.chat.completions.create(
        model='gpt-4o-mini',
        messages=[{'role': 'user', 'content': prompt}]
    )
    
    return response.choices[0].message.content

# Streamlit UI
st.title("🍔 Chat with Zomato Reviews")
question = st.text_input("Ask a question about customer reviews:")

if st.button("Search") and question:
    with st.spinner("Retrieving relevant reviews..."):
        top_reviews = retrieve_reviews(question)
    
    with st.spinner("Generating answer..."):
        answer = generate_answer(question, top_reviews)
    
    st.write("### Answer")
    st.write(answer)
    
    st.write("### Source Reviews")
    for i, (score, review_id, review_text) in enumerate(top_reviews):
        st.write(f"**Review {i+1}** (similarity: {score:.3f})")
        st.write(review_text[:200] + "...")
```

**Run:**
```bash
export SNOWFLAKE_ACCOUNT=... SNOWFLAKE_USER=... SNOWFLAKE_PASSWORD=... OPENAI_API_KEY=...
streamlit run ai/rag_chat.py
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

SCHEMA_CONTEXT = """
Available tables in ZOMATO.MARTS:

1. fct_orders: order_id, customer_id, restaurant_id, order_date, total_amount, status, is_delivered
2. dim_restaurants: restaurant_id, restaurant_name, city, cuisine_type, rating
3. dim_customer: customer_id, customer_name, email, age, age_segment
4. mart_city_revenue_daily: city, order_date, total_orders, gmv, aov, cancel_rate_pct
5. mart_restaurant_performance: restaurant_id, restaurant_name, total_orders, total_revenue, avg_rating
"""

def generate_sql(question):
    prompt = f"""You are a Snowflake SQL expert. Given this schema:

{SCHEMA_CONTEXT}

Generate a SELECT query for: {question}

Rules:
- Use only the tables/columns listed above
- Always qualify columns with table aliases
- Return only the SQL query, no explanation
- Use standard Snowflake SQL syntax"""

    response = client.chat.completions.create(
        model='gpt-4o-mini',
        messages=[{'role': 'user', 'content': prompt}],
        temperature=0
    )
    
    return response.choices[0].message.content.strip()

def validate_sql(sql):
    """Basic safety check - only allow SELECT"""
    sql_upper = sql.upper()
    forbidden = ['DROP', 'DELETE', 'UPDATE', 'INSERT', 'CREATE', 'ALTER', 'TRUNCATE']
    
    for keyword in forbidden:
        if keyword in sql_upper:
            return False, f"Forbidden keyword: {keyword}"
    
    if not sql_upper.strip().startswith('SELECT'):
        return False, "Only SELECT queries allowed"
    
    return True, "OK"

def execute_query(sql):
    conn = snowflake.connector.connect(
        account=os.getenv('SNOWFLAKE_ACCOUNT'),
        user=os.getenv('SNOWFLAKE_USER'),
        password=os.getenv('SNOWFLAKE_PASSWORD'),
        warehouse='ZOMATO_WH',
        database='ZOMATO',
        schema='MARTS',
        role='DBT_ROLE'
    )
    
    cur = conn.cursor()
    cur.execute(sql)
    results = cur.fetchall()
    columns = [desc[0] for desc in cur.description]
    
    cur.close()
    conn.close()
    
    return columns, results

# Streamlit UI
st.title("🗣️ Chat with Zomato Warehouse")
st.write("Ask questions in plain English, get SQL + results")

question = st.text_area("Your question:", height=100)

if st.button("Generate & Execute") and question:
    with st.spinner("Generating SQL..."):
        sql = generate_sql(question)
    
    st.write("### Generated SQL")
    st.code(sql, language='sql')
    
    # Validate
    is_safe, msg = validate_sql(sql)
    if not is_safe:
        st.error(f"❌ Query rejected: {msg}")
    else:
        with st.spinner("Executing query..."):
            try:
                columns, results = execute_query(sql)
                st.write("### Results")
                st.dataframe(results, column_config={i: col for i, col in enumerate(columns)})
            except Exception as e:
                st.error(f"Query failed: {e}")
```

**Run:**
```bash
streamlit run ai/text_to_sql.py
```

## Key Commands

### dbt
```bash
# Test connection
dbt debug

# Run specific models
dbt run --select staging.*
dbt run --select fct_orders
dbt run --select tag:daily

# Run with full-refresh (ignore incremental logic)
dbt run --select fct_orders --full-refresh

# Build models + run tests
dbt build

# Test only
dbt test

# Generate documentation
dbt docs generate
dbt docs serve
```

### Airflow
```bash
# Initialize database
airflow db migrate

# Create admin user
airflow users create --username admin --password admin --firstname Admin --lastname User --role Admin --email admin@example.com

# Test DAG
airflow dags test zomato_batch 2024-01-01

# List DAGs
airflow dags list

# Trigger run
airflow dags trigger zomato_batch
```

## Common Patterns

### Incremental Models with MERGE

For large fact tables, use incremental materialization to avoid full rebuilds:

```sql
{{
    config(
        materialized='incremental',
        unique_key='order_id',
        on_schema_change='append_new_columns',
        merge_update_columns=['status', 'total_amount']  -- only update these
    )
}}

SELECT * FROM {{ ref('stg_orders') }}
{% if is_incremental() %}
WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

### Custom Schema Naming

**`macros/generate_schema_name.sql`:**
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

This ensures `models/marts/core/fct_orders.sql` with `{{ config(schema='marts') }}` lands in `ZOMATO.MARTS.FCT_ORDERS`, not `ZOMATO.MARTS_MARTS.FCT_ORDERS`.

### Idempotent AI Enrichment

To avoid re-enriching the same reviews:

```python
# Only process reviews not already in AI table
cur.execute("""
    SELECT review_id, review_text 
    FROM ZOMATO.RAW.REVIEWS 
    WHERE review_id NOT IN (SELECT review_id FROM ZOMATO.AI.REVIEW_ENRICHED)
    LIMIT 1000
""")
```

### Environment-Based Configuration

**dbt profiles:**
```yaml
outputs:
  dev:
    schema: DEV_MARTS
  prod:
    schema: MARTS
```

**Airflow Variables:**
Store credentials in Airflow UI under Admin → Variables or use environment variables.

## Troubleshooting

### `COPY INTO` fails with access denied
- Verify IAM role trust policy includes Snowflake's `STORAGE_AWS_IAM_USER_ARN` (not `:root`)
- Check `STORAGE_AWS_EXTERNAL_ID` matches current integration (`DESC STORAGE INTEGRATION`)
- Never run `CREATE OR REPLACE STORAGE INTEGRATION` — it regenerates external ID

### dbt incremental models rebuild fully every run
- Ensure `unique_key` is specified
- Check `target/run/zomato/models/.../model.sql` to see compiled SQL
- Verify `is_incremental()` condition resolves correctly

### OpenAI rate limits
- Add exponential backoff in enrichment loop
- Use `SAMPLE_N` env var to limit batch size
- Consider batching with `client.embeddings.create(input=[list])`

### Airflow tasks fail with connection error
- Verify `AIRFLOW_CONN_SNOWFLAKE_DEFAULT` or create connection in UI
- Check environment variables are passed to containers via `.env`
- Test connection: `airflow connections test snowflake_default`

### dbt tests fail on relationships
- Ensure dimension tables load before fact tables
- Add `ref()` dependencies in staging models
- Use `dbt build` (not `dbt run`) to respect test dependencies

### Text-to-SQL generates invalid queries
- Improve schema context with example queries
- Add few-shot examples to prompt
- Implement query result validation before execution

## Testing

```bash
# dbt generic tests (defined in schema.yml)
dbt test --select fct_orders

# dbt singular tests (custom SQL in tests/)
dbt test --select test_order_revenue_reconciliation

# Full build with tests
dbt build --fail-fast

# Test Airflow DAG
airflow dags test zomato_batch 2024-01-01
```

## Performance Tips

1. **Incremental models**: Always use for tables >1M rows
2. **Clustering keys**: `ALTER TABLE fct_orders CLUSTER BY (order_date);`
3. **Warehouse sizing**: Start with MEDIUM, scale up for heavy transformations
4. **dbt threads**: Increase `threads: 8` in `profiles.yml` for parallel execution
5. **COPY INTO parallelism**: Split large files into chunks in S3
6. **OpenAI batch API**: For >1K reviews, use batch API (cheaper, async)

## References

- dbt incremental models: https://docs.getdbt.com/docs/build/incremental-models
- Snowflake storage integration: https://docs.snowflake.com/en/user-guide/data-load-s3-config-storage-integration
- Airflow Snowflake provider: https://airflow.apache.org/docs/apache-airflow-providers-snowflake/
- OpenAI embeddings: https://platform.openai.com/docs/guides/embeddings
