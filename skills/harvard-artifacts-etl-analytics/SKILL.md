---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for the Harvard Art Museums API
  - show me how to create a data engineering project with Harvard artifacts
  - how to use the Harvard artifacts collection analytics app
  - build a streamlit dashboard for art museum data
  - create SQL analytics for Harvard Art Museums API
  - extract and transform Harvard museum artifact data
  - set up ETL pipeline for art museum collections
  - visualize Harvard Art Museums data with plotly
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates building end-to-end data engineering pipelines using the Harvard Art Museums API. It covers API integration, ETL processes, SQL database design, analytics queries, and interactive Streamlit dashboards with Plotly visualizations.

## What It Does

The Harvard Artifacts Collection Data Engineering Analytics App:
- Extracts artifact data from Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into normalized relational database schema
- Loads data into MySQL/TiDB Cloud databases
- Provides 20+ predefined SQL analytics queries
- Visualizes results through interactive Streamlit dashboards

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get a free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file or configure environment variables:

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Schema

The project uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dated VARCHAR(200),
    period VARCHAR(200),
    accession_year INT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    alt_text TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### 1. Extract Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Fetch multiple pages with pagination
def collect_artifacts(total_records=500):
    """Collect artifacts across multiple pages"""
    artifacts = []
    page = 1
    size = 100
    
    while len(artifacts) < total_records:
        data = fetch_artifacts(page=page, size=size)
        artifacts.extend(data.get('records', []))
        
        if not data.get('info', {}).get('next'):
            break
            
        page += 1
    
    return artifacts[:total_records]
```

### 2. Transform Data

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform raw API data into structured DataFrames"""
    
    # Artifact metadata
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'accession_year': artifact.get('accessionyear')
        })
        
        # Extract media/images
        for image in artifact.get('images', []):
            media_records.append({
                'artifact_id': artifact.get('id'),
                'image_url': image.get('baseimageurl'),
                'alt_text': image.get('alttext')
            })
        
        # Extract color data
        for color in artifact.get('colors', []):
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'percentage': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### 3. Load Data to Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Load transformed data into SQL database"""
    
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata
        for _, row in metadata_df.iterrows():
            insert_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             technique, medium, dated, period, accession_year)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
            """
            cursor.execute(insert_query, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            insert_query = """
            INSERT INTO artifactmedia (artifact_id, image_url, alt_text)
            VALUES (%s, %s, %s)
            """
            cursor.execute(insert_query, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            insert_query = """
            INSERT INTO artifactcolors (artifact_id, color, percentage)
            VALUES (%s, %s, %s)
            """
            cursor.execute(insert_query, tuple(row))
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Error loading data: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

## Analytics Queries

### Sample SQL Analytics

```python
# Top 10 cultures by artifact count
query_1 = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Artifacts by century
query_2 = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Department distribution
query_3 = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
GROUP BY department
ORDER BY total_artifacts DESC
"""

# Color analysis
query_4 = """
SELECT c.color, COUNT(DISTINCT c.artifact_id) as artifact_count, 
       AVG(c.percentage) as avg_percentage
FROM artifactcolors c
GROUP BY c.color
ORDER BY artifact_count DESC
LIMIT 15
"""

# Media availability
query_5 = """
SELECT 
    COUNT(DISTINCT m.artifact_id) as with_images,
    (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
    ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / 
          (SELECT COUNT(*) FROM artifactmetadata), 2) as coverage_percent
