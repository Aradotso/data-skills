---
name: zomato-ai-data-engineering-pipeline
description: End-to-end batch data pipeline with Snowflake, dbt, Airflow, and OpenAI for food delivery analytics
triggers:
  - build a zomato data pipeline
  - set up snowflake medallion architecture
  - create dbt incremental models for zomato
  - orchestrate data pipeline with airflow
  - enrich reviews with openai llm
  - implement rag for text data
  - build text to sql with openai
  - configure s3 snowflake integration
---

# zomato-ai-data-engineering-pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

Complete batch data engineering pipeline that processes food delivery data through a medallion architecture (Bronze → Silver → Gold) using Amazon S3, Snowflake, dbt, Airflow orchestration, and OpenAI-powered AI capabilities (LLM enrichment, RAG, text-to-SQL).

## Project Overview

**Pipeline Flow:**
```
CSVs → S3 Data Lake → Snowflake RAW (Bronze) → dbt STAGING (Silver) → dbt MARTS (Gold) → AI Layer
```

**Architecture Layers:**
- **Bronze (RAW)**: Direct `COPY INTO` from S3 via storage integration
- **Silver (STAGING)**: dbt views for cleaning, typing, renaming
- **Gold (MARTS)**: Dimensions, incremental facts (MERGE), business aggregates, SCD2 snapshots
- **AI**: LLM enrichment, RAG chat, text-to-SQL queries

**Data Scale:**
- 10M orders
- 23M order items
- 300K text reviews
- 7 source tables (restaurants, users, food, menu, orders, order_items, reviews)

## Installation & Setup

### Prerequisites

```bash
# Clone and get dataset
git clone https://github.com/darshilparmar/zomato-ai-data-engineering-end-to-end-project
cd zomato-ai-data-engineering-end-to-end-project

# Download CSVs from Google Drive (link in README) → place in data/
```

### AWS S3 Setup

```bash
# 1. Create S3 bucket
aws s3 mb s3://your-zomato-bucket

# 2. Upload data to S3
aws s3 sync data/ s3://your-zomato-bucket/raw/ --exclude "*" \
  --include "restaurants/*" \
  --include "users/*" \
  --include "food/*" \
  --include "menu/*" \
  --include "orders/*" \
  --include "order_items/*" \
  --include "reviews/*"

# 3. Create IAM policy (use aws/iam/s3-read-policy.json)
aws iam create-policy \
  --policy-name zomato-s3-read \
  --policy-document file://aws/iam/s3-read-policy.json

# 4. Create IAM role with initial trust policy
aws iam create-role \
  --role-name snowflake-s3-role \
  --assume-role-policy-document file://aws/iam/snowflake-role-trust-policy-initial.json

# 5. Attach policy to role
aws iam attach-role-policy \
  --role-name snowflake-s3-role \
  --policy-arn arn:aws:iam::YOUR_ACCOUNT:policy/zomato-s3-read
```

### Snowflake Setup

