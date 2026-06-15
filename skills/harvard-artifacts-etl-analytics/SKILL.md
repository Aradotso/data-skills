---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with MySQL and Streamlit
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - set up a data engineering pipeline for museum artifacts
  - create analytics dashboard for Harvard Art Museums data
  - extract and transform Harvard museum collection data
  - build Streamlit app for artifact data visualization
  - query and analyze Harvard Art Museums API data
  - set up SQL database for museum artifacts ETL
  - visualize Harvard museum collection with Plotly
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It covers ETL pipeline construction, SQL database design, analytical query execution, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App demonstrates a complete data pipeline:

1. **Extract**: Fetch artifact data from Harvard Art Museums API with pagination
2. **Transform**: Convert nested JSON to relational database schema
3. **Load**: Batch insert into MySQL/TiDB Cloud
4. **Analyze**: Execute SQL queries for insights
5. **Visualize**: Display results in interactive Streamlit dashboards

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

Required packages:
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Connection
DB_HOST=your_db_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

Get your Harvard API key: https://docs.api.harvardartmuseums.org/

### Database Setup

Create the database schema:

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    creditline TEXT,
    primaryimageurl TEXT,
    totalpageviews INT,
    totaluniquepageviews INT
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl TEXT,
    renditionnumber VARCHAR(50),
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

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

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, num_pages=5, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        api_key: Harvard API key
        num_pages: Number of pages to fetch
        page_size: Items per page (max 100)
    
    Returns:
        List of artifact records
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only fetch artifacts with images
        }
        
        try:
            response = requests.get(base_url, params=params, timeout=30)
            response.raise_for_status()
            data = response.json()
            
            if 'records' in data:
                all_artifacts.extend(data['records'])
                print(f"Fetched page {page}/{num_pages}: {len(data['records'])} records")
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts
```

### 2. Data Transformation

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform nested JSON into relational dataframes
    
    Returns:
        tuple: (metadata_df, media_df, colors_df)
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'period': artifact.get('period', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'division': artifact.get('division', '')[:200],
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'creditline': artifact.get('creditline', ''),
            'primaryimageurl': artifact.get('primaryimageurl', ''),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_records.append(metadata)
        
        # Extract media (images)
        artifact_id = artifact.get('id')
        if artifact.get('images'):
            for img in artifact['images']:
                media = {
                    'artifact_id': artifact_id,
                    'iiifbaseuri': img.get('iiifbaseuri', '')[:500],
                    'baseimageurl': img.get('baseimageurl', ''),
                    'renditionnumber': img.get('renditionnumber', ''),
                    'format': img.get('format', '')
                }
                media_records.append(media)
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_data = {
                    'artifact_id': artifact_id,
                    'color': color.get('color', ''),
                    'spectrum': color.get('spectrum', ''),
                    'hue': color.get('hue', ''),
                    'percent': color.get('percent', 0.0)
                }
                color_records.append(color_data)
    
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
    """Create database connection from environment variables"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_to_database(metadata_df, media_df, colors_df):
    """
    Batch insert dataframes into SQL database
    
    Args:
        metadata_df: Artifact metadata DataFrame
        media_df: Media/images DataFrame
        colors_df: Colors DataFrame
    """
    conn = get_db_connection()
    if not conn:
        return False
    
    cursor = conn.cursor()
    
    try:
        # Insert metadata (batch)
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, classification, department, 
             dated, division, technique, medium, creditline, primaryimageurl, 
             totalpageviews, totaluniquepageviews)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        if not media_df.empty:
            media_query = """
                INSERT INTO artifactmedia 
                (artifact_id, iiifbaseuri, baseimageurl, renditionnumber, format)
                VALUES (%s, %s, %s, %s, %s)
            """
            cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        if not colors_df.empty:
            colors_query = """
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """
            cursor.executemany(colors_query, colors_df.values.tolist())
        
        conn.commit()
        print(f"Successfully loaded {len(metadata_df)} artifacts")
        return True
        
    except Error as e:
        print(f"Database load error: {e}")
        conn.rollback()
        return False
        
    finally:
        cursor.close()
        conn.close()
```

### 4. Analytical Queries