FROM artifactmedia m
"""
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🎨 Harvard Art Museums Collection Analytics")

# Sidebar configuration
st.sidebar.header("Configuration")
api_key_input = st.sidebar.text_input("Harvard API Key", type="password", 
                                       value=os.getenv('HARVARD_API_KEY', ''))

# ETL Pipeline Section
st.header("1️⃣ ETL Pipeline")

col1, col2 = st.columns(2)

with col1:
    num_records = st.number_input("Number of artifacts to fetch", 
                                   min_value=10, max_value=1000, value=100)
    
if st.button("Run ETL Pipeline"):
    with st.spinner("Fetching artifacts..."):
        artifacts = collect_artifacts(num_records)
        st.success(f"✅ Fetched {len(artifacts)} artifacts")
    
    with st.spinner("Transforming data..."):
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        st.success("✅ Data transformed")
    
    with st.spinner("Loading to database..."):
        load_to_database(metadata_df, media_df, colors_df)
        st.success("✅ Data loaded to database")

# Analytics Section
st.header("2️⃣ SQL Analytics")

queries = {
    "Top Cultures": query_1,
    "Artifacts by Century": query_2,
    "Department Distribution": query_3,
    "Color Analysis": query_4,
    "Media Availability": query_5
}

selected_query = st.selectbox("Select Analysis", list(queries.keys()))

if st.button("Run Query"):
    conn = get_db_connection()
    result_df = pd.read_sql(queries[selected_query], conn)
    conn.close()
    
    st.dataframe(result_df)
    
    # Auto-generate visualization
    if len(result_df.columns) >= 2:
        fig = px.bar(result_df, 
                     x=result_df.columns[0], 
                     y=result_df.columns[1],
                     title=selected_query)
        st.plotly_chart(fig, use_container_width=True)
```

### Visualization Components

```python
def create_culture_chart(df):
    """Create interactive culture distribution chart"""
    fig = px.bar(df, 
                 x='culture', 
                 y='artifact_count',
                 title='Top 10 Cultures by Artifact Count',
                 labels={'culture': 'Culture', 'artifact_count': 'Number of Artifacts'},
                 color='artifact_count',
                 color_continuous_scale='viridis')
    return fig

def create_color_pie_chart(df):
    """Create color distribution pie chart"""
    fig = px.pie(df, 
                 values='artifact_count', 
                 names='color',
                 title='Color Distribution Across Artifacts')
    return fig

# Usage in Streamlit
st.plotly_chart(create_culture_chart(culture_df), use_container_width=True)
```

## Common Patterns

### Batch Processing for Large Datasets

```python
def batch_insert(cursor, table, df, batch_size=1000):
    """Insert data in batches for better performance"""
    total_rows = len(df)
    
    for i in range(0, total_rows, batch_size):
        batch = df.iloc[i:i+batch_size]
        # Construct insert statement
        placeholders = ', '.join(['%s'] * len(batch.columns))
        columns = ', '.join(batch.columns)
        query = f"INSERT INTO {table} ({columns}) VALUES ({placeholders})"
        
        cursor.executemany(query, batch.values.tolist())
        print(f"Inserted {min(i+batch_size, total_rows)}/{total_rows} rows")
```

### Error Handling and Retry Logic

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(url, params, max_retries=3, delay=2):
    """Fetch data with retry logic"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except RequestException as e:
            if attempt < max_retries - 1:
                time.sleep(delay * (attempt + 1))
                continue
            else:
                raise Exception(f"Failed after {max_retries} attempts: {e}")
```

## Troubleshooting

### API Rate Limiting

```python
import time

def rate_limited_fetch(pages, delay=1):
    """Fetch with rate limiting to avoid API throttling"""
    results = []
    for page in range(1, pages + 1):
        data = fetch_artifacts(page=page)
        results.extend(data.get('records', []))
        time.sleep(delay)  # Wait between requests
    return results
```

### Database Connection Issues

```python
def safe_db_operation(operation_func):
    """Wrapper for safe database operations"""
    conn = None
    try:
        conn = get_db_connection()
        result = operation_func(conn)
        return result
    except Error as e:
        st.error(f"Database error: {e}")
        return None
    finally:
        if conn and conn.is_connected():
            conn.close()
```

### Missing Data Handling

```python
def safe_extract(artifact, key, default=None):
    """Safely extract data with fallback"""
    return artifact.get(key, default) or default

# Usage
metadata = {
    'id': artifact.get('id'),
    'title': safe_extract(artifact, 'title', 'Unknown'),
    'culture': safe_extract(artifact, 'culture', 'Unknown Culture'),
    'century': safe_extract(artifact, 'century', 'Unknown Century')
}
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Run with custom port
streamlit run app.py --server.port 8501

# Run ETL pipeline only (if separated)
python etl_pipeline.py
```

This skill enables AI agents to help developers build complete ETL pipelines for museum artifact data, implement SQL analytics, and create interactive dashboards using modern Python data engineering tools.
