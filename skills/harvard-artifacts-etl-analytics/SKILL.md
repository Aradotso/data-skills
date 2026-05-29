---
name: harvard-artifacts-etl-analytics
description: Build end-to-end data pipelines using Harvard Art Museums API with ETL, SQL storage, and Streamlit visualization
triggers:
  - build a data pipeline with Harvard Art Museums API
  - create ETL pipeline for museum artifacts
  - set up analytics dashboard for Harvard artifacts
  - extract and analyze Harvard museum data
  - build Streamlit app with SQL analytics
  - integrate Harvard Art Museums API with database
  - create artifact data warehouse
  - visualize museum collection analytics
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is an end-to-end data engineering project that demonstrates real-world ETL pipelines, SQL analytics, and interactive visualization. It extracts artifact data from the Harvard Art Museums API, transforms and loads it into a relational database (MySQL/TiDB), and provides an analytics dashboard built with Streamlit.

**Architecture Flow:** API → ETL → SQL Database → Analytics Queries → Streamlit Visualization

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
export DB_NAME="your_database_name"
```

**Required packages:**
- streamlit
- pandas
- requests
- mysql-connector-python (or pymysql)
- plotly
- python-dotenv

## Database Schema

The project uses three main tables with foreign key relationships:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    period VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    PRIMARY KEY (artifact_id)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(1000),
    primaryimageurl VARCHAR(1000),
    imagepermissionlevel INT,
    totalpageviews INT,
    totaluniquepageviews INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);
```

## Core ETL Pipeline

### 1. Extract Data from API

```python
import requests
import os
from typing import List, Dict

def fetch_artifacts(api_key: str, page: int = 1, size: int = 100) -> Dict:
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

def collect_all_artifacts(api_key: str, max_pages: int = 10) -> List[Dict]:
    """Collect artifacts across multiple pages with rate limiting"""
    import time
    
    all_artifacts = []
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(data.get('records', []))
        
        # Rate limiting
        time.sleep(1)
        
        # Check if we've reached the last page
        if page >= data.get('info', {}).get('pages', 0):
            break
    
    return all_artifacts
```

### 2. Transform Data

```python
import pandas as pd

def transform_artifacts(raw_data: List[Dict]) -> tuple:
    """Transform raw API data into structured dataframes"""
    
    # Metadata transformation
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        artifact_id = artifact.get('id')
        
        # Extract metadata
        metadata_records.append({
            'artifact_id': artifact_id,
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'technique': artifact.get('technique', ''),
            'period': artifact.get('period', ''),
            'division': artifact.get('division', ''),
            'dated': artifact.get('dated', '')
        })
        
        # Extract media information
        media_records.append({
            'artifact_id': artifact_id,
            'baseimageurl': artifact.get('baseimageurl', ''),
            'primaryimageurl': artifact.get('primaryimageurl', ''),
            'imagepermissionlevel': artifact.get('imagepermissionlevel', 0),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        })
        
        # Extract color information
        colors = artifact.get('colors', [])
        for color_data in colors:
            color_records.append({
                'artifact_id': artifact_id,
                'color': color_data.get('color', ''),
                'spectrum': color_data.get('spectrum', ''),
                'hue': color_data.get('hue', ''),
                'percent': color_data.get('percent', 0.0)
            })
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### 3. Load Data to Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def load_to_database(df_metadata: pd.DataFrame, 
                     df_media: pd.DataFrame, 
                     df_colors: pd.DataFrame):
    """Batch load dataframes into SQL database"""
    conn = create_database_connection()
    if not conn:
        return False
    
    cursor = conn.cursor()
    
    try:
        # Load metadata (parent table first)
        for _, row in df_metadata.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (artifact_id, title, culture, century, classification, 
                 department, technique, period, division, dated)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE
                title=VALUES(title), culture=VALUES(culture)
            """, tuple(row))
        
        # Load media
        for _, row in df_media.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (artifact_id, baseimageurl, primaryimageurl, 
                 imagepermissionlevel, totalpageviews, totaluniquepageviews)
                VALUES (%s, %s, %s, %s, %s, %s)
            """, tuple(row))
        
        # Load colors
        for _, row in df_colors.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
        
        conn.commit()
        print("Data loaded successfully!")
        return True
        
    except Error as e:
        print(f"Error loading data: {e}")
        conn.rollback()
        return False
    finally:
        cursor.close()
        conn.close()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Artifacts Analytics Dashboard")
    st.markdown("End-to-end data pipeline: API → ETL → SQL → Analytics")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "Analytics Dashboard", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection_page()
    elif page == "Analytics Dashboard":
        show_analytics_page()
    else:
        show_visualizations_page()

def show_data_collection_page():
    st.header("📥 Data Collection")
    
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=os.getenv('HARVARD_API_KEY', ''))
    max_pages = st.slider("Number of pages to collect", 1, 50, 10)
    
    if st.button("Start Collection"):
        with st.spinner("Collecting artifacts..."):
            artifacts = collect_all_artifacts(api_key, max_pages)
            st.success(f"Collected {len(artifacts)} artifacts!")
            
            # Transform
            df_metadata, df_media, df_colors = transform_artifacts(artifacts)
            
            # Load
            if load_to_database(df_metadata, df_media, df_colors):
                st.success("Data loaded to database!")
            else:
                st.error("Failed to load data")

def show_analytics_page():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture 
            ORDER BY count DESC 
            LIMIT 15
        """,
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century 
            ORDER BY count DESC 
            LIMIT 10
        """,
        "Most Common Colors": """
            SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
            FROM artifactcolors
            WHERE color IS NOT NULL
            GROUP BY color
            ORDER BY frequency DESC
            LIMIT 10
        """,
        "Department Distribution": """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY artifact_count DESC
        """,
        "Image Availability": """
            SELECT 
                CASE 
                    WHEN primaryimageurl IS NOT NULL AND primaryimageurl != '' 
                    THEN 'Has Image' 
                    ELSE 'No Image' 
                END as image_status,
                COUNT(*) as count
            FROM artifactmedia
            GROUP BY image_status
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        conn = create_database_connection()
        if conn:
            df_result = pd.read_sql(queries[selected_query], conn)
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) == 2:
                fig = px.bar(df_result, x=df_result.columns[0], 
                            y=df_result.columns[1],
                            title=selected_query)
                st.plotly_chart(fig)
            
            conn.close()

if __name__ == "__main__":
    main()
```

