---
name: zomato-ai-data-engineering-pipeline
description: End-to-end batch data pipeline with dbt, Snowflake, Airflow, and OpenAI for food delivery analytics
triggers:
  - set up zomato data pipeline
  - build food delivery data warehouse
  - configure dbt snowflake medallion architecture
  - integrate openai with snowflake
  - orchestrate batch pipeline with airflow
  - create rag chatbot for reviews
  - implement text to sql with llm
  - enrich data with openai
---

# Zomato AI Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project implements a complete batch data pipeline for food delivery analytics: **CSV → S3 → Snowflake → dbt (medallion) → Airflow → OpenAI (LLM enrichment, RAG, text-to-SQL)**. It demonstrates modern data engineering patterns including incremental loads, SCD2 snapshots, LLM-powered transformations, and AI-driven analytics.

## What It Does

- **Data Lake**: Uploads 7 CSV files (10M+ orders, 23M+ order items, 300K reviews) to S3
- **Bronze Layer**: Loads raw data into Snowflake via `COPY INTO` with keyless S3 storage integration
- **Silver Layer**: dbt staging views clean and standardize all sources
- **Gold Layer**: dbt marts create dimensional model with incremental facts, business aggregates, and SCD2 snapshots
- **AI Layer**: OpenAI enriches reviews (sentiment/topics), RAG for conversational search, text-to-SQL for natural language queries
- **Orchestration**: Airflow 3 DAG runs the full pipeline daily

## Architecture

```
CSVs → S3 → Snowflake RAW → dbt STAGING → dbt MARTS → AI enrichment
                ↑                                          ↓
            Storage Integration                    OpenAI (gpt-4o-mini)
                                                          ↓
                                            Streamlit (RAG + text-to-SQL)
```

## Prerequisites

- **AWS**: S3 bucket, IAM role with trust relationship to Snowflake
- **Snowflake**: Account with ACCOUNTADMIN access
- **OpenAI**: API key for GPT-4o-mini and embeddings
- **Docker**: For Airflow
- **Python 3.9+**: For dbt and AI scripts

## Installation

### 1. Clone and Get Data

```bash
git clone https://github.com/darshilparmar/zomato-ai-data-engineering-end-to-end-project.git
cd zomato-ai-data-engineering-end-to-end-project
```

Download CSVs from [Google Drive](https://drive.google.com/drive/folders/1FEnGWMHhHzzTUCZOw1-YnH2v3DMuM-rs) and place in `data/` directory.

### 2. AWS Setup (S3 + IAM)

Create S3 bucket and upload data:

```bash
aws s3 mb s3://your-zomato-bucket
aws s3 sync data/ s3://your-zomato-bucket/raw/ --recursive
```

Create IAM policy `zomato-s3-read` using `aws/iam/s3-read-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::your-zomato-bucket",
        "arn:aws:s3:::your-zomato-bucket/*"
      ]
    }
  ]
}
```

Create IAM role `snowflake-s3-role` with initial trust policy (`aws/iam/snowflake-role-trust-policy-initial.json`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Attach `zomato-s3-read` policy to the role.

### 3. Snowflake Setup

Create warehouse, database, schemas, and storage integration in Snowsight:

```sql
-- Warehouse
CREATE WAREHOUSE ZOMATO_WH 
  WAREHOUSE_SIZE = 'MEDIUM' 
  AUTO_SUSPEND = 60 
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
GRANT ROLE DBT_ROLE TO USER YOUR_USER;

-- Storage integration (replace with your IAM role ARN)
CREATE STORAGE INTEGRATION s3_zomato_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = S3
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake-s3-role'
  STORAGE_ALLOWED_LOCATIONS = ('s3://your-zomato-bucket/raw/');

-- Get Snowflake IAM user and external ID
DESC STORAGE INTEGRATION s3_zomato_integration;
-- Copy STORAGE_AWS_IAM_USER_ARN and STORAGE_AWS_EXTERNAL_ID
```

Update IAM role trust policy with Snowflake's IAM user ARN and external ID (`aws/iam/snowflake-role-trust-policy-final.json`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::SNOWFLAKE_ACCOUNT:user/SNOWFLAKE_USER"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "EXTERNAL_ID_FROM_DESC_INTEGRATION"
        }
      }
    }
  ]
}
```

Create external stage and RAW tables:

```sql
USE SCHEMA ZOMATO.RAW;

