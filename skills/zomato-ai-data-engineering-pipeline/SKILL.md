---
name: zomato-ai-data-engineering-pipeline
description: End-to-end batch data pipeline from S3 to Snowflake with dbt transformations, Airflow orchestration, and OpenAI-powered AI analytics (LLM enrichment, RAG, text-to-SQL)
triggers:
  - build a food delivery data pipeline with Snowflake and dbt
  - set up Zomato data engineering pipeline with AI features
  - create end-to-end data pipeline with S3, Snowflake, dbt, and Airflow
  - implement LLM enrichment for reviews in Snowflake
  - build RAG chat with Snowflake data
  - create text-to-SQL with OpenAI and Snowflake
  - orchestrate dbt with Airflow for batch pipeline
  - set up medallion architecture with dbt incremental models
---

# Zomato AI Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What It Does

This project implements a production-grade batch data pipeline for food delivery analytics:

- **Data Lake**: Raw CSVs → Amazon S3 (organized by table in `raw/<table>/` folders)
- **Warehouse**: S3 → Snowflake via keyless storage integration (no stored credentials)
- **Transformations**: dbt medallion architecture (Bronze → Silver → Gold)
  - **Bronze (RAW)**: `COPY INTO` from S3
  - **Silver (STAGING)**: Cleaning views
  - **Gold (MARTS)**: Dimensions, incremental facts (MERGE), business aggregates
- **Orchestration**: Apache Airflow daily DAG (load → transform → AI enrich)
- **AI Layer**: OpenAI-powered features
  - LLM enrichment (structured sentiment/topic extraction from reviews)
  - RAG chat (ask questions about reviews)
  - Text-to-SQL (query warehouse in plain English)

**Dataset**: 10M orders, ~23M order items, 300K reviews across restaurants, users, food, menu dimensions.

## Installation & Setup

### Prerequisites

```bash
# System requirements
- Python 3.8+
- Docker & Docker Compose (for Airflow)
- AWS account (S3 bucket)
- Snowflake account
- OpenAI API key
```

### Clone & Get Dataset

```bash
git clone https://github.com/darshilparmar/zomato-ai-data-engineering-end-to-end-project.git
cd zomato-ai-data-engineering-end-to-end-project

# Download CSVs from Google Drive (link in README) → place in data/ folder
# Expected structure:
# data/
#   restaurants.csv
#   users.csv
#   food.csv
#   menu.csv
#   orders.csv
#   order_items.csv
#   reviews.csv
```

### AWS Setup (S3 + IAM)

```bash
# 1. Create S3 bucket
aws s3 mb s3://your-zomato-bucket

# 2. Upload data
aws s3 sync data/ s3://your-zomato-bucket/raw/ --exclude "*" \
  --include "restaurants.csv" --include "users.csv" --include "food.csv" \
  --include "menu.csv" --include "orders.csv" --include "order_items.csv" \
  --include "reviews.csv"

# Folder structure in S3:
# s3://your-zomato-bucket/raw/restaurants/restaurants.csv
# s3://your-zomato-bucket/raw/users/users.csv
# (etc.)

# 3. Create IAM policy for S3 read access
# Use aws/iam/s3-read-policy.json, replace YOUR_BUCKET_NAME
aws iam create-policy --policy-name zomato-s3-read \
  --policy-document file://aws/iam/s3-read-policy.json

# 4. Create IAM role with initial trust policy
aws iam create-role --role-name snowflake-s3-role \
  --assume-role-policy-document file://aws/iam/snowflake-role-trust-policy-initial.json

# 5. Attach policy to role
aws iam attach-role-policy --role-name snowflake-s3-role \
  --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/zomato-s3-read

# 6. Get role ARN (needed for Snowflake)
aws iam get-role --role-name snowflake-s3-role --query 'Role.Arn' --output text
```

### Snowflake Setup

