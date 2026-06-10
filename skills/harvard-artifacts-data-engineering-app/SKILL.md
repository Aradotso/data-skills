---
name: harvard-artifacts-data-engineering-app
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline for museum artifacts
  - create ETL pipeline with Harvard Art Museums API
  - set up artifact collection analytics dashboard
  - implement SQL analytics for art museum data
  - visualize Harvard museum collection data
  - build Streamlit app for artifact data engineering
  - create end-to-end analytics app with museum API
  - analyze Harvard art collection with SQL queries
---

# Harvard Artifacts Data Engineering App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering Analytics App is an end-to-end data pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational tables, loads it into SQL databases (MySQL/TiDB), and provides interactive analytics through a Streamlit dashboard. The project demonstrates real-world ETL patterns, SQL analytics, and data visualization.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly sqlalchemy
```

## Configuration

### Environment Variables

Set up your credentials using environment variables:

```bash
# Harvard Art Museums API
export HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Connection
export DB_HOST=your_host
export DB_PORT=4000
export DB_USER=your_username
export DB_PASSWORD=your_password
export DB_NAME=your_database
```

### API Key Setup

Get your Harvard Art Museums API key from: https://docs.harvardartmuseums.org/

### Database Schema

The application creates three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    dated VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accession_number VARCHAR(100),
    url TEXT
);

CREATE TABLE artifactmedia (
    media_id INT PRIMARY KEY AUTO_INCREMENT,
    artifact_id INT,
    image_url TEXT,
    media_type VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT PRIMARY KEY AUTO_INCREMENT,
    artifact_id INT,
    color_name VARCHAR(100),
    color_hex VARCHAR(10),
    color_percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Core Components

### 1. API Integration

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': os.environ.get('HARVARD_API_KEY'),
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

def collect_paginated_data(api_key, max_pages=10):
    """Collect data across multiple pages"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(api_key, page=page)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_artifacts.extend(records)
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd

def extract_metadata(artifacts):
    """Extract artifact metadata"""
    metadata = []
    
    for artifact in artifacts:
        record = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'dated': artifact.get('dated'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accession_number': artifact.get('accessionyear'),
            'url': artifact.get('url')
        }
        metadata.append(record)
    
    return pd.DataFrame(metadata)

def extract_media(artifacts):
    """Extract media/image data"""
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media_records.append({
                'artifact_id': artifact_id,
                'image_url': image.get('baseimageurl'),
                'media_type': 'image'
            })
    
    return pd.DataFrame(media_records)

def extract_colors(artifacts):
    """Extract color data"""
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_records.append({
                'artifact_id': artifact_id,
                'color_name': color.get('color'),
                'color_hex': color.get('hex'),
                'color_percentage': color.get('percent')
            })
    
    return pd.DataFrame(color_records)
```

### 3. Database Operations

```python
import mysql.connector
from sqlalchemy import create_engine
import os

def get_db_connection():
    """Create database connection"""
    conn = mysql.connector.connect(
        host=os.environ.get('DB_HOST'),
        port=int(os.environ.get('DB_PORT', 3306)),
        user=os.environ.get('DB_USER'),
        password=os.environ.get('DB_PASSWORD'),
        database=os.environ.get('DB_NAME')
    )
    return conn

def load_to_sql(df, table_name, if_exists='append'):
    """Load DataFrame to SQL using SQLAlchemy"""
    engine = create_engine(
        f"mysql+mysqlconnector://{os.environ.get('DB_USER')}:"
        f"{os.environ.get('DB_PASSWORD')}@"
        f"{os.environ.get('DB_HOST')}:"
        f"{os.environ.get('DB_PORT', 3306)}/"
        f"{os.environ.get('DB_NAME')}"
    )
    
    df.to_sql(table_name, engine, if_exists=if_exists, index=False)
    print(f"Loaded {len(df)} records to {table_name}")

def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### 4. Sample Analytics Queries

```python
# Query 1: Artifacts by culture
query_by_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Query 2: Artifacts by century
query_by_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY century;
"""

# Query 3: Department distribution
query_by_department = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
GROUP BY department
ORDER BY total_artifacts DESC;
"""

