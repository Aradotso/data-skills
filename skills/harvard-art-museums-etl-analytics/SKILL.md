---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I extract data from the Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create analytics dashboard with Streamlit and SQL
  - process Harvard Art Museums API data with Python
  - set up artifact metadata database pipeline
  - visualize museum collection data with Plotly
  - implement pagination for Harvard API requests
  - transform nested JSON to relational database tables
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App is a complete data pipeline that:

- **Extracts** artifact data from the Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables (metadata, media, colors)
- **Loads** data into MySQL/TiDB Cloud databases with proper foreign key relationships
- **Analyzes** data using 20+ predefined SQL queries
- **Visualizes** results through interactive Plotly charts in a Streamlit interface

## Installation

### Prerequisites

```bash
# Install required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Environment Setup

Create a `.env` file with your credentials:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Setup

```sql
-- Create the database
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
    url VARCHAR(500),
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    imagepermissionlevel INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(total_records=100, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        total_records: Total number of records to fetch
        page_size: Number of records per API call (max 100)
    
    Returns:
        List of artifact dictionaries
    """
    artifacts = []
    pages = (total_records + page_size - 1) // page_size
    
    for page in range(1, pages + 1):
        params = {
            'apikey': API_KEY,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only fetch artifacts with images
        }
        
        try:
            response = requests.get(BASE_URL, params=params, timeout=30)
            response.raise_for_status()
            data = response.json()
            
            if 'records' in data:
                artifacts.extend(data['records'])
                print(f"Fetched page {page}/{pages}: {len(data['records'])} records")
            
            # Rate limiting - be respectful to the API
            if page < pages:
                import time
                time.sleep(0.5)
                
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            continue
    
    return artifacts
```

### 2. Data Transformation

```python
import pandas as pd

def transform_to_metadata(artifacts):
    """Transform raw API data into artifact metadata DataFrame"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],  # Truncate if too long
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'division': artifact.get('division', ''),
            'dated': artifact.get('dated', ''),
            'url': artifact.get('url', ''),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        })
    
    return pd.DataFrame(metadata)

def transform_to_media(artifacts):
    """Transform raw API data into artifact media DataFrame"""
    media = []
    
    for artifact in artifacts:
        media.append({
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl', ''),
            'primaryimageurl': artifact.get('primaryimageurl', ''),
            'iiifbaseuri': artifact.get('iiifbaseuri', ''),
            'imagepermissionlevel': artifact.get('imagepermissionlevel', 0)
        })
    
    return pd.DataFrame(media)

def transform_to_colors(artifacts):
    """Transform raw API data into artifact colors DataFrame"""
    colors = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        color_data = artifact.get('colors', [])
        
        if color_data:
            for color_entry in color_data:
                colors.append({
                    'artifact_id': artifact_id,
                    'color': color_entry.get('color', ''),
                    'spectrum': color_entry.get('spectrum', ''),
                    'percent': color_entry.get('percent', 0.0)
                })
    
    return pd.DataFrame(colors)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Establish database connection"""
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

def load_to_database(df_metadata, df_media, df_colors):
    """Load transformed DataFrames into SQL database"""
    connection = get_db_connection()
    if not connection:
        return False
    
    try:
        cursor = connection.cursor()
        
        # Clear existing data
        cursor.execute("DELETE FROM artifactcolors")
        cursor.execute("DELETE FROM artifactmedia")
        cursor.execute("DELETE FROM artifactmetadata")
        
        # Insert metadata
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, division, 
             dated, url, totalpageviews, totaluniquepageviews)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """
        metadata_values = df_metadata.values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, primaryimageurl, iiifbaseuri, imagepermissionlevel)
            VALUES (%s, %s, %s, %s, %s)
        """
        media_values = df_media.values.tolist()
        cursor.executemany(media_query, media_values)
        
        # Insert colors
        if not df_colors.empty:
            color_query = """
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, percent)
                VALUES (%s, %s, %s, %s)
            """
            color_values = df_colors.values.tolist()
            cursor.executemany(color_query, color_values)
        
        connection.commit()
        print(f"Successfully loaded {len(df_metadata)} artifacts")
        return True
        
    except Error as e:
        print(f"Database loading error: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

### 4. SQL Analytics Queries

```python
# Example analytical queries
ANALYTICS_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Most Popular Artifacts (by page views)": """
        SELECT title, culture, totalpageviews
        FROM artifactmetadata
        WHERE totalpageviews > 0
        ORDER BY totalpageviews DESC
        LIMIT 10
    """,
    
    "Color Distribution Across Artifacts": """
        SELECT spectrum, COUNT(DISTINCT artifact_id) as artifact_count,
               ROUND(AVG(percent), 2) as avg_percent
        FROM artifactcolors
        GROUP BY spectrum
        ORDER BY artifact_count DESC
    """,
    
    "Artifacts with Images by Department": """
        SELECT m.department, COUNT(*) as count
        FROM artifactmetadata m
        JOIN artifactmedia a ON m.id = a.artifact_id
        WHERE a.primaryimageurl IS NOT NULL AND a.primaryimageurl != ''
        GROUP BY m.department
        ORDER BY count DESC
    """
}

def execute_query(query_name):
    """Execute a named analytical query and return results"""
    connection = get_db_connection()
    if not connection:
        return None
    
    try:
        query = ANALYTICS_QUERIES[query_name]
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query execution error: {e}")
        return None
    finally:
        connection.close()
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for ETL controls
    with st.sidebar:
        st.header("ETL Pipeline")
        num_records = st.number_input("Records to fetch", 100, 1000, 100)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Fetching data from API..."):
                artifacts = fetch_artifacts(total_records=num_records)
            
            with st.spinner("Transforming data..."):
                df_metadata = transform_to_metadata(artifacts)
                df_media = transform_to_media(artifacts)
                df_colors = transform_to_colors(artifacts)
            
            with st.spinner("Loading to database..."):
                success = load_to_database(df_metadata, df_media, df_colors)
            
            if success:
                st.success(f"✅ Successfully processed {len(artifacts)} artifacts")
            else:
                st.error("❌ ETL pipeline failed")
    
    # Main analytics section
    st.header("📊 Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            results = execute_query(query_name)
        
        if results is not None and not results.empty:
            st.dataframe(results, use_container_width=True)
            
            # Auto-generate visualization
            if len(results.columns) >= 2:
                fig = px.bar(
                    results, 
                    x=results.columns[0], 
                    y=results.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)
        else:
            st.warning("No results found")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Workflow

```python
def run_complete_etl(num_records=100):
    """Execute complete ETL pipeline"""
    # Extract
    print("1. Extracting data from API...")
    artifacts = fetch_artifacts(total_records=num_records)
    
    # Transform
    print("2. Transforming data...")
    df_metadata = transform_to_metadata(artifacts)
    df_media = transform_to_media(artifacts)
    df_colors = transform_to_colors(artifacts)
    
    # Validate
    print(f"Metadata records: {len(df_metadata)}")
    print(f"Media records: {len(df_media)}")
    print(f"Color records: {len(df_colors)}")
    
    # Load
    print("3. Loading to database...")
    success = load_to_database(df_metadata, df_media, df_colors)
    
    return success
```

### Custom Query Execution

```python
def run_custom_query(sql_query):
    """Execute a custom SQL query"""
    connection = get_db_connection()
    if not connection:
        return None
    
    try:
        df = pd.read_sql(sql_query, connection)
        return df
    except Error as e:
        print(f"Error: {e}")
        return None
    finally:
        connection.close()

# Example usage
custom_sql = """
    SELECT m.classification, COUNT(*) as count,
           AVG(m.totalpageviews) as avg_views
    FROM artifactmetadata m
    WHERE m.century LIKE '%21st%'
    GROUP BY m.classification
    ORDER BY count DESC