```sql
-- 1. Create warehouse, database, schemas
CREATE WAREHOUSE ZOMATO_WH 
  WITH WAREHOUSE_SIZE = 'MEDIUM' 
  AUTO_SUSPEND = 60 
  AUTO_RESUME = TRUE;

CREATE DATABASE ZOMATO;

USE DATABASE ZOMATO;
CREATE SCHEMA RAW;
CREATE SCHEMA STAGING;
CREATE SCHEMA MARTS;
CREATE SCHEMA SNAPSHOTS;
CREATE SCHEMA AI;

-- 2. Create role and grant permissions
CREATE ROLE DBT_ROLE;
GRANT USAGE ON WAREHOUSE ZOMATO_WH TO ROLE DBT_ROLE;
GRANT ALL ON DATABASE ZOMATO TO ROLE DBT_ROLE;
GRANT ALL ON ALL SCHEMAS IN DATABASE ZOMATO TO ROLE DBT_ROLE;
GRANT ROLE DBT_ROLE TO USER YOUR_USER;

-- 3. Create storage integration (replace with your IAM role ARN)
CREATE STORAGE INTEGRATION s3_zomato_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'S3'
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::YOUR_ACCOUNT:role/snowflake-s3-role'
  STORAGE_ALLOWED_LOCATIONS = ('s3://your-zomato-bucket/raw/');

-- 4. Get Snowflake's IAM user ARN and external ID
DESC STORAGE INTEGRATION s3_zomato_integration;
-- Copy STORAGE_AWS_IAM_USER_ARN and STORAGE_AWS_EXTERNAL_ID

-- 5. Update IAM role trust policy with these values (aws/iam/snowflake-role-trust-policy-final.json)

-- 6. Create external stage
CREATE STAGE s3_stage
  STORAGE_INTEGRATION = s3_zomato_integration
  URL = 's3://your-zomato-bucket/raw/';

-- 7. Create RAW tables
CREATE TABLE RAW.RESTAURANTS (
  restaurant_id NUMBER,
  name VARCHAR,
  city VARCHAR,
  rating FLOAT,
  rating_count NUMBER,
  cost VARCHAR,
  cuisine VARCHAR,
  lic_no VARCHAR,
  link VARCHAR,
  address VARCHAR,
  menu VARCHAR
);

CREATE TABLE RAW.USERS (
  user_id NUMBER,
  name VARCHAR,
  email VARCHAR,
  password VARCHAR,
  age NUMBER,
  gender VARCHAR,
  marital_status VARCHAR,
  occupation VARCHAR,
  monthly_income NUMBER,
  educational_qualifications VARCHAR,
  family_size NUMBER
);

CREATE TABLE RAW.FOOD (
  food_id NUMBER,
  item VARCHAR,
  veg_or_non_veg VARCHAR
);

CREATE TABLE RAW.MENU (
  menu_id NUMBER,
  restaurant_id NUMBER,
  food_id NUMBER,
  cuisine VARCHAR,
  price NUMBER
);

CREATE TABLE RAW.ORDERS (
  order_id NUMBER,
  user_id NUMBER,
  restaurant_id NUMBER,
  order_date DATE,
  order_time TIME,
  order_status VARCHAR,
  order_value NUMBER
);

CREATE TABLE RAW.ORDER_ITEMS (
  order_item_id NUMBER,
  order_id NUMBER,
  food_id NUMBER,
  quantity NUMBER,
  price NUMBER
);

CREATE TABLE RAW.REVIEWS (
  review_id NUMBER,
  order_id NUMBER,
  restaurant_id NUMBER,
  user_id NUMBER,
  rating NUMBER,
  review_text VARCHAR,
  review_date DATE
);
```

### dbt Configuration

```bash
cd zomato

# Create profiles.yml (or update ~/.dbt/profiles.yml)
cat > profiles.yml <<EOF
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
EOF

# Set environment variables
export SNOWFLAKE_ACCOUNT=your_account.region
export SNOWFLAKE_USER=your_user
export SNOWFLAKE_PASSWORD=your_password

# Test connection
dbt debug

# Install dependencies
dbt deps
```

### Airflow Setup

```bash
cd airflow

# Create .env from example
cp example.env .env

# Edit .env with your credentials
# SNOWFLAKE_ACCOUNT=your_account.region
# SNOWFLAKE_USER=your_user
# SNOWFLAKE_PASSWORD=your_password
# SNOWFLAKE_ROLE=DBT_ROLE
# SNOWFLAKE_WAREHOUSE=ZOMATO_WH
# SNOWFLAKE_DATABASE=ZOMATO
# OPENAI_API_KEY=your_openai_key
# S3_BUCKET=your-zomato-bucket
# SAMPLE_N=1000

# Build and start Airflow
docker compose build
docker compose up -d

# Access Airflow UI
# http://localhost:8080 (admin/admin)
```

## Key dbt Models

### Staging (Silver Layer)

```yaml
# models/staging/schema.yml
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

```sql
-- models/staging/stg_restaurants.sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'restaurants') }}
),

cleaned AS (
    SELECT
        restaurant_id,
        TRIM(name) AS restaurant_name,
        LOWER(TRIM(city)) AS city,
        rating,
        rating_count,
        -- Parse cost: '₹ 200' → 200, '--' → NULL
        TRY_CAST(
            REPLACE(REPLACE(cost, '₹', ''), ' ', '')
            AS NUMBER
        ) AS avg_cost_for_two,
        TRIM(cuisine) AS cuisine,
        NULLIF(TRIM(lic_no), '--') AS license_number,
        link AS restaurant_url,
        address,
        menu AS menu_url
    FROM source
)

SELECT * FROM cleaned
```

```sql
-- models/staging/stg_orders.sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'orders') }}
),

