---
name: harvard-artifacts-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with Harvard artifacts
  - extract and transform museum artifact data
  - set up SQL database for art collection analytics
  - visualize Harvard Art Museums data with Streamlit
  - implement data engineering pipeline for museum artifacts
  - query and analyze Harvard art collection data
  - build interactive museum data visualization
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is an end-to-end data engineering solution that demonstrates real-world ETL pipelines, SQL analytics, and interactive visualization. It extracts artifact data from the Harvard Art Museums API, transforms it into relational tables, loads it into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics dashboards through Streamlit.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

The project creates three normalized tables:
- `artifactmetadata` - Core artifact information
- `artifactmedia` - Media and image details
- `artifactcolors` - Color analysis data

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Key dependencies:**
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

Create a `.env` file or set environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Cloud Connection
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Getting Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free for research/educational use)
3. Store it in your environment variables

### Database Setup

**MySQL/TiDB Cloud schema creation:**

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
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
    url TEXT,
    creditline TEXT,
    technique VARCHAR(500),
    medium VARCHAR(500)
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    iiifbaseuri TEXT,
    primaryimageurl TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

The Streamlit interface will open at `http://localhost:8501`

## Core ETL Pipeline

### 1. Extract Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """Extract artifacts from Harvard Art Museums API"""
    url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(url, params=params)
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
artifacts = data['records']
total_pages = data['info']['pages']
```

### 2. Transform Data

```python
import pandas as pd

def transform_metadata(artifacts):
    """Transform artifact data into normalized metadata"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'creditline': artifact.get('creditline'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_media(artifacts):
    """Extract media/image information"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        if images:
            for img in images:
                media = {
                    'artifact_id': artifact_id,
                    'baseimageurl': img.get('baseimageurl'),
                    'iiifbaseuri': img.get('iiifbaseuri'),
                    'primaryimageurl': artifact.get('primaryimageurl')
                }
                media_list.append(media)
        else:
            # Add entry even if no images
            media_list.append({
                'artifact_id': artifact_id,
                'baseimageurl': None,
                'iiifbaseuri': None,
                'primaryimageurl': artifact.get('primaryimageurl')
            })
    
    return pd.DataFrame(media_list)

def transform_colors(artifacts):
    """Extract color analysis data"""
    colors_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color_info in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color': color_info.get('color'),
                'spectrum': color_info.get('spectrum'),
                'hue': color_info.get('hue'),
                'percent': color_info.get('percent')
            }
            colors_list.append(color_data)
    
    return pd.DataFrame(colors_list)
```

### 3. Load Data to SQL

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df, connection):
    """Load metadata into SQL database"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, classification, 
     department, dated, url, creditline, technique, medium)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()

def load_media(df, connection):
    """Load media data"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, baseimageurl, iiifbaseuri, primaryimageurl)
    VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()

def load_colors(df, connection):
    """Load color data"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()

# Full ETL Pipeline
def run_etl_pipeline(num_pages=5):
    """Execute complete ETL pipeline"""
    api_key = os.getenv('HARVARD_API_KEY')
    connection = get_db_connection()
    
    try:
        for page in range(1, num_pages + 1):
            # Extract
            data = fetch_artifacts(api_key, page=page, size=100)
            artifacts = data['records']
            
            # Transform
            metadata_df = transform_metadata(artifacts)
            media_df = transform_media(artifacts)
            colors_df = transform_colors(artifacts)
            
            # Load
            load_metadata(metadata_df, connection)
            load_media(media_df, connection)
            load_colors(colors_df, connection)
            
            print(f"Loaded page {page}/{num_pages}")
    
    finally:
        connection.close()
```

## Analytics Queries

### Common SQL Queries

