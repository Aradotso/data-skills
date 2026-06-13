---
name: harvard-artifacts-etl-streamlit-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - create an ETL workflow for museum artifact data
  - set up analytics dashboard with Streamlit and SQL
  - extract and transform Harvard museum collection data
  - build end-to-end data engineering app with museum API
  - analyze Harvard artifacts data with SQL queries
  - visualize museum collection data using Plotly
  - implement batch data ingestion from Harvard API
---

# Harvard Artifacts ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards with Plotly visualizations.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Data Collection**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media details, and color information
- **SQL Database**: Stores structured data in relational tables (MySQL/TiDB Cloud)
- **Analytics Queries**: Pre-built SQL queries for artifact insights
- **Interactive Dashboard**: Streamlit-based UI with Plotly visualizations

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly
```

## Configuration

### Environment Variables

Set up your credentials using environment variables:

```bash
# Harvard Art Museums API
export HARVARD_API_KEY="your_api_key_here"

# Database credentials
export DB_HOST="your_db_host"
export DB_PORT="4000"
export DB_USER="your_username"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"
```

### API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

## Key Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, size=100, page=1):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        "apikey": api_key,
        "size": size,
        "page": page,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv("HARVARD_API_KEY")
data = fetch_artifacts(api_key, size=100, page=1)
artifacts = data.get("records", [])
total_records = data.get("info", {}).get("totalrecords", 0)
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class HarvardETL:
    def __init__(self, db_config):
        self.db_config = db_config
        self.connection = None
    
    def connect_db(self):
        """Establish database connection"""
        self.connection = mysql.connector.connect(
            host=self.db_config['host'],
            port=self.db_config['port'],
            user=self.db_config['user'],
            password=self.db_config['password'],
            database=self.db_config['database']
        )
        return self.connection
    
    def extract_metadata(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract artifact metadata into DataFrame"""
        metadata = []
        
        for artifact in artifacts:
            metadata.append({
                'artifact_id': artifact.get('id'),
                'title': artifact.get('title'),
                'culture': artifact.get('culture'),
                'century': artifact.get('century'),
                'department': artifact.get('department'),
                'classification': artifact.get('classification'),
                'dated': artifact.get('dated'),
                'description': artifact.get('description'),
                'medium': artifact.get('medium'),
                'technique': artifact.get('technique'),
                'dimensions': artifact.get('dimensions')
            })
        
        return pd.DataFrame(metadata)
    
    def extract_media(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract media/image information"""
        media_records = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            images = artifact.get('images', [])
            
            for img in images:
                media_records.append({
                    'artifact_id': artifact_id,
                    'image_id': img.get('imageid'),
                    'baseimageurl': img.get('baseimageurl'),
                    'width': img.get('width'),
                    'height': img.get('height'),
                    'format': img.get('format')
                })
        
        return pd.DataFrame(media_records)
    
    def extract_colors(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract color information"""
        color_records = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            colors = artifact.get('colors', [])
            
            for color in colors:
                color_records.append({
                    'artifact_id': artifact_id,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
        
        return pd.DataFrame(color_records)
    
    def load_to_sql(self, df: pd.DataFrame, table_name: str, if_exists='append'):
        """Load DataFrame to SQL database using batch insert"""
        cursor = self.connection.cursor()
        
        # Create insert query
        cols = ','.join(df.columns)
        placeholders = ','.join(['%s'] * len(df.columns))
        query = f"INSERT INTO {table_name} ({cols}) VALUES ({placeholders})"
        
        # Batch insert
        data_tuples = [tuple(x) for x in df.to_numpy()]
        cursor.executemany(query, data_tuples)
        self.connection.commit()
        cursor.close()
        
        print(f"Loaded {len(df)} records into {table_name}")
```

### 3. Database Schema Creation

```python
def create_database_schema(connection):
    """Create database tables for artifacts data"""
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            century VARCHAR(100),
            department VARCHAR(255),
            classification VARCHAR(255),
            dated VARCHAR(255),
            description TEXT,
            medium TEXT,
            technique VARCHAR(255),
            dimensions VARCHAR(255)
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_id INT,
            baseimageurl TEXT,
            width INT,
            height INT,
            format VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            hue VARCHAR(100),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    connection.commit()
    cursor.close()
    print("Database schema created successfully")
```

### 4. Complete ETL Workflow

