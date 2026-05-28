---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifact collections
  - extract and transform Harvard API data into SQL database
  - visualize art museum collection data with Streamlit
  - implement data engineering pipeline for cultural artifacts
  - analyze Harvard Art Museums collections with SQL queries
  - set up museum data warehouse with Python and MySQL
  - create interactive art collection analytics application
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact metadata, media, and color data from Harvard Art Museums API
- **ETL Pipeline**: Extract, transform, and load nested JSON into relational database tables
- **SQL Analytics**: Execute analytical queries on artifact collections
- **Interactive Dashboards**: Visualize insights using Streamlit and Plotly
- **Real-world Data Engineering**: Demonstrates pagination, rate limiting, batch processing, and database optimization

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
import json
```

## Configuration

### API Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

```python
import os

# API Configuration
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

```python
# MySQL/TiDB Connection
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

def get_db_connection():
    return mysql.connector.connect(**db_config)
```

## Database Schema

### Create Tables

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    dated VARCHAR(255),
    century VARCHAR(255),
    division VARCHAR(255),
    department VARCHAR(255),
    classification VARCHAR(255),
    medium VARCHAR(500),
    technique VARCHAR(500),
    people TEXT,
    url TEXT,
    creditline TEXT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    base_url TEXT,
    format VARCHAR(50),
    width INT,
    height INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
def fetch_artifacts(page: int = 1, size: int = 100) -> Dict:
    """Fetch artifacts from Harvard Art Museums API with pagination."""
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'page': page,
        'size': size
    }
    
    response = requests.get(BASE_URL, params=params)
    response.raise_for_status()
    return response.json()

def collect_all_artifacts(max_records: int = 1000) -> List[Dict]:
    """Collect artifacts with pagination handling."""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        data = fetch_artifacts(page=page, size=size)
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

### Transform: Parse Nested JSON

```python
def transform_artifact_metadata(artifact: Dict) -> Dict:
    """Extract metadata fields from artifact JSON."""
    return {
        'id': artifact.get('id'),
        'title': artifact.get('title', ''),
        'culture': artifact.get('culture', ''),
        'dated': artifact.get('dated', ''),
        'century': artifact.get('century', ''),
        'division': artifact.get('division', ''),
        'department': artifact.get('department', ''),
        'classification': artifact.get('classification', ''),
        'medium': artifact.get('medium', ''),
        'technique': artifact.get('technique', ''),
        'people': json.dumps(artifact.get('people', [])),
        'url': artifact.get('url', ''),
        'creditline': artifact.get('creditline', '')
    }

def transform_artifact_media(artifact: Dict) -> List[Dict]:
    """Extract media information from artifact."""
    media_list = []
    artifact_id = artifact.get('id')
    
    for media in artifact.get('images', []):
        media_list.append({
            'artifact_id': artifact_id,
            'media_type': 'image',
            'base_url': media.get('baseimageurl', ''),
            'format': media.get('format', ''),
            'width': media.get('width'),
            'height': media.get('height')
        })
    
    return media_list

def transform_artifact_colors(artifact: Dict) -> List[Dict]:
    """Extract color information from artifact."""
    colors_list = []
    artifact_id = artifact.get('id')
    
    for color in artifact.get('colors', []):
        colors_list.append({
            'artifact_id': artifact_id,
            'color': color.get('color', ''),
            'spectrum': color.get('spectrum', ''),
            'percentage': color.get('percent', 0.0)
        })
    
    return colors_list
```

### Load: Insert into Database

```python
def load_metadata_batch(metadata_list: List[Dict]):
    """Batch insert artifact metadata."""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, dated, century, division, department, 
     classification, medium, technique, people, url, creditline)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    values = [
        (m['id'], m['title'], m['culture'], m['dated'], m['century'],
         m['division'], m['department'], m['classification'], m['medium'],
         m['technique'], m['people'], m['url'], m['creditline'])
        for m in metadata_list
    ]
    
    cursor.executemany(query, values)
    conn.commit()
    cursor.close()
    conn.close()

def load_media_batch(media_list: List[Dict]):
    """Batch insert artifact media."""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    query = """
    INSERT INTO artifactmedia 
    (artifact_id, media_type, base_url, format, width, height)
    VALUES (%s, %s, %s, %s, %s, %s)
    """
    
    values = [
        (m['artifact_id'], m['media_type'], m['base_url'], 
         m['format'], m['width'], m['height'])
        for m in media_list
    ]
    
    cursor.executemany(query, values)
    conn.commit()
    cursor.close()
    conn.close()

def load_colors_batch(colors_list: List[Dict]):
    """Batch insert artifact colors."""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    query = """
    INSERT INTO artifactcolors 
    (artifact_id, color, spectrum, percentage)
    VALUES (%s, %s, %s, %s)
    """
    
    values = [
        (c['artifact_id'], c['color'], c['spectrum'], c['percentage'])
        for c in colors_list
    ]
    
    cursor.executemany(query, values)
    conn.commit()
    cursor.close()
    conn.close()
```

### Complete ETL Orchestration

