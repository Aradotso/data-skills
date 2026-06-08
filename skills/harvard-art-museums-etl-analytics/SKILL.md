---
name: harvard-art-museums-etl-analytics
description: ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering app with Harvard Art API
  - analyze Harvard Art Museums collection data
  - build a Streamlit analytics dashboard for museum data
  - extract and transform Harvard Art Museums API data
  - set up SQL database for art collection analytics
  - visualize Harvard Art Museums data with Python
  - create a museum artifacts data pipeline
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application for the Harvard Art Museums API. It demonstrates ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What It Does

- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables
- **Loads** data into MySQL/TiDB Cloud with proper schema design
- **Analyzes** using 20+ predefined SQL analytical queries
- **Visualizes** results through interactive Plotly charts in Streamlit

## Architecture

```
Harvard API → Python ETL → MySQL/TiDB → SQL Analytics → Streamlit Dashboard
```

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

# MySQL/TiDB Database
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Request an API key (free)
3. Add to `.env` file

### Database Setup

```python
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

# Connect to database
conn = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    port=int(os.getenv('DB_PORT', 3306)),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

# Create tables
cursor = conn.cursor()

# Artifact metadata table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(255),
    period VARCHAR(255),
    accession_year INT,
    url TEXT
)
""")

# Artifact media table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_url TEXT,
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
)
""")

# Artifact colors table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
)
""")

conn.commit()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py
```

Access the dashboard at `http://localhost:8501`

## ETL Pipeline Implementation