cleaned AS (
    SELECT
        order_id,
        user_id,
        restaurant_id,
        order_date,
        order_time,
        LOWER(TRIM(order_status)) AS order_status,
        order_value,
        -- Derive delivery flag
        CASE 
            WHEN order_status = 'delivered' THEN TRUE
            ELSE FALSE
        END AS is_delivered,
        -- Derive cancellation flag
        CASE 
            WHEN order_status IN ('cancelled', 'canceled') THEN TRUE
            ELSE FALSE
        END AS is_cancelled
    FROM source
)

SELECT * FROM cleaned
```

### Marts (Gold Layer)

```sql
-- models/marts/dim_restaurants.sql
{{ config(
    materialized='table'
) }}

SELECT
    restaurant_id,
    restaurant_name,
    city,
    rating,
    rating_count,
    avg_cost_for_two,
    cuisine,
    license_number,
    restaurant_url,
    address
FROM {{ ref('stg_restaurants') }}
```

```sql
-- models/marts/dim_customer.sql
{{ config(
    materialized='table'
) }}

WITH customers AS (
    SELECT
        user_id,
        name AS customer_name,
        LOWER(email) AS email,
        age,
        gender,
        marital_status,
        occupation,
        monthly_income,
        educational_qualifications,
        family_size,
        -- Age segmentation
        CASE
            WHEN age < 25 THEN '18-24'
            WHEN age BETWEEN 25 AND 34 THEN '25-34'
            WHEN age BETWEEN 35 AND 44 THEN '35-44'
            WHEN age BETWEEN 45 AND 54 THEN '45-54'
            WHEN age >= 55 THEN '55+'
            ELSE 'Unknown'
        END AS age_segment
    FROM {{ ref('stg_users') }}
)

SELECT * FROM customers
```

```sql
-- models/marts/fct_orders.sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    on_schema_change='append_new_columns'
) }}

WITH orders AS (
    SELECT
        order_id,
        user_id,
        restaurant_id,
        order_date,
        order_time,
        order_status,
        order_value,
        is_delivered,
        is_cancelled
    FROM {{ ref('stg_orders') }}
    
    {% if is_incremental() %}
    WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
    {% endif %}
)

SELECT * FROM orders
```

```sql
-- models/marts/fact_order_items.sql
{{ config(
    materialized='incremental',
    unique_key='order_item_id',
    on_schema_change='append_new_columns'
) }}

WITH order_items AS (
    SELECT
        oi.order_item_id,
        oi.order_id,
        oi.food_id,
        oi.quantity,
        oi.price,
        oi.quantity * oi.price AS line_total,
        o.order_date
    FROM {{ ref('stg_order_items') }} oi
    JOIN {{ ref('stg_orders') }} o ON oi.order_id = o.order_id
    
    {% if is_incremental() %}
    WHERE o.order_date > (SELECT MAX(order_date) FROM {{ this }})
    {% endif %}
)

SELECT * FROM order_items
```

```sql
-- models/marts/mart_daily_city_revenue.sql
{{ config(
    materialized='table'
) }}

WITH daily_metrics AS (
    SELECT
        o.order_date,
        r.city,
        COUNT(DISTINCT o.order_id) AS total_orders,
        COUNT(DISTINCT CASE WHEN o.is_delivered THEN o.order_id END) AS delivered_orders,
        COUNT(DISTINCT CASE WHEN o.is_cancelled THEN o.order_id END) AS cancelled_orders,
        SUM(CASE WHEN o.is_delivered THEN o.order_value ELSE 0 END) AS gmv,
        AVG(CASE WHEN o.is_delivered THEN o.order_value END) AS aov,
        COUNT(DISTINCT o.user_id) AS active_customers,
        COUNT(DISTINCT o.restaurant_id) AS active_restaurants
    FROM {{ ref('fct_orders') }} o
    JOIN {{ ref('dim_restaurants') }} r ON o.restaurant_id = r.restaurant_id
    GROUP BY o.order_date, r.city
)

SELECT
    order_date,
    city,
    total_orders,
    delivered_orders,
    cancelled_orders,
    ROUND(cancelled_orders::FLOAT / NULLIF(total_orders, 0) * 100, 2) AS cancellation_rate_pct,
    gmv,
    aov,
    active_customers,
    active_restaurants,
    ROUND(gmv / NULLIF(active_restaurants, 0), 2) AS revenue_per_restaurant