```sql
-- Run in Snowsight as ACCOUNTADMIN

-- 1. Create warehouse
CREATE WAREHOUSE ZOMATO_WH 
  WAREHOUSE_SIZE = 'XSMALL' 
  AUTO_SUSPEND = 60 
  AUTO_RESUME = TRUE;

-- 2. Create database and schemas
CREATE DATABASE ZOMATO;
CREATE SCHEMA ZOMATO.RAW;
CREATE SCHEMA ZOMATO.STAGING;
CREATE SCHEMA ZOMATO.MARTS;
CREATE SCHEMA ZOMATO.SNAPSHOTS;
CREATE SCHEMA ZOMATO.AI;

-- 3. Create role for dbt
CREATE ROLE DBT_ROLE;
GRANT USAGE ON WAREHOUSE ZOMATO_WH TO ROLE DBT_ROLE;
GRANT USAGE ON DATABASE ZOMATO TO ROLE DBT_ROLE;
GRANT ALL ON ALL SCHEMAS IN DATABASE ZOMATO TO ROLE DBT_ROLE;
GRANT CREATE SCHEMA ON DATABASE ZOMATO TO ROLE DBT_ROLE;

-- Grant to your user
GRANT ROLE DBT_ROLE TO USER YOUR_SNOWFLAKE_USER;

-- 4. Create storage integration (replace with your IAM role ARN)
CREATE STORAGE INTEGRATION s3_int
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = S3
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::YOUR_ACCOUNT_ID:role/snowflake-s3-role'
  STORAGE_ALLOWED_LOCATIONS = ('s3://your-zomato-bucket/raw/');

-- 5. Get Snowflake's IAM user and external ID
DESC STORAGE INTEGRATION s3_int;
-- Copy STORAGE_AWS_IAM_USER_ARN and STORAGE_AWS_EXTERNAL_ID

-- 6. Update IAM role trust policy in AWS
-- Edit aws/iam/snowflake-role-trust-policy-final.json with:
--   - STORAGE_AWS_IAM_USER_ARN as Principal.AWS
--   - STORAGE_AWS_EXTERNAL_ID as Condition.StringEquals.sts:ExternalId
-- Then update role:
```

```bash
aws iam update-assume-role-policy --role-name snowflake-s3-role \
  --policy-document file://aws/iam/snowflake-role-trust-policy-final.json
```

```sql
-- 7. Create stage (back in Snowflake)
USE SCHEMA ZOMATO.RAW;

CREATE STAGE zomato_stage
  STORAGE_INTEGRATION = s3_int
  URL = 's3://your-zomato-bucket/raw/'
  FILE_FORMAT = (TYPE = CSV FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1);

-- Test stage access
LIST @zomato_stage/restaurants/;

-- 8. Create RAW tables
CREATE TABLE RAW.RESTAURANTS (
  restaurant_id INT,
  restaurant_name VARCHAR,
  city VARCHAR,
  address VARCHAR,
  rating FLOAT,
  rating_count INT,
  cost_for_two VARCHAR,
  cuisine VARCHAR,
  lic_no VARCHAR,
  link VARCHAR,
  menu VARCHAR
);

CREATE TABLE RAW.USERS (
  user_id INT,
  name VARCHAR,
  email VARCHAR,
  password VARCHAR,
  age INT,
  gender VARCHAR,
  marital_status VARCHAR,
  occupation VARCHAR,
  monthly_income VARCHAR,
  educational_qualifications VARCHAR,
  family_size INT
);

CREATE TABLE RAW.FOOD (
  food_id INT,
  item VARCHAR,
  veg_or_non_veg VARCHAR
);

CREATE TABLE RAW.MENU (
  menu_id INT,
  restaurant_id INT,
  food_id INT,
  cuisine VARCHAR,
  price FLOAT
);

CREATE TABLE RAW.ORDERS (
  order_id INT,
  customer_id INT,
  restaurant_id INT,
  order_date DATE,
  delivery_date DATE,
  delivery_time TIME,
  order_value FLOAT,
  delivery_fee FLOAT,
  tip_amount FLOAT,
  order_status VARCHAR,
  discount_applied FLOAT,
  delivery_address VARCHAR,
  city VARCHAR,
  payment_method VARCHAR,
  payment_txn_id VARCHAR,
  customer_lat FLOAT,
  customer_long FLOAT,
  restaurant_lat FLOAT,
  restaurant_long FLOAT
);

CREATE TABLE RAW.ORDER_ITEMS (
  order_item_id INT,
  order_id INT,
  food_id INT,
  quantity INT,
  unit_price FLOAT,
  total_price FLOAT
);

CREATE TABLE RAW.REVIEWS (
  review_id INT,
  order_id INT,
  customer_id INT,
  restaurant_id INT,
  rating INT,
  review_text VARCHAR,
  review_date DATE
);

-- 9. Initial load (optional — Airflow will do this)
COPY INTO RAW.RESTAURANTS 
  FROM @zomato_stage/restaurants/ 
  FILE_FORMAT = (TYPE = CSV FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1);
-- Repeat for other tables...
```

