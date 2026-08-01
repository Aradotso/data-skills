---
name: zomato-ai-data-engineering-pipeline
description: End-to-end batch data pipeline with Snowflake, dbt, Airflow, and OpenAI for food delivery analytics
triggers:
  - "set up zomato data pipeline"
  - "configure snowflake s3 integration for zomato"
  - "run dbt models for zomato project"
  - "orchestrate zomato pipeline with airflow"
  - "enrich reviews with openai"
  - "implement text to sql for snowflake"
  - "build medallion architecture with dbt"
  - "create incremental facts in dbt"
---

# Zomato AI Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A complete batch data pipeline demonstrating medallion architecture (Bronze → Silver → Gold) using Snowflake as the warehouse, dbt for transformations, Airflow for orchestration, and OpenAI for AI-powered analytics including LLM enrichment, RAG, and text-to-SQL.

## Architecture Overview

**Data Flow:** CSV → S3 → Snowflake (Bronze) → dbt Staging (Silver) → dbt Marts (Gold) → AI Layer

**Layers:**
- **Bronze (RAW):** Direct `COPY INTO` from S3 via storage integration
- **Silver (STAGING):** dbt views for cleaning and standardization
- **Gold (MARTS):** Dimensions, incremental facts, business aggregates
- **AI:** LLM-enriched reviews, RAG, text-to-SQL

## Prerequisites

```bash
# Required accounts/services
- AWS account (S3 bucket)
- Snowflake account
- OpenAI API key
- Docker (for Airflow)

# Python dependencies
pip install dbt-snowflake apache-airflow openai pandas streamlit
```

## Project Structure

```
zomato-ai-data-engineering/
├── airflow/
│   ├── docker-compose.yaml
│   ├── Dockerfile
│   └── dags/zomato_batch.py
├── zomato/                    # dbt project
│   ├── dbt_project.yml
│   ├── profiles.yml
│   └── models/
│       ├── staging/
│       └── marts/
├── ai/
│   ├── enrich_reviews.py
│   ├── rag_chat.py
│   └── text_to_sql.py
└── aws/iam/                   # IAM policies for S3-Snowflake
```

## Setup Steps

### 1. Snowflake Objects

```sql
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
GRANT CREATE TABLE ON SCHEMA ZOMATO.RAW TO ROLE DBT_ROLE;
GRANT CREATE VIEW ON SCHEMA ZOMATO.STAGING TO ROLE DBT_ROLE;
GRANT CREATE TABLE ON SCHEMA ZOMATO.MARTS TO ROLE DBT_ROLE;
GRANT CREATE TABLE ON SCHEMA ZOMATO.AI TO ROLE DBT_ROLE;
```

### 2. S3-Snowflake Storage Integration

**Step 1:** Create IAM policy for S3 read access:

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
        "arn:aws:s3:::YOUR-BUCKET-NAME/*",
        "arn:aws:s3:::YOUR-BUCKET-NAME"
      ]
    }
  ]
}
```

**Step 2:** Create IAM role with initial trust policy:

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

**Step 3:** Create Snowflake storage integration:

```sql
CREATE STORAGE INTEGRATION s3_zomato_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'S3'
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::YOUR-ACCOUNT:role/snowflake-s3-role'
  STORAGE_ALLOWED_LOCATIONS = ('s3://YOUR-BUCKET-NAME/raw/');

-- Get Snowflake's IAM user and external ID
DESC STORAGE INTEGRATION s3_zomato_integration;
-- Note: STORAGE_AWS_IAM_USER_ARN and STORAGE_AWS_EXTERNAL_ID
```

**Step 4:** Update IAM role trust policy with Snowflake's values:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/abc123-s"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "ABC12345_SFCRole=1_xyz="
        }
      }
    }
  ]
}
```

**Step 5:** Create external stages for each table:

