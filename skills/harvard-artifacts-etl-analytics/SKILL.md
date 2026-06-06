---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums API
  - create a data engineering app with museum artifact data
  - set up Harvard artifacts collection analytics dashboard
  - build streamlit app for art museum data visualization
  - extract and analyze Harvard Art Museums API data
  - create SQL analytics for art collection metadata
  - implement ETL for museum artifact data with Python
  - visualize Harvard art collection data with Plotly
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App:
- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational database tables
- **Loads** data into MySQL/TiDB Cloud using batch inserts
- **Analyzes** data using 20+ predefined SQL queries
- **Visualizes** results through interactive Streamlit dashboards with Plotly charts

Architecture: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required Dependencies:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Get your free API key from: https://www.harvardartmuseums.org/collections/api

Store in environment variable:
```bash
export HARVARD_API_KEY="your_api_key_here"
```

Or use `.env` file:
```env
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

Set up MySQL or TiDB Cloud connection:
```env
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### 3. Database Schema Setup

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    accession_year INT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    url TEXT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    image_height INT,
    image_width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. ETL Pipeline Implementation

**Extract from API:**
```python
import requests
import os
from time import sleep

def fetch_artifacts(api_key, num_records=100, page_size=100):
    """Extract artifact data from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    
    params = {
        'apikey': api_key,
        'size': page_size,
        'page': 1
    }
    
    total_fetched = 0
    while total_fetched < num_records:
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            artifacts.extend(records)
            total_fetched += len(records)
            
            # Rate limiting
            sleep(0.5)
            
            if not data.get('info', {}).get('next'):
                break
                
            params['page'] += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

**Transform Data:**
```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform raw JSON into structured DataFrames"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for item in raw_data:
        # Extract metadata
        metadata = {
            'id': item.get('id'),
            'title': item.get('title'),
            'culture': item.get('culture'),
            'century': item.get('century'),
            'classification': item.get('classification'),
            'department': item.get('department'),
            'division': item.get('division'),
            'dated': item.get('dated'),
            'accession_year': item.get('accessionyear'),
            'technique': item.get('technique'),
            'medium': item.get('medium'),
            'url': item.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        if item.get('images'):
            for img in item.get('images', []):
                media = {
                    'artifact_id': item.get('id'),
                    'image_url': img.get('baseimageurl'),
                    'image_height': img.get('height'),
                    'image_width': img.get('width')
                }
                media_list.append(media)
        
        # Extract color data
        if item.get('colors'):
            for color in item.get('colors', []):
                color_data = {
                    'artifact_id': item.get('id'),
                    'color_hex': color.get('hex'),
                    'color_percent': color.get('percent')
                }
                colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

**Load to Database:**
```python
import mysql.connector
from mysql.connector import Error

def load_to_database(df_metadata, df_media, df_colors, db_config):
    """Load transformed data into MySQL database"""
    try:
        connection = mysql.connector.connect(
            host=db_config['host'],
            port=db_config['port'],
            user=db_config['user'],
            password=db_config['password'],
            database=db_config['database']
        )
        
        cursor = connection.cursor()
        
        # Insert metadata (batch insert for performance)
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             division, dated, accession_year, technique, medium, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        metadata_values = df_metadata.values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, image_url, image_height, image_width)
            VALUES (%s, %s, %s, %s)
        """
        media_values = df_media.values.tolist()
        cursor.executemany(media_query, media_values)
        
        # Insert colors
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color_hex, color_percent)
            VALUES (%s, %s, %s)
        """
        colors_values = df_colors.values.tolist()
        cursor.executemany(colors_query, colors_values)
        
        connection.commit()
        print(f"Inserted {len(metadata_values)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 2. SQL Analytics Queries

**Example analytical queries:**
```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN EXISTS (
                SELECT 1 FROM artifactmedia 
                WHERE artifactmedia.artifact_id = artifactmetadata.id
            ) THEN 'Has Images' ELSE 'No Images' END as media_status,
            COUNT(*) as count
        FROM artifactmetadata
        GROUP BY media_status
    """,
    
    "Top Colors Used": """
        SELECT color_hex, 
               COUNT(*) as usage_count,
               AVG(color_percent) as avg_percent
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts with Most Images": """
        SELECT m.title, m.culture, COUNT(a.media_id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia a ON m.id = a.artifact_id
        GROUP BY m.id, m.title, m.culture
        ORDER BY image_count DESC
        LIMIT 10
    """
}
```

### 3. Streamlit Dashboard

**Main app structure:**
```python
import streamlit as st
import plotly.express as px
import pandas as pd
from dotenv import load_dotenv
import os

load_dotenv()

st.set_page_config(
    page_title="Harvard Artifacts Analytics",
    page_icon="🏛️",
    layout="wide"
)

st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")

# Sidebar configuration
st.sidebar.header("Configuration")

# API Key input
api_key = st.sidebar.text_input(
    "Harvard API Key",
    type="password",
    value=os.getenv("HARVARD_API_KEY", "")
)

# Database configuration
db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts')
}

# ETL Section
st.header("📥 Data Collection & ETL")

num_records = st.number_input("Number of artifacts to fetch", 
                               min_value=10, max_value=1000, 
                               value=100, step=10)

if st.button("Run ETL Pipeline"):
    with st.spinner("Fetching artifacts..."):
        artifacts = fetch_artifacts(api_key, num_records)
        st.success(f"Fetched {len(artifacts)} artifacts")
    
    with st.spinner("Transforming data..."):
        df_meta, df_media, df_colors = transform_artifacts(artifacts)
        st.success("Data transformed")
    
    with st.spinner("Loading to database..."):
        load_to_database(df_meta, df_media, df_colors, db_config)
        st.success("Data loaded to database")

# Analytics Section
st.header("📊 SQL Analytics")

query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))

if st.button("Run Query"):
    try:
        connection = mysql.connector.connect(**db_config)
        df_result = pd.read_sql(ANALYTICS_QUERIES[query_name], connection)
        connection.close()
        
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, 
                        x=df_result.columns[0], 
                        y=df_result.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)
            
    except Exception as e:
        st.error(f"Query error: {e}")
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY="your_api_key"
export DB_HOST="your_host"
export DB_USER="your_user"
export DB_PASSWORD="your_password"

# Run Streamlit app
streamlit run app.py
```

## Common Patterns

### Pattern 1: Incremental ETL
```python
def incremental_etl(api_key, db_config, last_id=0):
    """Fetch only new artifacts since last run"""
    params = {
        'apikey': api_key,
        'after': last_id,
        'size': 100
    }
    # Continue ETL process...
```

### Pattern 2: Error Handling with Retry
```python
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def get_session_with_retry():
    session = requests.Session()
    retry = Retry(total=3, backoff_factor=1, 
                  status_forcelist=[429, 500, 502, 503, 504])
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('https://', adapter)
    return session
```

### Pattern 3: Data Quality Validation
```python
def validate_artifacts(df):
    """Validate data quality before loading"""
    # Check for required fields
    required_fields = ['id', 'title']
    missing = [f for f in required_fields if f not in df.columns]
    
    if missing:
        raise ValueError(f"Missing required fields: {missing}")
    
    # Remove duplicates
    df = df.drop_duplicates(subset=['id'])
    
    # Handle nulls
    df['title'] = df['title'].fillna('Untitled')
    
    return df
```

## Troubleshooting

**API Rate Limiting:**
```python
# Add exponential backoff
import time

def fetch_with_backoff(url, params, max_retries=3):
    for i in range(max_retries):
        response = requests.get(url, params=params)
        if response.status_code == 429:
            wait_time = 2 ** i
            time.sleep(wait_time)
            continue
        return response
    raise Exception("Max retries exceeded")
```

**Database Connection Errors:**
```python
# Use connection pooling
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    **db_config
)

connection = db_pool.get_connection()
```

**Memory Issues with Large Datasets:**
```python
# Process in chunks
def process_in_chunks(artifacts, chunk_size=100):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        df_meta, df_media, df_colors = transform_artifacts(chunk)
        load_to_database(df_meta, df_media, df_colors, db_config)
```

**Streamlit Performance:**
```python
# Cache database connections
@st.cache_resource
def get_db_connection():
    return mysql.connector.connect(**db_config)

# Cache query results
@st.cache_data(ttl=3600)
def run_cached_query(query):
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    return df
```
