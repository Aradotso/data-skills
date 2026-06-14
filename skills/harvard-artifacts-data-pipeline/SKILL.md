---
name: harvard-artifacts-data-pipeline
description: Build end-to-end data engineering pipelines for museum artifact data using Harvard Art Museums API, ETL processing, SQL analytics, and Streamlit visualization
triggers:
  - create a data pipeline for museum artifacts
  - build ETL pipeline with Harvard Art Museums API
  - set up artifact analytics dashboard with Streamlit
  - process museum collection data with SQL
  - visualize art museum data with Plotly
  - implement Harvard artifacts data engineering project
  - query and analyze museum artifact metadata
  - build museum data warehouse with Python
---

# Harvard Artifacts Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables you to build production-ready data engineering pipelines using the Harvard Art Museums API. It covers ETL workflows, relational database design, SQL analytics, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:
- **API Integration**: Fetch artifact metadata from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transform nested JSON into normalized relational tables
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboards**: Streamlit UI with Plotly visualizations
- **Database Schema**: Proper relational design with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables

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

### 1. Harvard Art Museums API Key

Get your API key from: https://www.harvardartmuseums.org/collections/api

```python
# .env file
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Connection

Configure MySQL/TiDB Cloud connection:

```python
# .env file
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### 3. Streamlit Secrets (Alternative)

```toml
# .streamlit/secrets.toml
[api]
harvard_api_key = "your_api_key_here"

[database]
host = "your_database_host"
port = 3306
user = "your_username"
password = "your_password"
database = "harvard_artifacts"
```

## Core Components

### API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
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
```

### ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

def transform_artifacts(raw_artifacts: List[Dict]) -> tuple:
    """Transform nested JSON into relational tables"""
    
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata = {
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'classification': artifact.get('classification'),
            'century': artifact.get('century'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'credit_line': artifact.get('creditline')
        }
        metadata_records.append(metadata)
        
        # Extract media (images)
        if artifact.get('images'):
            for img in artifact['images']:
                media = {
                    'artifact_id': artifact.get('id'),
                    'media_type': 'image',
                    'base_url': img.get('baseimageurl'),
                    'format': img.get('format'),
                    'width': img.get('width'),
                    'height': img.get('height')
                }
                media_records.append(media)
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_record = {
                    'artifact_id': artifact.get('id'),
                    'color_hex': color.get('hex'),
                    'color_name': color.get('color'),
                    'percentage': color.get('percent')
                }
                color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

# Usage
metadata_df, media_df, colors_df = transform_artifacts(artifacts)
```

### Database Schema Creation

```python
def create_database_schema(connection):
    """Create relational tables for artifact data"""
    
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            classification VARCHAR(255),
            century VARCHAR(100),
            department VARCHAR(255),
            division VARCHAR(255),
            dated VARCHAR(255),
            url TEXT,
            credit_line TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            media_type VARCHAR(50),
            base_url TEXT,
            format VARCHAR(50),
            width INT,
            height INT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_hex VARCHAR(7),
            color_name VARCHAR(100),
            percentage DECIMAL(5,2),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    connection.commit()
    cursor.close()

# Usage
conn = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)
create_database_schema(conn)
```

### Batch Data Loading

```python
def load_data_to_sql(connection, metadata_df, media_df, colors_df):
    """Batch insert data into SQL tables"""
    
    cursor = connection.cursor()
    
    # Insert metadata
    metadata_query = """
        INSERT INTO artifactmetadata 
        (artifact_id, title, culture, classification, century, department, division, dated, url, credit_line)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    cursor.executemany(metadata_query, metadata_df.values.tolist())
    
    # Insert media
    if not media_df.empty:
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, media_type, base_url, format, width, height)
            VALUES (%s, %s, %s, %s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
    
    # Insert colors
    if not colors_df.empty:
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color_hex, color_name, percentage)
            VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(colors_query, colors_df.values.tolist())
    
    connection.commit()
    cursor.close()
```

## SQL Analytics Queries

### Common Analytical Patterns

