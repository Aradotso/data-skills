---
name: harvard-art-museums-data-pipeline
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an art museum data pipeline
  - extract data from Harvard Art Museums API
  - create ETL pipeline for museum artifacts
  - analyze Harvard art collection with SQL
  - build museum data analytics dashboard
  - integrate Harvard Art Museums API with database
  - visualize art collection data with Streamlit
  - process museum artifact metadata
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Art Museums Data Pipeline is a complete data engineering application that demonstrates ETL processes, SQL analytics, and interactive visualization using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON data into relational tables, loads it into SQL databases, and provides analytics dashboards through Streamlit.

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

### API Key Setup

Get your Harvard Art Museums API key from: https://docs.api.harvardartmuseums.org/

Create a `.env` file or configure through Streamlit sidebar:

```python
HARVARD_API_KEY=your_api_key_here
```

### Database Configuration

The project supports MySQL/TiDB Cloud. Configure connection parameters:

```python
db_config = {
    'host': 'your-host.com',
    'port': 4000,
    'user': 'your_username',
    'password': 'your_password',
    'database': 'harvard_artifacts'
}
```

## Database Schema

The pipeline creates three main tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    dated VARCHAR(255),
    classification VARCHAR(255),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    department VARCHAR(255),
    division VARCHAR(255),
    accession_number VARCHAR(100),
    provenance TEXT,
    description TEXT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    image_url VARCHAR(1000),
    thumbnail_url VARCHAR(1000),
    media_type VARCHAR(100),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    hue VARCHAR(50),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Core ETL Pipeline

