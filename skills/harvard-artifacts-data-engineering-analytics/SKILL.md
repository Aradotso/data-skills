---
name: harvard-artifacts-data-engineering-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit and SQL
  - process Harvard museum collection data
  - set up artifact data engineering pipeline
  - visualize museum artifacts with Plotly
  - query Harvard Art Museums database
  - implement batch data loading for artifacts
---

# Harvard Artifacts Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates an end-to-end data engineering pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational structures, loads it into SQL databases, and provides interactive analytics dashboards using Streamlit. It showcases real-world ETL patterns, SQL analytics, and data visualization techniques.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Run the Streamlit app
streamlit run app.py
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

Obtain a free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Store credentials in environment variables or a `.env` file:

```bash
HARVARD_API_KEY=your_api_key_here
MYSQL_HOST=your_database_host
MYSQL_USER=your_username
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=artifacts_db
```

### Database Setup

Create the required database and tables:

```sql
CREATE DATABASE IF NOT EXISTS artifacts_db;
USE artifacts_db;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    url TEXT
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri TEXT,
    baseimageurl TEXT,
    primaryurl TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(7),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination.
    
    Args:
        api_key: Harvard API key
        page: Page number (default: 1)
        size: Number of records per page (max: 100)
    
    Returns:
        dict: JSON response containing artifact records
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Pages available: {data['info']['pages']}")
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class ArtifactETL:
    def __init__(self, db_config: Dict[str, str]):
        self.db_config = db_config
        
    def extract(self, api_key: str, num_pages: int = 5) -> List[Dict]:
        """Extract artifact data from API"""
        all_artifacts = []
        
        for page in range(1, num_pages + 1):
            data = fetch_artifacts(api_key, page=page, size=100)
            all_artifacts.extend(data['records'])
            print(f"Extracted page {page}/{num_pages}")
            
        return all_artifacts
    
    def transform(self, artifacts: List[Dict]) -> tuple:
        """Transform raw JSON into relational dataframes"""
        metadata_records = []
        media_records = []
        color_records = []
        
        for artifact in artifacts:
            # Transform metadata
            metadata = {
                'id': artifact.get('id'),
                'title': artifact.get('title'),
                'culture': artifact.get('culture'),
                'period': artifact.get('period'),
                'century': artifact.get('century'),
                'classification': artifact.get('classification'),
                'department': artifact.get('department'),
                'dated': artifact.get('dated'),
                'url': artifact.get('url')
            }
            metadata_records.append(metadata)
            
            # Transform media
            if artifact.get('primaryimageurl'):
                media = {
                    'artifact_id': artifact.get('id'),
                    'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri'),
                    'baseimageurl': artifact.get('images', [{}])[0].get('baseimageurl'),
                    'primaryurl': artifact.get('primaryimageurl')
                }
                media_records.append(media)
            
            # Transform colors
            for color in artifact.get('colors', []):
                color_record = {
                    'artifact_id': artifact.get('id'),
                    'color_hex': color.get('hex'),
                    'color_percent': color.get('percent')
                }
                color_records.append(color_record)
        
        df_metadata = pd.DataFrame(metadata_records)
        df_media = pd.DataFrame(media_records)
        df_colors = pd.DataFrame(color_records)
        
        return df_metadata, df_media, df_colors
    
    def load(self, df_metadata: pd.DataFrame, df_media: pd.DataFrame, 
             df_colors: pd.DataFrame):
        """Load dataframes into SQL database"""
        conn = mysql.connector.connect(**self.db_config)
        cursor = conn.cursor()
        
        # Load metadata
        for _, row in df_metadata.iterrows():
            query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, classification, department, dated, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
            """
            cursor.execute(query, tuple(row))
        
        # Load media
        for _, row in df_media.iterrows():
            query = """
            INSERT INTO artifactmedia 
            (artifact_id, iiifbaseuri, baseimageurl, primaryurl)
            VALUES (%s, %s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        # Load colors
        for _, row in df_colors.iterrows():
            query = """
            INSERT INTO artifactcolors 
            (artifact_id, color_hex, color_percent)
            VALUES (%s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        conn.commit()
        cursor.close()
        conn.close()
        
        print(f"Loaded {len(df_metadata)} metadata records")
        print(f"Loaded {len(df_media)} media records")
        print(f"Loaded {len(df_colors)} color records")

# Usage
db_config = {
    'host': os.getenv('MYSQL_HOST'),
    'user': os.getenv('MYSQL_USER'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': os.getenv('MYSQL_DATABASE')
}

etl = ArtifactETL(db_config)
artifacts = etl.extract(os.getenv('HARVARD_API_KEY'), num_pages=5)
df_meta, df_media, df_colors = etl.transform(artifacts)
etl.load(df_meta, df_media, df_colors)
```