### Extract: Fetching Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_artifacts=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    all_artifacts = []
    page = 1
    per_page = 100  # API max per page
    
    while len(all_artifacts) < num_artifacts:
        params = {
            'apikey': api_key,
            'size': per_page,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            all_artifacts.extend(records)
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_artifacts[:num_artifacts]
```

### Transform: Processing JSON to Relational Format

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform raw API data into structured DataFrames"""
    
    # Metadata DataFrame
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'technique': artifact.get('technique', ''),
            'period': artifact.get('period', ''),
            'accession_year': artifact.get('accessionyear'),
            'url': artifact.get('url', '')
        })
        
        # Extract media
        for image in artifact.get('images', []):
            media_records.append({
                'artifact_id': artifact.get('id'),
                'media_url': image.get('baseimageurl'),
                'media_type': 'image'
            })
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percent': color.get('percent')
            })
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### Load: Batch Insert into SQL

```python
def load_to_database(df_metadata, df_media, df_colors, conn):
    """Load transformed data into MySQL database"""
    cursor = conn.cursor()
    
    # Insert metadata
    insert_metadata = """
    INSERT IGNORE INTO artifactmetadata 
    (id, title, culture, century, classification, department, technique, period, accession_year, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    
    metadata_values = [
        tuple(row) for row in df_metadata.values
    ]
    cursor.executemany(insert_metadata, metadata_values)
    
    # Insert media
    if not df_media.empty:
        insert_media = """
        INSERT INTO artifactmedia (artifact_id, media_url, media_type)
        VALUES (%s, %s, %s)
        """
        media_values = [tuple(row) for row in df_media.values]
        cursor.executemany(insert_media, media_values)
    
    # Insert colors
    if not df_colors.empty:
        insert_colors = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, percent)
        VALUES (%s, %s, %s, %s)
        """
        color_values = [tuple(row) for row in df_colors.values]
        cursor.executemany(insert_colors, color_values)
    
    conn.commit()
    print(f"Loaded {len(df_metadata)} artifacts successfully")
```

## Analytical SQL Queries

### Query 1: Artifacts by Culture

```python
def query_artifacts_by_culture(conn):
    """Count artifacts grouped by culture"""
    query = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL AND culture != ''
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 20
    """
    df = pd.read_sql(query, conn)
    return df
```

### Query 2: Century Distribution

```python
def query_century_distribution(conn):
    """Analyze artifact distribution by century"""
    query = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL AND century != ''
    GROUP BY century
    ORDER BY count DESC
    """
    df = pd.read_sql(query, conn)
    return df
```

### Query 3: Color Analysis

```python
def query_dominant_colors(conn):
    """Find most common colors across artifacts"""
    query = """
    SELECT c.color, COUNT(DISTINCT c.artifact_id) as artifact_count,
           AVG(c.percent) as avg_percent
    FROM artifactcolors c
    WHERE c.color IS NOT NULL
    GROUP BY c.color
    ORDER BY artifact_count DESC
    LIMIT 15
    """
    df = pd.read_sql(query, conn)
    return df
```

### Query 4: Media Availability

```python
def query_media_stats(conn):
    """Analyze media availability per artifact"""
    query = """
    SELECT 
        a.classification,
        COUNT(DISTINCT a.id) as artifact_count,
        COUNT(m.media_id) as total_images,
        ROUND(COUNT(m.media_id) / COUNT(DISTINCT a.id), 2) as avg_images_per_artifact
    FROM artifactmetadata a
    LEFT JOIN artifactmedia m ON a.id = m.artifact_id
    GROUP BY a.classification
    HAVING artifact_count > 5
    ORDER BY avg_images_per_artifact DESC
    """
    df = pd.read_sql(query, conn)
    return df
```

## Streamlit Dashboard Example

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Art Analytics", layout="wide")

st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar for query selection
query_options = [
    "Artifacts by Culture",
    "Century Distribution",
    "Dominant Colors",
    "Media Availability",
    "Department Statistics"
]

selected_query = st.sidebar.selectbox("Select Analysis", query_options)

# Database connection
conn = get_database_connection()

# Execute selected query
if selected_query == "Artifacts by Culture":
    df = query_artifacts_by_culture(conn)
    st.subheader("Artifact Distribution by Culture")
    
    # Display table
    st.dataframe(df, use_container_width=True)
    
    # Visualization
    fig = px.bar(df, x='culture', y='artifact_count',
                 title='Top 20 Cultures by Artifact Count',
                 labels={'artifact_count': 'Number of Artifacts'})
    st.plotly_chart(fig, use_container_width=True)

elif selected_query == "Dominant Colors":
    df = query_dominant_colors(conn)
    st.subheader("Most Common Colors in Collection")
    
    col1, col2 = st.columns(2)
    
    with col1:
        st.dataframe(df)
    
    with col2:
        fig = px.pie(df, values='artifact_count', names='color',
                     title='Color Distribution')
        st.plotly_chart(fig)
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(url, params, delay=0.5):
    """Fetch data with rate limiting"""
    response = requests.get(url, params=params)
    time.sleep(delay)  # Respect API rate limits
    return response
```

### Incremental Data Loading

```python
def get_last_artifact_id(conn):
    """Get the last loaded artifact ID"""
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(conn, num_new_artifacts=50):
    """Load only new artifacts"""
    last_id = get_last_artifact_id(conn)
    
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'size': num_new_artifacts,
        'after': last_id  # Fetch artifacts after this ID
    }
    
    # Fetch and load new data
    artifacts = fetch_artifacts_with_params(params)
    # ... transform and load
```

### Error Handling

```python
def safe_etl_pipeline(num_artifacts=100):
    """ETL pipeline with error handling"""
    try:
        # Extract
        artifacts = fetch_artifacts(num_artifacts)
        if not artifacts:
            st.error("No artifacts fetched from API")
            return
        
        # Transform
        df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        
        # Load
        conn = get_database_connection()
        load_to_database(df_metadata, df_media, df_colors, conn)
        
        st.success(f"Successfully loaded {len(df_metadata)} artifacts")
        
    except requests.RequestException as e:
        st.error(f"API Error: {str(e)}")
    except mysql.connector.Error as e:
        st.error(f"Database Error: {str(e)}")
    except Exception as e:
        st.error(f"Unexpected Error: {str(e)}")
    finally:
        if 'conn' in locals() and conn.is_connected():
            conn.close()
```

## Troubleshooting

### API Key Issues

```python
# Verify API key is loaded
import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv('HARVARD_API_KEY')

if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment variables")

# Test API connection
response = requests.get(
    "https://api.harvardartmuseums.org/object",
    params={'apikey': api_key, 'size': 1}
)

print(f"API Status: {response.status_code}")
```

### Database Connection Issues

```python
# Test database connection
try:
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    print("✓ Database connected successfully")
    conn.close()
except mysql.connector.Error as e:
    print(f"✗ Database connection failed: {e}")
```

### Data Quality Checks

```python
def validate_data_quality(df_metadata):
    """Check data quality before loading"""
    
    # Check for duplicates
    duplicates = df_metadata['id'].duplicated().sum()
    if duplicates > 0:
        print(f"Warning: {duplicates} duplicate artifact IDs found")
    
    # Check for null values in critical fields
    null_counts = df_metadata[['id', 'title']].isnull().sum()
    print(f"Null values:\n{null_counts}")
    
    # Check data types
    if not pd.api.types.is_integer_dtype(df_metadata['id']):
        print("Warning: 'id' column is not integer type")
    
    return True
```

This skill enables AI coding agents to build, configure, and extend ETL pipelines for museum data analytics using the Harvard Art Museums API.
