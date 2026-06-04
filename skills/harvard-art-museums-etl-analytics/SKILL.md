---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering and analytics application using Harvard Art Museums API with ETL pipelines, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifacts
  - create a data analytics app with Harvard Art Museums API
  - set up artifact collection data engineering pipeline
  - analyze Harvard museum data with SQL queries
  - build Streamlit dashboard for museum artifacts
  - implement ETL for art collection metadata
  - create museum data visualization with Plotly
  - design SQL database for artifact analytics
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the Harvard Art Museums Collection Data Engineering and Analytics App. The project demonstrates production-grade ETL pipelines, extracting artifact metadata from the Harvard Art Museums API, transforming it into relational tables, loading into SQL databases, and visualizing insights through interactive Streamlit dashboards.

## What This Project Does

The application provides:
- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts nested JSON, transforms to relational schema, loads to SQL
- **Database Design**: Three normalized tables (artifactmetadata, artifactmedia, artifactcolors)
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Visualization**: Streamlit frontend with Plotly charts

Architecture flow: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    description TEXT,
    provenance TEXT,
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact media table
CREATE TABLE IF NOT EXISTS artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    imagepermissionlevel INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE IF NOT EXISTS artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core ETL Pipeline

### Extract Phase

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, num_artifacts=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    size = 100  # API limit per page
    
    for page in range(0, num_artifacts, size):
        params = {
            'apikey': api_key,
            'size': min(size, num_artifacts - page),
            'page': page // size + 1
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts = fetch_artifacts(api_key, num_artifacts=500)
```

### Transform Phase

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform nested JSON into normalized DataFrames
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'dated': artifact.get('dated', ''),
            'description': artifact.get('description', ''),
            'provenance': artifact.get('provenance', ''),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
        
        # Extract media info
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl', ''),
            'primaryimageurl': artifact.get('primaryimageurl', ''),
            'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri', '') if artifact.get('images') else '',
            'imagepermissionlevel': artifact.get('imagepermissionlevel', 0)
        }
        media_list.append(media)
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_entry = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_entry)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

# Usage
df_metadata, df_media, df_colors = transform_artifacts(artifacts)
```

### Load Phase

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(df_metadata, df_media, df_colors):
    """
    Load transformed data into MySQL database
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        cursor = connection.cursor()
        
        # Load metadata
        insert_metadata = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, dated, 
             description, provenance, totalpageviews, totaluniquepageviews)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(insert_metadata, df_metadata.values.tolist())
        
        # Load media
        insert_media = """
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, primaryimageurl, iiifbaseuri, imagepermissionlevel)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(insert_media, df_media.values.tolist())
        
        # Load colors
        insert_colors = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(insert_colors, df_colors.values.tolist())
        
        connection.commit()
        print(f"Loaded {len(df_metadata)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()

# Usage
load_to_database(df_metadata, df_media, df_colors)
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px
import mysql.connector

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

# Sidebar configuration
st.sidebar.title("Configuration")
api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))
num_artifacts = st.sidebar.slider("Number of Artifacts", 10, 1000, 100)

# Main tabs
tab1, tab2, tab3 = st.tabs(["Data Collection", "SQL Analytics", "Visualizations"])

with tab1:
    st.header("ETL Pipeline")
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(api_key, num_artifacts)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata, df_media, df_colors = transform_artifacts(artifacts)
            st.success("Data transformed")
        
        with st.spinner("Loading to database..."):
            load_to_database(df_metadata, df_media, df_colors)
            st.success("Data loaded successfully")
        
        st.dataframe(df_metadata.head())