### 3. SQL Analytics Queries

```python
def execute_analytics_query(query: str, db_config: Dict[str, str]) -> pd.DataFrame:
    """Execute SQL query and return results as DataFrame"""
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Sample analytical queries
ANALYTICS_QUERIES = {
    'artifacts_by_culture': """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    'artifacts_by_century': """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    'top_colors': """
        SELECT color_hex, COUNT(*) as usage_count, 
               AVG(color_percent) as avg_percent
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    'artifacts_with_images': """
        SELECT 
            COUNT(DISTINCT m.id) as total_artifacts,
            COUNT(DISTINCT a.artifact_id) as artifacts_with_media,
            ROUND(COUNT(DISTINCT a.artifact_id) * 100.0 / COUNT(DISTINCT m.id), 2) as percentage
        FROM artifactmetadata m
        LEFT JOIN artifactmedia a ON m.id = a.artifact_id
    """,
    
    'department_distribution': """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
}

# Execute query
df_results = execute_analytics_query(
    ANALYTICS_QUERIES['artifacts_by_culture'],
    db_config
)
print(df_results.head())
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar for query selection
query_options = list(ANALYTICS_QUERIES.keys())
selected_query = st.sidebar.selectbox("Select Analysis", query_options)

# Execute query button
if st.button("Run Analysis"):
    with st.spinner("Executing query..."):
        df = execute_analytics_query(
            ANALYTICS_QUERIES[selected_query],
            db_config
        )
        
        # Display results
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            x_col = df.columns[0]
            y_col = df.columns[1]
            
            fig = px.bar(
                df.head(20),
                x=x_col,
                y=y_col,
                title=f"{selected_query.replace('_', ' ').title()}",
                color=y_col,
                color_continuous_scale='viridis'
            )
            
            st.plotly_chart(fig, use_container_width=True)

# ETL Controls
st.sidebar.header("⚙️ ETL Pipeline")
num_pages = st.sidebar.slider("Pages to fetch", 1, 10, 5)

if st.sidebar.button("Run ETL Pipeline"):
    with st.spinner("Running ETL..."):
        etl = ArtifactETL(db_config)
        artifacts = etl.extract(os.getenv('HARVARD_API_KEY'), num_pages)
        df_meta, df_media, df_colors = etl.transform(artifacts)
        etl.load(df_meta, df_media, df_colors)
        st.sidebar.success("ETL completed successfully!")
```

## Common Patterns

### Batch Processing with Rate Limiting

```python
import time

def fetch_artifacts_batch(api_key: str, total_pages: int, delay: float = 0.5):
    """Fetch artifacts with rate limiting"""
    for page in range(1, total_pages + 1):
        data = fetch_artifacts(api_key, page=page)
        yield data['records']
        time.sleep(delay)  # Respect API rate limits

# Usage
for batch in fetch_artifacts_batch(api_key, total_pages=10):
    process_batch(batch)
```

### Error Handling and Retry Logic

```python
from functools import wraps
import time

def retry_on_failure(max_retries=3, delay=2):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_retries - 1:
                        raise
                    print(f"Attempt {attempt + 1} failed: {e}. Retrying...")
                    time.sleep(delay)
        return wrapper
    return decorator

@retry_on_failure(max_retries=3)
def fetch_artifacts_safe(api_key, page):
    return fetch_artifacts(api_key, page)
```

## Troubleshooting

### API Rate Limit Errors

**Problem**: HTTP 429 Too Many Requests

**Solution**: Add delays between requests

```python
import time
time.sleep(1)  # 1 second delay between API calls
```

### Database Connection Issues

**Problem**: Can't connect to MySQL/TiDB

**Solution**: Verify credentials and network access

```python
try:
    conn = mysql.connector.connect(**db_config)
    print("Connection successful")
except mysql.connector.Error as err:
    print(f"Error: {err}")
```

### Missing API Key

**Problem**: 401 Unauthorized

**Solution**: Ensure API key is properly set

```python
assert os.getenv('HARVARD_API_KEY'), "HARVARD_API_KEY not set"
```

### Memory Issues with Large Datasets

**Problem**: Out of memory when processing many records

**Solution**: Use chunked processing

```python
def process_in_chunks(artifacts, chunk_size=1000):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        yield chunk
```