### dbt Setup

```bash
cd zomato

# Install dbt-snowflake
pip install dbt-snowflake

# Configure profiles.yml (already set up to use env vars)
export SNOWFLAKE_ACCOUNT=your_account.region
export SNOWFLAKE_USER=your_user
export SNOWFLAKE_PASSWORD=your_password

# Test connection
dbt debug

# Run models (excluding AI models initially)
dbt build --exclude tag:ai

# Run with tests
dbt test

# Run specific model
dbt run --select fct_orders

# Run downstream from a model
dbt run --select fct_orders+
```

**dbt Project Structure**:

```
zomato/
├── dbt_project.yml
├── profiles.yml              # Uses env_var() for credentials
├── models/
│   ├── sources.yml          # Defines RAW schema sources
│   ├── staging/             # Silver layer
│   │   ├── stg_restaurants.sql
│   │   ├── stg_customer.sql
│   │   ├── stg_food.sql
│   │   ├── stg_menu.sql
│   │   ├── stg_orders.sql
│   │   ├── stg_order_items.sql
│   │   └── stg_reviews.sql
│   └── marts/               # Gold layer
│       ├── dimensions/
│       │   ├── dim_restaurants.sql
│       │   ├── dim_customer.sql
│       │   ├── dim_food.sql
│       │   └── dim_date.sql
│       ├── facts/
│       │   ├── fct_orders.sql         # incremental
│       │   └── fct_order_items.sql    # incremental
│       └── business/
│           ├── mart_daily_city_revenue.sql
│           ├── mart_restaurant_performance.sql
│           ├── mart_delivery_sla.sql
│           └── mart_review_insights.sql
└── macros/
    └── generate_schema_name.sql
```

### Airflow Setup

```bash
cd airflow

# Copy environment template
cp example.env .env

# Edit .env with your credentials
# SNOWFLAKE_ACCOUNT=your_account.region
# SNOWFLAKE_USER=your_user
# SNOWFLAKE_PASSWORD=your_password
# SNOWFLAKE_WAREHOUSE=ZOMATO_WH
# SNOWFLAKE_DATABASE=ZOMATO
# SNOWFLAKE_SCHEMA=RAW
# SNOWFLAKE_ROLE=DBT_ROLE
# OPENAI_API_KEY=sk-...
# SAMPLE_N=100  # Number of reviews to enrich per run

# Build and start Airflow
docker compose build
docker compose up -d

# Check services
docker compose ps

# View logs
docker compose logs -f api-server

# Access UI
# http://localhost:8080
# Default credentials: admin / admin

# Un-pause and trigger the DAG
# UI: Zomato Batch → Toggle → Trigger DAG
```

## Key Commands & Patterns

### dbt Incremental Models

**Incremental Fact Pattern** (`fct_orders.sql`):

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
        WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
    {% endif %}
)

SELECT
    order_id,
    customer_id,
    restaurant_id,
    order_date,
    order_value,
    is_delivered,
    -- ... other columns
FROM orders
```

**Why**: Processes only new rows on subsequent runs, avoiding full table scans. Uses MERGE strategy in Snowflake.

### AI Enrichment Pattern

**LLM Review Enrichment** (`ai/enrich_reviews.py`):

```python
import os
import snowflake.connector
from openai import OpenAI

# Environment setup
SAMPLE_N = int(os.getenv('SAMPLE_N', 100))
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

# Create enrichment table if not exists
cursor.execute("""
CREATE TABLE IF NOT EXISTS ZOMATO.AI.REVIEW_ENRICHED (
    review_id INT,
    sentiment VARCHAR,
    topic VARCHAR,
    enriched_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
)
""")

