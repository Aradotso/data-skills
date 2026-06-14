---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create a data engineering project with museum artifacts API
  - build analytics dashboard with Harvard Art Museums API
  - how to extract and load Harvard museum data to SQL
  - visualize art collection data with Streamlit
  - set up Harvard Art Museums data pipeline
  - analyze museum artifacts with SQL and Python
  - create interactive art data visualization dashboard
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates end-to-end data engineering: extracting artifact data from the Harvard Art Museums API, transforming it into relational structures, loading into SQL databases, and building interactive analytics dashboards with Streamlit.

## What This Project Does

- **Data Collection**: Fetches artifact metadata, media, and color data from Harvard Art Museums API
- **ETL Pipeline**: Extracts nested JSON, transforms to relational format, loads to MySQL/TiDB
- **SQL Analytics**: Runs predefined analytical queries on structured artifact data
- **Visualization**: Creates interactive Plotly charts and Streamlit dashboards
- **Real-world Simulation**: Models production data pipeline architecture

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Get your free API key from: https://www.harvardartmuseums.org/collections/api

Store it in environment variables or `.env` file:

```bash
# .env
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

Set up MySQL or TiDB Cloud connection:

```python
# config.py or .env
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### 3. Database Schema Setup

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    url TEXT,
    lastupdate DATETIME
);

-- Media table with foreign key
CREATE TABLE artifactmedia (
    artifact_id INT,
    media_id INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id),
    INDEX idx_artifact_id (artifact_id)
);

-- Colors table with foreign key
CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id),
    INDEX idx_artifact_id (artifact_id)
);
```

## Core ETL Pipeline

### Extract: Fetching from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Extract artifact data from Harvard Art Museums API
    Handles pagination and rate limiting
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = 'https://api.harvardartmuseums.org/object'
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status()
        data = response.json()
        
        return {
            'records': data.get('records', []),
            'total_pages': data.get('info', {}).get('pages', 1),
            'total_records': data.get('info', {}).get('totalrecords', 0)
        }
    except requests.exceptions.RequestException as e:
        print(f"API request failed: {e}")
        return None

# Fetch multiple pages
def fetch_all_artifacts(max_pages=10):
    """Collect artifacts across multiple pages"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        result = fetch_artifacts(page=page)
        if result and result['records']:
            all_artifacts.extend(result['records'])
            print(f"Fetched page {page}/{max_pages} - {len(result['records'])} records")
        else:
            break
    
    return all_artifacts
```