```sql
CREATE STAGE ZOMATO.RAW.s3_restaurants
  URL = 's3://YOUR-BUCKET-NAME/raw/restaurants/'
  STORAGE_INTEGRATION = s3_zomato_integration
  FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1);

CREATE STAGE ZOMATO.RAW.s3_users
  URL = 's3://YOUR-BUCKET-NAME/raw/users/'
  STORAGE_INTEGRATION = s3_zomato_integration
  FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1);

-- Repeat for: food, menu, orders, order_items, reviews
```

### 3. dbt Configuration

**profiles.yml:**

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

**dbt_project.yml:**

```yaml
name: 'zomato'
version: '1.0.0'
config-version: 2

profile: 'zomato'

model-paths: ["models"]
test-paths: ["tests"]
macro-paths: ["macros"]

models:
  zomato:
    staging:
      +materialized: view
      +schema: staging
    marts:
      +materialized: table
      +schema: marts
```

### 4. Creating RAW Tables

```sql
-- Example: orders table
CREATE TABLE ZOMATO.RAW.ORDERS (
    order_id NUMBER,
    user_id NUMBER,
    restaurant_id NUMBER,
    order_date TIMESTAMP,
    delivery_date TIMESTAMP,
    order_value NUMBER(10,2),
    delivery_fee NUMBER(10,2),
    tip_amount NUMBER(10,2),
    order_status VARCHAR,
    delivery_time_minutes NUMBER,
    city VARCHAR,
    payment_method VARCHAR
);

-- COPY INTO example
COPY INTO ZOMATO.RAW.ORDERS
FROM @ZOMATO.RAW.s3_orders
FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
ON_ERROR = 'ABORT_STATEMENT';
```

## dbt Models

### Staging Layer (Silver)

**models/staging/stg_restaurants.sql:**

```sql
WITH source AS (
    SELECT * FROM {{ source('zomato_raw', 'restaurants') }}
),

cleaned AS (
    SELECT
        restaurant_id,
        restaurant_name,
        CASE 
            WHEN cuisine = '--' THEN NULL 
            ELSE cuisine 
        END AS cuisine,
        city,
        CAST(REPLACE(REPLACE(avg_cost_for_two, '₹', ''), ' ', '') AS NUMBER) AS avg_cost_for_two,
        CASE 
            WHEN rating = '--' THEN NULL 
            ELSE CAST(rating AS NUMBER(2,1))
        END AS rating,
        CAST(votes AS NUMBER) AS votes,
        delivery_available::BOOLEAN AS delivery_available
    FROM source
)

SELECT * FROM cleaned
```

**models/staging/stg_orders.sql:**

```sql
WITH source AS (
    SELECT * FROM {{ source('zomato_raw', 'orders') }}
),

cleaned AS (
    SELECT
        order_id,
        user_id,
        restaurant_id,
        order_date::TIMESTAMP AS order_date,
        delivery_date::TIMESTAMP AS delivery_date,
        order_value,
        delivery_fee,
        tip_amount,
        LOWER(TRIM(order_status)) AS order_status,
        delivery_time_minutes,
        LOWER(TRIM(city)) AS city,
        LOWER(TRIM(payment_method)) AS payment_method,
        CASE 
            WHEN LOWER(order_status) = 'delivered' THEN TRUE 
            ELSE FALSE 
        END AS is_delivered
    FROM source
)

SELECT * FROM cleaned
```

**models/staging/sources.yml:**

```yaml
version: 2

sources:
  - name: zomato_raw
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

### Marts Layer (Gold)

**models/marts/dim_restaurants.sql:**

```sql
SELECT
    restaurant_id,
    restaurant_name,
    cuisine,
    city,
    avg_cost_for_two,
    rating,
    votes,
    delivery_available
FROM {{ ref('stg_restaurants') }}
```

**models/marts/fct_orders.sql (Incremental):**

```sql
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
        user_id,
        restaurant_id,
        order_date,
        delivery_date,
        order_value,
        delivery_fee,
        tip_amount,
        order_status,
        delivery_time_minutes,
        city,
        payment_method,
        is_delivered,
        order_value + delivery_fee + tip_amount AS total_amount
    FROM {{ ref('stg_orders') }}
    {% if is_incremental() %}
    WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
    {% endif %}
)