## Sample Analytics Queries

```python
# Top 10 most viewed artifacts
query_most_viewed = """
SELECT m.title, m.culture, med.totalpageviews
FROM artifactmetadata m
JOIN artifactmedia med ON m.artifact_id = med.artifact_id
ORDER BY med.totalpageviews DESC
LIMIT 10
"""

# Color distribution analysis
query_color_analysis = """
SELECT c.color, COUNT(DISTINCT c.artifact_id) as artifact_count,
       AVG(c.percent) as avg_percentage
FROM artifactcolors c
GROUP BY c.color
HAVING artifact_count > 5
ORDER BY artifact_count DESC
"""

# Artifacts without images
query_no_images = """
SELECT COUNT(*) as no_image_count
FROM artifactmedia
WHERE primaryimageurl IS NULL OR primaryimageurl = ''
"""

# Classification by department
query_dept_classification = """
SELECT department, classification, COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL AND classification IS NOT NULL
GROUP BY department, classification
ORDER BY department, count DESC
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def get_last_artifact_id(conn):
    """Get the highest artifact_id to resume collection"""
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(artifact_id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(api_key: str):
    """Load only new artifacts"""
    conn = create_database_connection()
    last_id = get_last_artifact_id(conn)
    conn.close()
    
    # Fetch artifacts with ID greater than last_id
    # Implementation depends on API capabilities
```

### Pattern 2: Error Handling in ETL

```python
def safe_etl_pipeline(api_key: str, max_pages: int):
    """ETL with comprehensive error handling"""
    try:
        # Extract
        artifacts = collect_all_artifacts(api_key, max_pages)
        if not artifacts:
            raise ValueError("No artifacts collected")
        
        # Transform
        df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        
        # Validate
        assert not df_metadata.empty, "Metadata is empty"
        assert df_metadata['artifact_id'].nunique() == len(df_metadata), "Duplicate IDs"
        
        # Load
        success = load_to_database(df_metadata, df_media, df_colors)
        return success
        
    except requests.RequestException as e:
        print(f"API Error: {e}")
        return False
    except Exception as e:
        print(f"Pipeline Error: {e}")
        return False
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session_with_retry():
    session = requests.Session()
    retry = Retry(total=5, backoff_factor=1, status_forcelist=[429, 500, 502, 503, 504])
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('https://', adapter)
    return session
```

### Database Connection Issues
```python
# Use connection pooling
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_connection():
    return db_pool.get_connection()
```

### Memory Management for Large Datasets
```python
def batch_process_artifacts(artifacts: List[Dict], batch_size: int = 1000):
    """Process artifacts in batches to manage memory"""
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        df_metadata, df_media, df_colors = transform_artifacts(batch)
        load_to_database(df_metadata, df_media, df_colors)
        print(f"Processed batch {i//batch_size + 1}")
```