### Transform: Data Cleaning and Structuring

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """
    Transform nested JSON into relational dataframes
    Returns three normalized dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'division': artifact.get('division', 'Unknown'),
            'dated': artifact.get('dated', ''),
            'url': artifact.get('url', ''),
            'lastupdate': artifact.get('lastupdate', '')
        }
        metadata_list.append(metadata)
        
        # Extract media (images)
        if artifact.get('images'):
            for image in artifact['images']:
                media = {
                    'artifact_id': artifact.get('id'),
                    'media_id': image.get('imageid'),
                    'baseimageurl': image.get('baseimageurl', ''),
                    'format': image.get('format', '')
                }
                media_list.append(media)
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_record = {
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color', ''),
                    'spectrum': color.get('spectrum', ''),
                    'percent': color.get('percent', 0.0)
                }
                colors_list.append(color_record)
    
    return {
        'metadata': pd.DataFrame(metadata_list),
        'media': pd.DataFrame(media_list),
        'colors': pd.DataFrame(colors_list)
    }
```

### Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def create_db_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT', 3306),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME', 'harvard_artifacts')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_to_database(dataframes):
    """
    Load transformed dataframes to SQL database
    Uses batch inserts for performance
    """
    connection = create_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Insert metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, division, dated, url, lastupdate)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture), century=VALUES(century)
        """
        metadata_values = dataframes['metadata'].values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        media_query = """
        INSERT INTO artifactmedia (artifact_id, media_id, baseimageurl, format)
        VALUES (%s, %s, %s, %s)
        """
        if not dataframes['media'].empty:
            media_values = dataframes['media'].values.tolist()
            cursor.executemany(media_query, media_values)
        
        # Insert colors
        colors_query = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, percent)
        VALUES (%s, %s, %s, %s)
        """
        if not dataframes['colors'].empty:
            colors_values = dataframes['colors'].values.tolist()
            cursor.executemany(colors_query, colors_values)
        
        connection.commit()
        print(f"Loaded {len(metadata_values)} artifacts to database")
        return True
        
    except Error as e:
        print(f"Database insert error: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

## Streamlit Analytics Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose Function",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    elif page == "Visualizations":
        show_visualization_page()

def show_etl_page():
    st.header("📥 ETL Pipeline")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Number of pages to fetch", 1, 100, 5)
    
    with col2:
        if st.button("🚀 Run ETL Pipeline"):
            with st.spinner("Extracting data from API..."):
                artifacts = fetch_all_artifacts(max_pages=num_pages)
                st.success(f"✅ Extracted {len(artifacts)} artifacts")
            
            with st.spinner("Transforming data..."):
                dataframes = transform_artifacts(artifacts)
                st.success("✅ Data transformed to relational format")
            
            with st.spinner("Loading to database..."):
                success = load_to_database(dataframes)
                if success:
                    st.success("✅ Data loaded to SQL database")
                else:
                    st.error("❌ Database load failed")

if __name__ == "__main__":
    main()
```

### SQL Analytics Queries

```python
# Predefined analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != 'Unknown'
        GROUP BY century
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts with Most Images": """
        SELECT m.id, m.title, COUNT(a.media_id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia a ON m.id = a.artifact_id
        GROUP BY m.id, m.title
        ORDER BY image_count DESC
        LIMIT 20
    """,
    
    "Color Spectrum Analysis": """
        SELECT spectrum, COUNT(*) as count, AVG(percent) as avg_coverage
        FROM artifactcolors
        GROUP BY spectrum
        ORDER BY count DESC
    """
}

def show_analytics_page():
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Execute Query"):
        connection = create_db_connection()
        if connection:
            df = pd.read_sql(ANALYTICS_QUERIES[query_name], connection)
            connection.close()
            
            st.subheader("Query Results")
            st.dataframe(df, use_container_width=True)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df.head(15), 
                    x=df.columns[0], 
                    y=df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Pattern 1: Full ETL Workflow

```python
def complete_etl_pipeline(num_pages=5):
    """End-to-end ETL execution"""
    # Extract
    print("Step 1: Extracting...")
    artifacts = fetch_all_artifacts(max_pages=num_pages)
    
    # Transform
    print("Step 2: Transforming...")
    dataframes = transform_artifacts(artifacts)
    
    # Load
    print("Step 3: Loading...")
    success = load_to_database(dataframes)
    
    return success
```

### Pattern 2: Incremental Data Updates

```python
def incremental_update():
    """Only fetch artifacts updated since last run"""
    connection = create_db_connection()
    cursor = connection.cursor()
    
    # Get last update timestamp
    cursor.execute("SELECT MAX(lastupdate) FROM artifactmetadata")
    last_update = cursor.fetchone()[0]
    
    # Fetch only new/updated records
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'updatedafter': last_update
    }
    # Continue with ETL...
```

### Pattern 3: Error-Resilient API Calls

```python
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            result = fetch_artifacts(page=page)
            return result
        except Exception as e:
            wait_time = 2 ** attempt
            print(f"Retry {attempt + 1} after {wait_time}s: {e}")
            time.sleep(wait_time)
    return None
```

## Troubleshooting

### Issue: API Rate Limiting

```python
# Add delay between requests
import time

def fetch_all_artifacts_safe(max_pages=10, delay=1):
    all_artifacts = []
    for page in range(1, max_pages + 1):
        result = fetch_artifacts(page=page)
        if result:
            all_artifacts.extend(result['records'])
        time.sleep(delay)  # Respect rate limits
    return all_artifacts
```

### Issue: Database Connection Timeout

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

def get_connection_from_pool():
    return db_pool.get_connection()
```

### Issue: Large Dataset Memory Issues

```python
# Process in chunks
def load_in_chunks(dataframes, chunk_size=1000):
    metadata_df = dataframes['metadata']
    
    for i in range(0, len(metadata_df), chunk_size):
        chunk = metadata_df.iloc[i:i+chunk_size]
        # Load chunk to database
        print(f"Processing records {i} to {i+chunk_size}")
```

### Issue: Missing Environment Variables

```python
# Validate configuration on startup
def validate_config():
    required_vars = ['HARVARD_API_KEY', 'DB_HOST', 'DB_USER', 'DB_PASSWORD']
    missing = [var for var in required_vars if not os.getenv(var)]
    
    if missing:
        raise EnvironmentError(f"Missing environment variables: {', '.join(missing)}")
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY=your_key
export DB_HOST=localhost
export DB_USER=root
export DB_PASSWORD=your_password

# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```
