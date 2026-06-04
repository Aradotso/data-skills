---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline with Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract Harvard Art Museums API data to SQL
  - analyze art museum collection data
  - set up Streamlit app for artifact visualization
  - query Harvard museum artifacts with SQL
  - transform Harvard API JSON to relational database
  - visualize art collection data with Plotly
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming nested JSON into relational structures, loading into SQL databases (MySQL/TiDB), running analytical queries, and visualizing results through an interactive Streamlit dashboard.

**Core Components:**
- API data extraction with pagination and rate limiting
- ETL pipeline transforming JSON to normalized SQL tables
- SQL analytics with 20+ predefined queries
- Interactive Streamlit dashboard with Plotly visualizations

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

**Required packages:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Project Architecture

**Data Flow:** API → Extract → Transform → Load → SQL → Analytics → Visualization

**Database Schema:**
- `artifactmetadata` - Core artifact information (id, title, culture, century, department)
- `artifactmedia` - Media files associated with artifacts (images, URLs)
- `artifactcolors` - Color palette data extracted from images

## API Integration

### Fetching Data from Harvard Art Museums API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv("HARVARD_API_KEY")
data = fetch_artifacts(api_key, page=1, size=50)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Pages: {data['info']['pages']}")
```

### Handling Pagination

```python
def fetch_all_artifacts(api_key, max_records=500):
    """
    Fetch multiple pages of artifacts with rate limiting
    """
    import time
    
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        try:
            data = fetch_artifacts(api_key, page=page, size=size)
            artifacts = data.get("records", [])
            
            if not artifacts:
                break
                
            all_artifacts.extend(artifacts)
            print(f"Fetched page {page}: {len(artifacts)} artifacts")
            
            page += 1
            time.sleep(0.5)  # Rate limiting
            
        except Exception as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts[:max_records]
```

## ETL Pipeline

### Extract: Parse API Response

```python
def extract_artifact_metadata(artifact):
    """
    Extract core metadata from artifact JSON
    """
    return {
        "artifact_id": artifact.get("id"),
        "title": artifact.get("title", "Unknown"),
        "culture": artifact.get("culture", "Unknown"),
        "century": artifact.get("century", "Unknown"),
        "classification": artifact.get("classification", "Unknown"),
        "department": artifact.get("department", "Unknown"),
        "dated": artifact.get("dated", "Unknown"),
        "medium": artifact.get("medium", "Unknown"),
        "creditline": artifact.get("creditline", "Unknown")
    }
```

### Transform: Normalize Nested Data

```python
import pandas as pd

def transform_artifacts_to_tables(artifacts):
    """
    Transform nested JSON into normalized relational tables
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get("id")
        
        # Metadata table
        metadata_list.append(extract_artifact_metadata(artifact))
        
        # Media table (one-to-many)
        images = artifact.get("images", [])
        for img in images:
            media_list.append({
                "artifact_id": artifact_id,
                "image_id": img.get("imageid"),
                "base_url": img.get("baseimageurl"),
                "format": img.get("format"),
                "width": img.get("width"),
                "height": img.get("height")
            })
        
        # Colors table (one-to-many)
        colors = artifact.get("colors", [])
        for color in colors:
            colors_list.append({
                "artifact_id": artifact_id,
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

### Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """
    Create MySQL database connection using environment variables
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv("DB_HOST"),
            user=os.getenv("DB_USER"),
            password=os.getenv("DB_PASSWORD"),
            database=os.getenv("DB_NAME")
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def create_tables(connection):
    """
    Create normalized tables with foreign key relationships
    """
    cursor = connection.cursor()
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            dated VARCHAR(255),
            medium TEXT,
            creditline TEXT
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_id INT,
            base_url VARCHAR(500),
            format VARCHAR(50),
            width INT,
            height INT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_hex VARCHAR(10),
            color_name VARCHAR(100),
            percentage FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    connection.commit()

def batch_insert_dataframe(connection, df, table_name):
    """
    Batch insert DataFrame into SQL table
    """
    if df.empty:
        return
    
    cursor = connection.cursor()
    
    # Replace NaN with None for SQL NULL
    df = df.where(pd.notnull(df), None)
    
    columns = ", ".join(df.columns)
    placeholders = ", ".join(["%s"] * len(df.columns))
    
    insert_query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} rows into {table_name}")
