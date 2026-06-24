---
name: harvard-art-museums-data-pipeline
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I fetch data from the Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create a data analytics dashboard with Streamlit
  - query and visualize Harvard Art Museums collection data
  - set up a SQL database for artifact metadata
  - process nested JSON from museum APIs into relational tables
  - analyze art collections with SQL and Python
  - build museum data engineering pipeline
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering solution for collecting, transforming, storing, and visualizing artifact data from the Harvard Art Museums API. It demonstrates production-grade ETL pipelines, SQL analytics, and interactive dashboards using Streamlit.

## What This Project Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts metadata, transforms nested JSON into relational format, loads into SQL database
- **Database Design**: Creates normalized tables (artifactmetadata, artifactmedia, artifactcolors)
- **Analytics**: Executes 20+ predefined SQL queries for cultural, temporal, and media analysis
- **Visualization**: Renders interactive Plotly charts based on query results

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY=your_api_key_here
export DB_HOST=your_database_host
export DB_USER=your_database_user
export DB_PASSWORD=your_database_password
export DB_NAME=your_database_name
```

## Core Components

### 1. API Data Collection

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    data = response.json()
    return data['records'], data['info']

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts, info = fetch_artifacts(api_key, page=1, size=100)

print(f"Total records: {info['totalrecords']}")
print(f"Pages: {info['pages']}")
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def extract_metadata(artifacts):
    """Extract artifact metadata into structured format"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'dated': artifact.get('dated'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'dimensions': artifact.get('dimensions'),
            'description': artifact.get('description')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def extract_media(artifacts):
    """Extract media/image data from nested structure"""
    media_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for image in images:
            media = {
                'objectid': objectid,
                'imageid': image.get('imageid'),
                'baseimageurl': image.get('baseimageurl'),
                'height': image.get('height'),
                'width': image.get('width'),
                'iiifbaseuri': image.get('iiifbaseuri')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def extract_colors(artifacts):
    """Extract color information from artifacts"""
    colors_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'objectid': objectid,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            colors_list.append(color_data)
    
    return pd.DataFrame(colors_list)
```

### 3. Database Setup

```python
def create_database_connection():
    """Create MySQL/TiDB connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def create_tables(connection):
    """Create normalized tables for artifact data"""
    cursor = connection.cursor()
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            dated VARCHAR(255),
            century VARCHAR(255),
            classification VARCHAR(255),
            medium TEXT,
            department VARCHAR(255),
            division VARCHAR(255),
            period VARCHAR(255),
            technique TEXT,
            dimensions TEXT,
            description TEXT
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            imageid INT,
            baseimageurl TEXT,
            height INT,
            width INT,
            iiifbaseuri TEXT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(255),
            spectrum VARCHAR(255),
            hue VARCHAR(255),
            percent FLOAT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    connection.commit()
    cursor.close()

def load_data(connection, df, table_name):
    """Batch insert data into SQL table"""
    cursor = connection.cursor()
    
    cols = ",".join([str(i) for i in df.columns.tolist()])
    
    for i, row in df.iterrows():
        sql = f"INSERT INTO {table_name} ({cols}) VALUES ({','.join(['%s']*len(row))})"
        sql += " ON DUPLICATE KEY UPDATE objectid=objectid"  # Handle duplicates
        cursor.execute(sql, tuple(row))
    
    connection.commit()
    cursor.close()
```

### 4. SQL Analytics Queries

