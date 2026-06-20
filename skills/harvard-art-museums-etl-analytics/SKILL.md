---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifact data
  - analyze Harvard art collection with SQL
  - create a data engineering app with Streamlit
  - fetch and transform Harvard Art Museums API data
  - visualize museum artifact analytics
  - design SQL schema for artifact metadata
  - build museum data collection dashboard
  - extract artwork data from Harvard API
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

This project provides a complete data engineering solution for the Harvard Art Museums API. It demonstrates:

- **API Integration**: Paginated data collection from Harvard Art Museums
- **ETL Pipeline**: Extract, transform, and load artifact data into relational databases
- **SQL Analytics**: 20+ predefined analytical queries
- **Interactive Dashboards**: Streamlit-based visualization with Plotly charts
- **Database Design**: Normalized schema with proper foreign key relationships

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests plotly mysql-connector-python python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_db_host
DB_PORT=4000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free)
3. Add to `.env` file

### Database Setup

Supports MySQL or TiDB Cloud. Create database schema:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    dated VARCHAR(255),
    classification VARCHAR(255),
    division VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(255),
    medium VARCHAR(500),
    dated_begin INT,
    dated_end INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    image_width INT,
    image_height INT,
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = 'https://api.harvardartmuseums.org/object'
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'fields': 'id,title,culture,period,century,dated,classification,division,department,technique,medium,images,colors,primaryimageurl'
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

# Paginated collection
def collect_artifacts(total_records=500):
    """Collect artifacts with pagination"""
    artifacts = []
    page = 1
    size = 100
    
    while len(artifacts) < total_records:
        data = fetch_artifacts(page=page, size=size)
        artifacts.extend(data['records'])
        
        if data['info']['next'] is None:
            break
        page += 1
    
    return artifacts[:total_records]
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON to normalized DataFrames"""
    
    # Metadata transformation
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Main metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', ''),
            'period': artifact.get('period', ''),
            'century': artifact.get('century', ''),
            'dated': artifact.get('dated', ''),
            'classification': artifact.get('classification', ''),
            'division': artifact.get('division', ''),
            'department': artifact.get('department', ''),
            'technique': artifact.get('technique', ''),
            'medium': artifact.get('medium', '')
        })
        
        # Images/Media
        images = artifact.get('images', [])
        for img in images:
            media_records.append({
                'artifact_id': artifact.get('id'),
                'image_url': img.get('baseimageurl', ''),
                'image_width': img.get('width'),
                'image_height': img.get('height'),
                'media_type': 'image'
            })
        
        # Colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex', ''),
                'color_percentage': color.get('percent', 0)
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Batch insert data into SQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, dated, classification, 
             division, department, technique, medium)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, image_url, image_width, image_height, media_type)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color_hex, color_percentage)
            VALUES (%s, %s, %s)
        """
        cursor.executemany(colors_query, colors_df.values.tolist())
        
        conn.commit()
        return True
        
    except Error as e:
        print(f"Database error: {e}")
        conn.rollback()
        return False
    finally:
        cursor.close()
        conn.close()
```

### 4. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Artifacts with Images": """
        SELECT 
            CASE WHEN EXISTS (
                SELECT 1 FROM artifactmedia m 
                WHERE m.artifact_id = a.id
            ) THEN 'Has Images' ELSE 'No Images' END as image_status,
            COUNT(*) as count
        FROM artifactmetadata a
        GROUP BY image_status
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND department != ''
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Color Distribution": """
        SELECT color_hex, COUNT(*) as usage_count,
               AVG(color_percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY usage_count DESC
        LIMIT 20
    """,
    
    "Artifacts by Classification": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 10
    """
}

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### 5. Streamlit Visualization

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """Main Streamlit dashboard"""
    st.title("🎨 Harvard Art Museums Analytics")
    
    # Sidebar - ETL Controls
    st.sidebar.header("Data Collection")
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Fetching artifacts..."):
            artifacts = collect_artifacts(500)
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            success = load_to_database(metadata_df, media_df, colors_df)
            
            if success:
                st.sidebar.success(f"Loaded {len(metadata_df)} artifacts!")
            else:
                st.sidebar.error("ETL failed")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    
    query_choice = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_choice]
        
        # Execute and display
        df = execute_query(query)
        st.dataframe(df)
        
        # Auto-visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=query_choice)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    create_dashboard()
```

## Common Patterns

### Rate Limiting

```python
import time

def fetch_with_rate_limit(page, size, delay=0.5):
    """Fetch with rate limiting"""
    data = fetch_artifacts(page, size)
    time.sleep(delay)  # Respect API limits
    return data
```

### Error Handling

```python
def safe_etl_pipeline(num_records=500):
    """ETL with comprehensive error handling"""
    try:
        artifacts = collect_artifacts(num_records)
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        # Validate data
        assert not metadata_df.empty, "No metadata collected"
        
        success = load_to_database(metadata_df, media_df, colors_df)
        return success, len(metadata_df)
        
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return False, 0
    except Error as e:
        print(f"Database Error: {e}")
        return False, 0
    except Exception as e:
        print(f"Unexpected Error: {e}")
        return False, 0
```

## Troubleshooting

### API Issues

**Problem**: 401 Unauthorized
```python
# Solution: Verify API key
api_key = os.getenv('HARVARD_API_KEY')
if not api_key:
    raise ValueError("HARVARD_API_KEY not set in .env")
```

**Problem**: Rate limiting
```python
# Solution: Add delays and retry logic
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

session = requests.Session()
retry = Retry(total=3, backoff_factor=1)
adapter = HTTPAdapter(max_retries=retry)
session.mount('https://', adapter)
```

### Database Issues

**Problem**: Connection timeout
```python
# Solution: Increase timeout
conn = mysql.connector.connect(
    connection_timeout=30,
    # ... other params
)
```

**Problem**: Duplicate key errors
```python
# Solution: Use INSERT ... ON DUPLICATE KEY UPDATE
# or clear tables before loading
cursor.execute("TRUNCATE TABLE artifactmedia")
cursor.execute("TRUNCATE TABLE artifactcolors")
```

### Streamlit Issues

**Problem**: Slow dashboard
```python
# Solution: Use caching
@st.cache_data(ttl=3600)
def cached_query(query):
    return execute_query(query)
```

## Advanced Usage

### Custom Analytics

```python
def add_custom_query(name, sql):
    """Add custom analytical query"""
    ANALYTICS_QUERIES[name] = sql
    
# Example
add_custom_query("Recent Acquisitions", """
    SELECT title, culture, department, created_at
    FROM artifactmetadata
    ORDER BY created_at DESC
    LIMIT 20
""")
```

### Export Results

```python
def export_to_csv(df, filename):
    """Export query results to CSV"""
    df.to_csv(filename, index=False)
    st.download_button(
        label="Download CSV",
        data=df.to_csv(index=False).encode('utf-8'),
        file_name=filename,
        mime='text/csv'
    )
```

This skill enables AI agents to help developers build complete ETL pipelines for museum data with SQL analytics and interactive visualizations.
