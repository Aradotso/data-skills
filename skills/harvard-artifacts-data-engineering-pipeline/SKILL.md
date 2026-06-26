---
name: harvard-artifacts-data-engineering-pipeline
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I fetch data from the Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create a Streamlit dashboard for artifact analytics
  - query Harvard museum collection with SQL
  - set up data engineering pipeline for art museum data
  - visualize Harvard Art Museums data with Plotly
  - process and store museum artifact metadata in database
  - analyze art collection data with SQL queries
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application for the Harvard Art Museums API. It demonstrates ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit. The architecture follows: **API → ETL → SQL → Analytics → Visualization**.

## What It Does

- Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into relational database tables (metadata, media, colors)
- Loads structured data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries on the collection
- Visualizes insights with interactive Plotly dashboards in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
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

## Configuration

### Environment Variables

Create a `.env` file or set environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database credentials
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### API Key Setup

Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

### Database Schema

The application creates three tables:

```sql
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accessionyear INT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    division VARCHAR(200)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    alttext TEXT,
    format VARCHAR(50),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Core Usage Patterns

### 1. Data Collection from API

```python
import requests
import os

def fetch_artifacts(api_key, size=100, page=1):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, size=100, page=1)
artifacts = data.get('records', [])
total_records = data.get('info', {}).get('totalrecords', 0)
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

def extract_metadata(artifacts: List[Dict]) -> pd.DataFrame:
    """Extract artifact metadata into DataFrame"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'accessionyear': artifact.get('accessionyear'),
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'division': artifact.get('division', '')[:200]
        })
    
    return pd.DataFrame(metadata)

def extract_media(artifacts: List[Dict]) -> pd.DataFrame:
    """Extract media information"""
    media_data = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for img in images:
            media_data.append({
                'objectid': objectid,
                'iiifbaseuri': img.get('iiifbaseuri', '')[:500],
                'baseimageurl': img.get('baseimageurl', '')[:500],
                'alttext': img.get('alttext', ''),
                'format': img.get('format', '')[:50]
            })
    
    return pd.DataFrame(media_data)

def extract_colors(artifacts: List[Dict]) -> pd.DataFrame:
    """Extract color information"""
    color_data = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data.append({
                'objectid': objectid,
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'hue': color.get('hue', '')[:50],
                'percent': color.get('percent', 0.0)
            })
    
    return pd.DataFrame(color_data)
```

### 3. Loading Data into SQL

```python
def load_to_database(df: pd.DataFrame, table_name: str, db_config: Dict):
    """Load DataFrame to MySQL database"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    # Prepare insert query based on table
    if table_name == 'artifactmetadata':
        query = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, century, classification, department, 
         dated, accessionyear, technique, medium, division)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
        """
    elif table_name == 'artifactmedia':
        query = """
        INSERT INTO artifactmedia 
        (objectid, iiifbaseuri, baseimageurl, alttext, format)
        VALUES (%s, %s, %s, %s, %s)
        """
    elif table_name == 'artifactcolors':
        query = """
        INSERT INTO artifactcolors 
        (objectid, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
    
    # Batch insert
    data_tuples = [tuple(row) for row in df.values]
    cursor.executemany(query, data_tuples)
    
    conn.commit()
    cursor.close()
    conn.close()

# Usage
db_config = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

# ETL process
metadata_df = extract_metadata(artifacts)
load_to_database(metadata_df, 'artifactmetadata', db_config)
```

### 4. Analytical SQL Queries

```python
def execute_analytical_query(query: str, db_config: Dict) -> pd.DataFrame:
    """Execute SQL query and return results as DataFrame"""
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Sample analytical queries
queries = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
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
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Color Analysis": """
        SELECT c.color, COUNT(DISTINCT c.objectid) as artifact_count,
               AVG(c.percent) as avg_percentage
        FROM artifactcolors c
        GROUP BY c.color
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Media Format Distribution": """
        SELECT format, COUNT(*) as image_count
        FROM artifactmedia
        WHERE format IS NOT NULL
        GROUP BY format
        ORDER BY image_count DESC
    """
}

# Execute query
result_df = execute_analytical_query(queries["Top 10 Cultures by Artifact Count"], db_config)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Options")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(queries.keys())
    )
    
    # Execute selected query
    if st.sidebar.button("Run Analysis"):
        with st.spinner("Executing query..."):
            result_df = execute_analytical_query(queries[query_name], db_config)
            
            st.subheader(f"Results: {query_name}")
            st.dataframe(result_df)
            
            # Auto-generate visualization
            if len(result_df.columns) >= 2:
                fig = px.bar(
                    result_df,
                    x=result_df.columns[0],
                    y=result_df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Paginated Data Collection

```python
def collect_all_artifacts(api_key: str, max_pages: int = 10):
    """Collect artifacts across multiple pages"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(api_key, size=100, page=page)
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            # Respect rate limits
            import time
            time.sleep(0.5)
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

### Incremental ETL

```python
def get_last_synced_id(db_config: Dict) -> int:
    """Get the last synced object ID"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    return result or 0

def incremental_load(api_key: str, db_config: Dict):
    """Load only new artifacts"""
    last_id = get_last_synced_id(db_config)
    
    # Fetch artifacts with objectid > last_id
    # Implementation depends on API capabilities
    pass
```

## Troubleshooting

**API Rate Limiting:**
- Add delays between requests: `time.sleep(0.5)`
- Implement exponential backoff on failures

**Database Connection Issues:**
- Verify credentials in environment variables
- Check firewall rules for cloud databases
- Ensure database exists before running ETL

**Memory Issues with Large Datasets:**
- Process data in batches
- Use chunked reads with pandas: `pd.read_sql(query, conn, chunksize=1000)`

**Missing Data Fields:**
- Use `.get()` with defaults when extracting from API responses
- Implement null checks before database insertion

**Streamlit Performance:**
- Cache expensive operations with `@st.cache_data`
- Limit query result sizes with SQL LIMIT clauses

```python
@st.cache_data(ttl=3600)
def cached_query(query: str):
    return execute_analytical_query(query, db_config)
```