-- External stage
CREATE STAGE s3_stage
  STORAGE_INTEGRATION = s3_zomato_integration
  URL = 's3://your-zomato-bucket/raw/';

-- RAW tables (example: orders)
CREATE TABLE orders (
  order_id INT,
  customer_id INT,
  restaurant_id INT,
  order_date TIMESTAMP,
  delivery_date TIMESTAMP,
  order_status VARCHAR,
  order_value DECIMAL(10,2)
);

-- Repeat for: restaurants, users, food, menu, order_items, reviews
```

### 4. dbt Setup

```bash
cd zomato

# Install dbt-snowflake
pip install dbt-snowflake

# Configure profiles.yml (already set to use env vars)
export SNOWFLAKE_ACCOUNT=your_account
export SNOWFLAKE_USER=your_user
export SNOWFLAKE_PASSWORD=your_password

# Test connection
dbt debug

# Build all models except AI layer
dbt build --exclude tag:ai
```

### 5. Airflow Setup

```bash
cd airflow

# Create .env from template
cp example.env .env
# Edit .env with your credentials:
# SNOWFLAKE_ACCOUNT=...
# SNOWFLAKE_USER=...
# SNOWFLAKE_PASSWORD=...
# OPENAI_API_KEY=sk-...
# SAMPLE_N=100  # Limit reviews for LLM enrichment

# Build and start
docker compose build
docker compose up -d

# Access UI at http://localhost:8080 (admin/admin)
```

### 6. AI Layer Setup

```bash
cd ai

# Install dependencies
pip install openai pandas snowflake-connector-python streamlit

# Configure environment
export OPENAI_API_KEY=sk-...
export SNOWFLAKE_ACCOUNT=your_account
export SNOWFLAKE_USER=your_user
export SNOWFLAKE_PASSWORD=your_password
export SAMPLE_N=100
```

## Key Commands

### dbt Commands

```bash
# Test connection
dbt debug

# Compile SQL without running
dbt compile

# Run all models
dbt run

# Run tests only
dbt test

# Build models and run tests in dependency order
dbt build

# Run specific models
dbt run --select staging.stg_orders
dbt run --select marts.fct_orders+  # model and downstream
dbt run --select +dim_customer  # model and upstream

# Run incremental models with full refresh
dbt run --select fct_orders --full-refresh

# Run models by tag
dbt build --exclude tag:ai
dbt run --select tag:daily

# Generate and serve documentation
dbt docs generate
dbt docs serve
```

### Airflow Commands

```bash
# Start services
docker compose up -d

# Stop services
docker compose down

# View logs
docker compose logs -f airflow-scheduler

# Access Airflow CLI
docker compose exec airflow-scheduler airflow dags list
docker compose exec airflow-scheduler airflow dags trigger zomato_batch

# Restart after code changes
docker compose restart
```

### Snowflake Commands

```sql
-- Check row counts
SELECT 'orders' AS table_name, COUNT(*) FROM ZOMATO.RAW.orders
UNION ALL
SELECT 'order_items', COUNT(*) FROM ZOMATO.RAW.order_items
UNION ALL
SELECT 'reviews', COUNT(*) FROM ZOMATO.RAW.reviews;

-- Manually trigger COPY INTO (for testing)
COPY INTO ZOMATO.RAW.orders
FROM @ZOMATO.RAW.s3_stage/orders/
FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"');

-- Check incremental load metadata
SELECT * FROM ZOMATO.MARTS.fct_orders 
WHERE _loaded_at > CURRENT_TIMESTAMP - INTERVAL '1 day';

-- Query business marts
SELECT * FROM ZOMATO.MARTS.mart_daily_city_gmv
WHERE order_date >= CURRENT_DATE - 7
ORDER BY gmv DESC;
```

## Configuration

### dbt profiles.yml

Located at `~/.dbt/profiles.yml` (or the project uses env vars):

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

### dbt_project.yml

```yaml
name: 'zomato'
version: '1.0.0'
config-version: 2