# Get unenriched reviews (idempotent)
cursor.execute(f"""
SELECT r.review_id, r.review_text
FROM ZOMATO.STAGING.STG_REVIEWS r
LEFT JOIN ZOMATO.AI.REVIEW_ENRICHED e ON r.review_id = e.review_id
WHERE e.review_id IS NULL
LIMIT {SAMPLE_N}
""")
reviews = cursor.fetchall()

# Enrich with LLM
for review_id, review_text in reviews:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "Extract sentiment (positive/negative/neutral) and topic (food_quality/delivery/service/pricing/ambiance) from this review. Return JSON: {\"sentiment\": \"...\", \"topic\": \"...\"}"},
            {"role": "user", "content": review_text}
        ],
        response_format={"type": "json_object"}
    )
    
    result = json.loads(response.choices[0].message.content)
    
    # Insert enrichment
    cursor.execute("""
    INSERT INTO ZOMATO.AI.REVIEW_ENRICHED (review_id, sentiment, topic)
    VALUES (%s, %s, %s)
    """, (review_id, result['sentiment'], result['topic']))

conn.commit()
cursor.close()
conn.close()
```

**Key Points**:
- Idempotent: `LEFT JOIN` ensures already-enriched reviews are skipped
- Sample-capped: `SAMPLE_N` env var controls cost
- dbt models can then reference `AI.REVIEW_ENRICHED` like any other table

### RAG Pattern

**Semantic Search on Reviews** (`ai/rag_chat.py`):

```python
import os
import streamlit as st
from openai import OpenAI
import snowflake.connector
import numpy as np

client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))