```python
def execute_query(query):
    """
    Execute SQL query and return results as DataFrame
    
    Args:
        query: SQL query string
    
    Returns:
        pandas.DataFrame or None
    """
    conn = get_db_connection()
    if not conn:
        return None
    
    try:
        df = pd.read_sql(query, conn)
        return df
    except Error as e:
        print(f"Query execution error: {e}")
        return None
    finally:
        conn.close()

# Sample analytical queries
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
    
    "Top Viewed Artifacts": """
        SELECT title, culture, century, totalpageviews
        FROM artifactmetadata
        WHERE totalpageviews > 0
        ORDER BY totalpageviews DESC
        LIMIT 15
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Artifacts with Most Images": """
        SELECT m.id, m.title, m.culture, COUNT(a.media_id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia a ON m.id = a.artifact_id
        GROUP BY m.id, m.title, m.culture
        ORDER BY image_count DESC
        LIMIT 10
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND department != ''
        GROUP BY department
        ORDER BY count DESC
    """
}
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar for ETL operations
    st.sidebar.header("ETL Pipeline")
    
    if st.sidebar.button("Fetch & Load Data"):
        with st.spinner("Fetching artifacts from API..."):
            api_key = os.getenv('HARVARD_API_KEY')
            raw_data = fetch_artifacts(api_key, num_pages=5)
            
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            
        with st.spinner("Loading to database..."):
            success = load_to_database(metadata_df, media_df, colors_df)
            
        if success:
            st.sidebar.success(f"✅ Loaded {len(metadata_df)} artifacts")
        else:
            st.sidebar.error("❌ Load failed")
    
    # Analytics section
    st.header("📊 Analytics Dashboard")
    
    # Query selector
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        
        with st.spinner("Executing query..."):
            df = execute_query(query)
        
        if df is not None and not df.empty:
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Auto-visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df.head(20), 
                    x=df.columns[0], 
                    y=df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)
        else:
            st.warning("No results found")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Pipeline

```python
from dotenv import load_dotenv
import os

def run_etl_pipeline(num_pages=10):
    """Execute complete ETL pipeline"""
    load_dotenv()
    
    # Extract
    print("1. Extracting data from API...")
    api_key = os.getenv('HARVARD_API_KEY')
    raw_data = fetch_artifacts(api_key, num_pages=num_pages)
    
    # Transform
    print("2. Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_data)
    
    # Load
    print("3. Loading to database...")
    success = load_to_database(metadata_df, media_df, colors_df)
    
    if success:
        print(f"✅ ETL complete: {len(metadata_df)} artifacts processed")
    else:
        print("❌ ETL failed")
    
    return success

# Run pipeline
if __name__ == "__main__":
    run_etl_pipeline(num_pages=5)
```

### Custom Analytics Query

```python
def analyze_culture_century_distribution():
    """Custom query for culture-century cross-analysis"""
    query = """
        SELECT 
            culture,
            century,
            COUNT(*) as artifact_count,
            AVG(totalpageviews) as avg_views
        FROM artifactmetadata
        WHERE culture IS NOT NULL 
          AND century IS NOT NULL
          AND culture != ''
          AND century != ''
        GROUP BY culture, century
        HAVING artifact_count > 5
        ORDER BY artifact_count DESC
        LIMIT 50
    """
    
    df = execute_query(query)
    return df
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_artifacts_with_retry(api_key, num_pages=5, retry_delay=2):
    """Fetch with automatic retry on rate limit"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        max_retries = 3
        for attempt in range(max_retries):
            try:
                response = requests.get(base_url, params=params, timeout=30)
                
                if response.status_code == 429:  # Rate limited
                    wait_time = retry_delay * (attempt + 1)
                    print(f"Rate limited. Waiting {wait_time}s...")
                    time.sleep(wait_time)
                    continue
                    
                response.raise_for_status()
                data = response.json()
                all_artifacts.extend(data.get('records', []))
                break
                
            except requests.exceptions.RequestException as e:
                if attempt == max_retries - 1:
                    print(f"Failed after {max_retries} attempts: {e}")
                time.sleep(retry_delay)
    
    return all_artifacts
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        if conn and conn.is_connected():
            cursor = conn.cursor()
            cursor.execute("SELECT VERSION()")
            version = cursor.fetchone()
            print(f"✅ Connected to MySQL: {version[0]}")
            cursor.close()
            conn.close()
            return True
    except Error as e:
        print(f"❌ Connection failed: {e}")
        return False
```

### Missing Data Handling

```python
def transform_artifacts_safe(raw_data):
    """Transform with comprehensive null handling"""
    metadata_records = []
    
    for artifact in raw_data:
        metadata = {
            'id': artifact.get('id', 0),
            'title': (artifact.get('title') or 'Untitled')[:500],
            'culture': (artifact.get('culture') or '')[:200],
            'period': (artifact.get('period') or '')[:200],
            'century': (artifact.get('century') or '')[:100],
            # ... handle all fields with defaults
        }
        
        # Validate required fields
        if metadata['id'] > 0:
            metadata_records.append(metadata)
    
    return pd.DataFrame(metadata_records)
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Run ETL pipeline only
python etl_pipeline.py

# Test database connection
python -c "from utils import test_db_connection; test_db_connection()"
```

This skill provides everything needed to build production-ready ETL pipelines and analytics dashboards for museum artifact data.
