---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics application using Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering project with museum artifacts
  - fetch and analyze Harvard Art Museums API data
  - set up SQL analytics for art collection data
  - visualize museum artifact data with Streamlit
  - implement artifact metadata extraction and transformation
  - create interactive dashboard for art museums analytics
  - design relational database for museum collections
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming it into relational structures, loading it into SQL databases, and visualizing analytics through an interactive Streamlit dashboard.

## What It Does

- **Extracts** artifact metadata, media, and color data from Harvard Art Museums API
- **Transforms** nested JSON responses into normalized relational tables
- **Loads** data into MySQL/TiDB Cloud with proper foreign key relationships
- **Analyzes** data with 20+ predefined SQL queries
- **Visualizes** results through interactive Plotly charts in Streamlit

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

### API Key Setup

Get your Harvard Art Museums API key from: https://harvardartmuseums.org/collections/api

```bash
# Set environment variable
export HARVARD_API_KEY="your_api_key_here"
```

### Database Configuration

```python
# config.py or environment variables
import os

DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

## Database Schema

The project uses three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    dated VARCHAR(200),
    accession_number VARCHAR(100),
    url TEXT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    caption TEXT,
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core ETL Implementation

### Extract: Fetching Data from API

```python
import requests
import os
from typing import Dict, List

def fetch_artifacts(api_key: str, page: int = 1, size: int = 100) -> Dict:
    """Extract artifact data from Harvard Art Museums API"""
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

def extract_all_artifacts(api_key: str, max_records: int = 1000) -> List[Dict]:
    """Paginate through API to extract multiple records"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        data = fetch_artifacts(api_key, page, size)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_artifacts.extend(records)
        page += 1
        
        # Respect rate limits
        import time
        time.sleep(0.5)
    
    return all_artifacts[:max_records]
```

### Transform: Normalizing Data

```python
import pandas as pd
from typing import List, Tuple

def transform_artifacts(raw_data: List[Dict]) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
    """Transform raw API data into normalized dataframes"""
    
    # Metadata extraction
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'period': artifact.get('period', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'technique': artifact.get('technique', '')[:500],
            'dated': artifact.get('dated', '')[:200],
            'accession_number': artifact.get('accessionNumber', '')[:100],
            'url': artifact.get('url', '')
        })
        
        # Extract media
        for image in artifact.get('images', []):
            media_records.append({
                'artifact_id': artifact.get('id'),
                'image_url': image.get('baseimageurl', ''),
                'caption': image.get('caption', ''),
                'media_type': 'image'
            })
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex', ''),
                'color_percent': color.get('percent', 0.0)
            })
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records) if media_records else pd.DataFrame()
    df_colors = pd.DataFrame(color_records) if color_records else pd.DataFrame()
    
    return df_metadata, df_media, df_colors
```

### Load: Inserting into SQL

```python
import mysql.connector
from mysql.connector import Error
import pandas as pd

def create_database_connection(config: Dict):
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(**config)
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def load_metadata(df: pd.DataFrame, connection):
    """Load artifact metadata into database"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, classification, department, 
     technique, dated, accession_number, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title = VALUES(title),
    culture = VALUES(culture)
    """
    
    records = df.to_records(index=False)
    cursor.executemany(insert_query, records.tolist())
    connection.commit()
    cursor.close()

def load_media(df: pd.DataFrame, connection):
    """Load artifact media into database"""
    if df.empty:
        return
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, image_url, caption, media_type)
    VALUES (%s, %s, %s, %s)
    """
    
    records = df.to_records(index=False)
    cursor.executemany(insert_query, records.tolist())
    connection.commit()
    cursor.close()

def load_colors(df: pd.DataFrame, connection):
    """Load artifact colors into database"""
    if df.empty:
        return
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
    VALUES (%s, %s, %s)
    """
    
    records = df.to_records(index=False)
    cursor.executemany(insert_query, records.tolist())
    connection.commit()
    cursor.close()
```

## Complete ETL Pipeline

```python
import os
from typing import Dict

def run_etl_pipeline(api_key: str, db_config: Dict, max_records: int = 1000):
    """Execute complete ETL pipeline"""
    
    print("Starting ETL Pipeline...")
    
    # EXTRACT
    print("Extracting data from API...")
    raw_artifacts = extract_all_artifacts(api_key, max_records)
    print(f"Extracted {len(raw_artifacts)} artifacts")
    
    # TRANSFORM
    print("Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(raw_artifacts)
    print(f"Transformed: {len(df_metadata)} metadata, {len(df_media)} media, {len(df_colors)} colors")
    
    # LOAD
    print("Loading data into database...")
    connection = create_database_connection(db_config)
    
    if connection:
        load_metadata(df_metadata, connection)
        load_media(df_media, connection)
        load_colors(df_colors, connection)
        connection.close()
        print("ETL Pipeline completed successfully!")
    else:
        print("Failed to connect to database")

# Execute pipeline
if __name__ == "__main__":
    api_key = os.getenv('HARVARD_API_KEY')
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    run_etl_pipeline(api_key, db_config, max_records=500)
```

## SQL Analytics Queries

### Common Analytical Queries

```python
# Sample analytical queries for the dashboard

ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture 
        ORDER BY count DESC 
        LIMIT 10
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century 
        ORDER BY count DESC
    """,
    
    "artifacts_by_department": """
        SELECT department, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE department IS NOT NULL
        GROUP BY department 
        ORDER BY count DESC
    """,
    
    "artifacts_with_images": """
        SELECT 
            CASE 
                WHEN media_count > 0 THEN 'With Images'
                ELSE 'Without Images'
            END as image_status,
            COUNT(*) as count
        FROM (
            SELECT a.id, COUNT(m.media_id) as media_count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY a.id
        ) as subquery
        GROUP BY image_status
    """,
    
    "top_colors": """
        SELECT color_hex, COUNT(*) as usage_count, 
               AVG(color_percent) as avg_percentage
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "artifacts_by_classification": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL AND classification != ''
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """
}