"""
results = run_custom_query(custom_sql)
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limiting errors:

```python
import time

def fetch_artifacts_with_retry(total_records=100, retry_delay=2):
    """Fetch artifacts with exponential backoff on rate limits"""
    artifacts = []
    page_size = 100
    pages = (total_records + page_size - 1) // page_size
    
    for page in range(1, pages + 1):
        max_retries = 3
        for attempt in range(max_retries):
            try:
                params = {
                    'apikey': API_KEY,
                    'size': page_size,
                    'page': page
                }
                response = requests.get(BASE_URL, params=params, timeout=30)
                
                if response.status_code == 429:  # Rate limited
                    wait_time = retry_delay * (2 ** attempt)
                    print(f"Rate limited. Waiting {wait_time}s...")
                    time.sleep(wait_time)
                    continue
                
                response.raise_for_status()
                data = response.json()
                artifacts.extend(data.get('records', []))
                break
                
            except requests.exceptions.RequestException as e:
                if attempt == max_retries - 1:
                    print(f"Failed after {max_retries} attempts: {e}")
                time.sleep(retry_delay)
        
        time.sleep(0.5)  # Polite delay between pages
    
    return artifacts
```

### Database Connection Issues

```python
def test_database_connection():
    """Test database connectivity and permissions"""
    try:
        connection = get_db_connection()
        if connection and connection.is_connected():
            cursor = connection.cursor()
            cursor.execute("SELECT VERSION()")
            version = cursor.fetchone()
            print(f"✅ Connected to MySQL version: {version[0]}")
            
            # Test table access
            cursor.execute("SHOW TABLES")
            tables = cursor.fetchall()
            print(f"✅ Found {len(tables)} tables")
            
            cursor.close()
            connection.close()
            return True
    except Error as e:
        print(f"❌ Connection test failed: {e}")
        return False
```

### Data Validation

```python
def validate_etl_data(df_metadata, df_media, df_colors):
    """Validate data quality before loading"""
    issues = []
    
    # Check for duplicates
    if df_metadata['id'].duplicated().any():
        issues.append("Duplicate artifact IDs found")
    
    # Check for nulls in critical fields
    if df_metadata['id'].isnull().any():
        issues.append("Null IDs in metadata")
    
    # Check foreign key integrity
    media_ids = set(df_media['artifact_id'])
    metadata_ids = set(df_metadata['id'])
    orphaned = media_ids - metadata_ids
    if orphaned:
        issues.append(f"{len(orphaned)} orphaned media records")
    
    if issues:
        print("⚠️ Data validation issues:")
        for issue in issues:
            print(f"  - {issue}")
        return False
    
    print("✅ Data validation passed")
    return True
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

This skill provides everything needed to build, extend, and troubleshoot Harvard Art Museums data pipelines and analytics applications.
