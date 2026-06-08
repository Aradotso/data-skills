---
name: harvard-artifacts-etl-streamlit-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create a Streamlit dashboard for museum artifacts analytics
  - fetch and analyze Harvard Art Museums API data
  - set up SQL database for artifact metadata and media
  - build a data engineering pipeline with Harvard API
  - create interactive visualizations for museum collection data
  - query and visualize Harvard artifacts with SQL
  - implement batch ETL processing for museum data
---

# Harvard Artifacts ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It demonstrates ETL pipeline construction, relational database design, SQL analytics, and interactive visualization using Streamlit. The architecture follows: **API → ETL → SQL → Analytics → Visualization**.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required dependencies:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

Create a `.env` file or configure via Streamlit secrets:

```python
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Setup

The project uses MySQL or TiDB Cloud. Create the database schema:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    copyright TEXT,
    url VARCHAR(500),
    lastupdate DATETIME
);

CREATE TABLE artifactmedia (
    artifact_id INT,
    media_id INT PRIMARY KEY,
    media_type VARCHAR(100),
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

The application will start on `http://localhost:8501`

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

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
data = fetch_artifacts(api_key, page=1, size=100)
artifacts = data.get("records", [])
total_records = data.get("info", {}).get("totalrecords", 0)
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from datetime import datetime

def extract_artifact_metadata(artifact):
    """
    Extract metadata from artifact JSON
    """
    return {
        "id": artifact.get("id"),
        "title": artifact.get("title", "")[:500],
        "culture": artifact.get("culture", "")[:200],
        "period": artifact.get("period", "")[:200],
        "century": artifact.get("century", "")[:100],
        "classification": artifact.get("classification", "")[:200],
        "department": artifact.get("department", "")[:200],
        "dated": artifact.get("dated", "")[:200],
        "technique": artifact.get("technique", "")[:500],
        "medium": artifact.get("medium", "")[:500],
        "dimensions": artifact.get("dimensions", "")[:500],
        "creditline": artifact.get("creditline", ""),
        "copyright": artifact.get("copyright", ""),
        "url": artifact.get("url", "")[:500],
        "lastupdate": datetime.now()
    }

def extract_artifact_media(artifact):
    """
    Extract media information from artifact JSON
    """
    media_list = []
    for media in artifact.get("images", []):
        media_list.append({
            "artifact_id": artifact.get("id"),
            "media_id": media.get("imageid"),
            "media_type": media.get("format", ""),
            "baseimageurl": media.get("baseimageurl", ""),
            "iiifbaseuri": media.get("iiifbaseuri", "")
        })
    return media_list

def extract_artifact_colors(artifact):
    """
    Extract color information from artifact JSON
    """
    colors_list = []
    for color in artifact.get("colors", []):
        colors_list.append({
            "artifact_id": artifact.get("id"),
            "color": color.get("color", ""),
            "percentage": color.get("percent", 0.0)
        })
    return colors_list

def transform_artifacts(artifacts):
    """
    Transform raw artifacts into structured dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        metadata_list.append(extract_artifact_metadata(artifact))
        media_list.extend(extract_artifact_media(artifact))
        colors_list.extend(extract_artifact_colors(artifact))
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

### 3. Database Loading

```python
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

def load_to_database(df_metadata, df_media, df_colors):
    """
    Batch insert data into MySQL database
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Insert metadata
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, 
         dated, technique, medium, dimensions, creditline, copyright, url, lastupdate)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture), lastupdate=VALUES(lastupdate)
    """
    
    cursor.executemany(metadata_query, df_metadata.values.tolist())
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, media_id, media_type, baseimageurl, iiifbaseuri)
        VALUES (%s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE media_type=VALUES(media_type)
    """
    
    if not df_media.empty:
        cursor.executemany(media_query, df_media.values.tolist())
    
    # Insert colors
    colors_query = """
        INSERT INTO artifactcolors (artifact_id, color, percentage)
        VALUES (%s, %s, %s)
        ON DUPLICATE KEY UPDATE percentage=VALUES(percentage)
    """
    
    if not df_colors.empty:
        cursor.executemany(colors_query, df_colors.values.tolist())
    
    conn.commit()
    cursor.close()
    conn.close()
    
    return len(df_metadata), len(df_media), len(df_colors)
```

### 4. Streamlit Analytics Dashboard

```python
import streamlit as st
import plotly.express as px