```

### Complete ETL Workflow

```python
def run_etl_pipeline(api_key, max_records=500):
    """
    Execute complete ETL pipeline
    """
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_all_artifacts(api_key, max_records)
    
    # Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts_to_tables(artifacts)
    
    # Load
    print("Loading data into database...")
    connection = create_database_connection()
    
    if connection:
        create_tables(connection)
        batch_insert_dataframe(connection, metadata_df, "artifactmetadata")
        batch_insert_dataframe(connection, media_df, "artifactmedia")
        batch_insert_dataframe(connection, colors_df, "artifactcolors")
        connection.close()
        print("ETL pipeline completed successfully!")
    else:
        print("Failed to connect to database")
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Query 1: Artifact count by culture
query_culture = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query 2: Artifacts with most images
query_images = """
    SELECT m.title, m.culture, COUNT(med.image_id) as image_count
    FROM artifactmetadata m
    JOIN artifactmedia med ON m.artifact_id = med.artifact_id
    GROUP BY m.artifact_id
    ORDER BY image_count DESC
    LIMIT 15
"""

# Query 3: Color distribution across artifacts
query_colors = """
    SELECT color_name, COUNT(DISTINCT artifact_id) as artifact_count,
           AVG(percentage) as avg_percentage
    FROM artifactcolors
    GROUP BY color_name
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query 4: Department-wise classification
query_dept = """
    SELECT department, classification, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department, classification
    ORDER BY department, count DESC
"""

def execute_query(connection, query):
    """
    Execute SQL query and return DataFrame
    """
    cursor = connection.cursor()
    cursor.execute(query)
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    # ETL Section
    st.header("1. Data Collection (ETL)")
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            run_etl_pipeline(api_key)
            st.success("ETL completed!")
    
    # Analytics Section
    st.header("2. SQL Analytics")
    
    queries = {
        "Artifacts by Culture": query_culture,
        "Top Artifacts by Image Count": query_images,
        "Color Distribution": query_colors,
        "Department Classification": query_dept
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Execute Query"):
        connection = create_database_connection()
        if connection:
            df = execute_query(connection, queries[selected_query])
            st.dataframe(df)
            
            # Visualization
            if len(df.columns) >= 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                            title=selected_query)
                st.plotly_chart(fig)
            
            connection.close()

if __name__ == "__main__":
    main()
```

### Running the Dashboard

```bash
streamlit run app.py
```

## Common Patterns

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, max_records=500):
    """
    ETL pipeline with comprehensive error handling
    """
    try:
        artifacts = fetch_all_artifacts(api_key, max_records)
        
        if not artifacts:
            raise ValueError("No artifacts fetched from API")
        
        metadata_df, media_df, colors_df = transform_artifacts_to_tables(artifacts)
        
        connection = create_database_connection()
        if not connection:
            raise ConnectionError("Failed to connect to database")
        
        create_tables(connection)
        batch_insert_dataframe(connection, metadata_df, "artifactmetadata")
        batch_insert_dataframe(connection, media_df, "artifactmedia")
        batch_insert_dataframe(connection, colors_df, "artifactcolors")
        
        connection.close()
        return True, "ETL completed successfully"
        
    except Exception as e:
        return False, f"ETL failed: {str(e)}"
```

### Data Validation

```python
def validate_artifact_data(df):
    """
    Validate transformed data before loading
    """
    issues = []
    
    if df.empty:
        issues.append("DataFrame is empty")
    
    if "artifact_id" in df.columns and df["artifact_id"].isnull().any():
        issues.append("Missing artifact IDs detected")
    
    if df.duplicated(subset=["artifact_id"]).any():
        issues.append("Duplicate artifact IDs found")
    
    return len(issues) == 0, issues
```

## Troubleshooting

**API Rate Limiting:**
- Add `time.sleep(0.5)` between requests
- Implement exponential backoff for failed requests
- Use smaller page sizes (50 instead of 100)

**Database Connection Issues:**
- Verify environment variables are set correctly
- Check firewall rules for cloud databases (TiDB)
- Ensure database exists before running ETL

**Memory Issues with Large Datasets:**
- Process data in chunks instead of all at once
- Use batch inserts with smaller batch sizes
- Clear DataFrame variables after loading

**Missing Data in API Response:**
- Always use `.get()` with default values for JSON fields
- Validate data before transformation
- Handle empty arrays for images and colors gracefully

**Streamlit Performance:**
- Use `@st.cache_data` for expensive database queries
- Limit result set sizes with SQL LIMIT clauses
- Close database connections promptly
