---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build a data pipeline for museum artifacts
  - create an ETL workflow with Harvard Art Museums API
  - set up artifact data collection and analytics
  - how do I use the Harvard museums data pipeline
  - build a Streamlit dashboard for art collections
  - implement SQL analytics for museum data
  - extract and transform Harvard Art API data
  - create visualization dashboards for artifact data
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is an end-to-end data pipeline that:
- Extracts artifact data from the Harvard Art Museums API
- Transforms nested JSON into relational database tables
- Loads structured data into MySQL/TiDB Cloud
- Provides interactive SQL analytics via Streamlit dashboards
- Visualizes query results using Plotly

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

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
export DB_NAME="harvard_artifacts"
```

**Required Dependencies:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Getting Started

### 1. Obtain Harvard Art Museums API Key

Register at: https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dOd0Z8MpY2rvQ/viewform

### 2. Configure Database Connection

```python
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

def get_db_connection():
    """Establish database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```

### 3. Run the Streamlit Application

```bash
streamlit run app.py
```

## Core Components

### ETL Pipeline

#### Extract: Fetch Data from Harvard API

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, size=100, page=1):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: Your Harvard API key
        size: Number of records per page (max 100)
        page: Page number for pagination
    
    Returns:
        dict: API response with artifact data
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, size=100, page=1)
artifacts = data.get('records', [])
```

#### Transform: Process Nested JSON

```python
def transform_artifact_metadata(artifacts):
    """
    Extract and flatten artifact metadata
    
    Returns:
        pandas.DataFrame: Normalized artifact metadata
    """
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'dimensions': artifact.get('dimensions'),
            'medium': artifact.get('medium'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear')
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)

def transform_artifact_media(artifacts):
    """Extract media/image data from artifacts"""
    media_records = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for img in images:
            media_records.append({
                'objectid': objectid,
                'imageid': img.get('imageid'),
                'baseimageurl': img.get('baseimageurl'),
                'width': img.get('width'),
                'height': img.get('height'),
                'format': img.get('format')
            })
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """Extract color data from artifacts"""
    color_records = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_records.append({
                'objectid': objectid,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(color_records)
```

#### Load: Insert Data into SQL Database

```python
def create_database_schema(connection):
    """Create database tables for artifact data"""
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            period VARCHAR(255),
            century VARCHAR(255),
            classification VARCHAR(255),
            department VARCHAR(255),
            division VARCHAR(255),
            dated VARCHAR(255),
            dimensions TEXT,
            medium TEXT,
            creditline TEXT,
            accessionyear INT
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            imageid INT,
            baseimageurl TEXT,
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
            percent FLOAT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    connection.commit()

def batch_insert_dataframe(connection, df, table_name):
    """
    Batch insert DataFrame into SQL table
    
    Args:
        connection: MySQL connection object
        df: pandas DataFrame
        table_name: Target table name
    """
    cursor = connection.cursor()
    
    if df.empty:
        return
    
    # Generate INSERT statement
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Convert DataFrame to list of tuples
    data = [tuple(row) for row in df.values]
    
    # Batch insert
    cursor.executemany(query, data)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} rows into {table_name}")
```

### Complete ETL Workflow

```python
def run_etl_pipeline(api_key, num_pages=5):
    """
    Execute complete ETL pipeline
    
    Args:
        api_key: Harvard API key
        num_pages: Number of API pages to fetch
    """
    connection = get_db_connection()
    create_database_schema(connection)
    
    all_artifacts = []
    
    # Extract
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, size=100, page=page)
        artifacts = data.get('records', [])
        all_artifacts.extend(artifacts)
    
    # Transform
    df_metadata = transform_artifact_metadata(all_artifacts)
    df_media = transform_artifact_media(all_artifacts)
    df_colors = transform_artifact_colors(all_artifacts)
    
    # Load
    batch_insert_dataframe(connection, df_metadata, 'artifactmetadata')
    batch_insert_dataframe(connection, df_media, 'artifactmedia')
    batch_insert_dataframe(connection, df_colors, 'artifactcolors')
    
    connection.close()
    print("ETL pipeline completed successfully!")
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
def get_artifacts_by_culture(connection):
    """Count artifacts grouped by culture"""
    query = """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """
    return pd.read_sql(query, connection)

def get_artifacts_by_century(connection):
    """Analyze artifact distribution by century"""
    query = """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """
    return pd.read_sql(query, connection)

def get_color_distribution(connection):
    """Analyze most common colors across artifacts"""
    query = """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """
    return pd.read_sql(query, connection)

def get_media_availability(connection):
    """Check image availability statistics"""
    query = """
        SELECT 
            COUNT(DISTINCT am.objectid) as total_artifacts,
            COUNT(DISTINCT me.objectid) as artifacts_with_images,
            (COUNT(DISTINCT me.objectid) * 100.0 / COUNT(DISTINCT am.objectid)) as percentage_with_images
        FROM artifactmetadata am
        LEFT JOIN artifactmedia me ON am.objectid = me.objectid
    """
    return pd.read_sql(query, connection)

def get_department_classification(connection):
    """Cross-tabulate departments and classifications"""
    query = """
        SELECT department, classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND classification IS NOT NULL
        GROUP BY department, classification
        ORDER BY department, count DESC
    """
    return pd.read_sql(query, connection)
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    try:
        connection = get_db_connection()
        st.sidebar.success("Database connected!")
    except Exception as e:
        st.sidebar.error(f"Database connection failed: {e}")
        return
    
    # Analytics selection
    analysis_type = st.sidebar.selectbox(
        "Select Analysis",
        [
            "Artifacts by Culture",
            "Artifacts by Century",
            "Color Distribution",
            "Media Availability",
            "Department Classification"
        ]
    )
    
    # Execute query based on selection
    if analysis_type == "Artifacts by Culture":
        df = get_artifacts_by_culture(connection)
        st.subheader("Artifacts Distribution by Culture")
        st.dataframe(df)
        
        fig = px.bar(df, x='culture', y='artifact_count', 
                     title='Top 20 Cultures by Artifact Count')
        st.plotly_chart(fig)
    
    elif analysis_type == "Artifacts by Century":
        df = get_artifacts_by_century(connection)
        st.subheader("Artifacts Distribution by Century")
        st.dataframe(df)
        
        fig = px.bar(df, x='century', y='count',
                     title='Artifact Distribution Across Centuries')
        st.plotly_chart(fig)
    
    elif analysis_type == "Color Distribution":
        df = get_color_distribution(connection)
        st.subheader("Most Common Colors in Artifacts")
        st.dataframe(df)
        
        fig = px.bar(df, x='color', y='frequency',
                     title='Top 15 Colors by Frequency')
        st.plotly_chart(fig)
    
    elif analysis_type == "Media Availability":
        df = get_media_availability(connection)
        st.subheader("Image Availability Statistics")
        st.dataframe(df)
        
        st.metric("Total Artifacts", df['total_artifacts'].values[0])
        st.metric("Artifacts with Images", df['artifacts_with_images'].values[0])
        st.metric("Percentage with Images", 
                  f"{df['percentage_with_images'].values[0]:.2f}%")
    
    elif analysis_type == "Department Classification":
        df = get_department_classification(connection)
        st.subheader("Artifacts by Department and Classification")
        st.dataframe(df)
        
        fig = px.sunburst(df, path=['department', 'classification'], 
                          values='count',
                          title='Department and Classification Hierarchy')
        st.plotly_chart(fig)
    
    connection.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(api_key, max_pages=10, delay=1):
    """Fetch data with rate limiting to avoid API throttling"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(api_key, size=100, page=page)
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            print(f"Fetched page {page}: {len(artifacts)} artifacts")
            time.sleep(delay)  # Rate limiting
            
        except requests.exceptions.HTTPError as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, num_pages=5):
    """ETL pipeline with comprehensive error handling"""
    try:
        connection = get_db_connection()
        create_database_schema(connection)
        
        all_artifacts = fetch_with_rate_limit(api_key, num_pages)
        
        if not all_artifacts:
            raise ValueError("No artifacts fetched from API")
        
        # Transform with validation
        df_metadata = transform_artifact_metadata(all_artifacts)
        df_metadata = df_metadata.dropna(subset=['objectid'])
        
        df_media = transform_artifact_media(all_artifacts)
        df_colors = transform_artifact_colors(all_artifacts)
        
        # Load with transaction
        try:
            batch_insert_dataframe(connection, df_metadata, 'artifactmetadata')
            batch_insert_dataframe(connection, df_media, 'artifactmedia')
            batch_insert_dataframe(connection, df_colors, 'artifactcolors')
            connection.commit()
        except Exception as e:
            connection.rollback()
            raise e
        
        return True
        
    except Exception as e:
        print(f"ETL pipeline failed: {e}")
        return False
    finally:
        if 'connection' in locals():
            connection.close()