profile: 'zomato'

model-paths: ["models"]
macro-paths: ["macros"]
test-paths: ["tests"]

models:
  zomato:
    staging:
      +materialized: view
      +schema: staging
    marts:
      +materialized: table
      +schema: marts
      fct_orders:
        +materialized: incremental
        +unique_key: order_id
      fct_order_items:
        +materialized: incremental
        +unique_key: ['order_id', 'food_id']
```

### Airflow DAG Configuration

From `airflow/dags/zomato_batch.py`:

```python
from airflow.decorators import dag, task
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from airflow.operators.bash import BashOperator
from datetime import datetime

@dag(
    dag_id='zomato_batch',
    start_date=datetime(2024, 1, 1),
    schedule='@daily',
    catchup=False,
    tags=['zomato', 'batch']
)
def zomato_pipeline():
    
    # Task 1: Load raw data from S3
    reload_raw = SnowflakeOperator(
        task_id='reload_raw',
        snowflake_conn_id='snowflake_default',
        sql="""
        TRUNCATE TABLE ZOMATO.RAW.orders;
        COPY INTO ZOMATO.RAW.orders
        FROM @ZOMATO.RAW.s3_stage/orders/
        FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1);
        
        -- Repeat for all tables
        """
    )
    
    # Task 2: Run dbt build (core models + tests)
    dbt_build_core = BashOperator(
        task_id='dbt_build_core',
        bash_command='cd /opt/airflow/dbt/zomato && dbt build --exclude tag:ai'
    )
    
    # Task 3: Enrich reviews with OpenAI
    @task
    def enrich_reviews():
        import subprocess
        subprocess.run([
            'python', '/opt/airflow/ai/enrich_reviews.py'
        ], check=True)
    
    # Task 4: Build AI marts
    dbt_build_ai = BashOperator(
        task_id='dbt_build_ai',
        bash_command='cd /opt/airflow/dbt/zomato && dbt run --select tag:ai'
    )
    
    reload_raw >> dbt_build_core >> enrich_reviews() >> dbt_build_ai

zomato_pipeline()
```

## Code Examples

### dbt Staging Model (Silver)

`zomato/models/staging/stg_restaurants.sql`:

```sql
with source as (
    select * from {{ source('zomato_raw', 'restaurants') }}
),

cleaned as (
    select
        restaurant_id,
        name,
        lower(trim(city)) as city,
        lower(trim(locality)) as locality,
        
        -- Parse currency: "₹ 200" → 200, "--" → null
        case
            when trim(average_cost_for_two) = '--' then null
            else cast(regexp_replace(average_cost_for_two, '[^0-9]', '') as int)
        end as average_cost_for_two,
        
        -- Clean rating
        case
            when rating = '--' then null
            else cast(rating as decimal(2,1))
        end as rating,
        
        lower(trim(cuisines)) as cuisines,
        has_online_delivery::boolean as has_online_delivery,
        has_table_booking::boolean as has_table_booking
        
    from source
)

select * from cleaned
```

### dbt Incremental Fact Model (Gold)

`zomato/models/marts/fct_orders.sql`:

```sql
{{
    config(
        materialized='incremental',
        unique_key='order_id',
        incremental_strategy='merge'
    )
}}

with orders as (
    select * from {{ ref('stg_orders') }}
    {% if is_incremental() %}
    where order_date > (select max(order_date) from {{ this }})
    {% endif %}
),

enriched as (
    select
        o.order_id,
        o.customer_id,
        o.restaurant_id,
        o.order_date,
        o.delivery_date,
        o.order_status,
        o.order_value,
        
        -- Derived fields
        case 
            when o.order_status = 'delivered' then true 
            else false 
        end as is_delivered,
        
        datediff('minute', o.order_date, o.delivery_date) as delivery_time_minutes,
        
        current_timestamp() as _loaded_at
        
    from orders o
)

