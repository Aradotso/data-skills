---
name: harvard-artifacts-data-engineering-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API data with Streamlit
triggers:
  - how do I build a data pipeline with the Harvard Art Museums API
  - create an ETL workflow for museum artifact data
  - analyze Harvard art collection data with SQL
  - build a Streamlit dashboard for museum artifacts
  - extract and transform Harvard API data into SQL database
  - visualize art museum collection analytics
  - set up data engineering pipeline for cultural heritage data
  - query and analyze artifact metadata from Harvard Museums
---

# Harvard Artifacts Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering and analytics application for working with the Harvard Art Museums API. It demonstrates production-grade ETL pipelines, SQL database design, and interactive visualization using Streamlit. The architecture follows: **API → ETL → SQL → Analytics → Visualization**.

Key capabilities:
- Extract artifact data from Harvard Art Museums API with pagination and rate limiting
- Transform nested JSON into relational database tables
- Load data into MySQL/TiDB Cloud with optimized batch inserts
- Execute analytical SQL queries on artifact metadata, media, and color data
- Visualize results with interactive Plotly charts in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages include:
# - streamlit
# - pandas
# - requests
# - mysql-connector-python
# - plotly
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from: https://docs.api.harvardartmuseums.org/

Store credentials using environment variables:

```bash
export HARVARD_API_KEY="your_api_key_here"
```

### Database Configuration

Set up MySQL/TiDB Cloud connection parameters:

```python
import os

DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts')
}
```

Environment variables:
```bash
export DB_HOST="your_database_host"
export DB_PORT="3306"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Database Schema

The ETL pipeline creates three relational tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    accessionyear INT,
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    format VARCHAR(50),
    description TEXT,
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(100),
    hue VARCHAR(100),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Code Examples

### Extract: Fetching Data from API

```python
import requests
import pandas as pd
import os

def fetch_artifacts(api_key, num_records=100):
    """
    Extract artifact data from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': min(size, num_records - len(artifacts)),
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            artifacts.extend(records)
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts_data = fetch_artifacts(api_key, num_records=500)
```

### Transform: Processing Nested JSON

```python
def transform_artifacts(artifacts_raw):
    """
    Transform nested JSON into relational dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts_raw:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'accessionyear': artifact.get('accessionyear'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
        
        # Extract media (images)
        for media in artifact.get('images', []):
            media_record = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': media.get('baseimageurl'),
                'format': media.get('format'),
                'description': media.get('description'),
                'iiifbaseuri': media.get('iiifbaseuri')
            }
            media_list.append(media_record)
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            colors_list.append(color_record)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

# Usage
df_metadata, df_media, df_colors = transform_artifacts(artifacts_data)
```

### Load: Batch Insert into SQL

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(df_metadata, df_media, df_colors, db_config):
    """
    Load transformed data into MySQL database with batch inserts
    """
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Insert metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, 
         technique, accessionyear, totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
        """
        
        metadata_values = [tuple(row) for row in df_metadata.values]
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, format, description, iiifbaseuri)
        VALUES (%s, %s, %s, %s, %s)
        """
        
        media_values = [tuple(row) for row in df_media.values]
        cursor.executemany(media_query, media_values)
        
        # Insert colors
        colors_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        
        colors_values = [tuple(row) for row in df_colors.values]
        cursor.executemany(colors_query, colors_values)
        
        connection.commit()
        print(f"Successfully loaded {len(metadata_values)} artifacts")
        
    except Error as e:
        print(f"Database error: {e}")
        connection.rollback()
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()

# Usage
load_to_database(df_metadata, df_media, df_colors, DB_CONFIG)
```

## Analytical SQL Queries

### Example Analytics Queries

```python
# Sample analytical queries for the dashboard

ANALYTICS_QUERIES = {
    "Artifacts by Century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Top Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY total_artifacts DESC
        LIMIT 10
    """,
    
    "Media Availability Analysis": """
        SELECT 
            CASE WHEN media_count > 0 THEN 'Has Images' 
                 ELSE 'No Images' END as media_status,
            COUNT(*) as artifact_count
        FROM (
            SELECT m.id, COUNT(med.media_id) as media_count
            FROM artifactmetadata m
            LEFT JOIN artifactmedia med ON m.id = med.artifact_id
            GROUP BY m.id
        ) as media_summary
        GROUP BY media_status
    """,
    
    "Color Distribution": """
        SELECT spectrum, COUNT(*) as usage_count, 
               ROUND(AVG(percent), 2) as avg_percent
        FROM artifactcolors
        WHERE spectrum IS NOT NULL
        GROUP BY spectrum
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Department Popularity": """
        SELECT department, 
               SUM(totalpageviews) as total_views,
               COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_views DESC
        LIMIT 10
    """
}
```