```python
def run_etl_pipeline(num_artifacts: int = 500):
    """Execute complete ETL pipeline."""
    st.info(f"Starting ETL pipeline for {num_artifacts} artifacts...")
    
    # Extract
    st.write("📥 Extracting data from API...")
    artifacts = collect_all_artifacts(max_records=num_artifacts)
    st.success(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    st.write("🔄 Transforming data...")
    metadata_list = [transform_artifact_metadata(a) for a in artifacts]
    media_list = [m for a in artifacts for m in transform_artifact_media(a)]
    colors_list = [c for a in artifacts for c in transform_artifact_colors(a)]
    
    st.success(f"Transformed {len(metadata_list)} metadata, {len(media_list)} media, {len(colors_list)} colors")
    
    # Load
    st.write("💾 Loading data to database...")
    load_metadata_batch(metadata_list)
    load_media_batch(media_list)
    load_colors_batch(colors_list)
    
    st.success("✅ ETL pipeline completed successfully!")
```

## Analytics Queries

### Sample SQL Analytics

```python
# Predefined analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN media_count > 0 THEN 'With Media' ELSE 'No Media' END as media_status,
            COUNT(*) as count
        FROM (
            SELECT m.id, COUNT(am.id) as media_count
            FROM artifactmetadata m
            LEFT JOIN artifactmedia am ON m.id = am.artifact_id
            GROUP BY m.id
        ) as media_stats
        GROUP BY media_status
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Artifacts with Most Images": """
        SELECT m.title, m.culture, COUNT(am.id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia am ON m.id = am.artifact_id
        GROUP BY m.id, m.title, m.culture
        ORDER BY image_count DESC
        LIMIT 10
    """
}

def execute_query(query: str) -> pd.DataFrame:
    """Execute SQL query and return DataFrame."""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard

### Complete App Example

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Engineering & Analytics")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    else:
        show_visualizations_page()

def show_etl_page():
    st.header("📊 ETL Pipeline")
    
    num_artifacts = st.number_input(
        "Number of artifacts to collect",
        min_value=10,
        max_value=5000,
        value=500,
        step=50
    )
    
    if st.button("Run ETL Pipeline"):
        run_etl_pipeline(num_artifacts)

def show_analytics_page():
    st.header("📈 SQL Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Execute Query"):
        query = ANALYTICS_QUERIES[query_name]
        
        st.code(query, language='sql')
        
        with st.spinner("Executing query..."):
            df = execute_query(query)
        
        st.dataframe(df, use_container_width=True)
        
        # Auto-generate visualization
        if len(df.columns) == 2 and df.shape[0] > 0:
            fig = px.bar(
                df,
                x=df.columns[0],
                y=df.columns[1],
                title=query_name
            )
            st.plotly_chart(fig, use_container_width=True)

def show_visualizations_page():
    st.header("📊 Interactive Visualizations")
    
    # Culture distribution
    df_culture = execute_query(ANALYTICS_QUERIES["Artifacts by Culture"])
    fig1 = px.bar(df_culture, x='culture', y='count', title='Top 20 Cultures')
    st.plotly_chart(fig1, use_container_width=True)
    
    # Color usage
    df_colors = execute_query(ANALYTICS_QUERIES["Top Colors Used"])
    fig2 = px.pie(df_colors, values='usage_count', names='color', title='Color Distribution')
    st.plotly_chart(fig2, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Error Handling in ETL

```python
def safe_etl_pipeline(num_artifacts: int):
    """ETL pipeline with error handling."""
    try:
        artifacts = collect_all_artifacts(max_records=num_artifacts)
        
        for artifact in artifacts:
            try:
                metadata = transform_artifact_metadata(artifact)
                load_metadata_batch([metadata])
            except Exception as e:
                st.warning(f"Failed to process artifact {artifact.get('id')}: {e}")
                continue
                
    except requests.RequestException as e:
        st.error(f"API request failed: {e}")
    except mysql.connector.Error as e:
        st.error(f"Database error: {e}")
```

### Incremental Loading

```python
def get_last_loaded_id() -> int:
    """Get the maximum artifact ID already loaded."""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return result[0] if result[0] else 0

def incremental_load():
    """Load only new artifacts since last run."""
    last_id = get_last_loaded_id()
    
    # Fetch artifacts with ID > last_id
    # Implement custom filtering logic
    pass
```

## Troubleshooting

### API Rate Limiting

```python
import time
from functools import wraps

def rate_limit(calls_per_second=2):
    """Decorator to rate limit API calls."""
    min_interval = 1.0 / calls_per_second
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            left_to_wait = min_interval - elapsed
            if left_to_wait > 0:
                time.sleep(left_to_wait)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_second=2)
def fetch_artifacts_limited(page: int, size: int):
    return fetch_artifacts(page, size)
```

### Database Connection Pooling

```python
from mysql.connector import pooling

connection_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    **db_config
)

def get_pooled_connection():
    return connection_pool.get_connection()
```

### Handling Missing Data

```python
def safe_get(data: Dict, key: str, default=''):
    """Safely extract nested values."""
    try:
        value = data.get(key, default)
        return value if value is not None else default
    except:
        return default

def transform_artifact_metadata_safe(artifact: Dict) -> Dict:
    """Transform with null handling."""
    return {
        'id': safe_get(artifact, 'id', 0),
        'title': safe_get(artifact, 'title'),
        'culture': safe_get(artifact, 'culture'),
        # ... other fields
    }
```

This skill provides everything needed to build, extend, and troubleshoot ETL pipelines and analytics applications using the Harvard Art Museums API.