FROM daily_metrics
```

### dbt Testing

```yaml
# models/marts/schema.yml
version: 2

models:
  - name: dim_restaurants
    description: Restaurant dimension
    columns:
      - name: restaurant_id
        description: Primary key
        tests:
          - unique
          - not_null
      - name: city
        tests:
          - not_null

  - name: fct_orders
    description: Orders fact table (incremental)
    columns:
      - name: order_id
        description: Primary key
        tests:
          - unique
          - not_null
      - name: user_id
        tests:
          - not_null
          - relationships:
              to: ref('dim_customer')
              field: user_id
      - name: restaurant_id
        tests:
          - not_null
          - relationships:
              to: ref('dim_restaurants')
              field: restaurant_id
      - name: order_status
        tests:
          - accepted_values:
              values: ['delivered', 'cancelled', 'pending', 'preparing']
```

## Airflow DAG

```python
# airflow/dags/zomato_batch.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator
import os

# Environment variables
SNOWFLAKE_CONN_ID = 'snowflake_default'
S3_BUCKET = os.getenv('S3_BUCKET')

default_args = {
    'owner': 'data-eng',
    'depends_on_past': False,
    'email_on_failure': False,
    'email_on_retry': False,
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
    tags=['zomato', 'batch', 'ai'],
) as dag:

    # Task 1: Reload RAW tables from S3
    reload_raw = SnowflakeOperator(
        task_id='reload_raw',
        snowflake_conn_id=SNOWFLAKE_CONN_ID,
        sql=f"""
        USE SCHEMA ZOMATO.RAW;
        
        COPY INTO RESTAURANTS FROM @s3_stage/restaurants/
        FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
        FORCE = TRUE;
        
        COPY INTO USERS FROM @s3_stage/users/
        FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
        FORCE = TRUE;
        
        COPY INTO FOOD FROM @s3_stage/food/
        FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
        FORCE = TRUE;
        
        COPY INTO MENU FROM @s3_stage/menu/
        FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
        FORCE = TRUE;
        
        COPY INTO ORDERS FROM @s3_stage/orders/
        FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
        FORCE = TRUE;
        
        COPY INTO ORDER_ITEMS FROM @s3_stage/order_items/
        FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
        FORCE = TRUE;
        
        COPY INTO REVIEWS FROM @s3_stage/reviews/
        FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
        FORCE = TRUE;
        """
    )

    # Task 2: dbt build (core models excluding AI)
    dbt_build_core = BashOperator(
        task_id='dbt_build_core',
        bash_command='cd /opt/airflow/dbt/zomato && dbt build --exclude tag:ai',
        env={
            'SNOWFLAKE_ACCOUNT': os.getenv('SNOWFLAKE_ACCOUNT'),
            'SNOWFLAKE_USER': os.getenv('SNOWFLAKE_USER'),
            'SNOWFLAKE_PASSWORD': os.getenv('SNOWFLAKE_PASSWORD'),
        }
    )

    # Task 3: Enrich reviews with OpenAI
    def run_enrich_reviews():
        import sys
        sys.path.append('/opt/airflow/ai')
        from enrich_reviews import enrich_reviews
        enrich_reviews()

    enrich_reviews_task = PythonOperator(
        task_id='enrich_reviews',
        python_callable=run_enrich_reviews,
        env={
            'OPENAI_API_KEY': os.getenv('OPENAI_API_KEY'),
            'SNOWFLAKE_ACCOUNT': os.getenv('SNOWFLAKE_ACCOUNT'),
            'SNOWFLAKE_USER': os.getenv('SNOWFLAKE_USER'),
            'SNOWFLAKE_PASSWORD': os.getenv('SNOWFLAKE_PASSWORD'),
            'SAMPLE_N': os.getenv('SAMPLE_N', '1000'),
        }
    )

    # Task 4: dbt build AI marts
    dbt_build_ai = BashOperator(
        task_id='dbt_build_ai',
        bash_command='cd /opt/airflow/dbt/zomato && dbt build --select tag:ai',
        env={
            'SNOWFLAKE_ACCOUNT': os.getenv('SNOWFLAKE_ACCOUNT'),
            'SNOWFLAKE_USER': os.getenv('SNOWFLAKE_USER'),
            'SNOWFLAKE_PASSWORD': os.getenv('SNOWFLAKE_PASSWORD'),
        }
    )

    # Dependencies
    reload_raw >> dbt_build_core >> enrich_reviews_task >> dbt_build_ai
