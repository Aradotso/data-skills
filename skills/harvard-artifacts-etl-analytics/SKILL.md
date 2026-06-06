---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create a data engineering pipeline with Harvard artifacts API
  - analyze Harvard museum collection data with SQL
  - build a Streamlit dashboard for museum artifacts
  - extract and transform Harvard Art Museums API data
  - set up analytics for Harvard museum collection
  - query Harvard artifacts database with SQL
  - visualize museum artifact data with Plotly
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It demonstrates ETL pipeline construction, SQL database design, analytical querying, and interactive visualization using Streamlit. The architecture follows a production-grade pattern: API → ETL → SQL → Analytics → Visualization.

## What It Does

- **Data Collection**: Fetches artifact metadata, media, and color information from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables
- **SQL Database**: Stores data in MySQL/TiDB Cloud with proper foreign key relationships
- **Analytics**: Executes 20+ predefined SQL queries for insights
- **Visualization**: Renders interactive Plotly charts via Streamlit dashboard

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Requirements

Create a `requirements.txt` with:

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free for research/educational use)
3. Add to `.env` file

### Database Setup

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    accession_number VARCHAR(100),
    division VARCHAR(255)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(50),
    base_url TEXT,
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core API Usage

### Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Fetch multiple pages
def collect_artifacts(total_pages=5):
    all_artifacts = []
    for page in range(1, total_pages + 1):
        data = fetch_artifacts(page=page)
        all_artifacts.extend(data.get('records', []))
    return all_artifacts
```

### Data Transformation

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON to relational format"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'accession_number': artifact.get('accessionyear'),
            'division': artifact.get('division')
        })
        
        # Extract media
        for img in artifact.get('images', []):
            media_list.append({
                'artifact_id': artifact.get('id'),
                'media_type': 'image',
                'base_url': img.get('baseimageurl'),
                'format': img.get('format')
            })
        
        # Extract colors
        for color in artifact.get('colors', []):
            colors_list.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Data Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    load_dotenv()
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df_metadata):
    """Batch insert artifact metadata"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, classification, 
     department, dated, accession_number, division)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    data = [tuple(row) for row in df_metadata.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()

def load_media(df_media):
    """Batch insert media data"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, media_type, base_url, format)
    VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df_media.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
```

## Streamlit Application

### Basic App Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for navigation
page = st.sidebar.selectbox(
    "Select Function",
    ["Data Collection", "SQL Analytics", "Visualizations"]
)

if page == "Data Collection":
    st.header("ETL Pipeline")
    
    num_pages = st.number_input("Number of pages to fetch", 1, 20, 5)
    
    if st.button("Run ETL"):
        with st.spinner("Fetching data..."):
            artifacts = collect_artifacts(num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
            st.success("Data transformed")
        
        with st.spinner("Loading to database..."):
            load_metadata(df_meta)
            load_media(df_media)
            load_colors(df_colors)
            st.success("Data loaded successfully!")

elif page == "SQL Analytics":
    st.header("SQL Query Dashboard")
    
    query_options = {
        "Top 10 Cultures": "SELECT culture, COUNT(*) as count FROM artifactmetadata WHERE culture IS NOT NULL GROUP BY culture ORDER BY count DESC LIMIT 10",
        "Artifacts by Century": "SELECT century, COUNT(*) as count FROM artifactmetadata WHERE century IS NOT NULL GROUP BY century ORDER BY count DESC",
        "Department Distribution": "SELECT department, COUNT(*) as count FROM artifactmetadata GROUP BY department",
        "Top Colors Used": "SELECT color, COUNT(*) as count FROM artifactcolors GROUP BY color ORDER BY count DESC LIMIT 10"
    }
    
    selected_query = st.selectbox("Select Query", list(query_options.keys()))
    
    if st.button("Execute Query"):
        conn = get_db_connection()
        df_result = pd.read_sql(query_options[selected_query], conn)
        conn.close()
        
        st.dataframe(df_result)
        
        # Auto-generate chart
        if len(df_result.columns) == 2:
            fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1])
            st.plotly_chart(fig, use_container_width=True)
```

## Common Analytical Queries

### Culture Analysis

```sql
-- Top cultures by artifact count
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
```

### Media Availability

```sql
-- Artifacts with media vs without
SELECT 
    CASE WHEN media_count > 0 THEN 'With Media' ELSE 'No Media' END as media_status,
    COUNT(*) as count
FROM (
    SELECT m.id, COUNT(a.id) as media_count
    FROM artifactmetadata m
    LEFT JOIN artifactmedia a ON m.id = a.artifact_id
    GROUP BY m.id
) as media_summary
GROUP BY media_status;
```

### Color Distribution

```sql
-- Average color usage by department
SELECT 
    m.department,
    c.color,
    AVG(c.percent) as avg_percent
FROM artifactmetadata m
JOIN artifactcolors c ON m.id = c.artifact_id
WHERE m.department IS NOT NULL
GROUP BY m.department, c.color
ORDER BY m.department, avg_percent DESC;
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise e
```

### Database Connection Issues

```python
def test_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        return result[0] == 1
    except Error as e:
        st.error(f"Database connection failed: {e}")
        return False
```

### Memory Management for Large Datasets

```python
def batch_process(artifacts, batch_size=100):
    """Process data in batches to manage memory"""
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        df_meta, df_media, df_colors = transform_artifacts(batch)
        load_metadata(df_meta)
        load_media(df_media)
        load_colors(df_colors)
        yield i + len(batch)
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```