### Extract: Fetching Data from API

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_pages=5, records_per_page=100):
    """
    Extract artifacts from Harvard Art Museums API with pagination
    
    Args:
        api_key: Your Harvard API key
        num_pages: Number of pages to fetch
        records_per_page: Records per API call (max 100)
    
    Returns:
        List of artifact records
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_records = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': records_per_page,
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_records.extend(data.get('records', []))
        else:
            print(f"Error on page {page}: {response.status_code}")
            break
    
    return all_records
```

### Transform: Processing Nested JSON

```python
def transform_artifact_data(records):
    """
    Transform raw API data into structured dataframes
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in records:
        # Extract metadata
        metadata = {
            'objectid': record.get('objectid'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'period': record.get('period'),
            'century': record.get('century'),
            'dated': record.get('dated'),
            'classification': record.get('classification'),
            'medium': record.get('medium'),
            'dimensions': record.get('dimensions'),
            'department': record.get('department'),
            'division': record.get('division'),
            'accession_number': record.get('accessionyear'),
            'provenance': record.get('provenance'),
            'description': record.get('description')
        }
        metadata_list.append(metadata)
        
        # Extract media/images
        images = record.get('images', [])
        for img in images:
            media = {
                'objectid': record.get('objectid'),
                'image_url': img.get('baseimageurl'),
                'thumbnail_url': img.get('thumbnailurl'),
                'media_type': img.get('format')
            }
            media_list.append(media)
        
        # Extract color data
        colors = record.get('colors', [])
        for color in colors:
            color_data = {
                'objectid': record.get('objectid'),
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent'),
                'hue': color.get('hue')
            }
            colors_list.append(color_data)
    
    metadata_df = pd.DataFrame(metadata_list)
    media_df = pd.DataFrame(media_list)
    colors_df = pd.DataFrame(colors_list)
    
    return metadata_df, media_df, colors_df
```

### Load: Inserting into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """
    Load dataframes into SQL database with batch inserts
    
    Args:
        metadata_df: Artifact metadata DataFrame
        media_df: Media/images DataFrame
        colors_df: Colors DataFrame
        db_config: Database connection configuration
    """
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Insert metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, period, century, dated, classification, 
         medium, dimensions, department, division, accession_number, 
         provenance, description)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        
        metadata_values = [tuple(row) for row in metadata_df.values]
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        media_query = """
        INSERT INTO artifactmedia (objectid, image_url, thumbnail_url, media_type)
        VALUES (%s, %s, %s, %s)
        """
        
        media_values = [tuple(row) for row in media_df.values]
        cursor.executemany(media_query, media_values)
        
        # Insert colors
        colors_query = """
        INSERT INTO artifactcolors (objectid, color_hex, color_percent, hue)
        VALUES (%s, %s, %s, %s)
        """
        
        colors_values = [tuple(row) for row in colors_df.values]
        cursor.executemany(colors_query, colors_values)
        
        connection.commit()
        print(f"Successfully loaded {len(metadata_df)} artifacts")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

## Analytics Queries

### Sample SQL Analytics

```python
ANALYTICS_QUERIES = {
    "Artifacts by Department": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 10;
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC;
    """,
    
    "Top Cultures": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15;
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN media_id IS NOT NULL THEN 'Has Media' ELSE 'No Media' END as media_status,
            COUNT(DISTINCT m.objectid) as artifact_count
        FROM artifactmetadata m
        LEFT JOIN artifactmedia am ON m.objectid = am.objectid
        GROUP BY media_status;
    """,
    
    "Top Color Hues": """
        SELECT hue, COUNT(*) as usage_count, AVG(color_percent) as avg_percent
        FROM artifactcolors
        WHERE hue IS NOT NULL
        GROUP BY hue
        ORDER BY usage_count DESC
        LIMIT 10;
    """,
    
    "Classification Distribution": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 10;
    """
}
```

### Executing Analytics Queries

```python
def execute_analytics_query(query, db_config):
    """
    Execute SQL analytics query and return results as DataFrame
    """
    try:
        connection = mysql.connector.connect(**db_config)
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return pd.DataFrame()
    finally:
        if connection.is_connected():
            connection.close()
```

## Streamlit Dashboard

### Basic Dashboard Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    db_host = st.sidebar.text_input("Database Host")
    db_user = st.sidebar.text_input("Database User")
    db_password = st.sidebar.text_input("Database Password", type="password")
    db_name = st.sidebar.text_input("Database Name")
    
    # ETL Section
    st.header("📥 Data Collection")
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=20, value=5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            records = fetch_artifacts(api_key, num_pages=num_pages)
            st.success(f"Fetched {len(records)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifact_data(records)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            db_config = {
                'host': db_host,
                'user': db_user,
                'password': db_password,
                'database': db_name
            }
            load_to_database(metadata_df, media_df, colors_df, db_config)
            st.success("Data loaded to database")
    
    # Analytics Section
    st.header("📊 Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        df = execute_analytics_query(query, db_config)
        
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(api_key, num_pages, delay=0.5):
    """Fetch data with rate limiting to avoid API throttling"""
    all_records = []
    
    for page in range(1, num_pages + 1):
        records = fetch_artifacts(api_key, num_pages=1, records_per_page=100)
        all_records.extend(records)
        time.sleep(delay)  # Rate limiting
    
    return all_records
```

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, db_config, num_pages=5):
    """ETL pipeline with comprehensive error handling"""
    try:
        # Extract
        records = fetch_artifacts(api_key, num_pages)
        if not records:
            raise ValueError("No records fetched from API")
        
        # Transform
        metadata_df, media_df, colors_df = transform_artifact_data(records)
        
        # Validate
        if metadata_df.empty:
            raise ValueError("No metadata extracted")
        
        # Load
        load_to_database(metadata_df, media_df, colors_df, db_config)
        
        return {
            'status': 'success',
            'records': len(records),
            'metadata_rows': len(metadata_df)
        }
    
    except Exception as e:
        return {
            'status': 'error',
            'error': str(e)
        }
```

## Troubleshooting

### API Connection Issues

```python
# Test API connectivity
def test_api_connection(api_key):
    """Verify API key and connection"""
    url = f"https://api.harvardartmuseums.org/object?apikey={api_key}&size=1"
    response = requests.get(url)
    
    if response.status_code == 200:
        print("✅ API connection successful")
        return True
    elif response.status_code == 401:
        print("❌ Invalid API key")
        return False
    else:
        print(f"❌ API error: {response.status_code}")
        return False
```

### Database Connection Issues

```python
def test_db_connection(db_config):
    """Test database connectivity"""
    try:
        connection = mysql.connector.connect(**db_config)
        if connection.is_connected():
            print("✅ Database connection successful")
            connection.close()
            return True
    except Error as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Handling Missing Data

```python
def clean_metadata(df):
    """Clean and handle missing values in metadata"""
    # Replace None with empty strings for text columns
    text_columns = ['title', 'culture', 'period', 'century', 'classification']
    df[text_columns] = df[text_columns].fillna('')
    
    # Ensure objectid is not null
    df = df[df['objectid'].notna()]
    
    return df
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

The application provides an interactive interface for:
1. Configuring API and database credentials
2. Running ETL pipelines with customizable parameters
3. Executing 20+ predefined analytics queries
4. Visualizing results with interactive Plotly charts
5. Exploring artifact metadata, media, and color distributions