```python
def get_artifacts_by_culture(connection, limit=10):
    """Get artifact count by culture"""
    query = """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT %s
    """
    return pd.read_sql(query, connection, params=(limit,))

def get_artifacts_by_century(connection):
    """Analyze artifact distribution by century"""
    query = """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY century
    """
    return pd.read_sql(query, connection)

def get_media_availability(connection):
    """Check image/media availability"""
    query = """
        SELECT 
            CASE WHEN m.objectid IS NOT NULL THEN 'Has Media' ELSE 'No Media' END as media_status,
            COUNT(DISTINCT a.objectid) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.objectid = m.objectid
        GROUP BY media_status
    """
    return pd.read_sql(query, connection)

def get_color_distribution(connection, limit=15):
    """Analyze color usage across artifacts"""
    query = """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT %s
    """
    return pd.read_sql(query, connection, params=(limit,))

def get_department_stats(connection):
    """Department-wise artifact distribution"""
    query = """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """
    return pd.read_sql(query, connection)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                     value=os.getenv('HARVARD_API_KEY', ''))
    
    # Database connection
    connection = create_database_connection()
    
    if connection:
        st.success("Database connected!")
        
        # Analytics section
        st.header("Analytics Queries")
        
        query_options = [
            "Artifacts by Culture",
            "Artifacts by Century",
            "Media Availability",
            "Color Distribution",
            "Department Statistics"
        ]
        
        selected_query = st.selectbox("Select Analysis", query_options)
        
        if st.button("Run Query"):
            if selected_query == "Artifacts by Culture":
                df = get_artifacts_by_culture(connection)
                st.dataframe(df)
                
                fig = px.bar(df, x='culture', y='count', 
                            title='Artifacts by Culture')
                st.plotly_chart(fig)
                
            elif selected_query == "Artifacts by Century":
                df = get_artifacts_by_century(connection)
                st.dataframe(df)
                
                fig = px.line(df, x='century', y='count', 
                             title='Artifact Timeline by Century')
                st.plotly_chart(fig)
                
            elif selected_query == "Color Distribution":
                df = get_color_distribution(connection)
                st.dataframe(df)
                
                fig = px.bar(df, x='color', y='frequency',
                            title='Most Common Colors in Collection')
                st.plotly_chart(fig)
    
    else:
        st.error("Failed to connect to database")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Launch Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Full ETL Workflow

```python
def run_etl_pipeline(api_key, num_pages=5):
    """Complete ETL pipeline execution"""
    connection = create_database_connection()
    create_tables(connection)
    
    all_artifacts = []
    
    # Extract
    for page in range(1, num_pages + 1):
        artifacts, info = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(artifacts)
        print(f"Fetched page {page}/{info['pages']}")
    
    # Transform
    metadata_df = extract_metadata(all_artifacts)
    media_df = extract_media(all_artifacts)
    colors_df = extract_colors(all_artifacts)
    
    # Load
    load_data(connection, metadata_df, 'artifactmetadata')
    load_data(connection, media_df, 'artifactmedia')
    load_data(connection, colors_df, 'artifactcolors')
    
    print(f"Loaded {len(metadata_df)} artifacts into database")
    connection.close()
```

### Incremental Data Updates

```python
def get_latest_objectid(connection):
    """Get the most recent objectid to avoid duplicates"""
    query = "SELECT MAX(objectid) as max_id FROM artifactmetadata"
    result = pd.read_sql(query, connection)
    return result['max_id'].iloc[0] or 0

def incremental_load(api_key, connection):
    """Only fetch new artifacts"""
    latest_id = get_latest_objectid(connection)
    
    artifacts, _ = fetch_artifacts(api_key, page=1, size=100)
    new_artifacts = [a for a in artifacts if a['objectid'] > latest_id]
    
    if new_artifacts:
        metadata_df = extract_metadata(new_artifacts)
        load_data(connection, metadata_df, 'artifactmetadata')
        print(f"Loaded {len(new_artifacts)} new artifacts")
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff on rate limits"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page=page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
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
        conn = create_database_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        return result[0] == 1
    except Exception as e:
        print(f"Connection test failed: {e}")
        return False
```

### Handling Missing Data

```python
def safe_extract(artifact, key, default=None):
    """Safely extract nested values"""
    return artifact.get(key, default)

def clean_metadata(df):
    """Clean and standardize metadata"""
    df['culture'] = df['culture'].fillna('Unknown')
    df['century'] = df['century'].fillna('Undated')
    df['description'] = df['description'].str[:5000]  # Truncate long text
    return df
```

## Configuration

Create a `.env` file:

```
HARVARD_API_KEY=your_api_key_here
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

Or configure via Streamlit secrets (`~/.streamlit/secrets.toml`):

```toml
HARVARD_API_KEY = "your_api_key_here"
DB_HOST = "your_host"
DB_USER = "your_user"
DB_PASSWORD = "your_password"
DB_NAME = "harvard_artifacts"
```
