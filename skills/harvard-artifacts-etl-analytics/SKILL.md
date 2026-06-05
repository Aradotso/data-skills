---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I fetch data from the Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create analytics dashboard with Streamlit and museum data
  - process Harvard Art Museums API responses into SQL
  - visualize artifact metadata with Plotly
  - set up data engineering pipeline for art museum collections
  - query Harvard artifacts database with SQL analytics
  - transform nested JSON art data into relational tables
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL operations, SQL database design, analytics queries, and interactive Streamlit visualizations for museum artifact collections.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into SQL databases
- **SQL Analytics**: Execute predefined analytical queries on artifact collections
- **Interactive Dashboards**: Visualize query results using Streamlit and Plotly
- **Database Schema**: Relational tables for artifact metadata, media, and colors with proper foreign keys

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

## Key Dependencies

```python
import streamlit as st
import pandas as pd
import requests
import mysql.connector
import plotly.express as px
from typing import List, Dict, Optional
```

## Configuration

### API Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

```python
# config.py
import os

HARVARD_API_KEY = os.getenv('HARVARD_API_KEY')
API_BASE_URL = "https://api.harvardartmuseums.org/object"

DB_CONFIG = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts')
}
```

### Database Schema

```sql
-- artifactmetadata table
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    url VARCHAR(500)
);

-- artifactmedia table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    primaryimageurl VARCHAR(500),
    baseimageurl VARCHAR(500),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- artifactcolors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    percentage DECIMAL(5,2),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## ETL Pipeline Patterns

### Extract: Fetch Data from API

```python
def fetch_artifacts(api_key: str, page: int = 1, size: int = 100) -> Dict:
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(API_BASE_URL, params=params)
    response.raise_for_status()
    return response.json()

def fetch_all_artifacts(api_key: str, max_records: int = 1000) -> List[Dict]:
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        data = fetch_artifacts(api_key, page=page, size=size)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_artifacts.extend(records)
        page += 1
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return all_artifacts[:max_records]
```

### Transform: Process Nested JSON

```python
def transform_artifact_metadata(artifact: Dict) -> Dict:
    """Extract metadata fields from artifact record"""
    return {
        'objectid': artifact.get('objectid'),
        'title': artifact.get('title', '')[:500],
        'culture': artifact.get('culture', '')[:200],
        'period': artifact.get('period', '')[:200],
        'century': artifact.get('century', '')[:100],
        'classification': artifact.get('classification', '')[:200],
        'department': artifact.get('department', '')[:200],
        'dated': artifact.get('dated', '')[:200],
        'url': artifact.get('url', '')[:500]
    }

def transform_artifact_media(artifact: Dict) -> Optional[Dict]:
    """Extract media/image information"""
    if not artifact.get('primaryimageurl'):
        return None
        
    return {
        'objectid': artifact.get('objectid'),
        'primaryimageurl': artifact.get('primaryimageurl', '')[:500],
        'baseimageurl': artifact.get('baseimageurl', '')[:500]
    }

def transform_artifact_colors(artifact: Dict) -> List[Dict]:
    """Extract color data from artifact"""
    colors = []
    color_data = artifact.get('colors', [])
    
    for color_entry in color_data:
        colors.append({
            'objectid': artifact.get('objectid'),
            'color': color_entry.get('color', '')[:50],
            'percentage': color_entry.get('percent', 0.0)
        })
    
    return colors