```

## AI Layer

### LLM Enrichment

```python
# ai/enrich_reviews.py
import os
import json
from openai import OpenAI
import snowflake.connector

# Environment variables
SNOWFLAKE_ACCOUNT = os.getenv('SNOWFLAKE_ACCOUNT')
SNOWFLAKE_USER = os.getenv('SNOWFLAKE_USER')
SNOWFLAKE_PASSWORD = os.getenv('SNOWFLAKE_PASSWORD')
OPENAI_API_KEY = os.getenv('OPENAI_API_KEY')
SAMPLE_N = int(os.getenv('SAMPLE_N', '1000'))

client = OpenAI(api_key=OPENAI_API_KEY)

def get_snowflake_connection():
    return snowflake.connector.connect(
        account=SNOWFLAKE_ACCOUNT,
        user=SNOWFLAKE_USER,
        password=SNOWFLAKE_PASSWORD,
        warehouse='ZOMATO_WH',
        database='ZOMATO',
        schema='RAW',
        role='DBT_ROLE'
    )

def enrich_review(review_text):
    """Use LLM to extract sentiment and topic from review text."""
    prompt = f"""
    Analyze this restaurant review and return JSON with:
    - sentiment: "positive", "negative", or "neutral"
    - topic: main topic like "food_quality", "service", "delivery", "price", "ambiance"
    
    Review: {review_text}
    
    Return only valid JSON, no explanation.
    """
    
    response = client.chat.completions.create(
        model='gpt-4o-mini',
        messages=[
            {'role': 'system', 'content': 'You are a sentiment and topic extraction expert. Return only JSON.'},
            {'role': 'user', 'content': prompt}
        ],
        temperature=0
    )
    
    result = json.loads(response.choices[0].message.content)
    return result['sentiment'], result['topic']