```

## Troubleshooting

### API Key Issues

```python
# Verify API key is working
def test_api_connection(api_key):
    """Test Harvard API connection"""
    try:
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'size': 1}
        )
        response.raise_for_status()
        print("API connection successful!")
        return True
    except requests.exceptions.HTTPError as e:
        print(f"API authentication failed: {e}")
        return False
```

### Database Connection Issues

```python
# Test database connectivity
def test_db_connection():
    """Test database connection"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        print("Database connection successful!")
        conn.close()
        return True
    except Exception as e:
        print(f"Database connection failed: {e}")
        return False
```

### Common Errors

**Error: 401 Unauthorized**
- Check your `HARVARD_API_KEY` environment variable
- Verify API key is active at Harvard Art Museums

**Error: Table doesn't exist**
- Run `create_database_schema()` before inserting data
- Verify database user has CREATE TABLE permissions

**Error: Duplicate entry for key 'PRIMARY'**
- Use `INSERT IGNORE` or `ON DUPLICATE KEY UPDATE`
- Clean existing data before re-running ETL

**Streamlit not loading**
```bash
# Check if running on correct port
streamlit run app.py --server.port 8501

# Clear Streamlit cache
streamlit cache clear
```

This project demonstrates industry-standard data engineering practices with real-world museum data, making it ideal for learning ETL workflows, SQL analytics, and interactive dashboard development.