def execute_query(connection, query_name: str) -> pd.DataFrame:
    """Execute analytical query and return results as DataFrame"""
    query = ANALYTICS_QUERIES.get(query_name)
    
    if not query:
        raise ValueError(f"Query '{query_name}' not found")
    
    return pd.read_sql(query, connection)
```

## Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px
import pandas as pd

def create_dashboard():
    """Create interactive Streamlit dashboard"""
    
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    db_config = {
        'host': st.sidebar.text_input("Database Host", value="localhost"),
        'user': st.sidebar.text_input("Database User", value="root"),
        'password': st.sidebar.text_input("Database Password", type="password"),
        'database': st.sidebar.text_input("Database Name", value="harvard_artifacts")
    }
    
    if st.sidebar.button("Connect to Database"):
        connection = create_database_connection(db_config)
        
        if connection:
            st.sidebar.success("Connected!")
            
            # Query selection
            query_name = st.selectbox(
                "Select Analysis",
                list(ANALYTICS_QUERIES.keys())
            )
            
            if st.button("Run Analysis"):
                # Execute query
                df_result = execute_query(connection, query_name)
                
                # Display results
                st.subheader("Query Results")
                st.dataframe(df_result)
                
                # Visualization
                if len(df_result.columns) == 2:
                    fig = px.bar(
                        df_result,
                        x=df_result.columns[0],
                        y=df_result.columns[1],
                        title=f"Analysis: {query_name}"
                    )
                    st.plotly_chart(fig, use_container_width=True)
            
            connection.close()
        else:
            st.sidebar.error("Connection failed")

if __name__ == "__main__":
    create_dashboard()
```

## Running the Application

```bash
# Run Streamlit dashboard
streamlit run app.py

# Run ETL pipeline separately
python etl_pipeline.py
```

## Common Patterns

### Batch Processing with Error Handling

```python
def batch_insert_with_retry(df: pd.DataFrame, connection, table_name: str, batch_size: int = 100):
    """Insert data in batches with error handling"""
    total_rows = len(df)
    
    for i in range(0, total_rows, batch_size):
        batch = df.iloc[i:i+batch_size]
        
        try:
            if table_name == 'metadata':
                load_metadata(batch, connection)
            elif table_name == 'media':
                load_media(batch, connection)
            elif table_name == 'colors':
                load_colors(batch, connection)
            
            print(f"Inserted batch {i//batch_size + 1} ({len(batch)} rows)")
        
        except Exception as e:
            print(f"Error in batch {i//batch_size + 1}: {e}")
            continue
```

### API Rate Limiting

```python
import time
from functools import wraps

def rate_limit(calls_per_second: float = 2):
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
def fetch_artifacts_limited(api_key: str, page: int = 1, size: int = 100):
    """Rate-limited API fetching"""
    return fetch_artifacts(api_key, page, size)
```

## Troubleshooting

### Connection Issues

```python
# Test database connection
def test_connection(config: Dict):
    try:
        conn = mysql.connector.connect(**config)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        conn.close()
        return result[0] == 1
    except Error as e:
        print(f"Connection test failed: {e}")
        return False
```

### API Key Validation

```python
def validate_api_key(api_key: str) -> bool:
    """Validate Harvard Art Museums API key"""
    try:
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'size': 1}
        )
        return response.status_code == 200
    except:
        return False
```

### Data Quality Checks

```python
def validate_dataframe(df: pd.DataFrame, required_columns: List[str]) -> bool:
    """Validate DataFrame before loading"""
    # Check for required columns
    if not all(col in df.columns for col in required_columns):
        print("Missing required columns")
        return False
    
    # Check for empty DataFrame
    if df.empty:
        print("DataFrame is empty")
        return False
    
    # Check for duplicate IDs
    if 'id' in df.columns and df['id'].duplicated().any():
        print("Warning: Duplicate IDs found")
    
    return True
```

This skill enables AI agents to help developers build complete ETL pipelines for museum data, create analytics dashboards, and implement data engineering best practices using the Harvard Art Museums API.