```python
def run_etl_pipeline(api_key, db_config, num_pages=5):
    """Execute complete ETL pipeline"""
    
    # Initialize ETL
    etl = HarvardETL(db_config)
    etl.connect_db()
    
    # Create schema
    create_database_schema(etl.connection)
    
    all_artifacts = []
    
    # Extract data from API with pagination
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, size=100, page=page)
        artifacts = data.get("records", [])
        all_artifacts.extend(artifacts)
    
    # Transform and Load
    print("Transforming metadata...")
    metadata_df = etl.extract_metadata(all_artifacts)
    etl.load_to_sql(metadata_df, 'artifactmetadata', if_exists='append')
    
    print("Transforming media data...")
    media_df = etl.extract_media(all_artifacts)
    etl.load_to_sql(media_df, 'artifactmedia', if_exists='append')
    
    print("Transforming color data...")
    colors_df = etl.extract_colors(all_artifacts)
    etl.load_to_sql(colors_df, 'artifactcolors', if_exists='append')
    
    print(f"ETL pipeline completed! Processed {len(all_artifacts)} artifacts")
```

### 5. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE culture IS NOT NULL 
        GROUP BY culture 
        ORDER BY count DESC 
        LIMIT 10
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century IS NOT NULL 
        GROUP BY century 
        ORDER BY count DESC
    """,
    
    "media_availability": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
            COUNT(*) as total_media_files
        FROM artifactmedia m
    """,
    
    "top_colors": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "artifacts_by_department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
}

def execute_analytics_query(connection, query_name):
    """Execute analytical query and return results"""
    cursor = connection.cursor(dictionary=True)
    query = ANALYTICS_QUERIES.get(query_name)
    
    cursor.execute(query)
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results)
```

### 6. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """Create Streamlit analytics dashboard"""
    
    st.title("Harvard Art Museums Analytics Dashboard")
    st.write("End-to-end Data Engineering & Analytics Application")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("API Key", type="password", 
                                     value=os.getenv("HARVARD_API_KEY", ""))
    
    # Database connection
    db_config = {
        'host': os.getenv("DB_HOST"),
        'port': int(os.getenv("DB_PORT", 3306)),
        'user': os.getenv("DB_USER"),
        'password': os.getenv("DB_PASSWORD"),
        'database': os.getenv("DB_NAME")
    }
    
    # ETL Section
    st.header("📥 Data Collection & ETL")
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            run_etl_pipeline(api_key, db_config, num_pages)
            st.success(f"ETL completed! Processed {num_pages * 100} artifacts")
    
    # Analytics Section
    st.header("📊 Analytics Dashboard")
    
    query_option = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Execute Query"):
        etl = HarvardETL(db_config)
        etl.connect_db()
        
        df = execute_analytics_query(etl.connection, query_option)
        
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=f"{query_option.replace('_', ' ').title()}")
            st.plotly_chart(fig)

if __name__ == "__main__":
    create_dashboard()
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Handling API Rate Limits

```python
import time

def fetch_with_rate_limit(api_key, pages, delay=1):
    """Fetch data with rate limiting"""
    all_data = []
    
    for page in range(1, pages + 1):
        data = fetch_artifacts(api_key, size=100, page=page)
        all_data.extend(data.get("records", []))
        
        if page < pages:
            time.sleep(delay)  # Respect rate limits
    
    return all_data
```

### Error Handling in ETL

```python
def safe_etl_run(api_key, db_config, num_pages):
    """ETL with error handling and rollback"""
    etl = HarvardETL(db_config)
    
    try:
        etl.connect_db()
        all_artifacts = fetch_with_rate_limit(api_key, num_pages)
        
        # Transform
        metadata_df = etl.extract_metadata(all_artifacts)
        media_df = etl.extract_media(all_artifacts)
        colors_df = etl.extract_colors(all_artifacts)
        
        # Load with transaction
        etl.load_to_sql(metadata_df, 'artifactmetadata')
        etl.load_to_sql(media_df, 'artifactmedia')
        etl.load_to_sql(colors_df, 'artifactcolors')
        
        return True
        
    except Exception as e:
        if etl.connection:
            etl.connection.rollback()
        print(f"ETL failed: {str(e)}")
        return False
    
    finally:
        if etl.connection:
            etl.connection.close()
```

## Troubleshooting

### API Connection Issues

```python
# Verify API key is valid
def test_api_connection(api_key):
    try:
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={"apikey": api_key, "size": 1}
        )
        return response.status_code == 200
    except Exception as e:
        print(f"Connection failed: {e}")
        return False
```

### Database Connection Problems

```python
# Test database connectivity
def test_db_connection(db_config):
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        return True
    except Exception as e:
        print(f"Database connection failed: {e}")
        return False
```

### Handle Missing Data

```python
def safe_extract(artifact, field, default=None):
    """Safely extract field with default value"""
    return artifact.get(field, default) or default

# Usage in extraction
metadata.append({
    'artifact_id': safe_extract(artifact, 'id', 0),
    'title': safe_extract(artifact, 'title', 'Unknown'),
    'culture': safe_extract(artifact, 'culture', 'Unknown')
})
```
