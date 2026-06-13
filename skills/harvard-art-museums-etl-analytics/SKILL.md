---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics app using Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for the Harvard Art Museums API
  - create a data engineering pipeline with Harvard artifacts
  - set up analytics dashboard for museum collection data
  - extract and analyze Harvard Art Museums data
  - build a Streamlit app for art collection analytics
  - implement SQL analytics for museum artifacts
  - create visualization for Harvard museum data
  - design ETL workflow for art collection API
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Connects to Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts artifact metadata, transforms nested JSON, loads into relational SQL tables
- **Database Design**: Three-table schema (metadata, media, colors) with foreign key relationships
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Visualization**: Streamlit dashboard with Plotly charts

**Architecture**: `API → ETL → SQL → Analytics → Visualization`

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# MySQL or TiDB Cloud database instance
```

### Setup Steps

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

### API Key Setup

Get your Harvard Art Museums API key from: https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dOd0Z4MpvuDBA/viewform

```python
# Use environment variables for sensitive data
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
API_BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

```python
import mysql.connector
import os

# Database connection
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts')
}

connection = mysql.connector.connect(**db_config)
cursor = connection.cursor()
```

### Database Schema

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    artifact_id INT,
    media_id INT PRIMARY KEY,
    baseimageurl VARCHAR(500),
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. ETL Pipeline

#### Extract Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=10, page_size=100):
    """
    Extract artifact data from Harvard Art Museums API with pagination
    """
    artifacts = []
    base_url = "https://api.harvardartmuseums.org/object"
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            artifacts.extend(data.get('records', []))
            
            # Rate limiting
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return artifacts
```

#### Transform Data

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform nested JSON into relational dataframes
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear'),
            'url': artifact.get('url')
        }
        metadata_records.append(metadata)
        
        # Extract media information
        for image in artifact.get('images', []):
            media = {
                'artifact_id': artifact.get('id'),
                'media_id': image.get('imageid'),
                'baseimageurl': image.get('baseimageurl'),
                'format': image.get('format'),
                'height': image.get('height'),
                'width': image.get('width')
            }
            media_records.append(media)
        
        # Extract color information
        for color_data in artifact.get('colors', []):
            color = {
                'artifact_id': artifact.get('id'),
                'color': color_data.get('color'),
                'spectrum': color_data.get('spectrum'),
                'percent': color_data.get('percent')
            }
            color_records.append(color)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

#### Load Data to SQL

```python
def load_to_sql(metadata_df, media_df, colors_df, connection):
    """
    Load transformed data into SQL database
    """
    cursor = connection.cursor()
    
    # Insert metadata (batch insert)
    metadata_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, classification, department, 
     dated, technique, medium, dimensions, creditline, accessionyear, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    cursor.executemany(metadata_query, metadata_df.values.tolist())
    
    # Insert media
    if not media_df.empty:
        media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, media_id, baseimageurl, format, height, width)
        VALUES (%s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE baseimageurl=VALUES(baseimageurl)
        """
        cursor.executemany(media_query, media_df.values.tolist())
    
    # Insert colors
    if not colors_df.empty:
        colors_query = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, percent)
        VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(colors_query, colors_df.values.tolist())
    
    connection.commit()
    cursor.close()
```

### 2. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            CASE 
                WHEN media_id IS NOT NULL THEN 'With Images'
                ELSE 'Without Images'
            END as media_status,
            COUNT(DISTINCT m.id) as count
        FROM artifactmetadata m
        LEFT JOIN artifactmedia am ON m.id = am.artifact_id
        GROUP BY media_status
    """,
    
    "Top Colors by Frequency": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Artifacts per Accession Year": """
        SELECT accessionyear, COUNT(*) as count
        FROM artifactmetadata
        WHERE accessionyear IS NOT NULL AND accessionyear > 1900
        GROUP BY accessionyear
        ORDER BY accessionyear DESC
    """
}

def execute_query(connection, query):
    """Execute SQL query and return results as DataFrame"""
    cursor = connection.cursor()
    cursor.execute(query)
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    cursor.close()
    return pd.DataFrame(results, columns=columns)
```

### 3. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for configuration
    with st.sidebar:
        st.header("Configuration")
        api_key = st.text_input("API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))
        num_pages = st.slider("Pages to Fetch", 1, 50, 10)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Fetching data from API..."):
                artifacts = fetch_artifacts(api_key, num_pages)
                st.success(f"Fetched {len(artifacts)} artifacts")
            
            with st.spinner("Transforming data..."):
                metadata_df, media_df, colors_df = transform_artifacts(artifacts)
                st.success("Data transformed")
            
            with st.spinner("Loading to database..."):
                connection = mysql.connector.connect(**db_config)
                load_to_sql(metadata_df, media_df, colors_df, connection)
                connection.close()
                st.success("Data loaded to SQL")
    
    # Analytics section
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        connection = mysql.connector.connect(**db_config)
        df = execute_query(connection, ANALYTICS_QUERIES[query_name])
        connection.close()
        
        # Display table
        st.dataframe(df, use_container_width=True)
        
        # Generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Workflow

```python
import os
from dotenv import load_dotenv
import mysql.connector

# Load environment variables
load_dotenv()

# Configuration
API_KEY = os.getenv('HARVARD_API_KEY')
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

# Full ETL pipeline
def run_full_pipeline(num_pages=10):
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_artifacts(API_KEY, num_pages)
    
    # Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(artifacts)
    
    # Load
    print("Loading to database...")
    connection = mysql.connector.connect(**db_config)
    load_to_sql(metadata_df, media_df, colors_df, connection)
    connection.close()
    
    print(f"Pipeline complete! Processed {len(artifacts)} artifacts")

# Run pipeline
run_full_pipeline(num_pages=20)
```

### Error Handling and Retry Logic

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(url, params, max_retries=3):
    """Fetch data with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
            time.sleep(wait_time)
```

## Troubleshooting

### API Rate Limiting

```python
# Add delays between requests
import time

for page in range(1, num_pages + 1):
    response = requests.get(url, params=params)
    time.sleep(1)  # 1 second delay between requests
```

### Database Connection Issues

```python
# Test database connection
try:
    connection = mysql.connector.connect(**db_config)
    print("Database connection successful")
    connection.close()
except mysql.connector.Error as err:
    print(f"Database error: {err}")
```

### Empty Data Handling

```python
# Check for empty dataframes before loading
if not metadata_df.empty:
    load_to_sql(metadata_df, media_df, colors_df, connection)
else:
    print("No data to load")
```

### Memory Management for Large Datasets

```python
# Process in chunks
def process_in_chunks(artifacts, chunk_size=1000):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        metadata_df, media_df, colors_df = transform_artifacts(chunk)
        load_to_sql(metadata_df, media_df, colors_df, connection)
        print(f"Processed {i + len(chunk)}/{len(artifacts)}")
```

## Running the Application

```bash
# Run Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```
