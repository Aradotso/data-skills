---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create a data engineering project with museum artifacts
  - set up analytics dashboard for Harvard Art Museums data
  - extract and load Harvard museum data into SQL database
  - build a Streamlit app with Harvard Art Museums API
  - analyze art museum collections with Python and SQL
  - create visualizations from Harvard Art Museums data
  - design relational database schema for museum artifacts
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering and analytics application that demonstrates:
- **ETL Pipeline**: Extract data from Harvard Art Museums API, transform nested JSON into relational format, load into SQL databases
- **SQL Analytics**: Run analytical queries on structured artifact data
- **Interactive Visualization**: Build Streamlit dashboards with Plotly charts
- **Database Design**: Implement normalized schemas with proper foreign key relationships

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

### Prerequisites
- Python 3.8+
- MySQL or TiDB Cloud account
- Harvard Art Museums API key (get from https://www.harvardartmuseums.org/collections/api)

### Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_db_host"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
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

## Database Schema

The project uses three main tables with relational integrity:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    creditline TEXT,
    url VARCHAR(500)
);

-- Artifact media table (one-to-many relationship)
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    publiccaption TEXT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Artifact colors table (one-to-many relationship)
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

## Core Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination.
    
    Args:
        api_key: Harvard API key from environment
        page: Current page number
        size: Number of records per page
    
    Returns:
        JSON response with artifact records
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
total_records = data['info']['totalrecords']
artifacts = data['records']
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