SELECT * FROM orders
```

**models/marts/mart_daily_city_revenue.sql:**

```sql
WITH orders AS (
    SELECT * FROM {{ ref('fct_orders') }}
),

daily_metrics AS (
    SELECT
        DATE(order_date) AS order_date,
        city,
        COUNT(*) AS total_orders,
        SUM(CASE WHEN is_delivered THEN 1 ELSE 0 END) AS delivered_orders,
        SUM(order_value) AS gmv,
        SUM(delivery_fee) AS total_delivery_fees,
        SUM(tip_amount) AS total_tips,
        AVG(order_value) AS avg_order_value,
        AVG(delivery_time_minutes) AS avg_delivery_time,
        ROUND(100.0 * SUM(CASE WHEN order_status = 'cancelled' THEN 1 ELSE 0 END) / COUNT(*), 2) AS cancel_rate_pct
    FROM orders
    GROUP BY 1, 2
)

SELECT * FROM daily_metrics
ORDER BY order_date DESC, city
```

**models/marts/mart_delivery_sla.sql (Percentiles):**

```sql
WITH orders AS (
    SELECT
        city,
        HOUR(order_date) AS order_hour,
        delivery_time_minutes
    FROM {{ ref('fct_orders') }}
    WHERE is_delivered = TRUE
)

SELECT
    city,
    order_hour,
    COUNT(*) AS total_deliveries,
    AVG(delivery_time_minutes) AS avg_delivery_time,
    PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY delivery_time_minutes) AS p50_delivery_time,
    PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY delivery_time_minutes) AS p90_delivery_time,
    MIN(delivery_time_minutes) AS min_delivery_time,
    MAX(delivery_time_minutes) AS max_delivery_time
FROM orders
GROUP BY city, order_hour
ORDER BY city, order_hour
```

### Schema Tests

**models/marts/schema.yml:**

```yaml
version: 2

models:
  - name: fct_orders
    columns:
      - name: order_id
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
          - relationships:
              to: ref('dim_restaurants')
              field: restaurant_id
      - name: order_status
        tests:
          - accepted_values:
              values: ['delivered', 'cancelled', 'pending']

  - name: dim_restaurants
    columns:
      - name: restaurant_id
        tests:
          - unique
          - not_null
```

### Custom Schema Macro

**macros/generate_schema_name.sql:**

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

## Airflow Orchestration

### Docker Setup

**airflow/docker-compose.yaml:**

```yaml
version: '3.8'

x-airflow-common:
  &airflow-common
  build: .
  environment:
    &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: LocalExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
    AIRFLOW__CORE__FERNET_KEY: ${AIRFLOW_FERNET_KEY}
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
    AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    SNOWFLAKE_ACCOUNT: ${SNOWFLAKE_ACCOUNT}
    SNOWFLAKE_USER: ${SNOWFLAKE_USER}
    SNOWFLAKE_PASSWORD: ${SNOWFLAKE_PASSWORD}
    OPENAI_API_KEY: ${OPENAI_API_KEY}
    SAMPLE_N: ${SAMPLE_N:-100}
    AIRFLOW_CONN_SNOWFLAKE_DEFAULT: snowflake://${SNOWFLAKE_USER}:${SNOWFLAKE_PASSWORD}@${SNOWFLAKE_ACCOUNT}
  volumes:
    - ./dags:/opt/airflow/dags
    - ./logs:/opt/airflow/logs
    - ../zomato:/opt/dbt/zomato

services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow

  airflow-init:
    <<: *airflow-common
    command: >
      bash -c "airflow db init &&
               airflow users create --username admin --password admin --firstname Admin --lastname User --role Admin --email admin@example.com"

  airflow-webserver:
    <<: *airflow-common
    command: webserver
    ports:
      - "8080:8080"
    depends_on:
      - postgres

  airflow-scheduler:
    <<: *airflow-common
    command: scheduler
    depends_on:
      - postgres
```

**airflow/Dockerfile:**

```dockerfile
FROM apache-airflow:3.0.0-python3.11

USER root
RUN apt-get update && apt-get install -y git && apt-get clean

USER airflow

