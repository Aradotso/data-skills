---
name: harvard-artifacts-data-engineering-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Streamlit, MySQL, and Plotly
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - create an ETL pipeline for museum artifact data
  - set up analytics dashboard for Harvard artifacts
  - extract and transform Harvard museum data
  - build Streamlit app for art museum analytics
  - query Harvard Art Museums API and store in database
  - create data engineering project with museum API
  - visualize Harvard artifacts data with SQL analytics
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytics queries, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering App provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data
- **SQL Database**: Structured relational schema with foreign key relationships
- **Analytics**: 20+ predefined SQL queries for data insights
- **Visualization**: Interactive Plotly dashboards in Streamlit

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly sqlalchemy
```

## Configuration

### 1. Harvard Art Museums API Key

Get your free API key from: https://www.harvardartmuseums.org/collections/api

Set environment variable:
```bash
export HARVARD_API_KEY="your_api_key_here"
```

### 2. Database Configuration

Set up MySQL/TiDB Cloud connection:
```bash
export DB_HOST="your_database_host"
export DB_PORT="3306"
export DB_USER="your_username"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"
```

### 3. Streamlit Secrets (Alternative)

Create `.streamlit/secrets.toml`:
```toml
[api]
harvard_api_key = "your_api_key_here"

[database]
host = "your_database_host"
port = 3306
user = "your_username"
password = "your_password"
database = "harvard_artifacts"
```

## Database Schema

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accession_number VARCHAR(100)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(500),
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

def fetch_all_artifacts(api_key, max_records=1000):
    """Fetch multiple pages with pagination"""
    all_records = []
    page = 1
    size = 100
    
    while len(all_records) < max_records:
        data = fetch_artifacts(api_key, page=page, size=size)
        records = data.get("records", [])
        
        if not records:
            break
            
        all_records.extend(records)
        page += 1
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return all_records[:max_records]
```

### Transform: Process JSON to DataFrames

```python
import pandas as pd

def transform_metadata(records):
    """Extract artifact metadata"""
    metadata = []
    
    for record in records:
        metadata.append({
            "id": record.get("id"),
            "title": record.get("title"),
            "culture": record.get("culture"),
            "period": record.get("period"),
            "century": record.get("century"),
            "classification": record.get("classification"),
            "department": record.get("department"),
            "dated": record.get("dated"),
            "accession_number": record.get("accessionNumber")
        })
    
    return pd.DataFrame(metadata)

def transform_media(records):
    """Extract media/image data"""
    media = []
    
    for record in records:
        artifact_id = record.get("id")
        images = record.get("images", [])
        
        for img in images:
            media.append({
                "artifact_id": artifact_id,
                "image_url": img.get("baseimageurl"),
                "media_type": "image"
            })
    
    return pd.DataFrame(media)

def transform_colors(records):
    """Extract color information"""
    colors = []
    
    for record in records:
        artifact_id = record.get("id")
        color_data = record.get("colors", [])
        
        for color in color_data:
            colors.append({
                "artifact_id": artifact_id,
                "color_hex": color.get("hex"),
                "color_percent": color.get("percent")
            })
    
    return pd.DataFrame(colors)
```

### Load: Insert into Database

```python
import mysql.connector
from sqlalchemy import create_engine

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv("DB_HOST"),
        port=int(os.getenv("DB_PORT", 3306)),
        user=os.getenv("DB_USER"),
        password=os.getenv("DB_PASSWORD"),
        database=os.getenv("DB_NAME")
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Load dataframes to MySQL database"""
    engine = create_engine(
        f"mysql+mysqlconnector://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@"
        f"{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
    )
    
    # Load with replace strategy to handle duplicates
    metadata_df.to_sql("artifactmetadata", engine, if_exists="append", index=False)
    media_df.to_sql("artifactmedia", engine, if_exists="append", index=False)
    colors_df.to_sql("artifactcolors", engine, if_exists="append", index=False)
    
    return True
```

## SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "artifacts_with_images": """
        SELECT 
            CASE WHEN COUNT(m.media_id) > 0 THEN 'With Images' ELSE 'No Images' END as status,
            COUNT(DISTINCT a.id) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY status
    """,
    
    "top_colors": """
        SELECT color_hex, COUNT(*) as frequency
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "department_distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
}

def execute_query(query):
    """Execute SQL query and return results"""
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    cursor.execute(query)
    results = cursor.fetchall()
    cursor.close()
    conn.close()
    return pd.DataFrame(results)
```

## Streamlit Application

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Artifacts Analytics Dashboard")
    
    # Sidebar for API key input
    api_key = st.sidebar.text_input(
        "Harvard API Key",
        value=os.getenv("HARVARD_API_KEY", ""),
        type="password"
    )
    
    # ETL Pipeline Section
    st.header("1. Data Collection")
    num_records = st.number_input("Number of records to fetch", 100, 5000, 500)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            records = fetch_all_artifacts(api_key, max_records=num_records)
            st.success(f"Fetched {len(records)} records")
        
        with st.spinner("Transforming data..."):
            metadata_df = transform_metadata(records)
            media_df = transform_media(records)
            colors_df = transform_colors(records)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
            st.success("Data loaded to database")
    
    # Analytics Section
    st.header("2. SQL Analytics")
    query_name = st.selectbox("Select Query", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        st.code(query, language="sql")
        
        results_df = execute_query(query)
        st.dataframe(results_df)
        
        # Auto-visualization
        if len(results_df.columns) == 2:
            fig = px.bar(
                results_df,
                x=results_df.columns[0],
                y=results_df.columns[1],
                title=query_name.replace("_", " ").title()
            )
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Complete ETL Workflow

```python
def run_complete_pipeline(api_key, num_records=1000):
    """Run full ETL pipeline"""
    # Extract
    print("Extracting data...")
    records = fetch_all_artifacts(api_key, max_records=num_records)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_metadata(records)
    media_df = transform_media(records)
    colors_df = transform_colors(records)
    
    # Load
    print("Loading to database...")
    load_to_database(metadata_df, media_df, colors_df)
    
    print(f"Pipeline complete: {len(records)} artifacts processed")
    return metadata_df, media_df, colors_df
```

### Incremental Data Loading

```python
def incremental_load(api_key, last_update_date):
    """Load only new artifacts since last update"""
    params = {
        "apikey": api_key,
        "updatedafter": last_update_date,
        "hasimage": 1
    }
    
    response = requests.get("https://api.harvardartmuseums.org/object", params=params)
    records = response.json().get("records", [])
    
    if records:
        metadata_df = transform_metadata(records)
        media_df = transform_media(records)
        colors_df = transform_colors(records)
        load_to_database(metadata_df, media_df, colors_df)
    
    return len(records)
```

## Troubleshooting

### API Rate Limiting
```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues
```python
def test_db_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        return True
    except Exception as e:
        print(f"Database connection failed: {e}")
        return False
```

### Handling Missing Data
```python
def safe_get(record, key, default=None):
    """Safely extract nested values"""
    return record.get(key, default) or default

def transform_metadata_safe(records):
    """Transform with null handling"""
    metadata = []
    for record in records:
        metadata.append({
            "id": safe_get(record, "id"),
            "title": safe_get(record, "title", "Untitled"),
            "culture": safe_get(record, "culture", "Unknown"),
            "period": safe_get(record, "period", "Unknown"),
            "century": safe_get(record, "century", "Unknown"),
            "classification": safe_get(record, "classification", "Unknown"),
            "department": safe_get(record, "department", "Unknown"),
            "dated": safe_get(record, "dated", "Unknown"),
            "accession_number": safe_get(record, "accessionNumber", "N/A")
        })
    return pd.DataFrame(metadata)
```
