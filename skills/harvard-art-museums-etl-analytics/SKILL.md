---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - set up Harvard Art Museums API data pipeline
  - implement artifact collection data engineering workflow
  - analyze Harvard museum data with SQL and visualization
  - extract and transform art museum API data
  - build Streamlit app for museum artifact analytics
  - query Harvard Art Museums collection database
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the Harvard Art Museums Data Engineering & Analytics application. The project demonstrates a complete ETL pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational tables, loads it into SQL databases (MySQL/TiDB), and provides interactive analytics through Streamlit dashboards.

## What This Project Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables (metadata, media, colors)
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper foreign key relationships
- **Analytics Queries**: Executes 20+ predefined analytical SQL queries
- **Visualization**: Interactive Plotly charts in Streamlit dashboards

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

Set up your credentials using environment variables (never hardcode):

```bash
# Harvard Art Museums API Key
export HARVARD_API_KEY="your_api_key_here"

# Database Configuration
export DB_HOST="your_database_host"
export DB_PORT="4000"
export DB_USER="your_username"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"
```

### Database Schema Setup

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(300),
    period VARCHAR(200),
    dated VARCHAR(200),
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    format VARCHAR(50),
    description TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "page": page,
        "size": size
    }
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv("HARVARD_API_KEY")
data = fetch_artifacts(api_key, page=1, size=100)
artifacts = data.get("records", [])
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform API response to metadata DataFrame"""
    metadata = []
    for artifact in artifacts:
        metadata.append({
            "id": artifact.get("id"),
            "title": artifact.get("title"),
            "culture": artifact.get("culture"),
            "century": artifact.get("century"),
            "classification": artifact.get("classification"),
            "department": artifact.get("department"),
            "technique": artifact.get("technique"),
            "period": artifact.get("period"),
            "dated": artifact.get("dated"),
            "url": artifact.get("url")
        })
    return pd.DataFrame(metadata)

def transform_artifact_media(artifacts):
    """Extract media information"""
    media = []
    for artifact in artifacts:
        artifact_id = artifact.get("id")
        for image in artifact.get("images", []):
            media.append({
                "artifact_id": artifact_id,
                "baseimageurl": image.get("baseimageurl"),
                "format": image.get("format"),
                "description": image.get("description")
            })
    return pd.DataFrame(media)

def transform_artifact_colors(artifacts):
    """Extract color information"""
    colors = []
    for artifact in artifacts:
        artifact_id = artifact.get("id")
        for color in artifact.get("colors", []):
            colors.append({
                "artifact_id": artifact_id,
                "color": color.get("color"),
                "spectrum": color.get("spectrum"),
                "hue": color.get("hue"),
                "percent": color.get("percent")
            })
    return pd.DataFrame(colors)
```

### 3. Database Loading

```python
import mysql.connector
import os

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv("DB_HOST"),
        port=int(os.getenv("DB_PORT", 3306)),
        user=os.getenv("DB_USER"),
        password=os.getenv("DB_PASSWORD"),
        database=os.getenv("DB_NAME")
    )

def load_metadata(df_metadata):
    """Load metadata into database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, technique, period, dated, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    for _, row in df_metadata.iterrows():
        cursor.execute(query, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()

def load_media(df_media):
    """Load media data into database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, format, description)
        VALUES (%s, %s, %s, %s)
    """
    
    for _, row in df_media.iterrows():
        cursor.execute(query, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
```

### 4. Streamlit Analytics Dashboard

```python
import streamlit as st
import plotly.express as px

# SQL Analytics Queries
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
        ORDER BY count DESC
    """,
    "Media Availability": """
        SELECT 
            CASE WHEN baseimageurl IS NOT NULL THEN 'Has Image' ELSE 'No Image' END as status,
            COUNT(DISTINCT artifact_id) as count
        FROM artifactmedia
        GROUP BY status
    """,
    "Top Colors": """
        SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY count DESC
        LIMIT 10
    """,
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
}

def run_query(query):
    """Execute SQL query and return DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Streamlit App
st.title("Harvard Art Museums Analytics Dashboard")

# Query Selection
query_name = st.selectbox("Select Analytics Query", list(ANALYTICS_QUERIES.keys()))

if st.button("Run Query"):
    query = ANALYTICS_QUERIES[query_name]
    df_result = run_query(query)
    
    # Display results
    st.dataframe(df_result)
    
    # Auto-generate visualization
    if len(df_result.columns) == 2:
        fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1])
        st.plotly_chart(fig)
```

## Common Patterns

### Complete ETL Pipeline

```python
import os
import time

def run_etl_pipeline(num_pages=5):
    """Run complete ETL pipeline"""
    api_key = os.getenv("HARVARD_API_KEY")
    
    all_artifacts = []
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page, size=100)
        all_artifacts.extend(data.get("records", []))
        time.sleep(1)  # Rate limiting
    
    # Transform
    df_metadata = transform_artifact_metadata(all_artifacts)
    df_media = transform_artifact_media(all_artifacts)
    df_colors = transform_artifact_colors(all_artifacts)
    
    # Load
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    
    print(f"ETL Complete: {len(all_artifacts)} artifacts processed")
```

### Batch Processing with Error Handling

```python
def safe_etl_batch(artifacts_batch):
    """ETL with error handling"""
    try:
        df_metadata = transform_artifact_metadata(artifacts_batch)
        load_metadata(df_metadata)
        return True
    except Exception as e:
        print(f"Error processing batch: {e}")
        return False

# Process in batches
batch_size = 100
for i in range(0, len(all_artifacts), batch_size):
    batch = all_artifacts[i:i+batch_size]
    safe_etl_batch(batch)
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
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
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues
```python
def test_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        print("Database connection successful")
        return True
    except Exception as e:
        print(f"Connection failed: {e}")
        return False
```

### Handling NULL Values
```python
def clean_dataframe(df):
    """Clean DataFrame for SQL insertion"""
    # Replace empty strings with None
    df = df.replace("", None)
    # Handle NaN values
    df = df.where(pd.notnull(df), None)
    return df
```

This skill enables comprehensive work with the Harvard Art Museums ETL pipeline, from API extraction through visualization.