select * from enriched
```

### dbt Business Mart (Gold)

`zomato/models/marts/mart_daily_city_gmv.sql`:

```sql
with orders as (
    select * from {{ ref('fct_orders') }}
),

daily_metrics as (
    select
        date_trunc('day', order_date) as order_date,
        city,
        
        -- Revenue metrics
        sum(order_value) as gmv,
        avg(order_value) as aov,
        count(distinct order_id) as total_orders,
        count(distinct customer_id) as unique_customers,
        
        -- Operational metrics
        sum(case when is_delivered then 1 else 0 end) as delivered_orders,
        sum(case when order_status = 'cancelled' then 1 else 0 end) as cancelled_orders,
        
        -- Rates
        (cancelled_orders::float / nullif(total_orders, 0)) * 100 as cancel_rate_pct,
        avg(delivery_time_minutes) as avg_delivery_time_min
        
    from orders
    group by 1, 2
)

select * from daily_metrics
order by order_date desc, gmv desc
```

### AI: LLM Enrichment Script

`ai/enrich_reviews.py`:

```python
import os
import pandas as pd
import snowflake.connector
from openai import OpenAI
import json

# Configuration
SAMPLE_N = int(os.getenv('SAMPLE_N', 100))
client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))

# Snowflake connection
conn = snowflake.connector.connect(
    account=os.getenv('SNOWFLAKE_ACCOUNT'),
    user=os.getenv('SNOWFLAKE_USER'),
    password=os.getenv('SNOWFLAKE_PASSWORD'),
    warehouse='ZOMATO_WH',
    database='ZOMATO',
    schema='RAW'
)

def enrich_review(review_text):
    """Call OpenAI to extract sentiment and topic from review."""
    prompt = f"""Analyze this food delivery review and return JSON with:
- sentiment: positive, negative, or neutral
- topic: food_quality, delivery_speed, packaging, price, or other

Review: {review_text}

Return ONLY valid JSON."""
    
    response = client.chat.completions.create(
        model='gpt-4o-mini',
        messages=[
            {'role': 'system', 'content': 'You are a review analyzer. Return only JSON.'},
            {'role': 'user', 'content': prompt}
        ],
        temperature=0
    )
    
    return json.loads(response.choices[0].message.content)

# Fetch unenriched reviews
query = f"""
SELECT review_id, review_text
FROM ZOMATO.RAW.reviews
WHERE review_id NOT IN (SELECT review_id FROM ZOMATO.AI.review_enriched)
LIMIT {SAMPLE_N}
"""

df = pd.read_sql(query, conn)
print(f"Enriching {len(df)} reviews...")

# Enrich in batches
results = []
for idx, row in df.iterrows():
    try:
        enrichment = enrich_review(row['review_text'])
        results.append({
            'review_id': row['review_id'],
            'sentiment': enrichment['sentiment'],
            'topic': enrichment['topic']
        })
        if (idx + 1) % 10 == 0:
            print(f"  Processed {idx + 1}/{len(df)}")
    except Exception as e:
        print(f"  Error on review {row['review_id']}: {e}")

# Write to Snowflake
enriched_df = pd.DataFrame(results)
from snowflake.connector.pandas_tools import write_pandas

cur = conn.cursor()
cur.execute("CREATE TABLE IF NOT EXISTS ZOMATO.AI.review_enriched (review_id INT, sentiment VARCHAR, topic VARCHAR)")

write_pandas(conn, enriched_df, 'review_enriched', database='ZOMATO', schema='AI')
print(f"✓ Wrote {len(enriched_df)} enriched reviews to ZOMATO.AI.review_enriched")

conn.close()
```

### AI: RAG Chat with Reviews

`ai/rag_chat.py`:

```python
import streamlit as st
import snowflake.connector
from openai import OpenAI
import numpy as np

client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))

# Snowflake connection
@st.cache_resource
def get_snowflake_conn():
    return snowflake.connector.connect(
        account=os.getenv('SNOWFLAKE_ACCOUNT'),
        user=os.getenv('SNOWFLAKE_USER'),
        password=os.getenv('SNOWFLAKE_PASSWORD'),
        warehouse='ZOMATO_WH',
        database='ZOMATO'
    )

