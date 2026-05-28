---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - connect to Harvard Art Museums API
  - create a data engineering pipeline with Streamlit
  - extract and transform art collection metadata
  - visualize museum artifact analytics
  - set up SQL database for art collections
  - analyze Harvard Art Museums data
  - build a museum data dashboard
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering pipelines using the Harvard Art Museums API, including ETL processes, SQL database design, and interactive analytics dashboards with Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is a complete data pipeline that:
- Extracts artifact data from the Harvard Art Museums API
- Transforms nested JSON into relational database tables
- Loads structured data into MySQL/TiDB Cloud
- Provides 20+ predefined analytical SQL queries
- Visualizes results through an interactive Streamlit dashboard

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Expected dependencies include:
# streamlit
# pandas
# requests
# mysql-connector-python
# plotly
# python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file or set environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_db_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Getting API Access

1. Register at [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Obtain your API key
3. Store it securely in environment variables

## Database Schema

The project uses three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    url VARCHAR(500)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    imagestotal INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## ETL Pipeline Implementation

### Extract: Fetching Data from API

```python
import requests
import os
from typing import Dict, List

def fetch_artifacts(api_key: str, page: int = 1, size: int = 100) -> Dict:
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

def collect_multiple_pages(api_key: str, num_pages: int = 5) -> List[Dict]:
    """
    Collect artifacts across multiple pages with rate limiting
    """
    import time
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(data.get('records', []))
        time.sleep(1)  # Rate limiting
    
    return all_artifacts
```

### Transform: Processing Nested JSON

```python
import pandas as pd
from typing import List, Dict, Tuple

def transform_artifacts(artifacts: List[Dict]) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
    """
    Transform raw API response into three normalized DataFrames
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'division': artifact.get('division', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'period': artifact.get('period', '')[:255],
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'dimensions': artifact.get('dimensions', '')[:500],
            'url': artifact.get('url', '')[:500]
        }
        metadata_records.append(metadata)
        
        # Extract media information
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl', '')[:500],
            'primaryimageurl': artifact.get('primaryimageurl', '')[:500],
            'imagestotal': artifact.get('totalpageviews', 0)
        }
        media_records.append(media)
        
        # Extract color information
        colors = artifact.get('colors', [])
        for color in colors:
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'hue': color.get('hue', '')[:50],
                'percent': color.get('percent', 0.0)
            }
            color_records.append(color_record)
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### Load: Batch Insert into SQL

```python
import mysql.connector
from mysql.connector import Error
import pandas as pd

def get_db_connection():
    """
    Create database connection using environment variables
    """
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    return connection

def batch_insert_metadata(df: pd.DataFrame):
    """
    Batch insert artifact metadata into database
    """
    connection = get_db_connection()
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, division, 
     dated, period, technique, medium, dimensions, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    records = df.to_records(index=False)
    data = list(records)
    
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} metadata records")
    cursor.close()
    connection.close()

def batch_insert_media(df: pd.DataFrame):
    """
    Batch insert artifact media into database
    """
    connection = get_db_connection()
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, baseimageurl, primaryimageurl, imagestotal)
    VALUES (%s, %s, %s, %s)
    """
    
    records = df.to_records(index=False)
    data = list(records)
    
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} media records")
    cursor.close()
    connection.close()

def batch_insert_colors(df: pd.DataFrame):
    """
    Batch insert artifact colors into database
    """
    connection = get_db_connection()
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    records = df.to_records(index=False)
    data = list(records)
    
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} color records")
    cursor.close()
    connection.close()
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import pandas as pd
import plotly.express as px
from dotenv import load_dotenv

load_dotenv()

st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar configuration
st.sidebar.header("Configuration")
api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))

# ETL Controls
st.sidebar.header("ETL Pipeline")
num_pages = st.sidebar.slider("Number of pages to fetch", 1, 10, 5)

if st.sidebar.button("Run ETL Pipeline"):
    with st.spinner("Fetching data from API..."):
        artifacts = collect_multiple_pages(api_key, num_pages)
        st.success(f"Fetched {len(artifacts)} artifacts")
    
    with st.spinner("Transforming data..."):
        df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        st.success("Data transformed successfully")
    
    with st.spinner("Loading into database..."):
        batch_insert_metadata(df_metadata)
        batch_insert_media(df_media)
        batch_insert_colors(df_colors)
        st.success("Data loaded successfully")

# Analytics Section
st.header("📊 Analytics Queries")

# Sample analytical queries
queries = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
        LIMIT 10
    """,
    "Top Departments": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    "Color Distribution": """
        SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY count DESC
        LIMIT 10
    """,
    "Media Availability": """
        SELECT 
            CASE WHEN imagestotal > 0 THEN 'Has Images' ELSE 'No Images' END as status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY status
    """
}

