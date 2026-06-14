---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - how do I build an ETL pipeline for the Harvard Art Museums API
  - create a data engineering pipeline with Harvard artifacts data
  - set up analytics dashboard for museum collection data
  - extract and transform Harvard Art Museums API data to SQL
  - build a Streamlit app for art collection analytics
  - query and visualize museum artifact data with SQL
  - implement batch data collection from Harvard museums API
  - create interactive visualizations for artifact metadata
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines with API data extraction, SQL storage, analytical querying, and interactive Streamlit dashboards.

## What This Project Does

The Harvard Art Museums ETL Analytics application:

- **Extracts** artifact data from the Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables (metadata, media, colors)
- **Loads** data into MySQL/TiDB Cloud using batch inserts
- **Analyzes** data with 20+ predefined SQL queries
- **Visualizes** results through interactive Plotly charts in a Streamlit dashboard

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

Required packages:
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

Create a `.env` file for sensitive credentials:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Database
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Setup

The application uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    primaryimageurl TEXT,
    url TEXT
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
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

## Key Components

### 1. API Data Collection

Extract artifacts with pagination:

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    per_page = 100
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': per_page,
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            artifacts.extend(records)
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### 2. Data Transformation

Transform nested JSON to structured DataFrames:

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform raw API data into normalized tables"""
    
    # Metadata table
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'url': artifact.get('url')
        })
        
        # Extract media
        for image in artifact.get('images', []):
            media_records.append({
                'artifact_id': artifact.get('id'),
                'baseimageurl': image.get('baseimageurl'),
                'format': image.get('format'),
                'height': image.get('height'),
                'width': image.get('width')
            })
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### 3. Database Loading

Batch insert data into SQL:

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 4000)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df_metadata):
    """Load metadata into database with batch insert"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, dated, classification, department, 
     technique, medium, dimensions, creditline, accessionyear, 
     primaryimageurl, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    values = df_metadata.values.tolist()
    cursor.executemany(insert_query, values)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    return cursor.rowcount

def load_media(df_media):
    """Load media data"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, baseimageurl, format, height, width)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    values = df_media.values.tolist()
    cursor.executemany(insert_query, values)
    
    conn.commit()
    cursor.close()
    conn.close()

def load_colors(df_colors):
    """Load color data"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    values = df_colors.values.tolist()
    cursor.executemany(insert_query, values)
    
    conn.commit()
    cursor.close()
    conn.close()
```

### 4. Analytics Queries

Execute analytical SQL queries:

```python
def execute_analytics_query(query):
    """Execute SQL query and return results as DataFrame"""
    conn = get_db_connection()
    
    try:
        df = pd.read_sql(query, conn)
        return df
    except Error as e:
        print(f"Error executing query: {e}")
        return None
    finally:
        conn.close()

# Example analytics queries
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
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE department IS NOT NULL 
        GROUP BY department 
        ORDER BY count DESC
    """,
    
    "Artifacts with Images": """
        SELECT 
            SUM(CASE WHEN primaryimageurl IS NOT NULL THEN 1 ELSE 0 END) as with_images,
            SUM(CASE WHEN primaryimageurl IS NULL THEN 1 ELSE 0 END) as without_images
        FROM artifactmetadata
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency 
        FROM artifactcolors 
        WHERE color IS NOT NULL 
        GROUP BY color 
        ORDER BY frequency DESC 
        LIMIT 10
    """,
    
    "Average Color Distribution": """
        SELECT color, AVG(percent) as avg_percent 
        FROM artifactcolors 
        WHERE percent IS NOT NULL 
        GROUP BY color 
        ORDER BY avg_percent DESC 
        LIMIT 10
    """
}
```

### 5. Streamlit Dashboard

Create interactive visualization dashboard:

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for ETL operations
    st.sidebar.header("ETL Operations")
    
    if st.sidebar.button("Fetch New Data"):
        with st.spinner("Fetching artifacts from API..."):
            artifacts = fetch_artifacts(num_records=100)
            st.sidebar.success(f"Fetched {len(artifacts)} artifacts")
            
            # Transform
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
            
            # Load
            load_metadata(df_meta)
            load_media(df_media)
            load_colors(df_colors)
            
            st.sidebar.success("Data loaded successfully!")
    
    # Analytics section
    st.header("📊 Analytics Dashboard")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        
        with st.spinner("Executing query..."):
            df_result = execute_analytics_query(query)
            
            if df_result is not None and not df_result.empty:
                st.subheader("Query Results")
                st.dataframe(df_result)
                
                # Auto-generate visualization
                if len(df_result.columns) == 2:
                    fig = px.bar(
                        df_result,
                        x=df_result.columns[0],
                        y=df_result.columns[1],
                        title=query_name
                    )
                    st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Complete ETL Pipeline

```python
def run_etl_pipeline(num_records=100):
    """Run complete ETL pipeline"""
    
    # Extract
    print("Extracting data from API...")
    raw_data = fetch_artifacts(num_records)
    
    # Transform
    print("Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(raw_data)
    
    # Load
    print("Loading data to database...")
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    
    print(f"ETL complete: {len(df_metadata)} artifacts loaded")
    
    return {
        'metadata_count': len(df_metadata),
        'media_count': len(df_media),
        'colors_count': len(df_colors)
    }
```

### Incremental Data Loading

```python
def get_latest_artifact_id():
    """Get the latest artifact ID in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return result[0] or 0

def fetch_new_artifacts():
    """Fetch only new artifacts not in database"""
    latest_id = get_latest_artifact_id()
    
    # Implement filtering logic based on latest_id
    # This depends on API capabilities
    pass
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
import time

def fetch_artifacts_with_retry(num_records=100, delay=1):
    """Fetch with rate limiting"""
    artifacts = []
    page = 1
    
    while len(artifacts) < num_records:
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            time.sleep(delay)  # Rate limiting
            page += 1
            
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:
                print("Rate limited, waiting...")
                time.sleep(60)
            else:
                raise
    
    return artifacts[:num_records]
```

### Database Connection Issues

```python
def get_db_connection_with_retry(max_retries=3):
    """Get database connection with retry logic"""
    for attempt in range(max_retries):
        try:
            conn = mysql.connector.connect(
                host=os.getenv('DB_HOST'),
                port=int(os.getenv('DB_PORT', 4000)),
                user=os.getenv('DB_USER'),
                password=os.getenv('DB_PASSWORD'),
                database=os.getenv('DB_NAME'),
                connect_timeout=10
            )
            return conn
        except Error as e:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)
            else:
                raise
```

### Handling Missing Data

```python
def safe_transform(artifact, field, default=None):
    """Safely extract field with default value"""
    return artifact.get(field, default)

def transform_artifacts_safe(raw_data):
    """Transform with null handling"""
    metadata_records = []
    
    for artifact in raw_data:
        metadata_records.append({
            'id': safe_transform(artifact, 'id', 0),
            'title': safe_transform(artifact, 'title', 'Unknown'),
            'culture': safe_transform(artifact, 'culture'),
            'century': safe_transform(artifact, 'century'),
            # ... handle all fields safely
        })
    
    return pd.DataFrame(metadata_records)
```

This skill provides everything needed to build production-ready ETL pipelines and analytics dashboards for museum collection data using modern data engineering practices.
