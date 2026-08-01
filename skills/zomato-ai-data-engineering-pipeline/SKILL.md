---
name: zomato-ai-data-engineering-pipeline
description: End-to-end data pipeline from S3 to Snowflake with dbt transformations, Airflow orchestration, and OpenAI-powered analytics
triggers:
  - "set up the Zomato data pipeline"
  - "configure Snowflake storage integration for S3"
  - "run dbt transformations for Zomato data"
  - "orchestrate the pipeline with Airflow"
  - "add AI enrichment to reviews"
  - "build text-to-SQL with OpenAI"
  - "create incremental fact tables in dbt"
  - "implement RAG for review analytics"
---

# zomato-ai-data-engineering-pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A production-ready batch data pipeline that processes food delivery data through a medallion architecture (Bronze/Silver/Gold) using S3, Snowflake, dbt, Airflow, and OpenAI for AI-powered analytics.

## Architecture Overview

**Data Flow:**
```
CSV → S3 (Lake) → Snowflake RAW (Bronze) → dbt STAGING (Silver) → dbt MARTS (Gold) → AI Layer
```

**Key Components:**
- **Source:** 7 CSVs (~2.3 GB) — 10M orders, 23M order items, 300K reviews
- **Storage:** Amazon S3 with keyless Snowflake integration
- **Warehouse:** Snowflake (RAW, STAGING, MARTS, SNAPSHOTS, AI schemas)
- **Transform:** dbt with incremental models, tests, and snapshots
- **Orchestrate:** Apache Airflow 3 (Docker) with daily DAG
- **AI:** OpenAI GPT-4o-mini for enrichment, RAG, and text-to-SQL

## Installation & Setup

### Prerequisites

```bash
# Required tools
python >= 3.9
docker & docker-compose
AWS account with S3
Snowflake account
OpenAI API key
```

### 1. Clone and Download Dataset

```bash
git clone https://github.com/darshilparmar/zomato-ai-data-engineering-end-to-end-project.git
cd zomato-ai-data-engineering-end-to-end-project

# Download CSVs from Google Drive (link in README) to data/ folder
# Expected files: restaurants.csv, users.csv, food.csv, menu.csv, orders.csv, order_items.csv, reviews.csv
```

### 2. AWS S3 Setup

```bash
# Create S3 bucket
aws s3 mb s3://your-zomato-data-bucket

# Upload data with folder structure
aws s3 cp data/restaurants.csv s3://your-zomato-data-bucket/raw/restaurants/
aws s3 cp data/users.csv s3://your-zomato-data-bucket/raw/users/
aws s3 cp data/food.csv s3://your-zomato-data-bucket/raw/food/
aws s3 cp data/menu.csv s3://your-zomato-data-bucket/raw/menu/
aws s3 cp data/orders.csv s3://your-zomato-data-bucket/raw/orders/
aws s3 cp data/order_items.csv s3://your-zomato-data-bucket/raw/order_items/
aws s3 cp data/reviews.csv s3://your-zomato-data-bucket/raw/reviews/
```

### 3. AWS IAM Configuration (Keyless Integration)

**Step 1: Create IAM Policy**

```bash
# Use aws/iam/s3-read-policy.json
aws iam create-policy \
  --policy-name zomato-s3-read \
  --policy-document file://aws/iam/s3-read-policy.json
```

**Step 2: Create IAM Role**

```bash
# Use aws/iam/snowflake-role-trust-policy-initial.json (placeholder)
aws iam create-role \
  --role-name snowflake-s3-role \
  --assume-role-policy-document file://aws/iam/snowflake-role-trust-policy-initial.json

# Attach policy to role
aws iam attach-role-policy \
  --role-name snowflake-s3-role \
  --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/zomato-s3-read
```

### 4. Snowflake Setup

**Create Objects in Snowsight:**

