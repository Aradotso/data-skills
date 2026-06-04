---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline using Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - connect to Harvard Art Museums API
  - create data analytics dashboard with Streamlit
  - set up SQL database for artifact collections
  - visualize museum artifact data
  - implement data engineering workflow for art collections
  - query and analyze Harvard museum artifacts
  - build museum data warehouse pipeline
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact collections.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON into relational SQL tables
- **Database Design**: Structured schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Visualization**: Interactive Streamlit dashboard with Plotly charts

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Get your API key from: https://docs.harvardartmuseums.org/

Store in environment variables:

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

Set up MySQL or TiDB Cloud credentials:

```bash
# .env file
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py
```

The application will be available at `http://localhost:8501`

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    """
    url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

def collect_all_artifacts(api_key, max_records=1000):
    """
    Collect artifacts with pagination handling
    """
    artifacts = []
    page = 1
    size = 100
    
    while len(artifacts) < max_records:
        data = fetch_artifacts(api_key, page=page, size=size)
        records = data.get('records', [])
        
        if not records:
            break
            
        artifacts.extend(records)
        page += 1
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return artifacts[:max_records]
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifacts(raw_data):
    """
    Transform nested JSON into relational tables
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Metadata table
        metadata_records.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'department': artifact.get('department'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'dated': artifact.get('dated')
        })
        
        # Media table
        for image in artifact.get('images', []):
            media_records.append({
                'objectid': artifact.get('objectid'),
                'imageid': image.get('imageid'),
                'baseimageurl': image.get('baseimageurl'),
                'iiifbaseuri': image.get('iiifbaseuri'),
                'copyright': image.get('copyright')
            })
        
        # Colors table
        for color in artifact.get('colors', []):
            color_records.append({
                'objectid': artifact.get('objectid'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return {
        'metadata': pd.DataFrame(metadata_records),
        'media': pd.DataFrame(media_records),
        'colors': pd.DataFrame(color_records)
    }

def create_database_connection():
    """
    Create MySQL database connection
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT', 3306),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def create_tables(connection):
    """
    Create database schema
    """
    cursor = connection.cursor()
    
    # Metadata table
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS artifactmetadata (
        objectid INT PRIMARY KEY,
        title TEXT,
        culture VARCHAR(255),
        period VARCHAR(255),
        century VARCHAR(255),
        department VARCHAR(255),
        classification VARCHAR(255),
        medium TEXT,
        dimensions VARCHAR(255),
        dated VARCHAR(255)
    )
    """)
    
    # Media table
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS artifactmedia (
        id INT AUTO_INCREMENT PRIMARY KEY,
        objectid INT,
        imageid INT,
        baseimageurl TEXT,
        iiifbaseuri TEXT,
        copyright TEXT,
        FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
    )
    """)
    
    # Colors table
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS artifactcolors (
        id INT AUTO_INCREMENT PRIMARY KEY,
        objectid INT,
        color VARCHAR(100),
        spectrum VARCHAR(100),
        hue VARCHAR(100),
        percent FLOAT,
        FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
    )
    """)
    
    connection.commit()
    cursor.close()

def load_data_to_sql(connection, dataframes):
    """
    Batch insert data into SQL tables
    """
    cursor = connection.cursor()
    
    # Insert metadata
    for _, row in dataframes['metadata'].iterrows():
        cursor.execute("""
        INSERT IGNORE INTO artifactmetadata 
        (objectid, title, culture, period, century, department, classification, medium, dimensions, dated)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert media
    for _, row in dataframes['media'].iterrows():
        cursor.execute("""
        INSERT INTO artifactmedia 
        (objectid, imageid, baseimageurl, iiifbaseuri, copyright)
        VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in dataframes['colors'].iterrows():
        cursor.execute("""
        INSERT INTO artifactcolors 
        (objectid, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    connection.commit()
    cursor.close()
```

### 3. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password", value=os.getenv('HARVARD_API_KEY', ''))
    
    # ETL Section
    st.header("📥 Data Collection & ETL")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_records = st.number_input("Number of records to fetch", min_value=100, max_value=10000, value=1000, step=100)
    
    with col2:
        if st.button("Start ETL Pipeline"):
            with st.spinner("Fetching data from API..."):
                artifacts = collect_all_artifacts(api_key, max_records=num_records)
                st.success(f"Fetched {len(artifacts)} artifacts")
            
            with st.spinner("Transforming data..."):
                dataframes = transform_artifacts(artifacts)
                st.success("Data transformed successfully")
            
            with st.spinner("Loading to database..."):
                conn = create_database_connection()
                create_tables(conn)
                load_data_to_sql(conn, dataframes)
                conn.close()
                st.success("ETL pipeline completed!")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    
    queries = {
        "Top 10 Cultures by Artifact Count": """
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
        "Top 10 Colors Used": """
            SELECT color, COUNT(*) as count 
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY count DESC 
            LIMIT 10
        """,
        "Media Availability": """
            SELECT 
                COUNT(DISTINCT objectid) as artifacts_with_images,
                COUNT(*) as total_images,
                AVG(images_per_artifact) as avg_images_per_artifact
            FROM (
                SELECT objectid, COUNT(*) as images_per_artifact
                FROM artifactmedia
                GROUP BY objectid
            ) as subquery
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        conn = create_database_connection()
        df_result = pd.read_sql(queries[selected_query], conn)
        conn.close()
        
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) == 2 and 'count' in df_result.columns:
            fig = px.bar(df_result, x=df_result.columns[0], y='count', 
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Analytical Queries

```sql
-- Artifacts without images
SELECT COUNT(*) FROM artifactmetadata 
WHERE objectid NOT IN (SELECT DISTINCT objectid FROM artifactmedia);

-- Average colors per artifact
SELECT AVG(color_count) FROM (
    SELECT objectid, COUNT(*) as color_count 
    FROM artifactcolors 
    GROUP BY objectid
) as subquery;

-- Most common classification
SELECT classification, COUNT(*) as count 
FROM artifactmetadata 
WHERE classification IS NOT NULL 
GROUP BY classification 
ORDER BY count DESC 
LIMIT 5;

-- Color spectrum distribution
SELECT spectrum, COUNT(*) as count 
FROM artifactcolors 
GROUP BY spectrum 
ORDER BY count DESC;

-- Artifacts by period and culture
SELECT period, culture, COUNT(*) as count 
FROM artifactmetadata 
WHERE period IS NOT NULL AND culture IS NOT NULL 
GROUP BY period, culture 
ORDER BY count DESC 
LIMIT 20;
```

## Troubleshooting

### API Rate Limiting

```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            response.raise_for_status()
            return response.json()
        except HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def safe_db_connection():
    max_attempts = 3
    for attempt in range(max_attempts):
        try:
            conn = create_database_connection()
            if conn and conn.is_connected():
                return conn
        except Exception as e:
            st.error(f"Connection attempt {attempt + 1} failed: {e}")
            time.sleep(2)
    return None
```

### Handling Missing Data

```python
def safe_transform(artifact, field, default=None):
    """Safely extract fields with defaults"""
    return artifact.get(field, default) or default

# Usage in transformation
metadata_records.append({
    'objectid': safe_transform(artifact, 'objectid'),
    'title': safe_transform(artifact, 'title', 'Untitled'),
    'culture': safe_transform(artifact, 'culture', 'Unknown')
})
```

## Best Practices

1. **Use environment variables** for all credentials
2. **Implement pagination** for large datasets
3. **Add retry logic** for API calls
4. **Batch insert** data for better performance
5. **Index foreign keys** in SQL tables
6. **Cache results** in Streamlit for faster dashboards
7. **Validate data** before database insertion