# Airflow providers
RUN pip install --no-cache-dir \
    apache-airflow-providers-snowflake \
    apache-airflow-providers-openai

# dbt in its own venv
RUN python -m venv /home/airflow/dbt_venv && \
    /home/airflow/dbt_venv/bin/pip install dbt-snowflake

ENV PATH="/home/airflow/dbt_venv/bin:$PATH"
```

**airflow/.env (example):**

```bash
SNOWFLAKE_ACCOUNT=abc12345.us-east-1
SNOWFLAKE_USER=DBT_USER
SNOWFLAKE_PASSWORD=your_password_here
OPENAI_API_KEY=sk-proj-...
SAMPLE_N=100
AIRFLOW_FERNET_KEY=your_fernet_key_here
```

### DAG Implementation

**airflow/dags/zomato_batch.py:**

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator

default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    'zomato_batch',
    default_args=default_args,
    description='Zomato end-to-end batch pipeline',
    schedule_interval='@daily',
    catchup=False,
)

# Task 1: Reload RAW tables from S3
reload_raw = SnowflakeOperator(
    task_id='reload_raw',
    snowflake_conn_id='snowflake_default',
    sql="""
    USE WAREHOUSE ZOMATO_WH;
    USE DATABASE ZOMATO;
    USE SCHEMA RAW;
    
    -- Truncate and reload
    TRUNCATE TABLE RESTAURANTS;
    COPY INTO RESTAURANTS FROM @s3_restaurants FILE_FORMAT = (TYPE='CSV' FIELD_OPTIONALLY_ENCLOSED_BY='"' SKIP_HEADER=1);
    
    TRUNCATE TABLE USERS;
    COPY INTO USERS FROM @s3_users FILE_FORMAT = (TYPE='CSV' FIELD_OPTIONALLY_ENCLOSED_BY='"' SKIP_HEADER=1);
    
    TRUNCATE TABLE FOOD;
    COPY INTO FOOD FROM @s3_food FILE_FORMAT = (TYPE='CSV' FIELD_OPTIONALLY_ENCLOSED_BY='"' SKIP_HEADER=1);
    
    TRUNCATE TABLE MENU;
    COPY INTO MENU FROM @s3_menu FILE_FORMAT = (TYPE='CSV' FIELD_OPTIONALLY_ENCLOSED_BY='"' SKIP_HEADER=1);
    
    TRUNCATE TABLE ORDERS;
    COPY INTO ORDERS FROM @s3_orders FILE_FORMAT = (TYPE='CSV' FIELD_OPTIONALLY_ENCLOSED_BY='"' SKIP_HEADER=1);
    
    TRUNCATE TABLE ORDER_ITEMS;
    COPY INTO ORDER_ITEMS FROM @s3_order_items FILE_FORMAT = (TYPE='CSV' FIELD_OPTIONALLY_ENCLOSED_BY='"' SKIP_HEADER=1);
    
    TRUNCATE TABLE REVIEWS;
    COPY INTO REVIEWS FROM @s3_reviews FILE_FORMAT = (TYPE='CSV' FIELD_OPTIONALLY_ENCLOSED_BY='"' SKIP_HEADER=1);
    """,
    dag=dag,
)

# Task 2: dbt build (staging + marts, exclude AI models)
dbt_build_core = BashOperator(
    task_id='dbt_build_core',
    bash_command='cd /opt/dbt/zomato && dbt build --exclude tag:ai',
    dag=dag,
)

# Task 3: AI enrichment
def run_enrich_reviews():
    import sys
    sys.path.append('/opt/airflow')
    from ai.enrich_reviews import main
    main()

enrich_reviews = PythonOperator(
    task_id='enrich_reviews',
    python_callable=run_enrich_reviews,
    dag=dag,
)

# Task 4: dbt build AI models
dbt_build_ai = BashOperator(
    task_id='dbt_build_ai',
    bash_command='cd /opt/dbt/zomato && dbt build --select tag:ai',
    dag=dag,
)

# Dependencies
reload_raw >> dbt_build_core >> enrich_reviews >> dbt_build_ai
```