```sql
-- Warehouse
CREATE WAREHOUSE ZOMATO_WH 
  WITH WAREHOUSE_SIZE = 'MEDIUM' 
  AUTO_SUSPEND = 300 
  AUTO_RESUME = TRUE;

-- Database and Schemas
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
GRANT ALL ON ALL SCHEMAS IN DATABASE ZOMATO TO ROLE DBT_ROLE;
GRANT CREATE SCHEMA ON DATABASE ZOMATO TO ROLE DBT_ROLE;
GRANT ROLE DBT_ROLE TO USER YOUR_DBT_USER;

-- Storage Integration
CREATE STORAGE INTEGRATION s3_zomato_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'S3'
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::YOUR_ACCOUNT_ID:role/snowflake-s3-role'
  STORAGE_ALLOWED_LOCATIONS = ('s3://your-zomato-data-bucket/raw/');

-- Get Snowflake's IAM user and external ID
DESC STORAGE INTEGRATION s3_zomato_integration;
-- Copy STORAGE_AWS_IAM_USER_ARN and STORAGE_AWS_EXTERNAL_ID
```

**Step 3: Update IAM Role Trust Policy**

```bash
# Edit aws/iam/snowflake-role-trust-policy-final.json with values from DESC INTEGRATION
# Replace STORAGE_AWS_IAM_USER_ARN and STORAGE_AWS_EXTERNAL_ID

aws iam update-assume-role-policy \
  --role-name snowflake-s3-role \
  --policy-document file://aws/iam/snowflake-role-trust-policy-final.json
```

**Create External Stage:**

```sql
CREATE STAGE ZOMATO.RAW.s3_stage
  STORAGE_INTEGRATION = s3_zomato_integration
  URL = 's3://your-zomato-data-bucket/raw/';

-- Test access
LIST @ZOMATO.RAW.s3_stage;
```

**Create RAW Tables:**

```sql
USE SCHEMA ZOMATO.RAW;

CREATE TABLE restaurants (
  restaurant_id INT,
  restaurant_name STRING,
  city STRING,
  address STRING,
  rating FLOAT,
  rating_count INT,
  cost_per_person STRING,
  cuisine STRING
);

CREATE TABLE users (
  user_id INT,
  name STRING,
  email STRING,
  phone STRING,
  date_of_birth DATE,
  city STRING
);

CREATE TABLE food (
  food_id INT,
  food_name STRING,
  category STRING,
  price FLOAT
);

CREATE TABLE menu (
  menu_id INT,
  restaurant_id INT,
  food_id INT
);

CREATE TABLE orders (
  order_id INT,
  user_id INT,
  restaurant_id INT,
  order_date TIMESTAMP,
  delivery_time TIMESTAMP,
  order_value FLOAT,
  delivery_fee FLOAT,
  tip_amount FLOAT,
  discount_applied FLOAT,
  status STRING,
  payment_method STRING
);

CREATE TABLE order_items (
  order_item_id INT,
  order_id INT,
  food_id INT,
  quantity INT,
  price FLOAT
);

CREATE TABLE reviews (
  review_id INT,
  order_id INT,
  user_id INT,
  restaurant_id INT,
  rating INT,
  review_text STRING,
  review_date TIMESTAMP
);
```

### 5. dbt Setup

**Install dbt:**

```bash
cd zomato
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install dbt-snowflake
```

**Configure profiles.yml:**

```yaml
# ~/.dbt/profiles.yml
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

**Set Environment Variables:**

```bash
export SNOWFLAKE_ACCOUNT=your_account.region
export SNOWFLAKE_USER=your_user
export SNOWFLAKE_PASSWORD=your_password
```

**Test Connection:**

```bash
dbt debug
```

### 6. Airflow Setup

```bash
cd airflow

# Copy and configure environment file
cp example.env .env

# Edit .env with your credentials:
# SNOWFLAKE_ACCOUNT=your_account.region
# SNOWFLAKE_USER=your_user
# SNOWFLAKE_PASSWORD=your_password
# SNOWFLAKE_DATABASE=ZOMATO
# SNOWFLAKE_SCHEMA=RAW
# SNOWFLAKE_WAREHOUSE=ZOMATO_WH
# SNOWFLAKE_ROLE=DBT_ROLE
# OPENAI_API_KEY=sk-...
# S3_BUCKET=your-zomato-data-bucket
# SAMPLE_N=1000  # Number of reviews to enrich

# Build and start Airflow
docker compose build
docker compose up -d