```python
# Query 1: Artifacts by culture
query_culture = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query 2: Department distribution
query_dept = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    GROUP BY department
    ORDER BY count DESC
"""

# Query 3: Color analysis
query_colors = """
    SELECT color_name, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
    FROM artifactcolors
    GROUP BY color_name
    ORDER BY usage_count DESC
    LIMIT 15
"""

# Query 4: Media format distribution
query_media = """
    SELECT format, COUNT(*) as count, AVG(width) as avg_width, AVG(height) as avg_height
    FROM artifactmedia
    GROUP BY format
"""

# Query 5: Century-wise artifacts
query_century = """
    SELECT century, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY artifact_count DESC
"""

def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)
```

## Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Artifacts Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                     value=st.secrets.get("api", {}).get("harvard_api_key", ""))
    
    # Database connection
    @st.cache_resource
    def get_connection():
        return mysql.connector.connect(
            host=st.secrets["database"]["host"],
            user=st.secrets["database"]["user"],
            password=st.secrets["database"]["password"],
            database=st.secrets["database"]["database"]
        )
    
    conn = get_connection()
    
    # Analytics section
    st.header("📊 Analytics")
    
    query_options = {
        "Top Cultures": "SELECT culture, COUNT(*) as count FROM artifactmetadata WHERE culture IS NOT NULL GROUP BY culture ORDER BY count DESC LIMIT 10",
        "Department Distribution": "SELECT department, COUNT(*) as count FROM artifactmetadata GROUP BY department",
        "Color Usage": "SELECT color_name, COUNT(*) as count FROM artifactcolors GROUP BY color_name ORDER BY count DESC LIMIT 15",
        "Century Analysis": "SELECT century, COUNT(*) as count FROM artifactmetadata WHERE century IS NOT NULL GROUP BY century ORDER BY count DESC"
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        df_result = pd.read_sql(query_options[selected_query], conn)
        
        st.dataframe(df_result)
        
        # Visualization
        if len(df_result.columns) == 2:
            fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1],
                        title=selected_query)
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Full ETL Pipeline

```python
def run_full_pipeline(api_key, db_config, num_pages=5):
    """Execute complete ETL pipeline"""
    
    # Extract
    all_artifacts = []
    for page in range(1, num_pages + 1):
        artifacts, _ = fetch_artifacts(api_key, page=page, size=100)
        all_artifacts.extend(artifacts)
        print(f"Fetched page {page}")
    
    # Transform
    metadata_df, media_df, colors_df = transform_artifacts(all_artifacts)
    
    # Load
    conn = mysql.connector.connect(**db_config)
    create_database_schema(conn)
    load_data_to_sql(conn, metadata_df, media_df, colors_df)
    conn.close()
    
    print(f"Pipeline complete: {len(all_artifacts)} artifacts loaded")
```

### Incremental Updates

```python
def get_latest_artifact_id(connection):
    """Get the most recent artifact ID in database"""
    query = "SELECT MAX(artifact_id) FROM artifactmetadata"
    cursor = connection.cursor()
    cursor.execute(query)
    result = cursor.fetchone()[0]
    cursor.close()
    return result or 0

def incremental_update(api_key, db_config):
    """Load only new artifacts"""
    conn = mysql.connector.connect(**db_config)
    latest_id = get_latest_artifact_id(conn)
    
    # Fetch artifacts with ID > latest_id
    # (Requires custom API filtering or client-side filtering)
    
    conn.close()
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1} after {wait_time}s")
                time.sleep(wait_time)
            else:
                raise
```

### Database Connection Pooling

```python
from mysql.connector import pooling

connection_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    database=os.getenv('DB_NAME'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD')
)

def get_connection():
    return connection_pool.get_connection()
```

### Handling NULL Values

```python
def clean_dataframe(df):
    """Handle NULL and missing values"""
    return df.fillna({
        'culture': 'Unknown',
        'classification': 'Unclassified',
        'century': 'Unknown',
        'department': 'General'
    })
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Run ETL pipeline (if separate script)
python etl_pipeline.py

# Query analytics
python analytics_queries.py
```

This skill provides everything needed to build, deploy, and extend museum data pipelines with modern data engineering practices.