### Executing Queries in Streamlit

```python
import streamlit as st
import pandas as pd
import plotly.express as px

def execute_query(query, db_config):
    """Execute SQL query and return results as DataFrame"""
    connection = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, connection)
    connection.close()
    return df

def visualize_results(df, title):
    """Create interactive Plotly visualization"""
    if len(df.columns) >= 2:
        fig = px.bar(
            df, 
            x=df.columns[0], 
            y=df.columns[1],
            title=title,
            labels={df.columns[0]: df.columns[0].title(), 
                    df.columns[1]: df.columns[1].title()}
        )
        st.plotly_chart(fig, use_container_width=True)

# Streamlit app structure
st.title("Harvard Artifacts Analytics Dashboard")

query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))

if st.button("Run Query"):
    query = ANALYTICS_QUERIES[query_name]
    results = execute_query(query, DB_CONFIG)
    
    st.subheader("Query Results")
    st.dataframe(results)
    
    visualize_results(results, query_name)
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(api_key, num_records, delay=0.5):
    """
    Fetch data with rate limiting to avoid API throttling
    """
    artifacts = []
    page = 1
    
    while len(artifacts) < num_records:
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'page': page, 'size': 100}
        )
        
        if response.status_code == 200:
            artifacts.extend(response.json().get('records', []))
            page += 1
            time.sleep(delay)  # Rate limiting
        elif response.status_code == 429:
            time.sleep(5)  # Longer wait for rate limit
        else:
            break
    
    return artifacts[:num_records]
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_pipeline(api_key, db_config, num_records=100):
    """
    ETL pipeline with comprehensive error handling
    """
    try:
        logger.info(f"Starting ETL for {num_records} records")
        
        # Extract
        artifacts = fetch_artifacts(api_key, num_records)
        logger.info(f"Extracted {len(artifacts)} artifacts")
        
        # Transform
        df_meta, df_media, df_colors = transform_artifacts(artifacts)
        logger.info("Transformation complete")
        
        # Load
        load_to_database(df_meta, df_media, df_colors, db_config)
        logger.info("Data loaded successfully")
        
        return True
        
    except requests.RequestException as e:
        logger.error(f"API request failed: {e}")
        return False
    except Error as e:
        logger.error(f"Database error: {e}")
        return False
    except Exception as e:
        logger.error(f"Unexpected error: {e}")
        return False
```

## Troubleshooting

### API Key Issues

```python
# Test API connection
def test_api_connection(api_key):
    response = requests.get(
        "https://api.harvardartmuseums.org/object",
        params={'apikey': api_key, 'size': 1}
    )
    if response.status_code == 200:
        print("✓ API connection successful")
        return True
    else:
        print(f"✗ API error: {response.status_code}")
        return False
```

### Database Connection Issues

```python
def test_database_connection(db_config):
    try:
        connection = mysql.connector.connect(**db_config)
        if connection.is_connected():
            print("✓ Database connection successful")
            connection.close()
            return True
    except Error as e:
        print(f"✗ Database error: {e}")
        return False
```

### Empty Query Results

Check data loading:
```sql
-- Verify data loaded correctly
SELECT COUNT(*) FROM artifactmetadata;
SELECT COUNT(*) FROM artifactmedia;
SELECT COUNT(*) FROM artifactcolors;

-- Check for NULL values
SELECT COUNT(*) FROM artifactmetadata WHERE culture IS NULL;
```

### Streamlit Port Conflicts

```bash
# Run on custom port
streamlit run app.py --server.port 8502

# Run with specific address
streamlit run app.py --server.address 0.0.0.0
```

## Best Practices

1. **Always use environment variables** for sensitive credentials
2. **Implement pagination** for large data extractions
3. **Use batch inserts** for better database performance
4. **Add indexes** on frequently queried columns (culture, century, department)
5. **Cache Streamlit results** using `@st.cache_data` for expensive queries
6. **Validate data** before inserting into database
7. **Log ETL pipeline steps** for debugging and monitoring
