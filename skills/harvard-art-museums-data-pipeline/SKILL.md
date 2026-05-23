---
name: harvard-art-museums-data-pipeline
description: End-to-end data engineering pipeline using Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline with the Harvard Art Museums API
  - create ETL pipeline for museum artifacts data
  - set up Harvard artifacts collection analytics app
  - implement art museum data engineering workflow
  - query and visualize Harvard Art Museums data
  - design museum artifact database schema
  - build Streamlit dashboard for art collection data
  - extract and analyze museum API data
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming it into relational tables, loading into SQL databases, and visualizing analytics through a Streamlit dashboard.

## What It Does

- **API Integration**: Fetches artifact metadata, media, and color information from Harvard Art Museums API
- **ETL Pipeline**: Transforms nested JSON into normalized relational data
- **Database Design**: Creates three related tables (metadata, media, colors) with proper foreign keys
- **SQL Analytics**: Executes 20+ analytical queries for insights
- **Interactive Visualization**: Streamlit dashboard with Plotly charts

Architecture: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Sign up at [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api) to get your API key.

Store it as an environment variable:
```bash
export HARVARD_API_KEY="your_api_key_here"
```

Or use a `.env` file:
```
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Connection

Configure your MySQL/TiDB Cloud connection:

```python
import os
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_art'),
    'port': int(os.getenv('DB_PORT', 3306))
}

conn = mysql.connector.connect(**db_config)
```

Environment variables needed:
```
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_art
DB_PORT=3306
```

## Database Schema

### Create Tables

```python
import mysql.connector

def create_tables(connection):
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS artifactmetadata (
        id INT PRIMARY KEY,
        title VARCHAR(500),
        culture VARCHAR(200),
        century VARCHAR(100),
        classification VARCHAR(200),
        department VARCHAR(200),
        technique VARCHAR(300),
        medium VARCHAR(300),
        dimensions VARCHAR(300),
        dated VARCHAR(200),
        url VARCHAR(500),
        INDEX idx_culture (culture),
        INDEX idx_century (century),
        INDEX idx_department (department)
    )
    """)
    
    # Artifact Media Table
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS artifactmedia (
        id INT AUTO_INCREMENT PRIMARY KEY,
        artifact_id INT,
        media_type VARCHAR(50),
        baseimageurl VARCHAR(500),
        iiifbaseuri VARCHAR(500),
        FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE,
        INDEX idx_artifact_id (artifact_id)
    )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS artifactcolors (
        id INT AUTO_INCREMENT PRIMARY KEY,
        artifact_id INT,
        color VARCHAR(50),
        spectrum VARCHAR(50),
        percent FLOAT,
        FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE,
        INDEX idx_artifact_id (artifact_id),
        INDEX idx_color (color)
    )
    """)
    
    connection.commit()
    cursor.close()
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os

def fetch_artifacts(api_key, size=100, page=1):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
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
        print(f"API Error: {e}")
        return None
```

### Transform: Process JSON to DataFrames

```python
import pandas as pd