# Access UI at http://localhost:8080
# Default credentials: admin/admin
```

## dbt Transformations

### Project Structure

```
zomato/
├── dbt_project.yml
├── models/
│   ├── staging/          # Silver layer (views)
│   │   ├── sources.yml   # Source definitions
│   │   ├── stg_restaurants.sql
│   │   ├── stg_users.sql
│   │   ├── stg_food.sql
│   │   ├── stg_menu.sql
│   │   ├── stg_orders.sql
│   │   ├── stg_order_items.sql
│   │   └── stg_reviews.sql
│   └── marts/            # Gold layer (tables)
│       ├── dim_restaurants.sql
│       ├── dim_customer.sql
│       ├── dim_food.sql
│       ├── dim_date.sql
│       ├── fct_orders.sql (incremental)
│       ├── fct_order_items.sql (incremental)
│       ├── mart_daily_city_metrics.sql
│       ├── mart_restaurant_performance.sql
│       ├── mart_delivery_sla.sql
│       └── mart_review_insights.sql
├── macros/
│   └── generate_schema_name.sql
└── tests/
    └── reconciliation_orders.sql
```

### Key dbt Commands

```bash
# Run all models and tests
dbt build

# Run only staging models
dbt run --select staging

# Run only marts (excluding AI models)
dbt run --select marts --exclude tag:ai

# Run incremental models (full refresh)
dbt run --select config.materialized:incremental --full-refresh

# Test data quality
dbt test

# Generate documentation
dbt docs generate
dbt docs serve  # View at http://localhost:8080
```

### Staging Layer Example

**stg_orders.sql:**

```sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'orders') }}
)

SELECT
    order_id,
    user_id,
    restaurant_id,
    order_date,
    delivery_time,
    order_value,
    delivery_fee,
    tip_amount,
    discount_applied,
    LOWER(status) AS status,
    CASE 
        WHEN LOWER(status) = 'delivered' THEN TRUE
        ELSE FALSE
    END AS is_delivered,
    payment_method,
    DATEDIFF('minute', order_date, delivery_time) AS delivery_time_minutes
FROM source
WHERE order_id IS NOT NULL
```

### Incremental Fact Table Example

**fct_orders.sql:**

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
)

SELECT
    order_id,
    user_id,
    restaurant_id,
    order_date,
    delivery_time,
    order_value,
    delivery_fee,
    tip_amount,
    discount_applied,
    is_delivered,
    payment_method,
    delivery_time_minutes
FROM orders

{% if is_incremental() %}
    WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

### Business Mart Example

**mart_daily_city_metrics.sql:**

```sql
WITH orders AS (
    SELECT * FROM {{ ref('fct_orders') }}
),

restaurants AS (
    SELECT * FROM {{ ref('dim_restaurants') }}
),

metrics AS (
    SELECT
        DATE(o.order_date) AS order_date,
        r.city,
        COUNT(DISTINCT o.order_id) AS total_orders,
        COUNT(DISTINCT CASE WHEN o.is_delivered THEN o.order_id END) AS delivered_orders,
        SUM(o.order_value) AS gmv,
        AVG(o.order_value) AS aov,
        SUM(o.delivery_fee) AS total_delivery_fees,
        AVG(o.delivery_time_minutes) AS avg_delivery_time,
        COUNT(DISTINCT CASE WHEN NOT o.is_delivered THEN o.order_id END) / COUNT(DISTINCT o.order_id)::FLOAT AS cancel_rate
    FROM orders o
    JOIN restaurants r ON o.restaurant_id = r.restaurant_id
    GROUP BY 1, 2
)

SELECT * FROM metrics
```

### Testing Example

**models/staging/schema.yml:**

```yaml
version: 2

models:
  - name: stg_orders
    description: Cleaned and standardized orders
    columns:
      - name: order_id
        description: Primary key
        tests:
          - unique
          - not_null
      - name: status
        tests:
          - accepted_values:
              values: ['delivered', 'cancelled', 'pending']
      - name: user_id
        tests:
          - relationships:
              to: ref('stg_users')
              field: user_id
```

## Airflow Orchestration

### DAG Structure

**dags/zomato_batch.py:**

```python
from airflow import DAG
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta
import os

default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'email_on_failure': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5)
}