# Query 4: Most common colors
query_colors = """
SELECT c.color_name, COUNT(*) as usage_count, AVG(c.color_percentage) as avg_percentage
FROM artifactcolors c
GROUP BY c.color_name
ORDER BY usage_count DESC
LIMIT 10;
"""

# Query 5: Artifacts with media
query_media_availability = """
SELECT 
    CASE WHEN m.artifact_id IS NOT NULL THEN 'Has Media' ELSE 'No Media' END as media_status,
    COUNT(DISTINCT a.id) as artifact_count
FROM artifactmetadata a
LEFT JOIN artifactmedia m ON a.id = m.artifact_id
GROUP BY media_status;
"""
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    st.title("Harvard Artifacts Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    
    queries = {
        "Artifacts by Culture": query_by_culture,
        "Artifacts by Century": query_by_century,
        "Department Distribution": query_by_department,
        "Color Analysis": query_colors,
        "Media Availability": query_media_availability
    }
    
    selected_query = st.sidebar.selectbox("Select Analysis", list(queries.keys()))
    
    if st.sidebar.button("Run Query"):
        with st.spinner("Executing query..."):
            df = execute_query(queries[selected_query])
            
            # Display results
            st.subheader(f"Results: {selected_query}")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df) > 0 and len(df.columns) >= 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1])
                st.plotly_chart(fig)

if __name__ == "__main__":
    create_dashboard()
```

## Complete ETL Workflow

```python
def run_etl_pipeline():
    """Complete ETL pipeline execution"""
    
    # Step 1: Extract
    st.info("Step 1: Extracting data from Harvard API...")
    api_key = os.environ.get('HARVARD_API_KEY')
    artifacts = collect_paginated_data(api_key, max_pages=5)
    
    # Step 2: Transform
    st.info("Step 2: Transforming data...")
    metadata_df = extract_metadata(artifacts)
    media_df = extract_media(artifacts)
    colors_df = extract_colors(artifacts)
    
    # Step 3: Load
    st.info("Step 3: Loading data to SQL...")
    load_to_sql(metadata_df, 'artifactmetadata', if_exists='replace')
    load_to_sql(media_df, 'artifactmedia', if_exists='replace')
    load_to_sql(colors_df, 'artifactcolors', if_exists='replace')
    
    st.success(f"ETL Complete! Processed {len(artifacts)} artifacts")
    
    return {
        'metadata': len(metadata_df),
        'media': len(media_df),
        'colors': len(colors_df)
    }
```

## Common Patterns

### Rate Limiting API Calls

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
            wait_time = min_interval - elapsed
            
            if wait_time > 0:
                time.sleep(wait_time)
            
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_second=2)
def fetch_artifact_by_id(artifact_id):
    url = f"https://api.harvardartmuseums.org/object/{artifact_id}"
    params = {'apikey': os.environ.get('HARVARD_API_KEY')}
    return requests.get(url, params=params).json()
```

### Batch Processing

```python
def batch_insert(df, table_name, batch_size=1000):
    """Insert data in batches for better performance"""
    total_rows = len(df)
    
    for i in range(0, total_rows, batch_size):
        batch = df.iloc[i:i+batch_size]
        load_to_sql(batch, table_name, if_exists='append')
        print(f"Inserted batch {i//batch_size + 1}: {len(batch)} rows")
```

### Error Handling

```python
def safe_api_call(func, max_retries=3):
    """Wrapper for API calls with retry logic"""
    for attempt in range(max_retries):
        try:
            return func()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
            continue
```

## Troubleshooting

**API Key Issues:**
- Verify `HARVARD_API_KEY` is set correctly
- Check API key validity at Harvard Art Museums developer portal
- Ensure no trailing spaces in environment variable

**Database Connection:**
- Verify all DB environment variables are set
- Test connection: `mysql -h $DB_HOST -P $DB_PORT -u $DB_USER -p`
- Check firewall rules for cloud databases (TiDB)

**Memory Issues with Large Datasets:**
- Use batch processing with smaller page sizes
- Process data in chunks: `pd.read_sql(query, conn, chunksize=1000)`
- Limit initial collection to fewer pages

**Streamlit Performance:**
- Cache expensive operations: `@st.cache_data`
- Use connection pooling for database queries
- Implement pagination for large result sets