def transform_artifacts(records):
    """
    Transform API records into normalized dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in records:
        artifact_id = record.get('id')
        
        # Extract metadata
        metadata_list.append({
            'id': artifact_id,
            'title': record.get('title', '')[:500],
            'culture': record.get('culture', '')[:200],
            'century': record.get('century', '')[:100],
            'classification': record.get('classification', '')[:200],
            'department': record.get('department', '')[:200],
            'technique': record.get('technique', '')[:300],
            'medium': record.get('medium', '')[:300],
            'dimensions': record.get('dimensions', '')[:300],
            'dated': record.get('dated', '')[:200],
            'url': record.get('url', '')[:500]
        })
        
        # Extract media
        images = record.get('images', [])
        for img in images:
            media_list.append({
                'artifact_id': artifact_id,
                'media_type': 'image',
                'baseimageurl': img.get('baseimageurl', '')[:500],
                'iiifbaseuri': img.get('iiifbaseuri', '')[:500]
            })
        
        # Extract colors
        colors = record.get('colors', [])
        for color in colors:
            colors_list.append({
                'artifact_id': artifact_id,
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'percent': color.get('percent', 0.0)
            })
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

### Load: Insert into Database

```python
def load_to_database(df_metadata, df_media, df_colors, connection):
    """
    Load dataframes into database tables using batch inserts
    """
    cursor = connection.cursor()
    
    try:
        # Insert metadata (use REPLACE to handle duplicates)
        if not df_metadata.empty:
            metadata_query = """
            REPLACE INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             technique, medium, dimensions, dated, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            """
            cursor.executemany(metadata_query, df_metadata.values.tolist())
        
        # Insert media
        if not df_media.empty:
            media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, media_type, baseimageurl, iiifbaseuri)
            VALUES (%s, %s, %s, %s)
            """
            cursor.executemany(media_query, df_media.values.tolist())
        
        # Insert colors
        if not df_colors.empty:
            colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, percent)
            VALUES (%s, %s, %s, %s)
            """
            cursor.executemany(colors_query, df_colors.values.tolist())
        
        connection.commit()
        return True
    except Exception as e:
        connection.rollback()
        print(f"Database Error: {e}")
        return False
    finally:
        cursor.close()
```

## Complete ETL Workflow

```python
def run_etl_pipeline(api_key, db_config, num_pages=5):
    """
    Execute complete ETL pipeline
    """
    # Connect to database
    conn = mysql.connector.connect(**db_config)
    create_tables(conn)
    
    total_loaded = 0
    
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}/{num_pages}...")
        
        # Extract
        result = fetch_artifacts(api_key, size=100, page=page)
        if not result or not result['records']:
            break
        
        # Transform
        df_meta, df_media, df_colors = transform_artifacts(result['records'])
        
        # Load
        success = load_to_database(df_meta, df_media, df_colors, conn)
        if success:
            total_loaded += len(df_meta)
            print(f"Loaded {len(df_meta)} artifacts")
    
    conn.close()
    print(f"ETL Complete: {total_loaded} total artifacts loaded")
```

## SQL Analytics Queries

### Example Analytical Queries

```python
# Top 10 cultures by artifact count
QUERY_TOP_CULTURES = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Artifacts by century distribution
QUERY_CENTURY_DISTRIBUTION = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL AND century != ''
GROUP BY century
ORDER BY count DESC
"""

# Most common colors across artifacts
QUERY_TOP_COLORS = """
SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
FROM artifactcolors
GROUP BY color
ORDER BY frequency DESC
LIMIT 10
"""

# Artifacts with media availability
QUERY_MEDIA_STATS = """
SELECT 
    COUNT(DISTINCT m.id) as total_artifacts,
    COUNT(DISTINCT med.artifact_id) as artifacts_with_media,
    ROUND(COUNT(DISTINCT med.artifact_id) * 100.0 / COUNT(DISTINCT m.id), 2) as media_percentage
FROM artifactmetadata m
LEFT JOIN artifactmedia med ON m.id = med.artifact_id
"""

# Department-wise classification
QUERY_DEPT_CLASSIFICATION = """
SELECT department, classification, COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL AND classification IS NOT NULL
GROUP BY department, classification
ORDER BY department, count DESC
"""
```

### Execute Queries

```python
def execute_analytics_query(query, connection):
    """
    Execute analytical query and return results as DataFrame
    """
    try:
        df = pd.read_sql(query, connection)
        return df
    except Exception as e:
        print(f"Query Error: {e}")
        return pd.DataFrame()
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import plotly.express as px
import mysql.connector
import os

st.set_page_config(page_title="Harvard Art Analytics", layout="wide")

# Database connection
@st.cache_resource
def get_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar for navigation
page = st.sidebar.selectbox(
    "Select Analysis",
    ["Overview", "Culture Analysis", "Color Patterns", "Department Stats"]
)

conn = get_connection()

