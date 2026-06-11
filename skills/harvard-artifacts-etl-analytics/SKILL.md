---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, Streamlit, and SQL
triggers:
  - how do I extract data from Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create analytics dashboard with Streamlit and Harvard API
  - set up SQL database for Harvard artifacts collection
  - visualize museum artifact data with Plotly
  - implement batch data loading for Harvard Art Museums
  - query and analyze artifact metadata with SQL
  - create end-to-end data engineering pipeline for art museums
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Collect artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON data into relational SQL tables
- **SQL Analytics**: Pre-built analytical queries for artifact insights
- **Interactive Dashboard**: Streamlit-based UI for data collection, query execution, and visualization
- **Data Visualization**: Plotly charts for real-time analytics

**Architecture Flow**: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required dependencies** (requirements.txt):
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Set it in the Streamlit app's sidebar or use environment variables:

```bash
export HARVARD_API_KEY="your_api_key_here"
```

### 2. Database Configuration

The app supports MySQL or TiDB Cloud. Configure connection in your code or environment:

```python
import os

DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts')
}
```

Environment variables:
```bash
export DB_HOST="your_db_host"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

## Running the Application

```bash
streamlit run app.py
```

The app will launch at `http://localhost:8501`

## Core Components

### 1. API Data Extraction

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_records=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    page = 1
    size = 100  # Max records per request
    
    while len(all_artifacts) < num_records:
        params = {
            'apikey': api_key,
            'page': page,
            'size': min(size, num_records - len(all_artifacts))
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            all_artifacts.extend(records)
            page += 1
        else:
            raise Exception(f"API Error: {response.status_code}")
    
    return all_artifacts[:num_records]
```

### 2. ETL Pipeline - Transform

```python
def transform_artifacts(raw_data):
    """Transform nested JSON into relational tables"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        artifact_id = artifact.get('id')
        
        # Metadata table
        metadata_records.append({
            'artifact_id': artifact_id,
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'technique': artifact.get('technique', 'Unknown'),
            'period': artifact.get('period', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown')
        })
        
        # Media table
        images = artifact.get('images', [])
        for img in images:
            media_records.append({
                'artifact_id': artifact_id,
                'media_type': 'image',
                'base_url': img.get('baseimageurl', ''),
                'alt_text': img.get('alttext', '')
            })
        
        # Colors table
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', ''),
                'color_percent': color.get('percent', 0.0),
                'color_name': color.get('color', 'Unknown')
            })
    
    return {
        'metadata': pd.DataFrame(metadata_records),
        'media': pd.DataFrame(media_records),
        'colors': pd.DataFrame(color_records)
    }
```

### 3. Load Data into SQL

```python
import mysql.connector
from mysql.connector import Error

def create_database_schema(connection):
    """Create database tables"""
    cursor = connection.cursor()
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            century VARCHAR(255),
            department VARCHAR(255),
            classification VARCHAR(255),
            technique VARCHAR(255),
            period VARCHAR(255),
            dated VARCHAR(255)
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            media_type VARCHAR(50),
            base_url VARCHAR(1000),
            alt_text TEXT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_hex VARCHAR(10),
            color_percent FLOAT,
            color_name VARCHAR(100),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    connection.commit()
    cursor.close()

def load_data_to_sql(dataframes, db_config):
    """Batch insert data into SQL tables"""
    try:
        connection = mysql.connector.connect(**db_config)
        create_database_schema(connection)
        cursor = connection.cursor()
        
        # Load metadata
        metadata_values = dataframes['metadata'].values.tolist()
        cursor.executemany("""
            INSERT INTO artifactmetadata 
            (artifact_id, title, culture, century, department, classification, technique, period, dated)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """, metadata_values)
        
        # Load media
        media_values = dataframes['media'].values.tolist()
        cursor.executemany("""
            INSERT INTO artifactmedia (artifact_id, media_type, base_url, alt_text)
            VALUES (%s, %s, %s, %s)
        """, media_values)
        
        # Load colors
        color_values = dataframes['colors'].values.tolist()
        cursor.executemany("""
            INSERT INTO artifactcolors (artifact_id, color_hex, color_percent, color_name)
            VALUES (%s, %s, %s, %s)
        """, color_values)
        
        connection.commit()
        cursor.close()
        connection.close()
        
        return True
    except Error as e:
        print(f"Database error: {e}")
        return False
```

### 4. SQL Analytics Queries

```python
def run_analytical_query(db_config, query_type):
    """Execute predefined analytical queries"""
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture != 'Unknown'
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE century != 'Unknown'
            GROUP BY century
            ORDER BY artifact_count DESC
        """,
        
        'media_availability': """
            SELECT 
                CASE WHEN EXISTS(
                    SELECT 1 FROM artifactmedia 
                    WHERE artifactmedia.artifact_id = artifactmetadata.artifact_id
                ) THEN 'Has Media' ELSE 'No Media' END as media_status,
                COUNT(*) as count
            FROM artifactmetadata
            GROUP BY media_status
        """,
        
        'top_colors': """
            SELECT color_name, COUNT(*) as usage_count
            FROM artifactcolors
            WHERE color_name != 'Unknown'
            GROUP BY color_name
            ORDER BY usage_count DESC
            LIMIT 10
        """,
        
        'artifacts_by_department': """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE department != 'Unknown'
            GROUP BY department
            ORDER BY artifact_count DESC
        """
    }
    
    connection = mysql.connector.connect(**db_config)
    df = pd.read_sql(queries[query_type], connection)
    connection.close()
    
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Artifacts Analytics Dashboard")
    
    # Sidebar for configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    num_records = st.sidebar.slider("Number of Records", 10, 1000, 100)
    
    # Data Collection Tab
    if st.button("Collect Data from API"):
        with st.spinner("Fetching artifacts..."):
            raw_data = fetch_artifacts(api_key, num_records)
            transformed = transform_artifacts(raw_data)
            
            if load_data_to_sql(transformed, DB_CONFIG):
                st.success(f"Successfully loaded {len(raw_data)} artifacts!")
            else:
                st.error("Failed to load data to database")
    
    # Analytics Tab
    st.header("Analytics Queries")
    
    query_options = {
        "Artifacts by Culture": "artifacts_by_culture",
        "Artifacts by Century": "artifacts_by_century",
        "Media Availability": "media_availability",
        "Top Colors": "top_colors",
        "Artifacts by Department": "artifacts_by_department"
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        result_df = run_analytical_query(DB_CONFIG, query_options[selected_query])
        
        st.subheader("Query Results")
        st.dataframe(result_df)
        
        # Visualization
        if len(result_df.columns) >= 2:
            fig = px.bar(result_df, x=result_df.columns[0], y=result_df.columns[1],
                        title=selected_query)
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(api_key, num_records, delay=1):
    """Fetch data with rate limiting"""
    all_data = []
    batch_size = 100
    
    for i in range(0, num_records, batch_size):
        batch = fetch_artifacts(api_key, min(batch_size, num_records - i))
        all_data.extend(batch)
        time.sleep(delay)  # Rate limit
    
    return all_data
```

### Handling Missing Data

```python
def clean_artifact_data(artifact):
    """Handle missing or null values"""
    defaults = {
        'title': 'Untitled',
        'culture': 'Unknown',
        'century': 'Unknown',
        'department': 'Unspecified',
        'classification': 'Other'
    }
    
    for key, default_value in defaults.items():
        if not artifact.get(key):
            artifact[key] = default_value
    
    return artifact
```

## Troubleshooting

**API Rate Limiting**: The Harvard API has rate limits. Add delays between requests or use batch processing.

**Database Connection Errors**: Ensure your database server is running and credentials are correct in environment variables.

**Empty Results**: Check that your API key is valid and the API endpoint is accessible.

**Memory Issues with Large Datasets**: Process data in batches rather than loading all records at once.

```python
# Process in chunks
def process_in_chunks(data, chunk_size=1000):
    for i in range(0, len(data), chunk_size):
        chunk = data[i:i + chunk_size]
        transformed = transform_artifacts(chunk)
        load_data_to_sql(transformed, DB_CONFIG)
```

**Foreign Key Constraint Errors**: Ensure metadata records are inserted before media and color records.

```python
# Correct order of insertion
load_data_to_sql({'metadata': metadata_df}, DB_CONFIG)
load_data_to_sql({'media': media_df, 'colors': colors_df}, DB_CONFIG)
```