with DAG(
    'zomato_batch',
    default_args=default_args,
    description='Zomato end-to-end batch pipeline',
    schedule_interval='@daily',
    catchup=False,
    tags=['zomato', 'batch']
) as dag:

    # Task 1: Load RAW tables from S3
    reload_raw = SnowflakeOperator(
        task_id='reload_raw',
        snowflake_conn_id='snowflake_default',
        sql="""
        -- Truncate and reload RAW tables
        TRUNCATE TABLE ZOMATO.RAW.restaurants;
        COPY INTO ZOMATO.RAW.restaurants 
        FROM @ZOMATO.RAW.s3_stage/restaurants/
        FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1);
        
        TRUNCATE TABLE ZOMATO.RAW.users;
        COPY INTO ZOMATO.RAW.users 
        FROM @ZOMATO.RAW.s3_stage/users/
        FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1);
        
        -- Repeat for all tables...
        """
    )

    # Task 2: Run dbt transformations (excluding AI models)
    dbt_build_core = BashOperator(
        task_id='dbt_build_core',
        bash_command='cd /opt/airflow/dbt/zomato && source venv/bin/activate && dbt build --exclude tag:ai',
        env={
            'SNOWFLAKE_ACCOUNT': os.getenv('SNOWFLAKE_ACCOUNT'),
            'SNOWFLAKE_USER': os.getenv('SNOWFLAKE_USER'),
            'SNOWFLAKE_PASSWORD': os.getenv('SNOWFLAKE_PASSWORD')
        }
    )

    # Task 3: AI enrichment
    def run_enrichment():
        import sys
        sys.path.append('/opt/airflow/ai')
        from enrich_reviews import enrich_reviews
        enrich_reviews(sample_n=int(os.getenv('SAMPLE_N', 1000)))

    enrich_reviews_task = PythonOperator(
        task_id='enrich_reviews',
        python_callable=run_enrichment
    )

    # Task 4: Build AI marts
    dbt_build_ai = BashOperator(
        task_id='dbt_build_ai',
        bash_command='cd /opt/airflow/dbt/zomato && source venv/bin/activate && dbt run --select tag:ai',
        env={
            'SNOWFLAKE_ACCOUNT': os.getenv('SNOWFLAKE_ACCOUNT'),
            'SNOWFLAKE_USER': os.getenv('SNOWFLAKE_USER'),
            'SNOWFLAKE_PASSWORD': os.getenv('SNOWFLAKE_PASSWORD')
        }
    )

    # Dependencies
    reload_raw >> dbt_build_core >> enrich_reviews_task >> dbt_build_ai
```

### Managing the DAG

```bash
# Trigger manually
docker exec -it airflow-scheduler airflow dags trigger zomato_batch

# View DAG status
docker exec -it airflow-scheduler airflow dags list

# Test specific task
docker exec -it airflow-scheduler airflow tasks test zomato_batch reload_raw 2024-01-01

# Check logs
docker logs airflow-scheduler -f
```

## AI Layer

### 1. LLM Enrichment

**ai/enrich_reviews.py:**

```python
import os
import json
from openai import OpenAI
import snowflake.connector