```

### Load: Insert into Database

```python
def load_metadata_batch(conn, metadata_list: List[Dict]):
    """Batch insert artifact metadata"""
    cursor = conn.cursor()
    
    query = """
        INSERT IGNORE INTO artifactmetadata 
        (objectid, title, culture, period, century, classification, department, dated, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    
    values = [
        (m['objectid'], m['title'], m['culture'], m['period'], 
         m['century'], m['classification'], m['department'], m['dated'], m['url'])
        for m in metadata_list
    ]
    
    cursor.executemany(query, values)
    conn.commit()
    cursor.close()

def load_media_batch(conn, media_list: List[Dict]):
    """Batch insert artifact media"""
    cursor = conn.cursor()
    
    query = """
        INSERT INTO artifactmedia (objectid, primaryimageurl, baseimageurl)
        VALUES (%s, %s, %s)
    """
    
    values = [
        (m['objectid'], m['primaryimageurl'], m['baseimageurl'])
        for m in media_list if m
    ]
    
    cursor.executemany(query, values)
    conn.commit()
    cursor.close()

def load_colors_batch(conn, colors_list: List[Dict]):
    """Batch insert artifact colors"""
    cursor = conn.cursor()
    
    query = """
        INSERT INTO artifactcolors (objectid, color, percentage)
        VALUES (%s, %s, %s)
    """
    
    values = [
        (c['objectid'], c['color'], c['percentage'])
        for c in colors_list
    ]
    
    cursor.executemany(query, values)
    conn.commit()
    cursor.close()
```

## Complete ETL Workflow

```python
import mysql.connector
from config import HARVARD_API_KEY, DB_CONFIG

def run_etl_pipeline(num_artifacts: int = 500):
    """Execute full ETL pipeline"""
    # Extract
    print(f"Fetching {num_artifacts} artifacts from API...")
    artifacts = fetch_all_artifacts(HARVARD_API_KEY, max_records=num_artifacts)
    
    # Transform
    print("Transforming data...")
    metadata_list = [transform_artifact_metadata(a) for a in artifacts]
    media_list = [transform_artifact_media(a) for a in artifacts if transform_artifact_media(a)]
    colors_list = []
    for a in artifacts:
        colors_list.extend(transform_artifact_colors(a))
    
    # Load
    print("Loading data into database...")
    conn = mysql.connector.connect(**DB_CONFIG)
    
    try:
        load_metadata_batch(conn, metadata_list)
        print(f"Loaded {len(metadata_list)} metadata records")
        
        load_media_batch(conn, media_list)
        print(f"Loaded {len(media_list)} media records")
        
        load_colors_batch(conn, colors_list)
        print(f"Loaded {len(colors_list)} color records")
    finally:
        conn.close()
    
    print("ETL pipeline completed successfully!")

# Run the pipeline
if __name__ == "__main__":
    run_etl_pipeline(num_artifacts=1000)
```

## SQL Analytics Queries

```python
# Sample analytics queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY artifact_count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Top Colors": """
        SELECT color, COUNT(*) as occurrences, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY occurrences DESC
        LIMIT 15
    """,
    
    "Artifacts with Images": """
        SELECT 
            CASE WHEN primaryimageurl IS NOT NULL THEN 'Has Image' ELSE 'No Image' END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "Classification Breakdown": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 20
    """
}

def execute_analytics_query(query_name: str) -> pd.DataFrame:
    """Execute an analytics query and return results as DataFrame"""
    conn = mysql.connector.connect(**DB_CONFIG)
    
    try:
        query = ANALYTICS_QUERIES[query_name]
        df = pd.read_sql(query, conn)
        return df
    finally:
        conn.close()
```

## Streamlit Dashboard

```python
# app.py
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for query selection
st.sidebar.header("Analytics Queries")
query_name = st.sidebar.selectbox(
    "Select Query",
    list(ANALYTICS_QUERIES.keys())
)

# Execute query button
if st.sidebar.button("Run Query"):
    with st.spinner("Executing query..."):
        df = execute_analytics_query(query_name)
        
        # Display results
        st.subheader(f"Results: {query_name}")
        st.dataframe(df, use_container_width=True)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(
                df,
                x=df.columns[0],
                y=df.columns[1],
                title=query_name,
                labels={df.columns[0]: df.columns[0].replace('_', ' ').title(),
                       df.columns[1]: df.columns[1].replace('_', ' ').title()}
            )
            st.plotly_chart(fig, use_container_width=True)

# ETL Pipeline section
st.sidebar.header("ETL Pipeline")
num_artifacts = st.sidebar.number_input("Number of artifacts to fetch", 
                                        min_value=10, max_value=5000, value=100)

if st.sidebar.button("Run ETL Pipeline"):
    with st.spinner(f"Fetching and processing {num_artifacts} artifacts..."):
        run_etl_pipeline(num_artifacts=num_artifacts)
        st.success(f"Successfully processed {num_artifacts} artifacts!")
```

## Common Patterns

### Handling API Pagination

```python
def fetch_with_pagination(api_key: str, total_records: int = 1000):
    """Robust pagination handler"""
    artifacts = []
    page = 1
    page_size = 100
    
    while len(artifacts) < total_records:
        try:
            response = fetch_artifacts(api_key, page=page, size=page_size)
            records = response.get('records', [])
            
            if not records:
                break
            
            artifacts.extend(records)
            
            # Check if we've reached the end
            total_available = response.get('info', {}).get('totalrecords', 0)
            if len(artifacts) >= total_available:
                break
            
            page += 1
            time.sleep(0.5)  # Rate limiting
            
        except requests.exceptions.RequestException as e:
            st.error(f"API error on page {page}: {e}")
            break
    
    return artifacts
```

### Error Handling and Validation

```python
def validate_artifact_data(artifact: Dict) -> bool:
    """Validate required fields exist"""
    required_fields = ['objectid']
    return all(field in artifact for field in required_fields)

def safe_transform(artifacts: List[Dict]) -> tuple:
    """Transform with error handling"""
    metadata, media, colors = [], [], []
    errors = []
    
    for artifact in artifacts:
        try:
            if not validate_artifact_data(artifact):
                errors.append(f"Invalid artifact: {artifact.get('objectid', 'unknown')}")
                continue
            
            metadata.append(transform_artifact_metadata(artifact))
            
            media_data = transform_artifact_media(artifact)
            if media_data:
                media.append(media_data)
            
            colors.extend(transform_artifact_colors(artifact))
            
        except Exception as e:
            errors.append(f"Error processing {artifact.get('objectid')}: {str(e)}")
    
    return metadata, media, colors, errors
```

## Troubleshooting

### API Rate Limiting

```python
import time
from functools import wraps

def rate_limit(calls_per_second=2):
    """Decorator to rate limit API calls"""
    min_interval = 1.0 / calls_per_second
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            if elapsed < min_interval:
                time.sleep(min_interval - elapsed)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_second=2)
def fetch_artifacts_rate_limited(api_key, page, size):
    return fetch_artifacts(api_key, page, size)
```

### Database Connection Issues

```python
def get_db_connection(retries=3):
    """Get database connection with retry logic"""
    for attempt in range(retries):
        try:
            conn = mysql.connector.connect(**DB_CONFIG)
            return conn
        except mysql.connector.Error as e:
            if attempt == retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

### Memory Management for Large Datasets

```python
def etl_with_batching(api_key: str, total_records: int, batch_size: int = 100):
    """Process large datasets in batches"""
    conn = get_db_connection()
    
    try:
        processed = 0
        while processed < total_records:
            # Fetch batch
            artifacts = fetch_all_artifacts(api_key, max_records=min(batch_size, total_records - processed))
            
            # Transform batch
            metadata, media, colors, errors = safe_transform(artifacts)
            
            # Load batch
            load_metadata_batch(conn, metadata)
            load_media_batch(conn, media)
            load_colors_batch(conn, colors)
            
            processed += len(artifacts)
            print(f"Processed {processed}/{total_records} artifacts")
            
    finally:
        conn.close()
```
