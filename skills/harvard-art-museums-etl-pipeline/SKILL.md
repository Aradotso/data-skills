---
name: harvard-art-museums-etl-pipeline
description: Build end-to-end ETL pipelines with Harvard Art Museums API, SQL analytics, and Streamlit visualization
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create a data engineering project with Harvard API
  - set up SQL analytics for art museum data
  - visualize Harvard Art Museums collection data
  - implement batch ETL with pandas and MySQL
  - query artifact metadata with SQL
  - build a Streamlit analytics dashboard for museum data
---

# Harvard Art Museums ETL Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering Analytics App is an end-to-end data pipeline that demonstrates:
- **ETL Operations**: Extract data from Harvard Art Museums API, transform nested JSON, load into SQL
- **Database Design**: Relational schema with artifact metadata, media, and color tables
- **Analytics**: Pre-built SQL queries for insights
- **Visualization**: Interactive Streamlit dashboards with Plotly charts

**Architecture**: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="your_database_name"
```

**Required packages**:
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Harvard Art Museums API Integration

### API Configuration

```python
import requests
import os

# API endpoint and authentication
API_BASE_URL = "https://api.harvardartmuseums.org"
API_KEY = os.getenv("HARVARD_API_KEY")

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    url = f"{API_BASE_URL}/object"
    params = {
        "apikey": API_KEY,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

# Fetch multiple pages
def collect_artifacts(num_pages=5):
    all_artifacts = []
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(page=page)
        all_artifacts.extend(data.get("records", []))
        print(f"Fetched page {page}, total artifacts: {len(all_artifacts)}")
    return all_artifacts
```

### Rate Limiting and Error Handling

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(page, max_retries=3):
    """
    Fetch data with retry logic and rate limiting
    """
    for attempt in range(max_retries):
        try:
            time.sleep(0.5)  # Rate limiting
            return fetch_artifacts(page=page)
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            print(f"Retry {attempt + 1} for page {page}")
            time.sleep(2 ** attempt)  # Exponential backoff
```

## ETL Pipeline

### Extract

```python
def extract_artifact_data(artifact):
    """
    Extract key fields from artifact JSON
    """
    return {
        "object_id": artifact.get("objectid"),
        "title": artifact.get("title", "Unknown"),
        "culture": artifact.get("culture"),
        "century": artifact.get("century"),
        "classification": artifact.get("classification"),
        "department": artifact.get("department"),
        "dated": artifact.get("dated"),
        "medium": artifact.get("medium"),
        "dimensions": artifact.get("dimensions"),
        "description": artifact.get("description"),
        "provenance": artifact.get("provenance")
    }
```

### Transform

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """
    Transform nested JSON into relational dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_artifacts:
        object_id = artifact.get("objectid")
        
        # Metadata table
        metadata_list.append(extract_artifact_data(artifact))
        
        # Media table (one-to-many)
        for image in artifact.get("images", []):
            media_list.append({
                "object_id": object_id,
                "image_url": image.get("baseimageurl"),
                "image_width": image.get("width"),
                "image_height": image.get("height"),
                "alt_text": image.get("alttext")
            })
        
        # Colors table (one-to-many)
        for color in artifact.get("colors", []):
            colors_list.append({
                "object_id": object_id,
                "color_hex": color.get("hex"),
                "color_name": color.get("color"),
                "percentage": color.get("percent")
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """
    Create MySQL database connection
    """
    return mysql.connector.connect(
        host=os.getenv("DB_HOST"),
        user=os.getenv("DB_USER"),
        password=os.getenv("DB_PASSWORD"),
        database=os.getenv("DB_NAME")
    )

def create_tables(conn):
    """
    Create database schema
    """
    cursor = conn.cursor()
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            object_id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            medium TEXT,
            dimensions VARCHAR(500),
            description TEXT,
            provenance TEXT
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            image_url VARCHAR(1000),
            image_width INT,
            image_height INT,
            alt_text TEXT,
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            color_hex VARCHAR(7),
            color_name VARCHAR(50),
            percentage FLOAT,
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    conn.commit()
    cursor.close()

def batch_insert_dataframe(conn, df, table_name):
    """
    Batch insert dataframe into SQL table
    """
    if df.empty:
        return
    
    cursor = conn.cursor()
    columns = ", ".join(df.columns)
    placeholders = ", ".join(["%s"] * len(df.columns))
    
    insert_query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Convert dataframe to list of tuples
    data = [tuple(row) for row in df.values]
    
    cursor.executemany(insert_query, data)
    conn.commit()
    cursor.close()
    print(f"Inserted {len(data)} rows into {table_name}")
```

## Complete ETL Pipeline

```python
def run_etl_pipeline(num_pages=10):
    """
    Execute full ETL pipeline
    """
    print("Starting ETL pipeline...")
    
    # Extract
    print("Extracting data from API...")
    raw_artifacts = collect_artifacts(num_pages=num_pages)
    
    # Transform
    print("Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(raw_artifacts)
    
    # Load
    print("Loading data into database...")
    conn = get_db_connection()
    create_tables(conn)
    
    batch_insert_dataframe(conn, df_metadata, "artifactmetadata")
    batch_insert_dataframe(conn, df_media, "artifactmedia")
    batch_insert_dataframe(conn, df_colors, "artifactcolors")
    
    conn.close()
    print("ETL pipeline completed successfully!")

# Run pipeline
if __name__ == "__main__":
    run_etl_pipeline(num_pages=5)
```

## SQL Analytics Queries

### Common Analytics Patterns

```python
def execute_query(query):
    """
    Execute SQL query and return results as dataframe
    """
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Query 1: Artifacts by century
query_by_century = """
    SELECT century, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query 2: Top cultures by artifact count
query_by_culture = """
    SELECT culture, COUNT(*) as count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY count DESC
    LIMIT 15
"""

# Query 3: Artifacts with most images
query_most_images = """
    SELECT m.object_id, m.title, COUNT(med.id) as image_count
    FROM artifactmetadata m
    JOIN artifactmedia med ON m.object_id = med.object_id
    GROUP BY m.object_id, m.title
    ORDER BY image_count DESC
    LIMIT 10
"""

# Query 4: Color distribution
query_color_distribution = """
    SELECT color_name, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
    FROM artifactcolors
    GROUP BY color_name
    ORDER BY usage_count DESC
    LIMIT 10
"""

# Query 5: Department statistics
query_department_stats = """
    SELECT department, 
           COUNT(*) as total_artifacts,
           COUNT(DISTINCT culture) as unique_cultures
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY total_artifacts DESC
"""
```

## Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    st.markdown("### ETL Pipeline & SQL Analytics")
    
    # Sidebar for ETL controls
    st.sidebar.header("ETL Pipeline")
    num_pages = st.sidebar.slider("Number of API pages", 1, 20, 5)
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            run_etl_pipeline(num_pages=num_pages)
            st.success("ETL completed!")
    
    # Analytics section
    st.header("SQL Analytics")
    
    query_options = {
        "Artifacts by Century": query_by_century,
        "Top Cultures": query_by_culture,
        "Most Documented Artifacts": query_most_images,
        "Color Distribution": query_color_distribution,
        "Department Statistics": query_department_stats
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Execute Query"):
        df = execute_query(query_options[selected_query])
        
        # Display results
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query)
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Configuration

Create a `.env` file:

```bash
HARVARD_API_KEY=your_api_key_from_harvard
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

Get your Harvard API key from: https://docs.harvardartmuseums.org/

## Troubleshooting

**API Rate Limiting**:
```python
# Add sleep between requests
import time
time.sleep(0.5)  # 500ms delay
```

**Database Connection Issues**:
```python
# Test connection
try:
    conn = get_db_connection()
    print("Connection successful")
    conn.close()
except Error as e:
    print(f"Connection failed: {e}")
```

**Empty API Response**:
```python
# Verify API key and check response
response = fetch_artifacts(page=1)
print(f"Total records: {response.get('info', {}).get('totalrecords')}")
```

**Foreign Key Constraint Errors**:
```python
# Ensure metadata is inserted before media/colors
# Use IGNORE to skip duplicates
cursor.execute("INSERT IGNORE INTO artifactmetadata ...")
```

**Memory Issues with Large Datasets**:
```python
# Process in smaller batches
def batch_process(artifacts, batch_size=100):
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        # Process batch
```