def enrich_reviews(sample_n=1000):
    """Enrich reviews with sentiment and topic using GPT-4o-mini"""
    
    client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))
    
    # Connect to Snowflake
    conn = snowflake.connector.connect(
        account=os.getenv('SNOWFLAKE_ACCOUNT'),
        user=os.getenv('SNOWFLAKE_USER'),
        password=os.getenv('SNOWFLAKE_PASSWORD'),
        warehouse='ZOMATO_WH',
        database='ZOMATO',
        schema='RAW'
    )
    
    cursor = conn.cursor()
    
    # Create enriched table if not exists
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS ZOMATO.AI.REVIEW_ENRICHED (
            review_id INT PRIMARY KEY,
            review_text STRING,
            sentiment STRING,
            topic STRING,
            enriched_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
        )
    """)
    
    # Get unenriched reviews
    cursor.execute(f"""
        SELECT r.review_id, r.review_text
        FROM ZOMATO.RAW.reviews r
        LEFT JOIN ZOMATO.AI.REVIEW_ENRICHED e ON r.review_id = e.review_id
        WHERE e.review_id IS NULL
        LIMIT {sample_n}
    """)
    
    reviews = cursor.fetchall()
    
    enriched_data = []
    for review_id, review_text in reviews:
        if not review_text:
            continue
            
        try:
            # Call OpenAI
            response = client.chat.completions.create(
                model="gpt-4o-mini",
                messages=[
                    {"role": "system", "content": "You are a review analyzer. Return JSON with 'sentiment' (positive/negative/neutral) and 'topic' (food/service/delivery/price/ambiance)."},
                    {"role": "user", "content": f"Analyze this restaurant review: {review_text}"}
                ],
                response_format={"type": "json_object"},
                temperature=0
            )
            
            result = json.loads(response.choices[0].message.content)
            enriched_data.append((
                review_id,
                review_text,
                result.get('sentiment', 'unknown'),
                result.get('topic', 'unknown')
            ))
            
        except Exception as e:
            print(f"Error enriching review {review_id}: {e}")
            continue
    
    # Bulk insert
    if enriched_data:
        cursor.executemany(
            "INSERT INTO ZOMATO.AI.REVIEW_ENRICHED (review_id, review_text, sentiment, topic) VALUES (%s, %s, %s, %s)",
            enriched_data
        )
        conn.commit()
    
    print(f"Enriched {len(enriched_data)} reviews")
    cursor.close()
    conn.close()

if __name__ == '__main__':
    enrich_reviews()
```

**Run Enrichment:**

```bash
export OPENAI_API_KEY=sk-...
export SNOWFLAKE_ACCOUNT=your_account
export SNOWFLAKE_USER=your_user
export SNOWFLAKE_PASSWORD=your_password

python ai/enrich_reviews.py
```

### 2. RAG (Chat with Reviews)

**ai/rag_chat.py:**

```python
import os
import streamlit as st
from openai import OpenAI
import snowflake.connector
import numpy as np

client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))

def get_embedding(text):
    """Get embedding for text"""
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return response.data[0].embedding

def search_reviews(query, top_k=5):
    """Search reviews using cosine similarity"""
    conn = snowflake.connector.connect(
        account=os.getenv('SNOWFLAKE_ACCOUNT'),
        user=os.getenv('SNOWFLAKE_USER'),
        password=os.getenv('SNOWFLAKE_PASSWORD'),
        warehouse='ZOMATO_WH',
        database='ZOMATO'
    )
    
    cursor = conn.cursor()
    
    # Get query embedding
    query_embedding = get_embedding(query)
    
    # Fetch all reviews with embeddings (in production, use vector DB)
    cursor.execute("""
        SELECT review_id, review_text, sentiment, topic
        FROM ZOMATO.AI.REVIEW_ENRICHED
    """)
    
    reviews = cursor.fetchall()
    
    # Calculate similarity (simplified - in production, use Snowflake's vector functions)
    results = []
    for review_id, text, sentiment, topic in reviews:
        review_embedding = get_embedding(text)
        similarity = np.dot(query_embedding, review_embedding) / (
            np.linalg.norm(query_embedding) * np.linalg.norm(review_embedding)
        )
        results.append((review_id, text, sentiment, topic, similarity))
    
    # Sort by similarity
    results.sort(key=lambda x: x[4], reverse=True)
    
    cursor.close()
    conn.close()
    
    return results[:top_k]

def generate_answer(query, context_reviews):
    """Generate answer using RAG"""
    context = "\n\n".join([
        f"Review {i+1} (sentiment: {r[2]}, topic: {r[3]}): {r[1]}" 
        for i, r in enumerate(context_reviews)
    ])
    
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "You are a helpful assistant that answers questions about restaurant reviews. Base your answers ONLY on the provided reviews."},
            {"role": "user", "content": f"Reviews:\n{context}\n\nQuestion: {query}"}
        ],
        temperature=0.7
    )
    
    return response.choices[0].message.content

# Streamlit UI
st.title("🍔 Chat with Your Reviews (RAG)")

query = st.text_input("Ask a question about customer reviews:")

if st.button("Search") and query:
    with st.spinner("Searching reviews..."):
        top_reviews = search_reviews(query)
        
        answer = generate_answer(query, top_reviews)
        
        st.subheader("Answer:")
        st.write(answer)
        
        st.subheader("Sources:")
        for i, (review_id, text, sentiment, topic, similarity) in enumerate(top_reviews):
            with st.expander(f"Review {i+1} (similarity: {similarity:.2f})"):
                st.write(f"**Sentiment:** {sentiment} | **Topic:** {topic}")
                st.write(text)
```

**Run RAG App:**

```bash
streamlit run ai/rag_chat.py
```

### 3. Text-to-SQL

**ai/text_to_sql.py:**

```python
import os
import streamlit as st
from openai import OpenAI
import snowflake.connector
import re

client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))

def get_schema():
    """Get mart schemas for context"""
    return """
    Available marts in ZOMATO.MARTS:
    
    1. mart_daily_city_metrics (order_date, city, total_orders, delivered_orders, gmv, aov, cancel_rate)
    2. mart_restaurant_performance (restaurant_id, restaurant_name, city, total_orders, avg_rating, total_revenue)
    3. mart_delivery_sla (city, hour_of_day, p50_delivery_time, p90_delivery_time, avg_delivery_time)
    4. mart_review_insights (restaurant_id, restaurant_name, total_reviews, avg_rating, positive_reviews, negative_reviews, top_topics)
    
    All date columns are DATE type. Use Snowflake SQL syntax.
    """

def is_safe_query(sql):
    """Validate query is SELECT-only"""
    sql_lower = sql.lower().strip()
    dangerous_keywords = ['insert', 'update', 'delete', 'drop', 'truncate', 'alter', 'create']
    
    if not sql_lower.startswith('select'):
        return False
    
    for keyword in dangerous_keywords:
        if keyword in sql_lower:
            return False
    
    return True

def text_to_sql(question):
    """Convert natural language to SQL"""
    schema = get_schema()
    
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": f"You are a Snowflake SQL expert. Convert questions to SELECT queries only. Schema:\n{schema}"},
            {"role": "user", "content": question}
        ],
        temperature=0
    )
    
    sql = response.choices[0].message.content
    
    # Extract SQL from markdown code blocks
    match = re.search(r'```sql\n(.*?)\n```', sql, re.DOTALL)
    if match:
        sql = match.group(1)
    
    return sql.strip()

def execute_query(sql):
    """Execute SQL and return results"""
    conn = snowflake.connector.connect(
        account=os.getenv('SNOWFLAKE_ACCOUNT'),
        user=os.getenv('SNOWFLAKE_USER'),
        password=os.getenv('SNOWFLAKE_PASSWORD'),
        warehouse='ZOMATO_WH',
        database='ZOMATO',
        schema='MARTS',
        role='DBT_ROLE'
    )
    
    cursor = conn.cursor()
    cursor.execute(sql)
    
    columns = [col[0] for col in cursor.description]
    results = cursor.fetchall()
    
    cursor.close()
    conn.close()
    
    return columns, results

# Streamlit UI
st.title("💬 Chat with Your Data Warehouse")

question = st.text_input("Ask a question in plain English:")

if st.button("Generate SQL") and question:
    with st.spinner("Generating SQL..."):
        sql = text_to_sql(question)
        
        st.subheader("Generated SQL:")
        st.code(sql, language="sql")
        
        if is_safe_query(sql):
            if st.button("Execute Query"):
                with st.spinner("Running query..."):
                    try:
                        columns, results = execute_query(sql)
                        
                        st.subheader("Results:")
                        st.dataframe(
                            data=results,
                            column_config={col: st.column_config.Column(col) for col in columns}
                        )
                        
                    except Exception as e:
                        st.error(f"Query failed: {e}")
        else:
            st.error("⚠️ Query contains unsafe operations. Only SELECT queries are allowed.")
```

**Run Text-to-SQL App:**

```bash
streamlit run ai/text_to_sql.py
```

## Common Patterns

### Incremental Load Pattern

```sql
-- Pattern for new records only
{{
    config(
        materialized='incremental',
        unique_key='id'
    )
}}

SELECT * FROM {{ ref('staging_model') }}

{% if is_incremental() %}
    WHERE created_at > (SELECT MAX(created_at) FROM {{ this }})
{% endif %}
```

### Late Arriving Dimensions (SCD Type 2)

```sql
-- Use dbt snapshots for S
