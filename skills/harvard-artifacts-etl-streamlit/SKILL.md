---
name: harvard-artifacts-etl-streamlit
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create a data engineering app with Harvard API and Streamlit
  - how to fetch and analyze Harvard artifacts with SQL
  - build analytics dashboard for museum collection data
  - extract transform load Harvard Art Museums artifacts
  - create interactive visualization for museum API data
  - set up TiDB pipeline for Harvard artifacts
  - query and visualize Harvard museum data
---

# Harvard Artifacts ETL & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables you to build end-to-end data engineering pipelines using the Harvard Art Museums API. It covers data extraction, transformation, SQL storage, analytics queries, and interactive Streamlit dashboards.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App demonstrates:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination
- **ETL Pipeline**: Extract nested JSON, transform to relational format, load into SQL
- **Database Design**: Store artifacts, media, and color data in normalized tables
- **SQL Analytics**: Execute 20+ predefined analytical queries
- **Interactive Dashboards**: Visualize query results with Plotly in Streamlit

**Architecture**: `API → ETL → SQL → Analytics → Visualization`

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
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

**Key Dependencies**:
- `streamlit` - Web app framework
- `pandas` - Data transformation
- `requests` - API calls
- `mysql-connector-python` or `pymysql` - Database connection
- `plotly` - Visualizations

## Configuration

### API Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

```python
import os
import requests

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts with pagination"""
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size
    }
    response = requests.get(BASE_URL, params=params)
    response.raise_for_status()
    return response.json()
```

### Database Connection

```python
import mysql.connector
from mysql.connector import Error

def create_db_connection():
    """Create MySQL/TiDB connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None
```

### Table Schema

```sql
-- Artifact metadata table
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(500),
    dated VARCHAR(100),
    url TEXT
);

-- Artifact media table
CREATE TABLE IF NOT EXISTS artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl TEXT,
    iiifbaseuri TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE IF NOT EXISTS artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import time

def extract_all_artifacts(max_pages=10, delay=1):
    """Extract artifacts with rate limiting"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(page=page, size=100)
            records = data.get('records', [])
            all_artifacts.extend(records)
            
            print(f"Fetched page {page}: {len(records)} artifacts")
            
            # Rate limiting
            time.sleep(delay)
            
            # Check if more pages exist
            if page >= data.get('info', {}).get('pages', 0):
                break
                
        except Exception as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts
```

### Transform: Convert to Relational Format

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """Transform nested JSON to relational DataFrames"""
    
    # Metadata DataFrame
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url')
        })
        
        # Extract media
        images = artifact.get('images', [])
        for img in images:
            media_list.append({
                'artifact_id': artifact.get('id'),
                'media_type': 'image',
                'baseimageurl': img.get('baseimageurl'),
                'iiifbaseuri': img.get('iiifbaseuri')
            })
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            colors_list.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Batch Insert to SQL

```python
def load_to_database(df_metadata, df_media, df_colors):
    """Load DataFrames to SQL database"""
    connection = create_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Insert metadata (batch)
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, period, technique, dated, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        metadata_values = df_metadata.values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, media_type, baseimageurl, iiifbaseuri)
        VALUES (%s, %s, %s, %s)
        """
        if not df_media.empty:
            media_values = df_media.values.tolist()
            cursor.executemany(media_query, media_values)
        
        # Insert colors
        colors_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        if not df_colors.empty:
            colors_values = df_colors.values.tolist()
            cursor.executemany(colors_query, colors_values)
        
        connection.commit()
        print(f"Loaded {len(df_metadata)} artifacts successfully")
        return True
        
    except Error as e:
        print(f"Database load error: {e}")
        connection.rollback()
        return False
        
    finally:
        cursor.close()
        connection.close()
```

## Analytics Queries