def transform_artifacts(raw_data: List[Dict]) -> tuple:
    """
    Transform nested JSON data into normalized dataframes.
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'period': artifact.get('period', '')[:255],
            'century': artifact.get('century', '')[:255],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'creditline': artifact.get('creditline', ''),
            'url': artifact.get('url', '')[:500]
        }
        metadata_list.append(metadata)
        
        # Extract media (one-to-many)
        for image in artifact.get('images', []):
            media = {
                'objectid': artifact.get('objectid'),
                'baseimageurl': image.get('baseimageurl', '')[:500],
                'iiifbaseuri': image.get('iiifbaseuri', '')[:500],
                'publiccaption': image.get('publiccaption', '')
            }
            media_list.append(media)
        
        # Extract colors (one-to-many)
        for color in artifact.get('colors', []):
            color_data = {
                'objectid': artifact.get('objectid'),
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'hue': color.get('hue', '')[:50],
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """
    Load transformed dataframes into MySQL database.
    """
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    # Insert metadata (parent table first)
    for _, row in metadata_df.iterrows():
        sql = """
        INSERT IGNORE INTO artifactmetadata 
        (objectid, title, culture, period, century, classification, department, dated, creditline, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """
        cursor.execute(sql, tuple(row))
    
    # Insert media (child table)
    for _, row in media_df.iterrows():
        sql = """
        INSERT INTO artifactmedia 
        (objectid, baseimageurl, iiifbaseuri, publiccaption)
        VALUES (%s, %s, %s, %s)
        """
        cursor.execute(sql, tuple(row))
    
    # Insert colors (child table)
    for _, row in colors_df.iterrows():
        sql = """
        INSERT INTO artifactcolors 
        (objectid, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.execute(sql, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
```

### 3. Complete ETL Workflow

```python
def run_etl_pipeline(api_key, db_config, num_pages=5):
    """
    Execute complete ETL pipeline: Extract, Transform, Load.
    """
    all_artifacts = []
    
    # Extract
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page, size=100)
        all_artifacts.extend(data['records'])
    
    print(f"Extracted {len(all_artifacts)} artifacts")
    
    # Transform
    metadata_df, media_df, colors_df = transform_artifacts(all_artifacts)
    print(f"Transformed data: {len(metadata_df)} metadata, {len(media_df)} media, {len(colors_df)} colors")
    
    # Load
    load_to_database(metadata_df, media_df, colors_df, db_config)
    print("Data loaded successfully!")

# Execute
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

run_etl_pipeline(os.getenv('HARVARD_API_KEY'), db_config, num_pages=5)
```

## Analytics Queries

### Common SQL Patterns

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "top_colors": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "artifacts_with_images": """
        SELECT 
            a.classification,
            COUNT(DISTINCT a.objectid) as total_artifacts,
            COUNT(DISTINCT m.objectid) as with_images,
            ROUND(COUNT(DISTINCT m.objectid) * 100.0 / COUNT(DISTINCT a.objectid), 2) as image_percentage
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.objectid = m.objectid
        WHERE a.classification IS NOT NULL
        GROUP BY a.classification
        ORDER BY total_artifacts DESC
        LIMIT 20
    """,
    
    "department_statistics": """
        SELECT 
            department,
            COUNT(*) as artifact_count,
            COUNT(DISTINCT period) as period_diversity,
            COUNT(DISTINCT culture) as culture_diversity
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """
}

def execute_query(query_name, db_config):
    """Execute analytical query and return results as DataFrame."""
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(ANALYTICS_QUERIES[query_name], conn)
    conn.close()
    return df
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    st.title("🎨 Harvard Art Museums Data Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    # Query selection
    query_options = list(ANALYTICS_QUERIES.keys())
    selected_query = st.sidebar.selectbox("Select Analysis", query_options)
    
    if st.sidebar.button("Run Analysis"):
        with st.spinner("Executing query..."):
            df = execute_query(selected_query, db_config)
            
            # Display results
            st.subheader(f"Results: {selected_query.replace('_', ' ').title()}")
            st.dataframe(df, use_container_width=True)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                            title=f"{selected_query.replace('_', ' ').title()}")
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

### Advanced Dashboard with ETL Controls

```python
def create_etl_tab():
    """ETL pipeline control interface."""
    st.header("ETL Pipeline Manager")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Number of pages to fetch", 
                                     min_value=1, max_value=100, value=5)
        api_key = os.getenv('HARVARD_API_KEY')
        
        if st.button("Start ETL Process"):
            progress_bar = st.progress(0)
            status_text = st.empty()
            
            try:
                # Extract
                status_text.text("Extracting data from API...")
                all_artifacts = []
                for i in range(num_pages):
                    data = fetch_artifacts(api_key, page=i+1)
                    all_artifacts.extend(data['records'])
                    progress_bar.progress((i + 1) / num_pages * 0.33)
                
                # Transform
                status_text.text("Transforming data...")
                metadata_df, media_df, colors_df = transform_artifacts(all_artifacts)
                progress_bar.progress(0.66)
                
                # Load
                status_text.text("Loading into database...")
                db_config = {
                    'host': os.getenv('DB_HOST'),
                    'user': os.getenv('DB_USER'),
                    'password': os.getenv('DB_PASSWORD'),
                    'database': os.getenv('DB_NAME')
                }
                load_to_database(metadata_df, media_df, colors_df, db_config)
                progress_bar.progress(1.0)
                
                status_text.text("ETL Complete!")
                st.success(f"Loaded {len(metadata_df)} artifacts successfully!")
                
            except Exception as e:
                st.error(f"ETL Error: {str(e)}")
    
    with col2:
        st.info("**ETL Pipeline Steps:**\n\n"
                "1. **Extract**: Fetch data from Harvard API\n"
                "2. **Transform**: Normalize JSON to relational format\n"
                "3. **Load**: Insert into MySQL database")

# Add to main app
def main():
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    
    tab1, tab2 = st.tabs(["Analytics Dashboard", "ETL Pipeline"])
    
    with tab1:
        # Analytics code here
        pass
    
    with tab2:
        create_etl_tab()
```

## Configuration

### Environment Variables

Create a `.env` file:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_db_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

Load in your application:

```python
from dotenv import load_dotenv
import os

load_dotenv()

# Access variables
api_key = os.getenv('HARVARD_API_KEY')
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

## Running the Application

```bash
# Run Streamlit dashboard
streamlit run app.py

# Run ETL pipeline standalone
python etl_pipeline.py

# Run with custom configuration
streamlit run app.py -- --server.port 8080
```

## Common Patterns

### Batch Processing with Rate Limiting

```python
import time

def batch_fetch_artifacts(api_key, total_pages, batch_size=10, delay=1):
    """
    Fetch artifacts in batches with rate limiting to avoid API throttling.
    """
    all_artifacts = []
    
    for batch_start in range(0, total_pages, batch_size):
        batch_end = min(batch_start + batch_size, total_pages)
        
        for page in range(batch_start + 1, batch_end + 1):
            data = fetch_artifacts(api_key, page=page)
            all_artifacts.extend(data['records'])
            time.sleep(delay)  # Rate limiting
        
        print(f"Completed batch {batch_start//batch_size + 1}")
    
    return all_artifacts
```

### Error Handling and Retry Logic

```python
from requests.exceptions import RequestException
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch data with exponential backoff retry logic."""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s...")
            time.sleep(wait_time)
```

### Incremental Data Loading

```python
def get_max_objectid(db_config):
    """Get the maximum objectid already loaded."""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    cursor.execute("SELECT COALESCE(MAX(objectid), 0) FROM artifactmetadata")
    max_id = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    return max_id

def incremental_etl(api_key, db_config):
    """Load only new artifacts not already in database."""
    max_existing_id = get_max_objectid(db_config)
    
    # Fetch all artifacts
    artifacts = batch_fetch_artifacts(api_key, total_pages=10)
    
    # Filter new artifacts
    new_artifacts = [a for a in artifacts if a['objectid'] > max_existing_id]
    
    if new_artifacts:
        metadata_df, media_df, colors_df = transform_artifacts(new_artifacts)
        load_to_database(metadata_df, media_df, colors_df, db_config)
        print(f"Loaded {len(new_artifacts)} new artifacts")
    else:
        print("No new artifacts to load")
```

## Troubleshooting

### API Rate Limiting
**Issue**: HTTP 429 Too Many Requests

**Solution**:
```python
# Add delay between requests
import time
time.sleep(1)  # 1 second delay

# Use batch processing with rate limiting
artifacts = batch_fetch_artifacts(api_key, total_pages=50, delay=2)
```

### Database Connection Issues
**Issue**: `mysql.connector.errors.DatabaseError`

**Solution**:
```python
# Test connection
try:
    conn = mysql.connector.connect(**db_config)
    print("Database connected successfully")
    conn.close()
except mysql.connector.Error as err:
    print(f"Database error: {err}")
    
# Use connection pooling for production
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    **db_config
)
```

### Memory Issues with Large Datasets
**Issue**: Out of memory when processing large result sets

**Solution**:
```python
# Process in chunks
def chunk_transform(artifacts, chunk_size=1000):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        metadata_df, media_df, colors_df = transform_artifacts(chunk)
        load_to_database(metadata_df, media_df, colors_df, db_config)
        print(f"Processed {i+len(chunk)}/{len(artifacts)}")
```

### Duplicate Data Prevention
**Issue**: Duplicate entries on re-runs

**Solution**:
```python
# Use INSERT IGNORE or REPLACE
cursor.execute("""
    INSERT IGNORE INTO artifactmetadata 
    (objectid, title, culture, ...)
    VALUES (%s, %s, %s, ...)
""", tuple(row))

# Or use ON DUPLICATE KEY UPDATE
cursor.execute("""
    INSERT INTO artifactmetadata (objectid, title, culture, ...)
    VALUES (%s, %s, %s, ...)
    ON DUPLICATE KEY UPDATE title=VALUES(title), culture=VALUES(culture)
""", tuple(row))
```

## Best Practices

1. **Always use environment variables** for sensitive data (API keys, DB credentials)
2. **Implement proper error handling** with retry logic for API calls
3. **Use batch processing** to handle large datasets efficiently
4. **Add logging** for production ETL pipelines
5. **Validate data** before loading into database
6. **Create indexes** on frequently queried columns (culture, century, classification)
7. **Use connection pooling** for database connections in production
8. **Implement incremental loads** to avoid reprocessing existing data