```python
def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    connection = get_db_connection()
    df = pd.read_sql(query, connection)
    connection.close()
    return df

# Top 10 artifacts by department
query_1 = """
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
GROUP BY department
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Artifacts with images
query_2 = """
SELECT 
    COUNT(DISTINCT m.artifact_id) as artifacts_with_images,
    (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
    ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / 
          (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
FROM artifactmedia m
WHERE m.primaryimageurl IS NOT NULL;
"""

# Top cultures by artifact count
query_3 = """
SELECT culture, COUNT(*) as count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY count DESC
LIMIT 15;
"""

# Color distribution
query_4 = """
SELECT color, COUNT(*) as frequency
FROM artifactcolors
GROUP BY color
ORDER BY frequency DESC
LIMIT 10;
"""

# Artifacts by century
query_5 = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC;
"""

# Classification breakdown
query_6 = """
SELECT classification, COUNT(*) as count
FROM artifactmetadata
WHERE classification IS NOT NULL
GROUP BY classification
ORDER BY count DESC
LIMIT 20;
"""

# Dominant color analysis
query_7 = """
SELECT c.color, AVG(c.percent) as avg_percent, COUNT(*) as frequency
FROM artifactcolors c
GROUP BY c.color
HAVING frequency > 10
ORDER BY avg_percent DESC
LIMIT 10;
"""
```

## Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")

# Sidebar for query selection
queries = {
    "Artifacts by Department": query_1,
    "Image Availability": query_2,
    "Top Cultures": query_3,
    "Color Distribution": query_4,
    "Artifacts by Century": query_5,
    "Classification Breakdown": query_6,
    "Dominant Colors": query_7
}

selected_query = st.sidebar.selectbox("Select Analysis", list(queries.keys()))

# Execute query
if st.button("Run Analysis"):
    with st.spinner("Querying database..."):
        df = execute_query(queries[selected_query])
        
        # Display results
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Visualize if applicable
        if len(df.columns) == 2 and df.shape[0] > 0:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

# ETL Control
st.sidebar.header("ETL Pipeline")
num_pages = st.sidebar.number_input("Pages to fetch", min_value=1, max_value=100, value=5)

if st.sidebar.button("Run ETL Pipeline"):
    with st.spinner("Running ETL..."):
        run_etl_pipeline(num_pages)
        st.success(f"Successfully loaded {num_pages} pages of data!")
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_artifacts_with_rate_limit(api_key, page, delay=0.5):
    """Fetch with rate limiting to respect API limits"""
    data = fetch_artifacts(api_key, page)
    time.sleep(delay)  # Wait between requests
    return data
```

### Incremental Data Loading

```python
def get_max_artifact_id(connection):
    """Get highest artifact ID already loaded"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_etl(start_id):
    """Load only new artifacts since last run"""
    api_key = os.getenv('HARVARD_API_KEY')
    # Filter API results by ID > start_id
    # Implementation depends on API capabilities
```

### Error Handling

```python
def safe_etl_pipeline(num_pages):
    """ETL with error handling and logging"""
    connection = get_db_connection()
    
    for page in range(1, num_pages + 1):
        try:
            data = fetch_artifacts(api_key, page)
            artifacts = data['records']
            
            metadata_df = transform_metadata(artifacts)
            load_metadata(metadata_df, connection)
            
        except Exception as e:
            print(f"Error on page {page}: {str(e)}")
            # Log to file or monitoring system
            continue
    
    connection.close()
```

## Troubleshooting

### API Connection Issues

```python
# Test API connectivity
import requests

def test_api_connection():
    api_key = os.getenv('HARVARD_API_KEY')
    url = f"https://api.harvardartmuseums.org/object?apikey={api_key}&size=1"
    
    try:
        response = requests.get(url, timeout=10)
        if response.status_code == 200:
            print("✓ API connection successful")
            return True
        else:
            print(f"✗ API returned status {response.status_code}")
            return False
    except Exception as e:
        print(f"✗ Connection error: {str(e)}")
        return False
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        connection = get_db_connection()
        cursor = connection.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        connection.close()
        print("✓ Database connection successful")
        return True
    except Error as e:
        print(f"✗ Database error: {str(e)}")
        return False
```

### Missing Data Handling

```python
def clean_null_values(df):
    """Handle missing values in dataframe"""
    # Replace None with appropriate defaults
    df = df.fillna({
        'culture': 'Unknown',
        'period': 'Unknown',
        'century': 'Unknown',
        'classification': 'Unclassified'
    })
    return df
```

### Performance Optimization

```python
# Use batch inserts for better performance
def batch_insert(df, table_name, connection, batch_size=1000):
    """Insert data in batches"""
    total_rows = len(df)
    
    for i in range(0, total_rows, batch_size):
        batch = df[i:i+batch_size]
        # Insert batch
        load_metadata(batch, connection)
        print(f"Inserted {min(i+batch_size, total_rows)}/{total_rows}")
```
