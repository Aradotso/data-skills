---
name: harvard-artifacts-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - set up analytics dashboard for museum artifact data
  - extract and transform Harvard Art Museums data into SQL
  - create visualizations from Harvard museum collections
  - run SQL analytics on art museum artifacts
  - build a Streamlit app for museum data engineering
  - implement batch data collection from Harvard API
  - analyze artifact metadata with SQL queries
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates a production-ready ETL pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into normalized relational tables, loads it into SQL databases (MySQL/TiDB), and visualizes analytics through an interactive Streamlit dashboard.

**Key capabilities:**
- API data extraction with pagination and rate limiting
- ETL transformations for nested JSON to relational schema
- SQL database design with proper foreign key relationships
- 20+ prebuilt analytical SQL queries
- Interactive Plotly visualizations

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

### 1. API Key Setup

Get your Harvard Art Museums API key from: https://docs.harvardartmuseums.org/

Create a `.env` file or configure environment variables:

```bash
# .env
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

Configure MySQL/TiDB connection in your application:

```python
import os
from dotenv import load_dotenv
import mysql.connector

load_dotenv()

db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

connection = mysql.connector.connect(**db_config)
```

## Database Schema

The ETL pipeline creates three normalized tables:

```sql
-- Artifact metadata
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    department VARCHAR(200),
    classification VARCHAR(200),
    dated VARCHAR(200),
    url TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact media
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    iiifbaseuri TEXT,
    baseimageurl TEXT,
    primaryimageurl TEXT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Artifact colors
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## ETL Pipeline Implementation

### Extract: API Data Collection

```python
import requests
import time

def extract_artifacts(api_key, num_records=100, page_size=100):
    """
    Extract artifact data from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    
    pages_needed = (num_records + page_size - 1) // page_size
    
    for page in range(1, pages_needed + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}/{pages_needed}")
            
            # Rate limiting
            time.sleep(0.5)
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### Transform: Data Normalization

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform nested JSON into normalized dataframes
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        objectid = artifact.get('objectid')
        
        # Metadata extraction
        metadata_records.append({
            'objectid': objectid,
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'period': artifact.get('period', ''),
            'century': artifact.get('century', ''),
            'department': artifact.get('department', ''),
            'classification': artifact.get('classification', ''),
            'dated': artifact.get('dated', ''),
            'url': artifact.get('url', '')
        })
        
        # Media extraction
        media_records.append({
            'objectid': objectid,
            'iiifbaseuri': artifact.get('iiifbaseuri', ''),
            'baseimageurl': artifact.get('baseimageurl', ''),
            'primaryimageurl': artifact.get('primaryimageurl', '')
        })
        
        # Color extraction
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'objectid': objectid,
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load: Batch Database Insert

```python
def load_to_database(connection, metadata_df, media_df, colors_df):
    """
    Load transformed data into SQL database with batch inserts
    """
    cursor = connection.cursor()
    
    # Insert metadata
    metadata_query = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, period, century, department, classification, dated, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    metadata_values = metadata_df.values.tolist()
    cursor.executemany(metadata_query, metadata_values)
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia 
        (objectid, iiifbaseuri, baseimageurl, primaryimageurl)
        VALUES (%s, %s, %s, %s)
    """
    media_values = media_df.values.tolist()
    cursor.executemany(media_query, media_values)
    
    # Insert colors
    if not colors_df.empty:
        colors_query = """
            INSERT INTO artifactcolors 
            (objectid, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        colors_values = colors_df.values.tolist()
        cursor.executemany(colors_query, colors_values)
    
    connection.commit()
    cursor.close()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums ETL Analytics")
    
    # Sidebar navigation
    menu = st.sidebar.selectbox(
        "Navigation",
        ["Home", "Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if menu == "Home":
        show_home()
    elif menu == "Data Collection":
        show_data_collection()
    elif menu == "SQL Analytics":
        show_sql_analytics()
    elif menu == "Visualizations":
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection")
    
    api_key = st.text_input("Harvard API Key", type="password")
    num_records = st.number_input("Number of records", min_value=10, max_value=1000, value=100)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Extracting data..."):
            artifacts = extract_artifacts(api_key, num_records)
            st.success(f"Extracted {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            connection = mysql.connector.connect(**db_config)
            load_to_database(connection, metadata_df, media_df, colors_df)
            connection.close()
            st.success("Data loaded to database")
```

## Analytical SQL Queries

### Example Queries

```python
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY artifact_count DESC
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as color_count, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY color_count DESC
        LIMIT 10
    """,
    
    "Artifacts with Images": """
        SELECT 
            CASE 
                WHEN primaryimageurl != '' THEN 'Has Image'
                ELSE 'No Image'
            END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND department != ''
        GROUP BY department
        ORDER BY artifact_count DESC
    """
}

def execute_query(connection, query):
    """Execute SQL query and return results as DataFrame"""
    return pd.read_sql(query, connection)
```

### Query Execution with Visualization

```python
def show_sql_analytics():
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Run Query"):
        connection = mysql.connector.connect(**db_config)
        
        # Show SQL query
        with st.expander("View SQL Query"):
            st.code(ANALYTICAL_QUERIES[query_name], language="sql")
        
        # Execute and display results
        df = execute_query(connection, ANALYTICAL_QUERIES[query_name])
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(
                df, 
                x=df.columns[0], 
                y=df.columns[1],
                title=query_name
            )
            st.plotly_chart(fig, use_container_width=True)
        
        connection.close()
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def get_latest_objectid(connection):
    """Get the latest objectid to avoid duplicate fetches"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def extract_new_artifacts(api_key, last_objectid):
    """Only fetch artifacts newer than last_objectid"""
    # Implementation with filtering
    pass
```

### Pattern 2: Error Handling in ETL

```python
def safe_etl_pipeline(api_key, num_records):
    """ETL pipeline with comprehensive error handling"""
    try:
        artifacts = extract_artifacts(api_key, num_records)
        if not artifacts:
            raise ValueError("No artifacts extracted")
        
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        connection = mysql.connector.connect(**db_config)
        load_to_database(connection, metadata_df, media_df, colors_df)
        connection.close()
        
        return True, f"Successfully processed {len(artifacts)} artifacts"
    
    except requests.RequestException as e:
        return False, f"API Error: {str(e)}"
    except mysql.connector.Error as e:
        return False, f"Database Error: {str(e)}"
    except Exception as e:
        return False, f"Unexpected Error: {str(e)}"
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# With custom port
streamlit run app.py --server.port 8501

# With auto-reload
streamlit run app.py --server.runOnSave true
```

## Troubleshooting

**API Rate Limiting:**
- Add `time.sleep()` between requests (0.5-1 second recommended)
- Implement exponential backoff for 429 errors

**Database Connection Issues:**
- Verify credentials in environment variables
- Check firewall rules for TiDB Cloud connections
- Ensure SSL/TLS settings if required

**Empty Color Data:**
- Not all artifacts have color information
- Use `WHERE` clauses to filter null/empty values
- Check `hasimage=1` parameter in API calls

**Memory Issues with Large Datasets:**
- Process data in batches (100-500 records)
- Use `cursor.executemany()` for bulk inserts
- Clear DataFrames after loading: `del metadata_df`