## AI Layer

### 1. LLM Enrichment

**ai/enrich_reviews.py:**

```python
import os
import snowflake.connector
import openai
import json
from typing import Dict, List

# Configuration
SNOWFLAKE_ACCOUNT = os.getenv('SNOWFLAKE_ACCOUNT')
SNOWFLAKE_USER = os.getenv('SNOWFLAKE_USER')
SNOWFLAKE_PASSWORD = os.getenv('SNOWFLAKE_PASSWORD')
OPENAI_API_KEY = os.getenv('OPENAI_API_KEY')
SAMPLE_N = int(os.getenv('SAMPLE_N', 100))

openai.api_key = OPENAI_API_KEY

def get_snowflake_connection():
    return snowflake.connector.connect(
        user=SNOWFLAKE_USER,
        password=SNOWFLAKE_PASSWORD,
        account=SNOWFLAKE_ACCOUNT,
        warehouse='ZOMATO_WH',
        database='ZOMATO',
        schema='RAW'
    )

def fetch_reviews_to_enrich(conn, sample_n: int) -> List[Dict]:
    """Fetch reviews not yet enriched."""
    cursor = conn.cursor()
    
    # Create enriched table if not exists
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS ZOMATO.AI.REVIEW_ENRICHED (
            review_id NUMBER,
            review_text VARCHAR,
            sentiment VARCHAR,
            topic VARCHAR,
            enriched_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
            PRIMARY KEY (review_id)
        )
    """)
    
    # Fetch unenriched reviews
    query = f"""
    SELECT r.review_id, r.review_text
    FROM ZOMATO.RAW.REVIEWS r
    LEFT JOIN ZOMATO.AI.REVIEW_ENRICHED e ON r.review_id = e.review_id
    WHERE e.review_id IS NULL
    LIMIT {sample_n}
    """
    
    cursor.execute(query)
    reviews = [{'review_id': row[0], 'review_text': row[1]} for row in cursor.fetchall()]
    cursor.close()
    return reviews

def enrich_with_llm(review_text: str) -> Dict:
    """Call OpenAI to extract sentiment and topic."""
    prompt = f"""Analyze this restaurant review and return a JSON object with two fields:
- sentiment: one of [positive, negative, neutral]
- topic: one of [food_quality, service, delivery_speed, price, packaging, other]

Review: "{review_text}"

Return only valid JSON, no markdown or explanation."""

    try:
        response = openai.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "You are a review analyzer. Always return valid JSON."},
                {"role": "user", "content": prompt}
            ],
            temperature=0,
            max_tokens=100
        )
        
        result = json.loads(response.choices[0].message.content.strip())
        return {
            'sentiment': result.get('sentiment', 'neutral'),
            'topic': result.get('topic', 'other')
        }
    except Exception as e:
        print(f"Error enriching review: {e}")
        return {'sentiment': 'neutral', 'topic': 'other'}

def write_enriched_reviews(conn, enriched: List[Dict]):
    """Write enriched reviews back to Snowflake."""
    cursor = conn.cursor()
    
    for item in enriched:
        cursor.execute("""
            INSERT INTO ZOMATO.AI.REVIEW_ENRICHED (review_id, review_text, sentiment, topic)
            VALUES (%s, %s, %s, %s)
        """, (
            item['review_id'],
            item['review_text'],
            item['sentiment'],
            item['topic']
        ))
    
    conn.commit()
    cursor.close()

def main():
    print(f"Starting review enrichment (sample_n={SAMPLE_N})...")
    conn = get_snowflake_connection()
    
    # Fetch reviews
    reviews = fetch_reviews_to_enrich(conn, SAMPLE_N)
    print(f"Found {len(reviews)} reviews to enrich")
    
    if not reviews:
        print("No reviews to enrich. Exiting.")
        return
    
    # Enrich with LLM
    enriched = []
    for idx, review in enumerate(reviews, 1):
        print(f"Enriching review {idx}/{len(reviews)}...")
        enrichment = enrich_with_llm(review['review_text'])
        enriched.append({
            'review_id': review['review_id'],
            'review_text': review['review_text'],
            'sentiment': enrichment['sentiment'],
            'topic': enrichment['topic']
        })
    
    # Write back
    write_enriched_reviews(conn, enriched)
    print(f"Successfully enriched {len(enriched)} reviews")
    
    conn.close()

if __name__ == '__main__':
    main()
```

