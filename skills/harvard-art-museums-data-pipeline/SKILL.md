---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - how do I extract data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit and museum data
  - work with Harvard Art Museums collection data
  - implement SQL analytics on artifact collections
  - visualize museum data with Plotly and Streamlit
  - set up data engineering pipeline for art collections
  - query and analyze Harvard Art Museums artifacts
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Extract artifact metadata from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transform nested JSON into relational database tables
- **SQL Storage**: Store artifacts, media, and color data in MySQL/TiDB Cloud
- **Analytics Queries**: 20+ predefined SQL queries for insights
- **Interactive Dashboard**: Streamlit UI with Plotly visualizations

Architecture: `API → ETL → SQL → Analytics → Visualization`

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install requirements
pip install -r requirements.txt
```

### Environment Configuration

Create a `.env` file in the project root:

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

Get your Harvard Art Museums API key from: https://docs.harvardartmuseums.org/

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Extract artifact data from Harvard Art Museums API
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
print(f"Total records: {data['info']['totalrecords']}")
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd

def transform_artifact_metadata(raw_data):
    """
    Transform API response into structured metadata
    """
    artifacts = []
    
    for record in raw_data.get('records', []):
        artifact = {
            'objectid': record.get('objectid'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'period': record.get('period'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'medium': record.get('medium'),
            'dimensions': record.get('dimensions'),
            'dated': record.get('dated'),
            'department': record.get('department'),
            'division': record.get('division'),
            'contact': record.get('contact'),
            'creditline': record.get('creditline')
        }
        artifacts.append(artifact)
    
    return pd.DataFrame(artifacts)

def transform_artifact_media(raw_data):
    """
    Extract media information for each artifact
    """
    media_records = []
    
    for record in raw_data.get('records', []):
        objectid = record.get('objectid')
        images = record.get('images', [])
        
        for image in images:
            media = {
                'objectid': objectid,
                'imageid': image.get('imageid'),
                'baseimageurl': image.get('baseimageurl'),
                'width': image.get('width'),
                'height': image.get('height'),
                'format': image.get('format')
            }
            media_records.append(media)
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(raw_data):
    """
    Extract color palette data
    """
    color_records = []
    
    for record in raw_data.get('records', []):
        objectid = record.get('objectid')
        colors = record.get('colors', [])
        
        for color in colors:
            color_data = {
                'objectid': objectid,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(color_data)
    
    return pd.DataFrame(color_records)
```

### 3. SQL Database Operations

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """
    Create connection to MySQL/TiDB database
    """
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

def create_tables(connection):
    """
    Create database schema
    """
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            period VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            medium TEXT,
            dimensions VARCHAR(500),
            dated VARCHAR(255),
            department VARCHAR(255),
            division VARCHAR(255),
            contact VARCHAR(500),
            creditline TEXT
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            imageid INT,
            baseimageurl VARCHAR(1000),
            width INT,
            height INT,
            format VARCHAR(50),
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            hue VARCHAR(100),
            percent DECIMAL(5,2),
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    connection.commit()