# Embed and store reviews (one-time)
def embed_reviews():
    conn = snowflake.connector.connect(...)
    cursor = conn.cursor()
    
    # Create embeddings table
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS ZOMATO.AI.REVIEW_EMBEDDINGS (
        review_id INT,
        review_text VARCHAR,
        embedding ARRAY
    )
    """)
    
    # Get reviews
    cursor.execute("SELECT review_id, review_text FROM ZOMATO.STAGING.STG_REVIEWS LIMIT 1000")
    reviews = cursor.fetchall()
    
    for review_id, text in reviews:
        response = client.embeddings.create(
            model="text-embedding-3-small",
            input=text
        )
        embedding = response.data[0].embedding
        
        cursor.execute("""
        INSERT INTO ZOMATO.AI.REVIEW_EMBEDDINGS VALUES (%s, %s, %s)
        """, (review_id, text, embedding))
    
    conn.commit()

# RAG query
def rag_query(question):
    # 1. Embed question
    q_embedding = client.embeddings.create(
        model="text-embedding-3-small",
        input=question
    ).data[0].embedding
    
    # 2. Retrieve similar reviews (cosine similarity in Python — Snowflake's VECTOR type is in preview)
    conn = snowflake.connector.connect(...)
    cursor = conn.cursor()
    cursor.execute("SELECT review_id, review_text, embedding FROM ZOMATO.AI.REVIEW_EMBEDDINGS")
    
    similarities = []
    for review_id, text, embedding in cursor.fetchall():
        sim = np.dot(q_embedding, embedding) / (np.linalg.norm(q_embedding) * np.linalg.norm(embedding))
        similarities.append((review_id, text, sim))
    
    # Top 5
    top_reviews = sorted(similarities, key=lambda x: x[2], reverse=True)[:5]
    context = "\n\n".join([f"Review {r[0]}: {r[1]}" for r in top_reviews])
    
    # 3. Generate answer
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": f"Answer the question based on these reviews:\n{context}"},
            {"role": "user", "content": question}
        ]
    )
    
    return response.choices[0].message.content, top_reviews

# Streamlit app
st.title("Chat with Zomato Reviews")
question = st.text_input("Ask about reviews:")
if question:
    answer, sources = rag_query(question)
    st.write("**Answer:**", answer)
    st.write("**Sources:**")
    for review_id, text, score in sources:
        st.write(f"- Review {review_id} (score: {score:.2f}): {text[:200]}...")
```

### Text-to-SQL Pattern

**Safe LLM-Generated Queries** (`ai/text_to_sql.py`):

```python
import os
import re
import streamlit as st
from openai import OpenAI
import snowflake.connector

client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))

# Get schema context
def get_schema():
    conn = snowflake.connector.connect(...)
    cursor = conn.cursor()
    
    schema_info = []
    cursor.execute("SHOW TABLES IN SCHEMA ZOMATO.MARTS")
    tables = cursor.fetchall()
    
    for table in tables:
        table_name = table[1]
        cursor.execute(f"DESCRIBE TABLE ZOMATO.MARTS.{table_name}")
        columns = cursor.fetchall()
        schema_info.append(f"{table_name}: {', '.join([c[0] for c in columns])}")
    
    return "\n".join(schema_info)

# Validate SQL is SELECT-only
def is_safe_query(sql):
    sql_upper = sql.upper().strip()
    dangerous = ['INSERT', 'UPDATE', 'DELETE', 'DROP', 'CREATE', 'ALTER', 'TRUNCATE', 'GRANT', 'REVOKE']
    return sql_upper.startswith('SELECT') and not any(d in sql_upper for d in dangerous)

# Text-to-SQL
def text_to_sql(question):
    schema = get_schema()
    
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": f"You are a Snowflake SQL expert. Schema:\n{schema}\n\nGenerate a SELECT query to answer the user's question. Return ONLY the SQL, no markdown."},
            {"role": "user", "content": question}
        ]
    )
    
    sql = response.choices[0].message.content.strip()
    
    # Clean markdown if present
    sql = re.sub(r'^```sql\n|^```\n|```$', '', sql, flags=re.MULTILINE).strip()
    
    if not is_safe_query(sql):
        return None, "Unsafe query detected"
    
    # Execute
    conn = snowflake.connector.connect(...)
    cursor = conn.cursor()
    cursor.execute(sql)
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    
    return results, columns, sql

# Streamlit app
st.title("Chat with Zomato Data Warehouse")
question = st.text_input("Ask a question about the data:")
if question:
    results, columns, sql = text_to_sql(question)
    if results:
        st.code(sql, language='sql')
        st.dataframe(pd.DataFrame(results, columns=columns))
    else:
        st.error(columns)  # Error message
```

**Security**: SELECT-only validation prevents destructive operations. Run as `DBT_ROLE` with read-only grants.

## Airflow DAG Structure

**The Pipeline DAG** (`airflow/dags/zomato_batch.py`):

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from airflow.operators.bash import BashOperator

default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

with DAG(
    'zomato_batch',
    default_args=default_args,
    description='Zomato data pipeline: S3 → Snowflake → dbt → AI',
    schedule_interval='@daily',
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=['zomato', 'batch', 'ai'],
) as dag:

    # Task 1: Reload RAW tables from S3
    reload_raw = SnowflakeOperator(
        task_id='reload_raw',
        snowflake_conn_id='snowflake_default',
        sql="""
        TRUNCATE TABLE ZOMATO.RAW.ORDERS;
        COPY INTO ZOMATO.RAW.ORDERS FROM @ZOMATO.RAW.ZOMATO_STAGE/orders/;
        -- Repeat for other tables...
        """
    )

    # Task 2: dbt build (staging + marts, excluding AI)
    dbt_build_core = BashOperator(
        task_id='dbt_build_core',
        bash_command='cd /opt/airflow/dbt/zomato && source venv/bin/activate && dbt build --exclude tag:ai'
    )

    # Task 3: LLM enrichment
    enrich_reviews = BashOperator(
        task_id='enrich_reviews',
        bash_command='cd /opt/airflow && python ai/enrich_reviews.py',
        env={
            'SNOWFLAKE_ACCOUNT': '{{ var.value.SNOWFLAKE_ACCOUNT }}',
            'OPENAI_API_KEY': '{{ var.value.OPENAI_API_KEY }}',
            'SAMPLE_N': '{{ var.value.SAMPLE_N }}'
        }
    )

    # Task 4: Build AI mart
    dbt_build_ai = BashOperator(
        task_id='dbt_build_ai',
        bash_command='cd /opt/airflow/dbt/zomato && source venv/bin/activate && dbt run --select tag:ai'
    )

    # Dependencies
    reload_raw >> dbt_build_core >> enrich_reviews >> dbt_build_ai
```

**Key Points**:
- Env vars injected via docker-compose (never hardcoded)
- dbt runs in its own virtualenv inside the container
- Idempotent: re-run safe due to incremental models and LEFT JOIN enrichment logic

## Configuration

### dbt Profiles (`zomato/profiles.yml`)

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
      client_session_keep_alive: False
```

### dbt Project Config (`zomato/dbt_project.yml`)

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
      dimensions:
        +materialized: table
        +schema: marts
      facts:
        +materialized: incremental
        +schema: marts
        +on_schema_change: append_new_columns
      business:
        +materialized: table
        +schema: marts
        +tags: ['mart']
```

### Environment Variables Reference

```bash
# Snowflake (required everywhere)
export SNOWFLAKE_ACCOUNT=your_account.region
export SNOWFLAKE_USER=your_user
export SNOWFLAKE_PASSWORD=your_password
export SNOWFLAKE_WAREHOUSE=ZOMATO_WH
export SNOWFLAKE_DATABASE=ZOMATO
export SNOWFLAKE_ROLE=DBT_ROLE

# OpenAI (required for AI features)
export OPENAI_API_KEY=sk-...

# AI enrichment control
export SAMPLE_N=100  # Reviews to enrich per run
```

## Common Patterns

### Adding a New Dimension

1. Create staging view in `models/staging/stg_new_dim.sql`:

```sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'new_dim') }}
)

SELECT
    id,
    LOWER(TRIM(name)) AS name,
    COALESCE(category, 'Unknown') AS category
FROM source
```

2. Create dimension in `models/marts/dimensions/dim_new_dim.sql`:

```sql
{{ config(materialized='table') }}

WITH staging AS (
    SELECT * FROM {{ ref('stg_new_dim') }}
)

SELECT
    id AS new_dim_key,
    name,
    category,
    CURRENT_TIMESTAMP() AS dw_created_at
FROM staging
```

3. Add tests in `models/marts/dimensions/schema.yml`:

```yaml
models:
  - name: dim_new_dim
    columns:
      - name: new_dim_key
        tests:
          - unique
          - not_null
```

### Adding an Incremental Fact

```sql
{{
    config(
        materialized='incremental',
        unique_key='fact_key',
        on_schema_change='append_new_columns',
        tags=['fact']
    )
}}

WITH staging AS (
    SELECT * FROM {{ ref('stg_fact') }}
    {% if is_incremental() %}
        WHERE event_date > (SELECT MAX(event_date) FROM {{ this }})
    {% endif %}
)

SELECT
    {{ dbt_utils.generate_surrogate_key(['order_id', 'item_id']) }} AS fact_key,
    order_id,
    item_id,
    event_date,
    measure_value
FROM staging
```

### Creating a Business Mart

**Daily City Revenue** (`models/marts/business/mart_daily_city_revenue.sql`):

```sql
{{ config(materialized='table', tags=['mart']) }}

WITH orders AS (
    SELECT * FROM {{ ref('fct_orders') }}
),

restaurants AS (
    SELECT * FROM {{ ref('dim_restaurants') }}
)

SELECT
    o.order_date,
    r.city,
    COUNT(DISTINCT o.order_id) AS total_orders,
    SUM(o.order_value) AS gmv,
    AVG(o.order_value) AS aov,
    SUM(CASE WHEN o.is_delivered THEN 1 ELSE 0 END) / COUNT(*) AS delivery_rate,
    SUM(CASE WHEN o.order_status = 'cancelled' THEN 1 ELSE 0 END) / COUNT(*) AS cancel_rate
FROM orders o
JOIN restaurants r ON o.restaurant_id = r.restaurant_key
GROUP BY o.order_date, r.city
ORDER BY o.order_date DESC, gmv DESC
```

## Troubleshooting

### Snowflake Storage Integration Issues

**Symptom**: `COPY INTO` fails with "Access Denied"

**Fix**:
1. Verify IAM role ARN in storage integration matches your role
2. Check trust policy has Snowflake's IAM user ARN (from `DESC INTEGRATION`)
3. Never `CREATE OR REPLACE` integration after initial setup — it regenerates external ID

```sql
-- Check integration
DESC STORAGE INTEGRATION s3_int;

-- Test stage
LIST @zomato_stage/orders/;
```

### dbt Incremental Model Not Updating

**Symptom**: New data in source but incremental model unchanged

**Fix**:
1. Check `is_incremental()` logic — ensure filter column exists in source
2. Force full refresh: `