### Sample SQL Queries

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY century
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Artifacts with Images": """
        SELECT 
            CASE WHEN m.artifact_id IS NOT NULL THEN 'With Images' ELSE 'Without Images' END as image_status,
            COUNT(DISTINCT a.id) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY image_status
    """
}

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    connection = create_db_connection()
    if not connection:
        return None
    
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query execution error: {e}")
        return None
    finally:
        connection.close()
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar navigation
    menu = st.sidebar.selectbox(
        "Navigation",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if menu == "ETL Pipeline":
        show_etl_page()
    elif menu == "SQL Analytics":
        show_analytics_page()
    elif menu == "Visualizations":
        show_viz_page()

def show_etl_page():
    """ETL Pipeline Interface"""
    st.header("ETL Pipeline")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Number of pages to fetch", 1, 50, 5)
    
    with col2:
        delay = st.number_input("Delay between requests (sec)", 0.5, 5.0, 1.0)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            raw_data = extract_all_artifacts(max_pages=num_pages, delay=delay)
            st.success(f"Extracted {len(raw_data)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_meta, df_media, df_colors = transform_artifacts(raw_data)
            st.success(f"Transformed into {len(df_meta)} metadata, {len(df_media)} media, {len(df_colors)} color records")
        
        with st.spinner("Loading to database..."):
            success = load_to_database(df_meta, df_media, df_colors)
            if success:
                st.success("✅ Data loaded successfully!")

def show_analytics_page():
    """SQL Analytics Interface"""
    st.header("SQL Analytics Dashboard")
    
    query_name = st.selectbox("Select Query", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Execute Query"):
        query = ANALYTICS_QUERIES[query_name]
        
        with st.spinner("Executing query..."):
            df = execute_query(query)
            
            if df is not None and not df.empty:
                st.dataframe(df)
                
                # Auto-generate visualization
                if len(df.columns) == 2:
                    fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                                title=query_name)
                    st.plotly_chart(fig, use_container_width=True)
            else:
                st.warning("No results found")

def show_viz_page():
    """Custom Visualizations"""
    st.header("Interactive Visualizations")
    
    # Example: Color spectrum analysis
    df = execute_query("SELECT spectrum, COUNT(*) as count FROM artifactcolors GROUP BY spectrum")
    
    if df is not None:
        fig = px.pie(df, values='count', names='spectrum', 
                    title='Color Spectrum Distribution')
        st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Incremental ETL

```python
def incremental_etl(last_update_timestamp):
    """Load only new/updated artifacts"""
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'updatedafter': last_update_timestamp
    }
    response = requests.get(BASE_URL, params=params)
    return response.json()
```

### Error Handling with Retry Logic

```python
from time import sleep

def fetch_with_retry(url, params, max_retries=3):
    """Fetch data with exponential backoff"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Retry {attempt + 1} after {wait_time}s...")
            sleep(wait_time)
```

### Data Quality Checks

```python
def validate_artifacts(df):
    """Validate data quality"""
    checks = {
        'null_ids': df['id'].isnull().sum(),
        'duplicate_ids': df['id'].duplicated().sum(),
        'missing_titles': df['title'].isnull().sum()
    }
    
    for check, count in checks.items():
        if count > 0:
            print(f"⚠️ {check}: {count}")
    
    return all(v == 0 for v in checks.values())
```

## Troubleshooting

### API Rate Limiting
```python
# Add delays between requests
time.sleep(1)  # 1 second delay

# Or use exponential backoff
import backoff

@backoff.on_exception(backoff.expo, requests.exceptions.RequestException, max_tries=5)
def robust_fetch(url, params):
    return requests.get(url, params=params)
```

### Database Connection Issues
```python
# Use connection pooling
from mysql.connector import pooling

connection_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_connection():
    return connection_pool.get_connection()
```

### Memory Issues with Large Datasets
```python
# Process in chunks
def process_in_chunks(artifacts, chunk_size=1000):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        df_meta, df_media, df_colors = transform_artifacts(chunk)
        load_to_database(df_meta, df_media, df_colors)
        print(f"Processed chunk {i // chunk_size + 1}")
```

### Streamlit Caching for Performance
```python
@st.cache_data(ttl=3600)
def cached_query(query):
    """Cache query results for 1 hour"""
    return execute_query(query)
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Key Features to Implement

1. **Data Collection**: Paginated API fetching with rate limiting
2. **ETL**: Transform nested JSON → relational tables
3. **Storage**: MySQL/TiDB with proper schema design
4. **Analytics**: Pre-built and custom SQL queries
5. **Visualization**: Plotly charts integrated with Streamlit
6. **Monitoring**: Log ETL runs, track data quality metrics

This skill enables you to build production-ready data engineering pipelines for museum collections or any similar API-based data sources.