selected_query = st.selectbox("Select Query", list(queries.keys()))

if st.button("Execute Query"):
    connection = get_db_connection()
    
    try:
        df_result = pd.read_sql(queries[selected_query], connection)
        
        st.subheader("Query Results")
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) == 2:
            fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)
    
    except Error as e:
        st.error(f"Database error: {e}")
    
    finally:
        connection.close()
```

## Advanced Analytics Queries

```python
# Complex analytical queries for deeper insights

ADVANCED_QUERIES = {
    "Artifacts with Most Colors": """
        SELECT 
            m.title,
            m.culture,
            COUNT(c.color_id) as color_count,
            GROUP_CONCAT(DISTINCT c.color ORDER BY c.percent DESC) as colors
        FROM artifactmetadata m
        JOIN artifactcolors c ON m.id = c.artifact_id
        GROUP BY m.id, m.title, m.culture
        ORDER BY color_count DESC
        LIMIT 20
    """,
    
    "Department Color Preferences": """
        SELECT 
            m.department,
            c.color,
            COUNT(*) as usage_count,
            AVG(c.percent) as avg_percent
        FROM artifactmetadata m
        JOIN artifactcolors c ON m.id = c.artifact_id
        WHERE m.department IS NOT NULL
        GROUP BY m.department, c.color
        HAVING COUNT(*) > 5
        ORDER BY m.department, usage_count DESC
    """,
    
    "Century Evolution": """
        SELECT 
            m.century,
            COUNT(DISTINCT m.id) as artifact_count,
            COUNT(DISTINCT m.culture) as culture_count,
            COUNT(DISTINCT c.color) as color_diversity
        FROM artifactmetadata m
        LEFT JOIN artifactcolors c ON m.id = c.artifact_id
        WHERE m.century IS NOT NULL AND m.century != ''
        GROUP BY m.century
        ORDER BY m.century
    """
}
```

## Common Patterns

### Complete ETL Workflow

```python
from typing import Optional
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class HarvardETLPipeline:
    """
    Complete ETL pipeline for Harvard Art Museums data
    """
    
    def __init__(self, api_key: str):
        self.api_key = api_key
    
    def run_pipeline(self, num_pages: int = 5) -> bool:
        """
        Execute complete ETL pipeline
        """
        try:
            # Extract
            logger.info("Starting extraction...")
            artifacts = collect_multiple_pages(self.api_key, num_pages)
            logger.info(f"Extracted {len(artifacts)} artifacts")
            
            # Transform
            logger.info("Starting transformation...")
            df_metadata, df_media, df_colors = transform_artifacts(artifacts)
            logger.info("Transformation complete")
            
            # Load
            logger.info("Starting load...")
            batch_insert_metadata(df_metadata)
            batch_insert_media(df_media)
            batch_insert_colors(df_colors)
            logger.info("Load complete")
            
            return True
            
        except Exception as e:
            logger.error(f"Pipeline failed: {e}")
            return False

# Usage
pipeline = HarvardETLPipeline(os.getenv('HARVARD_API_KEY'))
success = pipeline.run_pipeline(num_pages=3)
```

## Troubleshooting

### API Rate Limiting

```python
import time
from functools import wraps

def rate_limited(max_calls: int = 50, period: int = 60):
    """
    Decorator to rate limit API calls
    """
    calls = []
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            now = time.time()
            calls[:] = [c for c in calls if c > now - period]
            
            if len(calls) >= max_calls:
                sleep_time = period - (now - calls[0])
                time.sleep(sleep_time)
            
            calls.append(time.time())
            return func(*args, **kwargs)
        return wrapper
    return decorator

@rate_limited(max_calls=50, period=60)
def fetch_artifacts_with_limit(api_key: str, page: int = 1) -> Dict:
    return fetch_artifacts(api_key, page)
```

### Database Connection Pooling

```python
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    port=int(os.getenv('DB_PORT', 3306)),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_pooled_connection():
    return db_pool.get_connection()
```

### Handling Missing Data

```python
def safe_transform_artifacts(artifacts: List[Dict]) -> Tuple:
    """
    Transform with proper null handling
    """
    for artifact in artifacts:
        # Set defaults for missing fields
        artifact.setdefault('title', 'Untitled')
        artifact.setdefault('culture', 'Unknown')
        artifact.setdefault('colors', [])
        
        # Clean text fields
        for key in ['title', 'medium', 'technique']:
            if key in artifact and artifact[key]:
                artifact[key] = str(artifact[key]).strip()
    
    return transform_artifacts(artifacts)
```

This skill provides everything needed to build, deploy, and extend the Harvard Art Museums ETL and analytics pipeline.