**dbt model using enriched data:**

**models/marts/mart_review_insights.sql:**

```sql
{{ config(tags=['ai']) }}

WITH enriched AS (
    SELECT * FROM {{ source('zomato_ai', 'review_enriched') }}
),

aggregated AS (
    SELECT
        sentiment,
        topic,
        COUNT(*) AS review_count,
        COUNT(*) * 100.0 / SUM(COUNT(*)) OVER () AS pct_of_total
    FROM enriched
    GROUP BY sentiment, topic
)

SELECT * FROM aggregated
ORDER BY review_count DESC
```

### 2. RAG (Chat with Reviews)

**ai/rag_chat.py:**

```python
import os
import streamlit as st
import snowflake.connector
import openai
from typing import List, Dict

SNOWFLAKE_ACCOUNT = os.getenv('SNOWFLAKE_ACCOUNT')
SNOWFLAKE_USER = os.getenv('SNOWFLAKE_USER')
SNOWFLAKE_PASSWORD = os.getenv('SNOWFLAKE_PASSWORD')
OPENAI_API_KEY = os.getenv('OPENAI_API_KEY')

openai.api_key = OPENAI_API_KEY

@st.cache_resource
def get_snowflake_connection():
    return snowflake.connector.connect(
        user=SNOWFLAKE_USER,
        password=SNOWFLAKE_PASSWORD,
        account=SNOWFLAKE_ACCOUNT,
        warehouse='ZOMATO_WH',
        database='ZOMATO',
        schema='AI'
    )

def create_embedding(text: str) -> List[float]:
    """Create embedding using OpenAI."""
    response = openai.embeddings.create(
        input=text,
        model="text-embedding-3-small"
    )
    return response.data[0].embedding

def setup_vector_table(conn):
    """Create table with embeddings if not exists."""
    cursor = conn.cursor()
    
    # Create table with VECTOR type (Snowflake Cortex)
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS REVIEW_EMBEDDINGS (
            review_id NUMBER,
            review_text VARCHAR,
            sentiment VARCHAR,
            topic VARCHAR,
            embedding VECTOR(FLOAT, 1536),
            PRIMARY KEY (review_id)
        )
    """)
    
    # Check if embeddings need to be generated
    cursor.execute("SELECT COUNT(*) FROM REVIEW_EMBEDDINGS")
    count = cursor.fetchone()[0]
    
    if count == 0:
        st.info("Generating embeddings for reviews... This may take a minute.")
        cursor.execute("""
            SELECT review_id, review_text, sentiment, topic 
            FROM REVIEW_ENRICHED 
            LIMIT 1000
        """)
        reviews = cursor.fetchall()
        
        for review in reviews:
            review_id, review_text, sentiment, topic = review
            embedding = create_embedding(review_text)
            
            cursor.execute("""
                INSERT INTO REVIEW_EMBEDDINGS (review_id, review_text, sentiment, topic, embedding)
                VALUES (%s, %s, %s, %s, %s)
            """, (review_id, review_text, sentiment, topic, str(embedding)))
        
        conn.commit()
        st.success(f"Generated embeddings for {len(reviews)} reviews")
    
    cursor.close()

def retrieve_similar_reviews(conn, question: str, top_k: int = 5) -> List[Dict]:
    """Retrieve most similar reviews using cosine similarity."""
    question_embedding = create_embedding(question)
    
    cursor = conn.cursor()
    
    # Use Snowflake's vector similarity function
    cursor.execute("""
        SELECT 
            review_id,
            review_text,
            sentiment,
            topic,
            VECTOR_COSINE_SIMILARITY(embedding, PARSE_JSON(%s)::VECTOR(FLOAT, 1536)) AS similarity
        FROM REVIEW
