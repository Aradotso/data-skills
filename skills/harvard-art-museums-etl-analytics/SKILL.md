---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - create a data engineering project with Harvard API
  - set up artifact collection analytics dashboard
  - extract and transform art museum data
  - build a Streamlit analytics app with SQL
  - create data visualizations from Harvard Art Museums
  - design a relational database for art artifacts
  - implement batch ETL with pandas and MySQL
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a complete data engineering and analytics application that demonstrates real-world ETL pipelines using the Harvard Art Museums API. It extracts artifact data, transforms nested JSON into relational structures, loads into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics dashboards via Streamlit.

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

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
# Create .env file or export variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

### Dependencies

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Configuration

The Harvard Art Museums API requires authentication. Configure your API key:

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = 'https://api.harvardartmuseums.org/object'
```

### Database Configuration

Set up MySQL/TiDB connection:

```python
import mysql.connector
import os

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'port': int(os.getenv('DB_PORT', 3306))
}

connection = mysql.connector.connect(**db_config)
```

## Database Schema

### Create Tables

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    classification VARCHAR(200),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    department VARCHAR(200),
    division VARCHAR(200),
    creditline TEXT,
    accession_number VARCHAR(100),
    provenance TEXT,
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    media_type VARCHAR(100),
    format VARCHAR(100),
    description TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline

### Extract: Fetching Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=5, page_size=100):
    """
    Fetch artifact data from Harvard Art Museums API with pagination
    """
    all_records = []
    base_url = 'https://api.harvardartmuseums.org/object'
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            if 'records' in data:
                all_records.extend(data['records'])
                print(f"Fetched page {page}: {len(data['records'])} records")
            
            # Rate limiting
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_records
```

### Transform: Processing Nested JSON

```python
import pandas as pd

def transform_metadata(records):
    """
    Transform raw API records into structured metadata
    """
    metadata_list = []
    
    for record in records:
        metadata = {
            'id': record.get('id'),
            'title': record.get('title', '')[:500],
            'culture': record.get('culture', '')[:200],
            'period': record.get('period', '')[:200],
            'century': record.get('century', '')[:100],
            'dated': record.get('dated', '')[:200],
            'classification': record.get('classification', '')[:200],
            'medium': record.get('medium', '')[:500],
            'dimensions': record.get('dimensions', '')[:500],
            'department': record.get('department', '')[:200],
            'division': record.get('division', '')[:200],
            'creditline': record.get('creditline', ''),
            'accession_number': record.get('accessionyear', ''),
            'provenance': record.get('provenance', ''),
            'url': record.get('url', '')[:500]
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_media(records):
    """
    Extract media/image data from artifacts
    """
    media_list = []
    
    for record in records:
        artifact_id = record.get('id')
        images = record.get('images', [])
        
        for img in images:
            media = {
                'artifact_id': artifact_id,
                'image_url': img.get('baseimageurl', ''),
                'media_type': 'image',
                'format': img.get('format', ''),
                'description': img.get('description', '')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_colors(records):
    """
    Extract color data from artifacts
    """
    color_list = []
    
    for record in records:
        artifact_id = record.get('id')
        colors = record.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### Load: Batch Insert into SQL

```python
def load_to_database(df, table_name, connection):
    """
    Batch insert DataFrame into SQL database
    """
    cursor = connection.cursor()
    
    # Prepare INSERT statement
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    insert_query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Batch insert
    data_tuples = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} rows into {table_name}")
    except Exception as e:
        connection.rollback()
        print(f"Error inserting into {table_name}: {e}")
    finally:
        cursor.close()

# Complete ETL execution
def run_etl_pipeline(api_key, db_config, num_pages=5):
    """
    Execute complete ETL pipeline
    """
    # Extract
    print("Extracting data from API...")
    records = fetch_artifacts(api_key, num_pages)
    
    # Transform
    print("Transforming data...")
    df_metadata = transform_metadata(records)
    df_media = transform_media(records)
    df_colors = transform_colors(records)
    
    # Load
    print("Loading data to database...")
    connection = mysql.connector.connect(**db_config)
    
    load_to_database(df_metadata, 'artifactmetadata', connection)
    load_to_database(df_media, 'artifactmedia', connection)
    load_to_database(df_colors, 'artifactcolors', connection)
    
    connection.close()
    print("ETL pipeline completed!")
