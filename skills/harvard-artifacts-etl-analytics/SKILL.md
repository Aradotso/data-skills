---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create a data engineering project with Harvard artifacts API
  - set up analytics dashboard for museum collection data
  - extract and transform Harvard Art Museums API data
  - build SQL analytics for art museum artifacts
  - create Streamlit app for Harvard collection visualization
  - implement ETL workflow for museum artifact data
  - query and visualize Harvard Art Museums metadata
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates an end-to-end data engineering pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational structures, loads it into SQL databases, and provides interactive analytics dashboards using Streamlit.

## What It Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts nested JSON, transforms into normalized tables, loads into SQL
- **Database Design**: Creates relational schema with artifact metadata, media, and color tables
- **Analytics**: Executes 20+ predefined SQL queries for insights
- **Visualization**: Interactive dashboards with Plotly charts in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages**:
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard API Key

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file:
```bash
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

Set up MySQL or TiDB Cloud credentials in `.env`:
```bash
DB_HOST=your_database_host
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

## Database Schema

The ETL pipeline creates three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dated VARCHAR(200),
    division VARCHAR(200)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    image_width INT,
    image_height INT,
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    color_spectrum VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);
```

## Core ETL Pipeline

### Extract: Fetch API Data

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Extract artifacts from Harvard API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    url = f"https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Fetch multiple pages
all_artifacts = []
for page in range(1, 6):  # First 5 pages
    records, info = fetch_artifacts(page=page)
    all_artifacts.extend(records)
    print(f"Fetched page {page}: {len(records)} artifacts")
```

### Transform: Normalize Data

```python
import pandas as pd

def transform_metadata(artifacts):
    """Transform artifact data into metadata table format"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown')[:500],
            'culture': artifact.get('culture', 'Unknown')[:200],
            'period': artifact.get('period', 'Unknown')[:200],
            'century': artifact.get('century', 'Unknown')[:100],
            'classification': artifact.get('classification', 'Unknown')[:200],
            'department': artifact.get('department', 'Unknown')[:200],
            'technique': artifact.get('technique', 'Unknown')[:500],
            'medium': artifact.get('medium', 'Unknown')[:500],
            'dated': artifact.get('dated', 'Unknown')[:200],
            'division': artifact.get('division', 'Unknown')[:200]
        })
    
    return pd.DataFrame(metadata)

def transform_media(artifacts):
    """Extract media/image data"""
    media = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media.append({
                'artifact_id': artifact_id,
                'image_url': img.get('baseimageurl'),
                'image_width': img.get('width'),
                'image_height': img.get('height'),
                'format': img.get('format')
            })
    
    return pd.DataFrame(media)

def transform_colors(artifacts):
    """Extract color data"""
    colors = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        color_data = artifact.get('colors', [])
        
        for color in color_data:
            colors.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent'),
                'color_spectrum': color.get('spectrum')
            })
    
    return pd.DataFrame(colors)
```

### Load: Insert into SQL

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME'),
        port=int(os.getenv('DB_PORT', 3306))
    )

def load_metadata(df):
    """Batch insert metadata into SQL"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (artifact_id, title, culture, period, century, classification, 
     department, technique, medium, dated, division)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
    print(f"Loaded {len(df)} metadata records")

def load_media(df):
    """Batch insert media data"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, image_url, image_width, image_height, format)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
    print(f"Loaded {len(df)} media records")
```

## Streamlit Analytics App

### Main Application Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar navigation
page = st.sidebar.selectbox(
    "Select Page",
    ["ETL Pipeline", "SQL Analytics", "Visualizations"]
)

if page == "ETL Pipeline":
    st.header("Data Collection & ETL")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Number of pages to fetch", 1, 10, 5)
        
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = []
            for page in range(1, num_pages + 1):
                records, _ = fetch_artifacts(page=page)
                artifacts.extend(records)
            
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata = transform_metadata(artifacts)
            df_media = transform_media(artifacts)
            df_colors = transform_colors(artifacts)
            
            st.success("Data transformed successfully")
        
        with st.spinner("Loading into database..."):
            load_metadata(df_metadata)
            load_media(df_media)
            load_colors(df_colors)
            
            st.success("ETL Pipeline completed!")
```

### Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Top 10 Cultures": """
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
        WHERE century != 'Unknown'
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN image_url IS NOT NULL THEN 'Has Image' ELSE 'No Image' END as media_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY media_status
    """,
    
    "Top Color Palettes": """
        SELECT color_hex, color_spectrum, COUNT(*) as usage_count
        FROM artifactcolors
        GROUP BY color_hex, color_spectrum
        ORDER BY usage_count DESC
        LIMIT 20
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department != 'Unknown'
        GROUP BY department
        ORDER BY artifact_count DESC
    """
}

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# In Streamlit app
if page == "SQL Analytics":
    st.header("SQL Analytics")
    
    query_name = st.selectbox("Select Query", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Execute Query"):
        query = ANALYTICS_QUERIES[query_name]
        
        st.code(query, language='sql')
        
        df_result = execute_query(query)
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(
                df_result, 
                x=df_result.columns[0], 
                y=df_result.columns[1],
                title=query_name
            )
            st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(pages, delay=1):
    """Fetch API data with rate limiting"""
    artifacts = []
    
    for page in range(1, pages + 1):
        records, info = fetch_artifacts(page=page)
        artifacts.extend(records)
        
        if page < pages:
            time.sleep(delay)  # Respect API rate limits
    
    return artifacts
```

### Handling Missing Data

```python
def safe_get(data, key, default='Unknown', max_length=None):
    """Safely extract data with defaults and length limits"""
    value = data.get(key, default)
    
    if value is None:
        value = default
    
    if max_length and isinstance(value, str):
        value = value[:max_length]
    
    return value
```

## Troubleshooting

**API Key Issues**:
```python
# Verify API key is loaded
import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv('HARVARD_API_KEY')
if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment")
```

**Database Connection Errors**:
```python
try:
    conn = get_db_connection()
    print("Database connected successfully")
    conn.close()
except Error as e:
    print(f"Database connection failed: {e}")
```

**Foreign Key Constraint Violations**:
```python
# Always load metadata before media/colors
load_metadata(df_metadata)  # Load parent table first
load_media(df_media)        # Then child tables
load_colors(df_colors)
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

The application provides a complete workflow for building production-grade ETL pipelines with real-world museum data.