with tab2:
    st.header("SQL Analytics")
    
    queries = {
        "Artifacts by Culture": """
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
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century 
            ORDER BY count DESC 
            LIMIT 10
        """,
        "Top Viewed Artifacts": """
            SELECT title, culture, totalpageviews 
            FROM artifactmetadata 
            ORDER BY totalpageviews DESC 
            LIMIT 10
        """,
        "Color Distribution": """
            SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY frequency DESC 
            LIMIT 10
        """,
        "Department Analysis": """
            SELECT department, COUNT(*) as artifact_count,
                   AVG(totalpageviews) as avg_views
            FROM artifactmetadata 
            WHERE department IS NOT NULL AND department != ''
            GROUP BY department 
            ORDER BY artifact_count DESC
        """
    }
    
    query_name = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        result_df = execute_query(queries[query_name])
        st.dataframe(result_df)
        
        # Auto-generate visualization
        if len(result_df.columns) >= 2:
            fig = px.bar(result_df, x=result_df.columns[0], y=result_df.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```

## Analytical SQL Queries

### Advanced Analytics Queries

```sql
-- 1. Artifacts with complete metadata and images
SELECT 
    m.id, m.title, m.culture, m.century,
    med.primaryimageurl,
    COUNT(c.id) as color_count
FROM artifactmetadata m
JOIN artifactmedia med ON m.id = med.artifact_id
LEFT JOIN artifactcolors c ON m.id = c.artifact_id
WHERE m.description IS NOT NULL 
    AND med.primaryimageurl IS NOT NULL
GROUP BY m.id, m.title, m.culture, m.century, med.primaryimageurl
HAVING color_count > 0;

-- 2. Popular artifacts by department with image availability
SELECT 
    department,
    COUNT(*) as total_artifacts,
    SUM(CASE WHEN med.primaryimageurl IS NOT NULL THEN 1 ELSE 0 END) as with_images,
    AVG(totalpageviews) as avg_views
FROM artifactmetadata m
LEFT JOIN artifactmedia med ON m.id = med.artifact_id
GROUP BY department
ORDER BY avg_views DESC;

-- 3. Color trends by century
SELECT 
    m.century,
    c.color,
    COUNT(*) as usage_count,
    AVG(c.percent) as avg_coverage
FROM artifactmetadata m
JOIN artifactcolors c ON m.id = c.artifact_id
WHERE m.century IS NOT NULL
GROUP BY m.century, c.color
ORDER BY m.century, usage_count DESC;

-- 4. Most diverse color palettes
SELECT 
    m.id,
    m.title,
    m.culture,
    COUNT(DISTINCT c.color) as unique_colors,
    GROUP_CONCAT(c.color ORDER BY c.percent DESC LIMIT 5) as top_colors
FROM artifactmetadata m
JOIN artifactcolors c ON m.id = c.artifact_id
GROUP BY m.id, m.title, m.culture
HAVING unique_colors >= 5
ORDER BY unique_colors DESC
LIMIT 20;
```

## Common Patterns

### Batch Processing with Rate Limiting

```python
import time

def fetch_artifacts_with_rate_limit(api_key, num_artifacts, delay=1):
    """
    Fetch artifacts with rate limiting to avoid API throttling
    """
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    page_size = 100
    
    for offset in range(0, num_artifacts, page_size):
        params = {
            'apikey': api_key,
            'size': min(page_size, num_artifacts - offset),
            'page': offset // page_size + 1
        }
        
        try:
            response = requests.get(base_url, params=params, timeout=10)
            response.raise_for_status()
            
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            # Rate limiting
            time.sleep(delay)
            
        except requests.exceptions.RequestException as e:
            print(f"Request failed: {e}")
            break
    
    return artifacts
```

### Incremental ETL Updates

```python
def incremental_load(api_key, last_update_date):
    """
    Load only new or updated artifacts since last ETL run
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'size': 100,
        'sort': 'lastupdate',
        'sortorder': 'desc',
        'q': f'lastupdate:>{last_update_date}'
    }
    
    response = requests.get(base_url, params=params)
    new_artifacts = response.json().get('records', [])
    
    return new_artifacts
```

## Troubleshooting

### API Connection Issues

```python
def validate_api_connection(api_key):
    """Test API connectivity and key validity"""
    test_url = "https://api.harvardartmuseums.org/object"
    params = {'apikey': api_key, 'size': 1}
    
    try:
        response = requests.get(test_url, params=params, timeout=5)
        if response.status_code == 200:
            return True, "API connection successful"
        elif response.status_code == 401:
            return False, "Invalid API key"
        else:
            return False, f"API error: {response.status_code}"
    except Exception as e:
        return False, f"Connection error: {str(e)}"
```

### Database Connection Pooling

```python
from mysql.connector import pooling

def create_connection_pool():
    """Create database connection pool for better performance"""
    try:
        pool = pooling.MySQLConnectionPool(
            pool_name="harvard_pool",
            pool_size=5,
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return pool
    except Error as e:
        print(f"Error creating connection pool: {e}")
        return None
```

### Handling Missing Data

```python
def clean_artifact_data(artifact):
    """Handle null/missing values in artifact data"""
    defaults = {
        'title': 'Untitled',
        'culture': 'Unknown',
        'century': 'Unknown',
        'classification': 'Unclassified',
        'department': 'General',
        'dated': '',
        'description': '',
        'provenance': '',
        'totalpageviews': 0,
        'totaluniquepageviews': 0
    }
    
    return {key: artifact.get(key, default) for key, default in defaults.items()}
```

## Running the Application

```bash
# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501

# Run with custom port
streamlit run app.py --server.port 8080

# Run ETL pipeline standalone
python etl_pipeline.py
```
