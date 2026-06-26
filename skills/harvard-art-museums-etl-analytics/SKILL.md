---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifact data
  - create a data engineering project with Harvard Art Museums API
  - set up artifact analytics dashboard with Streamlit
  - extract and analyze art museum collections data
  - implement SQL analytics for cultural heritage data
  - visualize museum artifact metadata with plotly
  - build a data pipeline from API to database to dashboard
  - create an art collection data warehouse
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for cultural heritage data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App:
- **Extracts** artifact data from the Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables (metadata, media, colors)
- **Loads** structured data into MySQL/TiDB Cloud databases
- **Analyzes** data through 20+ predefined SQL analytical queries
- **Visualizes** insights using Plotly charts in an interactive Streamlit interface

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

### Prerequisites
```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Environment Setup
Create a `.env` file in the project root:
```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup
```sql
CREATE DATABASE harvard_artifacts;

USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    department VARCHAR(255),
    classification VARCHAR(255),
    dated VARCHAR(255),
    division VARCHAR(255),
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Data Collection

```python
import requests
import time
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, num_pages=10, page_size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            print(f"Fetched page {page}: {len(artifacts)} artifacts")
        else:
            print(f"Error on page {page}: {response.status_code}")
            break
        
        # Rate limiting - respect API limits
        time.sleep(0.5)
    
    return all_artifacts

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts = fetch_artifacts(api_key, num_pages=5)
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifacts(artifacts):
    """Transform raw API data into normalized dataframes"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown')[:500],
            'culture': artifact.get('culture', 'Unknown')[:255],
            'period': artifact.get('period', 'Unknown')[:255],
            'century': artifact.get('century', 'Unknown')[:255],
            'department': artifact.get('department', 'Unknown')[:255],
            'classification': artifact.get('classification', 'Unknown')[:255],
            'dated': artifact.get('dated', 'Unknown')[:255],
            'division': artifact.get('division', 'Unknown')[:255],
            'url': artifact.get('url', '')[:500]
        }
        metadata_list.append(metadata)
        
        # Extract media
        if artifact.get('primaryimageurl'):
            media = {
                'artifact_id': artifact.get('id'),
                'media_type': 'primary_image',
                'baseimageurl': artifact.get('primaryimageurl', '')[:500],
                'iiifbaseuri': artifact.get('iiifbaseuri', '')[:500]
            }
            media_list.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color_data in colors:
            color = {
                'artifact_id': artifact.get('id'),
                'color': color_data.get('color', 'Unknown')[:50],
                'percentage': float(color_data.get('percent', 0))
            }
            colors_list.append(color)
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

### 3. Database Loading

```python
def load_to_database(df_metadata, df_media, df_colors):
    """Load transformed data into MySQL database"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        cursor = connection.cursor()
        
        # Insert metadata (with REPLACE to handle duplicates)
        for _, row in df_metadata.iterrows():
            query = """
            REPLACE INTO artifactmetadata 
            (id, title, culture, period, century, department, 
             classification, dated, division, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        # Insert media
        for _, row in df_media.iterrows():
            query = """
            INSERT INTO artifactmedia 
            (artifact_id, media_type, baseimageurl, iiifbaseuri)
            VALUES (%s, %s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        # Insert colors
        for _, row in df_colors.iterrows():
            query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, percentage)
            VALUES (%s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        connection.commit()
        print(f"Loaded {len(df_metadata)} metadata, {len(df_media)} media, {len(df_colors)} colors")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 4. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture != 'Unknown'
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != 'Unknown'
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as with_media,
            (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
            ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / 
                  (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
        FROM artifactmedia m
    """,
    
    "Top 10 Colors Across All Artifacts": """
        SELECT color, 
               COUNT(*) as occurrences,
               ROUND(AVG(percentage), 2) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY occurrences DESC
        LIMIT 10
    """,
    
    "Artifacts with Multiple Colors": """
        SELECT a.artifact_id, 
               m.title,
               COUNT(*) as color_count,
               GROUP_CONCAT(a.color ORDER BY a.percentage DESC) as colors
        FROM artifactcolors a
        JOIN artifactmetadata m ON a.artifact_id = m.id
        GROUP BY a.artifact_id, m.title
        HAVING color_count > 3
        ORDER BY color_count DESC
        LIMIT 20
    """
}

def execute_query(query_name):
    """Execute analytical query and return results"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    query = ANALYTICS_QUERIES[query_name]
    df = pd.read_sql(query, connection)
    connection.close()
    
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar for data collection
    with st.sidebar:
        st.header("Data Collection")
        
        num_pages = st.slider("Number of pages to fetch", 1, 20, 5)
        
        if st.button("Fetch & Load Data"):
            with st.spinner("Fetching artifacts..."):
                api_key = os.getenv('HARVARD_API_KEY')
                artifacts = fetch_artifacts(api_key, num_pages=num_pages)
                
                st.success(f"Fetched {len(artifacts)} artifacts")
                
                df_metadata, df_media, df_colors = transform_artifacts(artifacts)
                load_to_database(df_metadata, df_media, df_colors)
                
                st.success("Data loaded successfully!")
    
    # Main analytics section
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            df_result = execute_query(query_name)
            
            st.subheader("Query Results")
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) >= 2:
                st.subheader("Visualization")
                
                x_col = df_result.columns[0]
                y_col = df_result.columns[1]
                
                fig = px.bar(
                    df_result.head(15),
                    x=x_col,
                    y=y_col,
                    title=query_name
                )
                
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Full ETL Workflow
```python
# Complete pipeline from API to database
def run_full_etl_pipeline(api_key, num_pages=10):
    """Execute complete ETL workflow"""
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_artifacts(api_key, num_pages)
    
    # Transform
    print("Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(artifacts)
    
    # Load
    print("Loading to database...")
    load_to_database(df_metadata, df_media, df_colors)
    
    print(f"ETL complete: {len(artifacts)} artifacts processed")
    
    return {
        'artifacts': len(artifacts),
        'metadata_records': len(df_metadata),
        'media_records': len(df_media),
        'color_records': len(df_colors)
    }
```

### Batch Processing for Large Datasets
```python
def batch_insert(dataframe, table_name, batch_size=1000):
    """Insert data in batches for better performance"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    cursor = connection.cursor()
    
    for start in range(0, len(dataframe), batch_size):
        batch = dataframe.iloc[start:start + batch_size]
        # Insert batch...
        connection.commit()
        print(f"Inserted batch {start//batch_size + 1}")
    
    cursor.close()
    connection.close()
```

## Running the Application

```bash
# Run the Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Troubleshooting

**API Rate Limiting:**
```python
# Adjust sleep time between requests
time.sleep(1)  # Increase to 1 second if getting 429 errors
```

**Database Connection Issues:**
```python
# Test connection
def test_db_connection():
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD')
        )
        print("Connection successful!")
        connection.close()
    except Error as e:
        print(f"Connection failed: {e}")
```

**Missing API Key:**
```python
# Validate environment variables
required_vars = ['HARVARD_API_KEY', 'DB_HOST', 'DB_USER', 'DB_PASSWORD']
for var in required_vars:
    if not os.getenv(var):
        raise ValueError(f"Missing required environment variable: {var}")
```

**Large Dataset Memory Issues:**
```python
# Use chunked processing
def fetch_artifacts_chunked(api_key, num_pages, chunk_size=5):
    for chunk_start in range(0, num_pages, chunk_size):
        chunk_end = min(chunk_start + chunk_size, num_pages)
        artifacts = fetch_artifacts(api_key, num_pages=chunk_end-chunk_start)
        # Process chunk immediately
        df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        load_to_database(df_metadata, df_media, df_colors)
```

This skill provides a complete framework for building data engineering pipelines with museum APIs, implementing ETL workflows, and creating analytics dashboards.