def embed_text(text):
    """Generate embedding using OpenAI."""
    response = client.embeddings.create(
        model='text-embedding-3-small',
        input=text
    )
    return response.data[0].embedding

def retrieve_similar_reviews(question, top_k=5):
    """Vector similarity search in Snowflake."""
    conn = get_snowflake_conn()
    question_embedding = embed_text(question)
    
    # Assumes you have pre-embedded reviews in ZOMATO.AI.review_embeddings
    # with columns: review_id, review_text, embedding (ARRAY)
    query = f"""
    SELECT review_text,
           VECTOR_COSINE_SIMILARITY(embedding, {question_embedding}) AS similarity
    FROM ZOMATO.AI.review_embeddings
    ORDER BY similarity DESC
    LIMIT {top_k}
    """
    
    cur = conn.cursor()
    cur.execute(query)
    return cur.fetchall()

def generate_answer(question, context_reviews):
    """Generate answer using retrieved reviews as context."""
    context = "\n\n".join([f"Review: {r[0]}" for r in context_reviews])
    
    response = client.chat.completions.create(
        model='gpt-4o-mini',
        messages=[
            {'role': 'system', 'content': 'Answer questions based on the provided reviews.'},
            {'role': 'user', 'content': f"Context:\n{context}\n\nQuestion: {question}"}
        ]
    )
    
    return response.choices[0].message.content

# Streamlit UI
st.title("🍕 Chat with Zomato Reviews")
question = st.text_input("Ask a question about customer reviews:")

if question:
    with st.spinner("Searching reviews..."):
        similar = retrieve_similar_reviews(question)
    
    with st.spinner("Generating answer..."):
        answer = generate_answer(question, similar)
    
    st.write("### Answer")
    st.write(answer)
    
    with st.expander("📄 Source Reviews"):
        for idx, (review, score) in enumerate(similar, 1):
            st.write(f"**{idx}.** (similarity: {score:.3f})")
            st.write(review)
```

### AI: Text-to-SQL

`ai/text_to_sql.py`:

```python
import streamlit as st
import snowflake.connector
from openai import OpenAI
import os

client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))

# Schema context for LLM
SCHEMA_CONTEXT = """
Available tables in ZOMATO.MARTS:

1. mart_daily_city_gmv (order_date, city, gmv, aov, total_orders, cancel_rate_pct)
2. mart_restaurant_performance (restaurant_id, name, city, total_orders, avg_rating, revenue)
3. fct_orders (order_id, customer_id, restaurant_id, order_date, order_value, is_delivered)
4. dim_customer (customer_id, name, email, age, age_segment)
5. dim_restaurants (restaurant_id, name, city, cuisines, rating)

Write Snowflake SQL queries. Use only SELECT statements.
"""

def generate_sql(question):
    """Generate SQL from natural language question."""
    response = client.chat.completions.create(
        model='gpt-4o-mini',
        messages=[
            {'role': 'system', 'content': f'You are a Snowflake SQL expert.\n\n{SCHEMA_CONTEXT}'},
            {'role': 'user', 'content': f'Write SQL for: {question}\n\nReturn ONLY the SQL query.'}
        ],
        temperature=0
    )
    
    return response.choices[0].message.content.strip()

def is_safe_sql(sql):
    """Basic validation - only allow SELECT."""
    forbidden = ['DROP', 'DELETE', 'UPDATE', 'INSERT', 'CREATE', 'ALTER', 'TRUNCATE']
    return not any(kw in sql.upper() for kw in forbidden)

def execute_sql(sql):
    """Run SQL as DBT_ROLE (read-only)."""
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
    return cur.fetchall(), [desc[0] for desc in cur.description]

# Streamlit UI
st.title("💬 Chat with Zomato Data Warehouse")
st.caption("Ask questions in plain English — AI will write SQL for you")

question = st.text_area("Your question:", placeholder="What are the top 5 cities by revenue last week?")