def batch_insert_artifacts(connection, df):
    """
    Batch insert artifact metadata
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, period, century, classification, 
         medium, dimensions, dated, department, division, contact, creditline)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    records = df.values.tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    
    return cursor.rowcount
```

### 4. Analytics Queries

```python
def run_analytics_query(connection, query_name):
    """
    Execute predefined analytics queries
    """
    queries = {
        'top_cultures': """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        
        'century_distribution': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'media_availability': """
            SELECT 
                CASE WHEN imageid IS NOT NULL THEN 'Has Media' ELSE 'No Media' END as media_status,
                COUNT(DISTINCT objectid) as artifact_count
            FROM artifactmetadata am
            LEFT JOIN artifactmedia media ON am.objectid = media.objectid
            GROUP BY media_status
        """,
        
        'color_distribution': """
            SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY color
            ORDER BY usage_count DESC
            LIMIT 15
        """,
        
        'department_breakdown': """
            SELECT department, COUNT(*) as total_artifacts
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY total_artifacts DESC
        """
    }
    
    cursor = connection.cursor(dictionary=True)
    cursor.execute(queries[query_name])
    results = cursor.fetchall()
    
    return pd.DataFrame(results)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """
    Main Streamlit dashboard application
    """
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                     value=os.getenv('HARVARD_API_KEY', ''))
    
    # Database connection
    conn = create_database_connection()
    
    if conn:
        st.sidebar.success("✅ Database Connected")
        
        # Data Collection Section
        st.header("📥 Data Collection")
        col1, col2 = st.columns(2)
        
        with col1:
            num_pages = st.number_input("Number of pages to fetch", 1, 100, 5)
        
        with col2:
            if st.button("Fetch & Load Data"):
                with st.spinner("Fetching data from API..."):
                    for page in range(1, num_pages + 1):
                        raw_data = fetch_artifacts(api_key, page=page)
                        
                        # Transform
                        metadata_df = transform_artifact_metadata(raw_data)
                        media_df = transform_artifact_media(raw_data)
                        colors_df = transform_artifact_colors(raw_data)
                        
                        # Load
                        batch_insert_artifacts(conn, metadata_df)
                        
                    st.success(f"✅ Loaded {num_pages} pages successfully!")
        
        # Analytics Section
        st.header("📊 Analytics Queries")
        
        query_option = st.selectbox(
            "Select Analysis",
            ["Top Cultures", "Century Distribution", "Media Availability", 
             "Color Distribution", "Department Breakdown"]
        )
        
        query_map = {
            "Top Cultures": "top_cultures",
            "Century Distribution": "century_distribution",
            "Media Availability": "media_availability",
            "Color Distribution": "color_distribution",
            "Department Breakdown": "department_breakdown"
        }
        
        if st.button("Run Query"):
            df_results = run_analytics_query(conn, query_map[query_option])
            
            # Display results
            st.dataframe(df_results)
            
            # Visualization
            if not df_results.empty:
                x_col = df_results.columns[0]
                y_col = df_results.columns[1]
                
                fig = px.bar(df_results, x=x_col, y=y_col, 
                            title=f"{query_option} Analysis")
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    create_dashboard()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Complete ETL Workflow

```python
def full_etl_pipeline(api_key, db_connection, num_pages=10):
    """
    Complete ETL pipeline from API to database
    """
    total_loaded = 0
    
    for page in range(1, num_pages + 1):
        # Extract
        raw_data = fetch_artifacts(api_key, page=page, size=100)
        
        # Transform
        metadata_df = transform_artifact_metadata(raw_data)
        media_df = transform_artifact_media(raw_data)
        colors_df = transform_artifact_colors(raw_data)
        
        # Load
        if not metadata_df.empty:
            batch_insert_artifacts(db_connection, metadata_df)
            total_loaded += len(metadata_df)
        
        print(f"Processed page {page}/{num_pages}")
    
    return total_loaded
```

### Error Handling and Retry Logic

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """
    API fetch with exponential backoff
    """
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1} after {wait_time}s: {e}")
                time.sleep(wait_time)
            else:
                raise
```

## Troubleshooting

**API Rate Limiting**
- Harvard API has rate limits; add delays between requests
- Use `time.sleep(1)` between pagination calls

**Database Connection Issues**
- Verify environment variables are set correctly
- Check firewall rules for TiDB Cloud connections
- Ensure database exists before running schema creation

**Missing Data**
- Some artifacts may have null fields; handle with `df.fillna('')`
- Use `WHERE field IS NOT NULL` in SQL queries

**Memory Issues with Large Datasets**
- Process data in batches rather than loading all at once
- Use `cursor.fetchmany()` for large query results

**Streamlit Caching**
```python
@st.cache_data
def load_query_results(query_name):
    conn = create_database_connection()
    return run_analytics_query(conn, query_name)
```

This skill provides the foundation for building production-ready data engineering pipelines with museum artifact data, SQL analytics, and interactive visualizations.