if page == "Overview":
    col1, col2, col3 = st.columns(3)
    
    # Total artifacts
    total = pd.read_sql("SELECT COUNT(*) as total FROM artifactmetadata", conn)
    col1.metric("Total Artifacts", total['total'].iloc[0])
    
    # Unique cultures
    cultures = pd.read_sql("SELECT COUNT(DISTINCT culture) as count FROM artifactmetadata", conn)
    col2.metric("Unique Cultures", cultures['count'].iloc[0])
    
    # Artifacts with images
    with_images = pd.read_sql("SELECT COUNT(DISTINCT artifact_id) as count FROM artifactmedia", conn)
    col3.metric("With Images", with_images['count'].iloc[0])

elif page == "Culture Analysis":
    st.subheader("Top Cultures by Artifact Count")
    
    df = pd.read_sql(QUERY_TOP_CULTURES, conn)
    
    fig = px.bar(df, x='culture', y='artifact_count',
                 title='Top 10 Cultures',
                 labels={'artifact_count': 'Number of Artifacts'})
    st.plotly_chart(fig, use_container_width=True)
    
    st.dataframe(df, use_container_width=True)
```

### Visualization Patterns

```python
# Bar chart for categorical data
def create_bar_chart(df, x_col, y_col, title):
    fig = px.bar(df, x=x_col, y=y_col, title=title)
    fig.update_layout(xaxis_tickangle=-45)
    return fig

# Pie chart for proportions
def create_pie_chart(df, names_col, values_col, title):
    fig = px.pie(df, names=names_col, values=values_col, title=title)
    return fig

# Histogram for distributions
def create_histogram(df, column, title):
    fig = px.histogram(df, x=column, title=title)
    return fig
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY="your_api_key"
export DB_HOST="localhost"
export DB_USER="root"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_art"

# Run Streamlit app
streamlit run app.py
```

## Common Patterns

### Rate Limiting for API Calls

```python
import time

def fetch_with_rate_limit(api_key, pages, delay=1):
    """
    Fetch multiple pages with rate limiting
    """
    all_records = []
    for page in range(1, pages + 1):
        result = fetch_artifacts(api_key, page=page)
        if result:
            all_records.extend(result['records'])
        time.sleep(delay)  # Respect API rate limits
    return all_records
```

### Error Handling in ETL

```python
def safe_etl_run(api_key, db_config):
    """
    ETL with comprehensive error handling
    """
    conn = None
    try:
        conn = mysql.connector.connect(**db_config)
        create_tables(conn)
        
        result = fetch_artifacts(api_key)
        if not result:
            raise Exception("Failed to fetch data from API")
        
        df_meta, df_media, df_colors = transform_artifacts(result['records'])
        
        if df_meta.empty:
            raise Exception("No data to load")
        
        load_to_database(df_meta, df_media, df_colors, conn)
        
        return {"status": "success", "records": len(df_meta)}
        
    except Exception as e:
        return {"status": "error", "message": str(e)}
    finally:
        if conn:
            conn.close()
```

## Troubleshooting

### API Connection Issues

```python
# Test API connection
def test_api_connection(api_key):
    try:
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'size': 1},
            timeout=10
        )
        if response.status_code == 200:
            print("✓ API connection successful")
            return True
        elif response.status_code == 401:
            print("✗ Invalid API key")
        else:
            print(f"✗ API returned status code: {response.status_code}")
        return False
    except Exception as e:
        print(f"✗ Connection error: {e}")
        return False
```

### Database Connection Issues

```python
# Test database connection
def test_db_connection(db_config):
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        print("✓ Database connection successful")
        return True
    except mysql.connector.Error as e:
        print(f"✗ Database error: {e}")
        return False
```

### Data Quality Checks

```python
def validate_data_quality(df_metadata):
    """
    Check data quality before loading
    """
    issues = []
    
    if df_metadata['id'].duplicated().any():
        issues.append("Duplicate artifact IDs found")
    
    if df_metadata['id'].isnull().any():
        issues.append("Null artifact IDs found")
    
    if len(df_metadata) == 0:
        issues.append("Empty dataset")
    
    return issues if issues else None
```

This skill provides a complete reference for building and operating the Harvard Art Museums data pipeline with ETL, SQL analytics, and interactive dashboards.