def enrich_reviews():
    """Main enrichment function."""
    conn = get_snowflake_connection()
    cursor = conn.cursor()
    
    # Create enriched table if not exists
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS ZOMATO.AI.REVIEW_ENRICHED (
        review_id NUMBER,
        order_id NUMBER,
        restaurant_id NUMBER,
        user_id NUMBER,
        rating NUMBER,
        review_text VARCHAR,
        review_date DATE,
        sentiment VARCHAR,
        topic VARCHAR,
        enriched_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
    )
    """)
    
    # Get reviews not yet enriched (idempotent)
    cursor.execute(f"""
    SELECT r.review_id, r.order_id, r.restaurant_id, r.user_id, 
           r.rating, r.review_text, r.review_date
    FROM ZOMATO.RAW.REVIEWS r
    LEFT JOIN ZOMATO.AI.REVIEW_ENRICHED e ON r.review_id = e.review_id
    WHERE e.review_id IS NULL
    AND r.review_text IS NOT NULL
    LIMIT {SAMPLE_N}
    """)
    
    reviews = cursor.fetchall()
    print(f"Enriching {len(reviews)} reviews...")
    
    for review in reviews:
        review_id, order_id, restaurant_id, user_id, rating, review_text, review_date = review
        
        try:
            sentiment, topic = enrich_review(review_text)
            
            cursor.execute("""
            INSERT INTO ZOMATO.AI.REVIEW_ENRICHED 
            (review_id, order_id, restaurant_id, user_id, rating, review_text, review_date, sentiment, topic)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
            """, (review_id, order_id, restaurant_id, user_id, rating, review_text, review_date, sentiment, topic))
            
            print(f"✓ Enriched review {review_id}: {sentiment} / {topic}")
        except Exception as e:
            print(f"✗ Failed review {review_id}: {e}")
    
    conn.commit()
    cursor.close()
    conn.close()
    print("Enrichment complete!")

if __name__ == '__main__':
    enrich_reviews()
```

### RAG Chat

```python
# ai/rag_chat.py
import os
import streamlit as st
from openai import OpenAI
import snowflake.connector
import numpy as np

OPENAI_API_KEY = os.getenv('OPENAI_API_KEY')
client = OpenAI(api_key=OPENAI_API_KEY)

def get_snowflake_connection():
    return snowflake.connector.connect(
        account=os.getenv('SNOWFLAKE_ACCOUNT'),
        user=os.getenv('SNOWFLAKE_USER'),
        password=os.getenv('SNOWFLAKE_PASSWORD'),
        warehouse='ZOMATO_WH',
        database='ZOMATO',
        schema='AI',
        role='DBT_ROLE'
    )

def get_embedding(text):
    """Generate embedding for text."""
    response = client.embeddings.create(
        model='text-embedding-3-small',
        input=text
    )
    return response.data[0].embedding

def cosine_similarity(a, b):
    """Calculate cosine similarity between two vectors."""
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

def retrieve_reviews(question, top_k=5):
    """Retrieve most relevant reviews for a question."""
    question_embedding = get_embedding(question)
    
    conn = get_snowflake_connection()
    cursor = conn.cursor()
    
    # Get all enriched reviews
    cursor.execute("""
    SELECT review_id, review_text, sentiment, topic, rating
    FROM REVIEW_ENRICHED
    """)
    
    reviews = cursor.fetchall()
    cursor.close()
    conn.close()
    
    # Calculate similarity for each review
    scored_reviews = []
    for review in reviews:
        review_id, review_text, sentiment, topic, rating = review
        review_embedding = get_embedding(review_text)
        similarity = cosine_similarity(question_embedding, review_embedding)
        scored_reviews.append((similarity, review_id, review_text, sentiment, topic, rating))
    
    # Sort by similarity and return top K
    scored_reviews.sort(reverse=True, key=lambda x: x[0])
    return scored_reviews[:top_k]

def generate_answer(question, context_reviews):
    """Generate answer using retrieved reviews as context."""
    context = "\n\n".join([
        f"Review {i+1} (Rating {r[5]}, {r[3]} / {r[4]}): {r[2]}"
        for i, r in enumerate(context_reviews)
    ])
    
    prompt = f"""
    Based on these restaurant reviews:
    
    {context}
    
    Answer this question: {question}
    
    Provide a concise answer grounded in the reviews above.
    """
    
    response = client.chat.completions.create(
        model='gpt-4o-mini',
        messages=[
            {'role': 'system', 'content': 'You are a helpful assistant that answers questions based on restaurant reviews.'},
            {'role': 'user', 'content': prompt}
        ]
    )
    
    return response.choices[0].message.content

# Streamlit UI
st.title("🍔 Chat with Zomato Reviews (RAG)")
st.write("Ask questions about restaurant reviews — powered by retrieval-augmented generation.")

question = st.text_input("Your question:", placeholder="What do customers say about food quality?")

if st.button("Ask"):
    if question:
        with st.spinner("Retrieving relevant reviews..."):
            relevant_reviews = retrieve_reviews(question, top_k=5)
        
        with st.spinner("Generating answer..."):
            answer = generate_answer(question, relevant_reviews)
        
        st.success("Answer:")
        st.write(answer)
        
        st.subheader("Source Reviews:")
        for i, (score, review_id, review_text, sentiment, topic, rating) in enumerate(relevant_reviews):
            st.write(f"**Review {i+1}** (ID: {review_id}, Similarity: {score:.3f})")
            st.write(f"Rating: {rating} ⭐ | Sentiment: {sentiment} | Topic: {topic}")
            st.write(f"> {review_text}")
            st.write("---")
```

### Text-to-SQL

```python
# ai/text_to_sql.py
import os
import streamlit as st
from openai import OpenAI
import snowflake.connector
import pandas as pd
import re

OPENAI_API_KEY = os.getenv('OPENAI_API_KEY')
client = OpenAI(api_key=OPENAI_API_KEY)

def get_snowflake_connection():
    return snowflake.connector.connect(
        account=os.getenv('SNOWFLAKE_ACCOUNT'),
        user=os.getenv('SNOWFLAKE_USER'),
        password=os.getenv('SNOWFLAKE_PASSWORD'),
        warehouse='ZOMATO_WH',
        database='ZOMATO',
        schema='MARTS',
        role='DBT_ROLE'
    )

def get_schema_info():
    """Get schema information for marts."""
    conn = get_snowflake_connection()
    cursor = conn.cursor()
    
    cursor.execute("""
    SELECT table_name, column_name, data_type
    FROM ZOMATO.INFORMATION_SCHEMA.COLUMNS
    WHERE table_schema = 'MARTS'
    ORDER BY table_name, ordinal_position
    """)
    
    schema = {}
    