def run_sql_query(query):
    """
    Execute SQL query and return results as DataFrame
    """
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Streamlit app structure
st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar for data collection
with st.sidebar:
    st.header("Data Collection")
    api_key = st.text_input("Harvard API Key", type="password", 
                             value=os.getenv("HARVARD_API_KEY", ""))
    num_records = st.number_input("Number of Records", min_value=10, 
                                   max_value=1000, value=100)
    
    if st.button("Fetch & Load Data"):
        with st.spinner("Fetching data from API..."):
            data = fetch_artifacts(api_key, page=1, size=num_records)
            artifacts = data.get("records", [])
            
            df_metadata, df_media, df_colors = transform_artifacts(artifacts)
            
            counts = load_to_database(df_metadata, df_media, df_colors)
            
            st.success(f"Loaded: {counts[0]} artifacts, {counts[1]} media, {counts[2]} colors")

# Main analytics section
st.header("📊 SQL Analytics")

# Predefined queries
queries = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE culture IS NOT NULL AND culture != ''
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
    "Department Distribution": """
        SELECT department, COUNT(*) as count 
        FROM artifactmetadata 
        GROUP BY department 
        ORDER BY count DESC
    """,
    "Top 10 Colors Across Artifacts": """
        SELECT color, COUNT(*) as frequency, AVG(percentage) as avg_percentage
        FROM artifactcolors 
        GROUP BY color 
        ORDER BY frequency DESC 
        LIMIT 10
    """,
    "Artifacts with Most Images": """
        SELECT m.title, m.culture, COUNT(media.media_id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia media ON m.id = media.artifact_id
        GROUP BY m.id, m.title, m.culture
        ORDER BY image_count DESC
        LIMIT 10
    """
}

selected_query = st.selectbox("Select Analysis", list(queries.keys()))

if st.button("Run Query"):
    query = queries[selected_query]
    
    with st.spinner("Executing query..."):
        df_result = run_sql_query(query)
        
        st.subheader("Query Results")
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, 
                         x=df_result.columns[0], 
                         y=df_result.columns[1],
                         title=selected_query)
            st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Pagination Handling

```python
def fetch_all_artifacts(api_key, total_records=1000):
    """
    Fetch multiple pages of artifacts
    """
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < total_records:
        data = fetch_artifacts(api_key, page=page, size=size)
        artifacts = data.get("records", [])
        
        if not artifacts:
            break
            
        all_artifacts.extend(artifacts)
        page += 1
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return all_artifacts[:total_records]
```

### Error Handling in ETL

```python
def safe_etl_pipeline(artifacts):
    """
    ETL pipeline with error handling
    """
    try:
        df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        
        # Validate data
        assert not df_metadata.empty, "No metadata extracted"
        assert df_metadata['id'].notna().all(), "Missing artifact IDs"
        
        counts = load_to_database(df_metadata, df_media, df_colors)
        return {"status": "success", "counts": counts}
        
    except mysql.connector.Error as db_err:
        return {"status": "error", "message": f"Database error: {db_err}"}
    except Exception as e:
        return {"status": "error", "message": f"ETL error: {e}"}
```

## Troubleshooting

### API Rate Limiting
```python
# Add delays between requests
import time
time.sleep(1)  # Wait 1 second between API calls
```

### Database Connection Issues
```python
# Test connection
try:
    conn = get_db_connection()
    conn.close()
    print("Database connection successful")
except Exception as e:
    print(f"Connection failed: {e}")
```

### Missing or Null Data
```python
# Handle missing fields gracefully
def safe_get(dictionary, key, default="", max_length=None):
    value = dictionary.get(key, default)
    if max_length and value:
        return str(value)[:max_length]
    return value if value else default
```

### Memory Issues with Large Datasets
```python
# Process in batches
def batch_etl(api_key, total_records, batch_size=100):
    for i in range(0, total_records, batch_size):
        artifacts = fetch_artifacts(api_key, page=i//batch_size + 1, size=batch_size)
        df_m, df_media, df_c = transform_artifacts(artifacts)
        load_to_database(df_m, df_media, df_c)
        print(f"Processed batch {i//batch_size + 1}")
```

## Key SQL Analytics Queries

```sql
-- Artifacts without images
SELECT COUNT(*) FROM artifactmetadata m
LEFT JOIN artifactmedia media ON m.id = media.artifact_id
WHERE media.media_id IS NULL;

-- Color diversity by culture
SELECT m.culture, COUNT(DISTINCT c.color) as color_count
FROM artifactmetadata m
JOIN artifactcolors c ON m.id = c.artifact_id
GROUP BY m.culture
ORDER BY color_count DESC
LIMIT 10;

-- Recent additions
SELECT title, department, lastupdate
FROM artifactmetadata
ORDER BY lastupdate DESC
LIMIT 20;
```
