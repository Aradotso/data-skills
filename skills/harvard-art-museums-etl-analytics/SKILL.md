---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics app using Harvard Art Museums API with Streamlit dashboards
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - fetch and analyze Harvard museum collection data
  - set up data engineering workflow with museum API
  - visualize art museum data with Streamlit
  - query Harvard artifacts database
  - implement museum data collection pipeline
  - analyze art collection metadata with SQL
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL workflows, SQL analytics, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection application:
- Fetches artifact data from Harvard Art Museums API with pagination
- Transforms nested JSON into relational database tables
- Loads data into MySQL/TiDB Cloud databases
- Provides 20+ prebuilt analytical SQL queries
- Visualizes results with interactive Plotly charts in Streamlit

**Architecture**: API → ETL (Extract/Transform/Load) → SQL Database → Analytics → Streamlit Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Run the Streamlit app
streamlit run app.py
```

## Configuration

### API Key Setup

Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Set the API key as an environment variable:

```bash
export HARVARD_API_KEY="your_api_key_here"
```

Or configure it in the Streamlit app's sidebar when you first run it.

### Database Configuration

The app supports MySQL and TiDB Cloud. Set these environment variables:

```bash
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
export DB_PORT="3306"
```

## Database Schema

The ETL pipeline creates three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    objectnumber VARCHAR(100),
    url TEXT,
    lastupdate DATETIME
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri TEXT,
    baseimageurl TEXT,
    renditionnumber VARCHAR(50),
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    css3 VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Usage

### Extract Data from API

```python
import requests
import os

def fetch_artifacts(api_key, size=100, page=1):
    """Fetch artifacts from Harvard Art Museums API"""
    url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "size": size,
        "page": page,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv("HARVARD_API_KEY")
data = fetch_artifacts(api_key, size=50, page=1)
print(f"Total records: {data['info']['totalrecords']}")
```

### Transform Data with Pandas

```python
import pandas as pd

def transform_artifacts(api_response):
    """Transform API response into relational dataframes"""
    artifacts = api_response['records']
    
    # Metadata table
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'objectnumber': artifact.get('objectnumber'),
            'url': artifact.get('url'),
            'lastupdate': artifact.get('lastupdate')
        })
        
        # Extract media information
        if artifact.get('images'):
            for image in artifact['images']:
                media_list.append({
                    'artifact_id': artifact['id'],
                    'iiifbaseuri': image.get('iiifbaseuri'),
                    'baseimageurl': image.get('baseimageurl'),
                    'renditionnumber': image.get('renditionnumber'),
                    'format': image.get('format')
                })
        
        # Extract color information
        if artifact.get('colors'):
            for color in artifact['colors']:
                colors_list.append({
                    'artifact_id': artifact['id'],
                    'color': color.get('color'),
                    'css3': color.get('css3'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load Data to SQL

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv("DB_HOST"),
        user=os.getenv("DB_USER"),
        password=os.getenv("DB_PASSWORD"),
        database=os.getenv("DB_NAME"),
        port=int(os.getenv("DB_PORT", 3306))
    )

def load_data(metadata_df, media_df, colors_df):
    """Load dataframes into SQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (id, title, culture, century, classification, department, 
                 division, dated, objectnumber, url, lastupdate)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE
                title=VALUES(title), culture=VALUES(culture)
            """, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (artifact_id, iiifbaseuri, baseimageurl, renditionnumber, format)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (artifact_id, color, css3, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

## Analytics Queries

### Example SQL Queries

```python
# Top 10 cultures by artifact count
query_cultures = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Artifacts with most color variations
query_colors = """
SELECT m.title, m.culture, COUNT(c.color_id) as color_count
FROM artifactmetadata m
JOIN artifactcolors c ON m.id = c.artifact_id
GROUP BY m.id, m.title, m.culture
ORDER BY color_count DESC
LIMIT 10
"""

# Artifacts by century
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Department distribution
query_departments = """
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY artifact_count DESC
"""
```

### Executing Queries with Pandas

```python
def execute_query(query):
    """Execute SQL query and return dataframe"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Usage
df_cultures = execute_query(query_cultures)
print(df_cultures)
```

## Streamlit Dashboard

### Basic Streamlit App Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar configuration
with st.sidebar:
    st.header("Configuration")
    api_key = st.text_input("Harvard API Key", type="password", 
                             value=os.getenv("HARVARD_API_KEY", ""))
    
    num_artifacts = st.slider("Number of artifacts to fetch", 10, 200, 50)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data..."):
            data = fetch_artifacts(api_key, size=num_artifacts)
            metadata_df, media_df, colors_df = transform_artifacts(data)
            load_data(metadata_df, media_df, colors_df)
            st.success(f"Loaded {len(metadata_df)} artifacts!")

# Analytics section
st.header("📊 Analytics")

query_option = st.selectbox("Select Analysis", [
    "Top Cultures",
    "Artifacts by Century",
    "Department Distribution",
    "Color Analysis"
])

if st.button("Run Query"):
    if query_option == "Top Cultures":
        df = execute_query(query_cultures)
        fig = px.bar(df, x='culture', y='artifact_count', 
                     title='Top 10 Cultures by Artifact Count')
        st.plotly_chart(fig, use_container_width=True)
    
    elif query_option == "Artifacts by Century":
        df = execute_query(query_century)
        fig = px.bar(df, x='century', y='count',
                     title='Artifact Distribution by Century')
        st.plotly_chart(fig, use_container_width=True)
    
    st.dataframe(df, use_container_width=True)
```

## Common Patterns

### Pagination for Large Datasets

```python
def fetch_all_artifacts(api_key, total_records=500):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    page_size = 100
    total_pages = (total_records // page_size) + 1
    
    for page in range(1, total_pages + 1):
        data = fetch_artifacts(api_key, size=page_size, page=page)
        all_artifacts.extend(data['records'])
        
        if len(all_artifacts) >= total_records:
            break
    
    return all_artifacts[:total_records]
```

### Error Handling for API Requests

```python
import time

def fetch_with_retry(api_key, size=100, page=1, max_retries=3):
    """Fetch data with retry logic"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, size, page)
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            print(f"Retry {attempt + 1}/{max_retries} after error: {e}")
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

**API Rate Limiting**: Harvard API has rate limits. Add delays between requests:
```python
import time
time.sleep(0.5)  # 500ms delay between requests
```

**Database Connection Issues**: Verify environment variables are set correctly and database is accessible.

**Empty Results**: Check that `hasimage=1` parameter isn't too restrictive for your queries.

**Streamlit Port Conflicts**: If port 8501 is busy:
```bash
streamlit run app.py --server.port 8502
```

**Memory Issues with Large Datasets**: Process in batches:
```python
batch_size = 100
for i in range(0, len(artifacts), batch_size):
    batch = artifacts[i:i+batch_size]
    # Process batch
```