```

## Analytics Queries

### Sample SQL Analytics

```python
# Top 10 cultures by artifact count
query_1 = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Artifacts by century
query_2 = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL AND century != ''
GROUP BY century
ORDER BY count DESC
"""

# Department distribution
query_3 = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY total_artifacts DESC
"""

# Most common colors
query_4 = """
SELECT color, COUNT(*) as frequency
FROM artifactcolors
GROUP BY color
ORDER BY frequency DESC
LIMIT 15
"""

# Media availability analysis
query_5 = """
SELECT 
    COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
    COUNT(*) as total_media_records
FROM artifactmedia m
"""

def execute_query(query, connection):
    """
    Execute SQL query and return results as DataFrame
    """
    try:
        df = pd.read_sql(query, connection)
        return df
    except Exception as e:
        print(f"Query error: {e}")
        return pd.DataFrame()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics")
    st.markdown("End-to-end Data Engineering & Analytics Pipeline")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    try:
        connection = mysql.connector.connect(**db_config)
        st.sidebar.success("✅ Database connected")
    except Exception as e:
        st.sidebar.error(f"❌ Connection failed: {e}")
        return
    
    # ETL Section
    st.header("📥 ETL Pipeline")
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            run_etl_pipeline(API_KEY, db_config, num_pages=3)
        st.success("ETL completed!")
    
    # Analytics Section
    st.header("📊 Analytics Dashboard")
    
    query_options = {
        "Top Cultures": query_1,
        "Century Distribution": query_2,
        "Department Analysis": query_3,
        "Color Frequency": query_4,
        "Media Availability": query_5
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        df = execute_query(query_options[selected_query], connection)
        
        st.subheader("Results")
        st.dataframe(df)
        
        # Visualization
        if len(df) > 0 and len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=f"{selected_query} Analysis")
            st.plotly_chart(fig, use_container_width=True)
    
    connection.close()

if __name__ == "__main__":
    main()
```

### Running the App

```bash
streamlit run app.py
```

## Common Patterns

### Error Handling in ETL

```python
def safe_etl_execution(api_key, db_config):
    """
    ETL with comprehensive error handling
    """
    try:
        records = fetch_artifacts(api_key)
        if not records:
            raise ValueError("No records fetched from API")
        
        connection = mysql.connector.connect(**db_config)
        
        # Transform and load with rollback on failure
        try:
            df_metadata = transform_metadata(records)
            load_to_database(df_metadata, 'artifactmetadata', connection)
        except Exception as e:
            connection.rollback()
            raise Exception(f"Load failed: {e}")
        finally:
            connection.close()
            
    except requests.exceptions.RequestException as e:
        print(f"API request failed: {e}")
    except mysql.connector.Error as e:
        print(f"Database error: {e}")
```

### Incremental Data Loading

```python
def incremental_load(api_key, db_config, last_updated_date):
    """
    Load only new artifacts since last update
    """
    params = {
        'apikey': api_key,
        'updatedafter': last_updated_date  # ISO format: 2024-01-01
    }
    
    response = requests.get(BASE_URL, params=params)
    new_records = response.json().get('records', [])
    
    # Process only new records
    # ...
```

## Troubleshooting

### API Rate Limiting

If you hit rate limits, increase the sleep time:

```python
time.sleep(1)  # Increase from 0.5 to 1 second
```

### Database Connection Issues

Verify connection parameters:

```python
try:
    connection = mysql.connector.connect(**db_config)
    connection.ping(reconnect=True)
except mysql.connector.Error as e:
    print(f"Error code: {e.errno}")
    print(f"Error message: {e.msg}")
```

### Memory Issues with Large Datasets

Process in smaller batches:

```python
def chunked_load(df, chunk_size=1000):
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i+chunk_size]
        load_to_database(chunk, 'artifactmetadata', connection)
```

### Empty Query Results

Check data availability:

```python
cursor.execute("SELECT COUNT(*) FROM artifactmetadata")
count = cursor.fetchone()[0]
print(f"Total records: {count}")
```

## Best Practices

1. **Always use environment variables** for sensitive data (API keys, DB credentials)
2. **Implement retry logic** for API requests with exponential backoff
3. **Use batch inserts** for better performance (1000+ rows at a time)
4. **Index foreign keys** for faster joins
5. **Validate data** before loading (null checks, type conversions)
6. **Log ETL execution** with timestamps and record counts
7. **Cache API responses** to avoid redundant calls during development
