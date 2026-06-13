---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - show me how to use the Harvard artifacts analytics app
  - help me set up data engineering pipeline with Harvard API
  - how to create analytics dashboard with museum artifacts data
  - extract and analyze Harvard Art Museums collection data
  - build SQL analytics for art museum artifacts
  - set up Streamlit dashboard for Harvard Art Museums
  - create data pipeline for museum collection API
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to work with the Harvard Art Museums API to build end-to-end data engineering pipelines, perform ETL operations, store data in SQL databases, and create interactive analytics dashboards using Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides a complete data pipeline that:
- Extracts artifact data from Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into relational database tables
- Loads data into MySQL/TiDB Cloud with proper schema design
- Runs 20+ analytical SQL queries for insights
- Visualizes results through interactive Plotly charts in Streamlit

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
DB_HOST=your_db_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
```

## Key Components

### 1. API Data Extraction

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
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

def collect_all_artifacts(api_key, max_records=1000):
    """Collect multiple pages of artifacts with rate limiting"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        data = fetch_artifacts(api_key, page=page, size=size)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_artifacts.extend(records)
        page += 1
        
        # Rate limiting - be respectful of API
        time.sleep(0.5)
        
        print(f"Collected {len(all_artifacts)} artifacts...")
    
    return all_artifacts[:max_records]
```

### 2. ETL Pipeline

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON into relational tables"""
    
    # Extract metadata
    metadata = []
    for artifact in raw_data:
        metadata.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear')
        })
    
    # Extract media information
    media = []
    for artifact in raw_data:
        objectid = artifact.get('objectid')
        if artifact.get('images'):
            for img in artifact['images']:
                media.append({
                    'objectid': objectid,
                    'imageid': img.get('imageid'),
                    'baseimageurl': img.get('baseimageurl'),
                    'format': img.get('format'),
                    'width': img.get('width'),
                    'height': img.get('height')
                })
    
    # Extract color information
    colors = []
    for artifact in raw_data:
        objectid = artifact.get('objectid')
        if artifact.get('colors'):
            for color in artifact['colors']:
                colors.append({
                    'objectid': objectid,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata),
        pd.DataFrame(media),
        pd.DataFrame(colors)
    )
```

### 3. Database Schema & Loading

```python
import mysql.connector
from mysql.connector import Error

def create_database_schema(connection):
    """Create tables for artifact data"""
    cursor = connection.cursor()
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            dated VARCHAR(200),
            classification VARCHAR(200),
            department VARCHAR(200),
            division VARCHAR(200),
            technique VARCHAR(500),
            medium VARCHAR(500),
            dimensions VARCHAR(500),
            creditline TEXT,
            accessionyear INT
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            imageid INT,
            baseimageurl VARCHAR(500),
            format VARCHAR(50),
            width INT,
            height INT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    connection.commit()
    cursor.close()

def load_data_to_db(connection, metadata_df, media_df, colors_df):
    """Batch insert data into database"""
    cursor = connection.cursor()
    
    # Insert metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT IGNORE INTO artifactmetadata 
            (objectid, title, culture, century, dated, classification, 
             department, division, technique, medium, dimensions, 
             creditline, accessionyear)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (objectid, imageid, baseimageurl, format, width, height)
            VALUES (%s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (objectid, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    connection.commit()
    cursor.close()

def get_db_connection():
    """Create database connection from environment variables"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```

### 4. Analytical SQL Queries

```python
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
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as occurrences, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY occurrences DESC
        LIMIT 15
    """,
    
    "Image Format Distribution": """
        SELECT format, COUNT(*) as count
        FROM artifactmedia
        GROUP BY format
        ORDER BY count DESC
    """,
    
    "Artifacts with Multiple Images": """
        SELECT m.objectid, m.title, COUNT(media.imageid) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia media ON m.objectid = media.objectid
        GROUP BY m.objectid, m.title
        HAVING image_count > 1
        ORDER BY image_count DESC
        LIMIT 20
    """
}

def execute_query(connection, query_name):
    """Execute analytical query and return results"""
    cursor = connection.cursor(dictionary=True)
    cursor.execute(ANALYTICS_QUERIES[query_name])
    results = cursor.fetchall()
    cursor.close()
    return pd.DataFrame(results)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for configuration
    with st.sidebar:
        st.header("Configuration")
        api_key = st.text_input("Harvard API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))
        
        st.header("ETL Pipeline")
        num_records = st.number_input("Records to Fetch", 100, 5000, 1000)
        
        if st.button("Run ETL Pipeline"):
            run_etl_pipeline(api_key, num_records)
    
    # Analytics section
    st.header("📊 Analytics Queries")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        connection = get_db_connection()
        df = execute_query(connection, query_name)
        connection.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)

def run_etl_pipeline(api_key, num_records):
    """Execute complete ETL pipeline"""
    with st.spinner("Extracting data from API..."):
        raw_data = collect_all_artifacts(api_key, num_records)
    
    with st.spinner("Transforming data..."):
        metadata_df, media_df, colors_df = transform_artifacts(raw_data)
    
    with st.spinner("Loading to database..."):
        connection = get_db_connection()
        create_database_schema(connection)
        load_data_to_db(connection, metadata_df, media_df, colors_df)
        connection.close()
    
    st.success(f"✅ Successfully loaded {len(metadata_df)} artifacts!")

if __name__ == "__main__":
    main()
```

### Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Incremental Data Loading

```python
def get_latest_objectid(connection):
    """Get the highest objectid already in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(objectid) as max_id FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def fetch_new_artifacts_only(api_key, last_id):
    """Fetch only artifacts newer than last_id"""
    # Add filtering logic based on your needs
    pass
```

### Error Handling for API Calls

```python
def safe_api_fetch(api_key, page, retries=3):
    """Fetch with retry logic"""
    for attempt in range(retries):
        try:
            return fetch_artifacts(api_key, page)
        except Exception as e:
            if attempt == retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
            continue
```

## Troubleshooting

**API Rate Limiting**: Add `time.sleep(0.5)` between requests or implement exponential backoff.

**Database Connection Issues**: Verify environment variables are set correctly and database is accessible.

**Missing Data**: Not all artifacts have all fields. Use `.get()` method with defaults when parsing JSON.

**Memory Issues**: Process data in smaller batches instead of loading all at once.

**Foreign Key Constraints**: Insert metadata before media/colors tables to satisfy foreign key relationships.