if st.button("Run Query"):
    with st.spinner("Generating SQL..."):
        sql = generate_sql(question)
    
    st.code(sql, language='sql')
    
    if not is_safe_sql(sql):
        st.error("⚠️ Query contains forbidden operations. Only SELECT is allowed.")
    else:
        with st.spinner("Executing..."):
            try:
                rows, columns = execute_sql(sql)
                st.success(f"✓ Returned {len(rows)} rows")
                
                import pandas as pd
                df = pd.DataFrame(rows, columns=columns)
                st.dataframe(df)
            except Exception as e:
                st.error(f"Query failed: {e}")
```

## Common Patterns

### 1. Incremental Loading Pattern

For large fact tables, use incremental materialization:

```sql
{{
    config(
        materialized='incremental',
        unique_key='order_id',
        incremental_strategy='merge',
        on_schema_change='append_new_columns'
    )
}}

select * from {{ ref('stg_orders') }}

{% if is_incremental() %}
where order_date > (select coalesce(max(order_date), '1900-01-01') from {{ this }})
{% endif %}
```

### 2. Idempotent AI Enrichment

Always check what's already enriched to avoid re-processing:

```python
query = f"""
SELECT review_id, review_text
FROM raw.reviews
WHERE review_id NOT IN (
    SELECT review_id FROM ai.review_enriched
)
LIMIT {SAMPLE_N}
"""
```

### 3. Custom dbt Schema Naming

Override default schema behavior in `macros/generate_schema_name.sql`:

```sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- if custom_schema_name is none -%}
        {{ target.schema }}
    {%- else -%}
        {{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

This makes `+schema: marts` create `ZOMATO.marts` instead of `ZOMATO_DEV_marts`.

### 4. Airflow with dbt in Docker

Install dbt in a virtual environment inside the Airflow container:

```dockerfile
# airflow/Dockerfile
FROM apache/airflow:3.0.0-python3.9

USER root
RUN apt-get update && apt-get install -y git

USER airflow
RUN python -m venv /opt/airflow/dbt_venv && \
    /opt/airflow/dbt_venv/bin/pip install dbt-snowflake

# Use venv in BashOperator:
# bash_command='/opt/airflow/dbt_venv/bin/dbt build'
```

### 5. SCD Type 2 Snapshots

Track dimension history with dbt snapshots:

```sql
-- snapshots/restaurant_snapshot.sql
{% snapshot restaurant_snapshot %}

{{
    config(
      target_schema='snapshots',
      unique_key='restaurant_id',
      strategy='check',
      check_cols=['rating', 'average_cost_for_two']
    )
}}

select * from {{ ref('dim_restaurants') }}

{% endsnapshot %}
```

Run with `dbt snapshot`.

## Troubleshooting

### Issue: Storage integration shows `STORAGE_AWS_IAM_USER_ARN` but S3 access fails

**Cause**: IAM role trust policy not updated with Snowflake's IAM user ARN and external ID.

**Solution**: 
1. Run `DESC STORAGE INTEGRATION s3_zomato_integration;`
2. Copy `STORAGE_AWS_IAM_USER_ARN` and `STORAGE_AWS_EXTERNAL_ID`
3. Update IAM role trust policy (`aws/iam/snowflake-role-trust-policy-final.json`)
4. **Do not** re-run `CREATE OR REPLACE STORAGE INTEGRATION` after this — it regenerates the external ID

### Issue: dbt incremental models rebuild every run

**Cause**: Missing or incorrect `unique_key` configuration.

**Solution**: Ensure unique_key matches the table's primary key:

```sql
{{
    config(
        materialized='incremental',
        unique_key='order_id',  -- Must be unique
        incremental_strategy='merge'
    )
}}
```

For composite keys:

```sql
unique_key=['order_id', 'food_id']
```

### Issue: OpenAI enrichment hits rate limits

**Cause**: Processing too many reviews at once.

**Solution**: 
1. Reduce `SAMPLE_N` in `.env`
2. Add retry logic with exponential backoff:

```python
from tenacity import retry, wait_exponential, stop_after_attempt

@retry(wait=wait_exponential(min=1, max=60), stop=stop_after_attempt(5))
def enrich_review(review_text):
    return client.chat.completions.create(...)
```

3. Batch process in smaller chunks:
