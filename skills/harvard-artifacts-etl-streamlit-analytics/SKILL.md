---
name: harvard-artifacts-etl-streamlit-analytics
description: Build end-to-end ETL pipelines with Harvard Art Museums API, SQL databases, and Streamlit dashboards for artifact analytics
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create a streamlit dashboard for museum artifact data
  - extract and load Harvard museum data into SQL
  - build analytics app with Harvard Art Museums collection
  - set up artifact data pipeline with Python and MySQL
  - visualize museum collection data with Plotly and Streamlit
  - analyze Harvard Art Museums API data with SQL queries
  - create museum artifact analytics dashboard
---

# Harvard Artifacts ETL & Streamlit Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming it into relational tables, loading into SQL databases (MySQL/TiDB), running analytics queries, and visualizing results through an interactive Streamlit dashboard.

The application handles pagination, rate limiting, nested JSON transformation, and batch SQL operations for real-world ETL scenarios.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### API Key Setup

1. Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api
2. Store credentials in environment variables or `.env` file:

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Setup

```python
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

# Database connection configuration
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Create tables
def create_tables():
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            period VARCHAR(200),
            url VARCHAR(500)
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            media_type VARCHAR(100),
            media_url VARCHAR(500),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            percentage FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    conn.commit()
    cursor.close()
    conn.close()
```

## API Data Extraction

### Fetch Artifacts with Pagination

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    Handles pagination and rate limiting
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

def collect_multiple_pages(api_key, num_pages=5):
    """Collect data from multiple pages"""
    all_records = []
    
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(api_key, page=page)
        all_records.extend(data.get('records', []))
        print(f"Fetched page {page}: {len(data.get('records', []))} records")
    
    return all_records
```

## ETL Pipeline

### Transform and Load Data

```python
import pandas as pd

def transform_artifact_data(records):
    """Transform nested JSON into relational dataframes"""
    
    metadata_list = []
    media_list = []
    color_list = []
    
    for record in records:
        # Extract metadata
        metadata = {
            'id': record.get('id'),
            'title': record.get('title', '')[:500],
            'culture': record.get('culture', '')[:200],
            'century': record.get('century', '')[:100],
            'classification': record.get('classification', '')[:200],
            'department': record.get('department', '')[:200],
            'dated': record.get('dated', '')[:200],
            'period': record.get('period', '')[:200],
            'url': record.get('url', '')[:500]
        }
        metadata_list.append(metadata)
        
        # Extract media
        for image in record.get('images', []):
            media = {
                'artifact_id': record.get('id'),
                'media_type': 'image',
                'media_url': image.get('baseimageurl', '')[:500]
            }
            media_list.append(media)
        
        # Extract colors
        for color in record.get('colors', []):
            color_data = {
                'artifact_id': record.get('id'),
                'color': color.get('color', '')[:50],
                'percentage': color.get('percent', 0.0)
            }
            color_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(color_list)
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Batch insert dataframes into SQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Insert metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, dated, period, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """, tuple(row))
    
    # Insert media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia (artifact_id, media_type, media_url)
            VALUES (%s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors (artifact_id, color, percentage)
            VALUES (%s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
    print(f"Loaded {len(metadata_df)} artifacts to database")
```

## SQL Analytics Queries

### Predefined Analytics Queries

```python
ANALYTICS_QUERIES = {
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
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Most Common Colors": """
        SELECT color, COUNT(*) as frequency, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT am.id) as total_artifacts,
            COUNT(DISTINCT media.artifact_id) as artifacts_with_media
        FROM artifactmetadata am
        LEFT JOIN artifactmedia media ON am.id = media.artifact_id
    """,
    
    "Artifacts with Most Images": """
        SELECT 
            am.title,
            am.culture,
            COUNT(media.id) as image_count
        FROM artifactmetadata am
        JOIN artifactmedia media ON am.id = media.artifact_id
        GROUP BY am.id, am.title, am.culture
        ORDER BY image_count DESC
        LIMIT 10
    """
}

def execute_query(query_name):
    """Execute analytics query and return results as dataframe"""
    conn = get_db_connection()
    query = ANALYTICS_QUERIES[query_name]
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard

### Main Application

```python
import streamlit as st
import plotly.express as px
from dotenv import load_dotenv

load_dotenv()

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar for ETL operations
with st.sidebar:
    st.header("ETL Operations")
    
    api_key = os.getenv('HARVARD_API_KEY')
    num_pages = st.number_input("Pages to fetch", min_value=1, max_value=20, value=5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            records = collect_multiple_pages(api_key, num_pages)
            st.success(f"Fetched {len(records)} records")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifact_data(records)
            st.success("Data transformed")
        
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
            st.success("ETL Complete!")

# Main analytics section
st.header("📊 Analytics Dashboard")

query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))

if st.button("Run Query"):
    df = execute_query(query_name)
    
    # Display table
    st.subheader("Results")
    st.dataframe(df)
    
    # Generate visualization
    if len(df) > 0 and len(df.columns) >= 2:
        fig = px.bar(
            df.head(15),
            x=df.columns[0],
            y=df.columns[1],
            title=query_name,
            labels={df.columns[0]: df.columns[0].title(), df.columns[1]: "Count"}
        )
        st.plotly_chart(fig, use_container_width=True)
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Incremental ETL

```python
def get_latest_artifact_id():
    """Get the most recent artifact ID in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    return result or 0

def incremental_load(api_key):
    """Only load new artifacts"""
    latest_id = get_latest_artifact_id()
    # Fetch only artifacts with ID > latest_id
    # Implementation depends on API filtering capabilities
```

### Error Handling

```python
def safe_etl_pipeline(api_key, num_pages):
    """ETL with error handling and logging"""
    try:
        records = collect_multiple_pages(api_key, num_pages)
        metadata_df, media_df, colors_df = transform_artifact_data(records)
        load_to_database(metadata_df, media_df, colors_df)
        return True, f"Successfully loaded {len(records)} records"
    except requests.exceptions.RequestException as e:
        return False, f"API Error: {str(e)}"
    except mysql.connector.Error as e:
        return False, f"Database Error: {str(e)}"
    except Exception as e:
        return False, f"Unexpected Error: {str(e)}"
```

## Troubleshooting

**API Rate Limiting**: Add delays between requests
```python
import time
time.sleep(0.5)  # 500ms delay between API calls
```

**Database Connection Issues**: Check environment variables and network access
```python
# Test connection
try:
    conn = get_db_connection()
    print("Database connection successful")
    conn.close()
except Exception as e:
    print(f"Connection failed: {e}")
```

**Large Dataset Memory Issues**: Process in smaller batches
```python
def batch_insert(df, batch_size=100):
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        # Insert batch
```

**Missing Data Fields**: Handle null values in transformation
```python
metadata['culture'] = record.get('culture') or 'Unknown'
```
