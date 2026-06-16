---
name: harvard-art-museums-data-pipeline
description: Build end-to-end data engineering pipelines with Harvard Art Museums API, ETL workflows, SQL analytics, and Streamlit dashboards
triggers:
  - build a data pipeline with Harvard Art Museums API
  - create an ETL pipeline for museum artifact data
  - set up Harvard artifacts analytics with Streamlit
  - query and visualize Harvard Art Museums collection data
  - design a data engineering project with museum APIs
  - implement SQL analytics for artifact collections
  - extract and analyze Harvard museum data
  - build interactive dashboards for art museum data
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build complete data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations for museum artifact collections.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is an end-to-end data pipeline that:

- **Extracts** artifact data from the Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational database tables
- **Loads** data into MySQL/TiDB Cloud with batch inserts
- **Analyzes** data using predefined SQL queries
- **Visualizes** results through interactive Plotly charts in a Streamlit dashboard

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Get your API key from: https://www.harvardartmuseums.org/collections/api

Store it in environment variables or a `.env` file:

```bash
# .env
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

Set up MySQL or TiDB Cloud connection:

```python
import os
from dotenv import load_dotenv

load_dotenv()

DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

### 3. Database Schema

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
    description TEXT,
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    media_url VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts, info = fetch_artifacts(api_key, page=1, size=50)

print(f"Total records: {info['totalrecords']}")
print(f"Pages: {info['pages']}")
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

class HarvardETL:
    def __init__(self, db_config, api_key):
        self.db_config = db_config
        self.api_key = api_key
        self.connection = None
    
    def connect_db(self):
        """Establish database connection"""
        try:
            self.connection = mysql.connector.connect(**self.db_config)
            return self.connection
        except Error as e:
            print(f"Database connection error: {e}")
            return None
    
    def extract_artifacts(self, num_pages=5):
        """Extract artifacts from API"""
        all_artifacts = []
        
        for page in range(1, num_pages + 1):
            artifacts, info = fetch_artifacts(self.api_key, page=page)
            all_artifacts.extend(artifacts)
            print(f"Extracted page {page}/{num_pages}")
        
        return all_artifacts
    
    def transform_artifacts(self, artifacts):
        """Transform nested JSON to relational format"""
        metadata_list = []
        media_list = []
        colors_list = []
        
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
                'description': artifact.get('description'),
                'url': artifact.get('url')
            }
            metadata_list.append(metadata)
            
            # Extract media
            if artifact.get('images'):
                for img in artifact['images']:
                    media_list.append({
                        'artifact_id': artifact['id'],
                        'media_type': 'image',
                        'media_url': img.get('baseimageurl')
                    })
            
            # Extract colors
            if artifact.get('colors'):
                for color in artifact['colors']:
                    colors_list.append({
                        'artifact_id': artifact['id'],
                        'color_hex': color.get('hex'),
                        'color_percentage': color.get('percent')
                    })
        
        return (
            pd.DataFrame(metadata_list),
            pd.DataFrame(media_list),
            pd.DataFrame(colors_list)
        )
    
    def load_data(self, df_metadata, df_media, df_colors):
        """Load data into database with batch inserts"""
        cursor = self.connection.cursor()
        
        # Load metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, dated, description, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, df_metadata.values.tolist())
        
        # Load media
        media_query = """
        INSERT INTO artifactmedia (artifact_id, media_type, media_url)
        VALUES (%s, %s, %s)
        """
        cursor.executemany(media_query, df_media.values.tolist())
        
        # Load colors
        colors_query = """
        INSERT INTO artifactcolors (artifact_id, color_hex, color_percentage)
        VALUES (%s, %s, %s)
        """
        cursor.executemany(colors_query, df_colors.values.tolist())
        
        self.connection.commit()
        print(f"Loaded {len(df_metadata)} artifacts")
    
    def run_pipeline(self, num_pages=5):
        """Execute complete ETL pipeline"""
        self.connect_db()
        
        # Extract
        artifacts = self.extract_artifacts(num_pages)
        
        # Transform
        df_metadata, df_media, df_colors = self.transform_artifacts(artifacts)
        
        # Load
        self.load_data(df_metadata, df_media, df_colors)
        
        self.connection.close()
        print("ETL pipeline completed successfully")
```

### 3. SQL Analytics Queries

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY century
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Media Coverage": """
        SELECT 
            COUNT(DISTINCT am.artifact_id) as artifacts_with_media,
            COUNT(DISTINCT amd.id) as total_artifacts,
            ROUND(COUNT(DISTINCT am.artifact_id) * 100.0 / COUNT(DISTINCT amd.id), 2) as coverage_pct
        FROM artifactmetadata amd
        LEFT JOIN artifactmedia am ON amd.id = am.artifact_id
    """,
    
    "Top Colors": """
        SELECT color_hex, COUNT(*) as usage_count, AVG(color_percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Classification Distribution": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 10
    """
}

def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Database connection
    connection = mysql.connector.connect(**DB_CONFIG)
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    selected_query = st.sidebar.selectbox(
        "Choose Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Execute query
    if st.sidebar.button("Run Analysis"):
        with st.spinner("Executing query..."):
            query = ANALYTICS_QUERIES[selected_query]
            df_result = execute_query(connection, query)
            
            # Display results
            st.subheader(f"Results: {selected_query}")
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) >= 2:
                fig = px.bar(
                    df_result,
                    x=df_result.columns[0],
                    y=df_result.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)
    
    # ETL Pipeline controls
    st.sidebar.header("ETL Pipeline")
    num_pages = st.sidebar.slider("Number of pages to fetch", 1, 10, 5)
    
    if st.sidebar.button("Run ETL Pipeline"):
        api_key = os.getenv('HARVARD_API_KEY')
        etl = HarvardETL(DB_CONFIG, api_key)
        
        with st.spinner("Running ETL pipeline..."):
            etl.run_pipeline(num_pages)
        st.success("ETL pipeline completed!")
    
    connection.close()

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Rate-Limited API Calls

```python
import time

def fetch_with_rate_limit(api_key, pages, delay=1):
    """Fetch data with rate limiting"""
    results = []
    
    for page in range(1, pages + 1):
        artifacts, info = fetch_artifacts(api_key, page)
        results.extend(artifacts)
        
        if page < pages:
            time.sleep(delay)  # Respect API rate limits
    
    return results
```

### Incremental Data Loading

```python
def get_last_artifact_id(connection):
    """Get the last loaded artifact ID"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(etl, connection):
    """Load only new artifacts"""
    last_id = get_last_artifact_id(connection)
    
    # Fetch artifacts with ID > last_id
    # Transform and load only new records
    pass
```

## Troubleshooting

### API Key Issues

```python
# Verify API key is loaded
import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv('HARVARD_API_KEY')

if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment variables")
```

### Database Connection Errors

```python
# Test database connection
try:
    connection = mysql.connector.connect(**DB_CONFIG)
    print("Database connection successful")
    connection.close()
except mysql.connector.Error as e:
    print(f"Error: {e}")
    print("Check DB_HOST, DB_USER, DB_PASSWORD, DB_NAME in .env")
```

### Empty Query Results

```python
# Verify data exists
cursor = connection.cursor()
cursor.execute("SELECT COUNT(*) FROM artifactmetadata")
count = cursor.fetchone()[0]
print(f"Total artifacts in database: {count}")

if count == 0:
    print("Run ETL pipeline first to load data")
```

### Memory Issues with Large Datasets

```python
# Use chunked processing
def load_in_chunks(df, connection, table_name, chunk_size=1000):
    """Load DataFrame in chunks to avoid memory issues"""
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i+chunk_size]
        chunk.to_sql(table_name, connection, if_exists='append', index=False)
        print(f"Loaded chunk {i//chunk_size + 1}")
```

This skill provides complete guidance for building production-ready data engineering pipelines with museum APIs, SQL analytics, and interactive dashboards.
