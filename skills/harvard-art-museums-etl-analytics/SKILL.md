---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and transform Harvard Art Museums API data
  - visualize museum collection data with Streamlit
  - set up SQL database for artifact metadata
  - analyze art museum collections with Python
  - build data engineering pipeline for art museums
  - create interactive museum data visualization
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build complete data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualizations with Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is an end-to-end data pipeline that:

- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into structured relational tables
- **Loads** data into SQL databases (MySQL/TiDB Cloud)
- **Analyzes** collections using 20+ predefined SQL queries
- **Visualizes** insights through interactive Plotly charts in Streamlit

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

## Installation

### Prerequisites

```bash
# Install required Python packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Create environment file
touch .env
```

### Environment Configuration

Create a `.env` file with the following variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Database Connection
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Database Schema

The application creates three relational tables:

### artifactmetadata
```sql
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    department VARCHAR(200),
    classification VARCHAR(200),
    technique VARCHAR(300),
    dated VARCHAR(200),
    url TEXT
);
```

### artifactmedia
```sql
CREATE TABLE artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

### artifactcolors
```sql
CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(100),
    spectrum VARCHAR(100),
    hue VARCHAR(100),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Core ETL Pipeline

### Extract: Fetching Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = 'https://api.harvardartmuseums.org/object'
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

def collect_all_artifacts(max_records=1000):
    """Collect artifacts with pagination handling"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        data = fetch_artifacts(page=page, size=size)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_artifacts.extend(records)
        page += 1
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return all_artifacts[:max_records]
```

### Transform: Processing JSON to Dataframes

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform raw JSON into structured dataframes"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata_records.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'century': artifact.get('century', '')[:100],
            'department': artifact.get('department', '')[:200],
            'classification': artifact.get('classification', '')[:200],
            'technique': artifact.get('technique', '')[:300],
            'dated': artifact.get('dated', '')[:200],
            'url': artifact.get('url', '')
        })
        
        # Extract media information
        images = artifact.get('images', [])
        for img in images:
            media_records.append({
                'objectid': artifact.get('objectid'),
                'baseimageurl': img.get('baseimageurl', ''),
                'format': img.get('format', '')[:50],
                'height': img.get('height'),
                'width': img.get('width')
            })
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'objectid': artifact.get('objectid'),
                'color': color.get('color', '')[:100],
                'spectrum': color.get('spectrum', '')[:100],
                'hue': color.get('hue', '')[:100],
                'percent': color.get('percent')
            })
    
    return {
        'metadata': pd.DataFrame(metadata_records),
        'media': pd.DataFrame(media_records),
        'colors': pd.DataFrame(color_records)
    }
```

### Load: Insert into SQL Database

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

def load_to_database(dataframes):
    """Batch insert dataframes into SQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Load metadata
        metadata_sql = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, century, department, classification, technique, dated, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        metadata_values = dataframes['metadata'].values.tolist()
        cursor.executemany(metadata_sql, metadata_values)
        
        # Load media
        media_sql = """
        INSERT INTO artifactmedia 
        (objectid, baseimageurl, format, height, width)
        VALUES (%s, %s, %s, %s, %s)
        """
        media_values = dataframes['media'].values.tolist()
        cursor.executemany(media_sql, media_values)
        
        # Load colors
        color_sql = """
        INSERT INTO artifactcolors 
        (objectid, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        color_values = dataframes['colors'].values.tolist()
        cursor.executemany(color_sql, color_values)
        
        conn.commit()
        print(f"Loaded {len(metadata_values)} artifacts successfully")
        
    except Error as e:
        print(f"Error loading data: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

## Analytical SQL Queries

### Sample Analytics Queries

```python
# Query 1: Artifacts by culture
query_by_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 15
"""

# Query 2: Artifacts by century
query_by_century = """
SELECT century, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE century IS NOT NULL AND century != ''
GROUP BY century
ORDER BY artifact_count DESC
"""

# Query 3: Department distribution
query_by_department = """
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY artifact_count DESC
"""

# Query 4: Color analysis
query_color_distribution = """
SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
FROM artifactcolors
WHERE color IS NOT NULL
GROUP BY color
ORDER BY usage_count DESC
LIMIT 20
"""

# Query 5: Media format analysis
query_media_formats = """
SELECT format, COUNT(*) as image_count, 
       AVG(height) as avg_height, AVG(width) as avg_width
FROM artifactmedia
WHERE format IS NOT NULL
GROUP BY format
ORDER BY image_count DESC
"""

# Query 6: Classification breakdown
query_classification = """
SELECT classification, COUNT(*) as count
FROM artifactmetadata
WHERE classification IS NOT NULL AND classification != ''
GROUP BY classification
ORDER BY count DESC
LIMIT 10
"""
```

### Execute and Visualize

```python
import plotly.express as px

def execute_and_visualize(query, title):
    """Execute SQL query and create visualization"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    
    # Create interactive bar chart
    if not df.empty and len(df.columns) >= 2:
        fig = px.bar(
            df.head(15),
            x=df.columns[0],
            y=df.columns[1],
            title=title,
            labels={df.columns[0]: df.columns[0].replace('_', ' ').title(),
                   df.columns[1]: df.columns[1].replace('_', ' ').title()}
        )
        fig.update_layout(xaxis_tickangle=-45)
        return df, fig
    
    return df, None
```

## Streamlit Dashboard Integration

```python
import streamlit as st

def main():
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    st.title("🎨 Harvard Art Museums Data Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a section",
        ["Data Collection", "ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        st.header("Data Collection from API")
        
        num_records = st.number_input("Number of records to fetch", 
                                      min_value=10, max_value=1000, value=100)
        
        if st.button("Fetch Artifacts"):
            with st.spinner("Fetching data from Harvard API..."):
                artifacts = collect_all_artifacts(max_records=num_records)
                st.success(f"Fetched {len(artifacts)} artifacts")
                st.json(artifacts[0])  # Show sample
    
    elif page == "SQL Analytics":
        st.header("SQL Analytics Queries")
        
        queries = {
            "Artifacts by Culture": query_by_culture,
            "Artifacts by Century": query_by_century,
            "Department Distribution": query_by_department,
            "Color Analysis": query_color_distribution,
            "Media Formats": query_media_formats
        }
        
        selected_query = st.selectbox("Select Analysis", list(queries.keys()))
        
        if st.button("Run Query"):
            df, fig = execute_and_visualize(
                queries[selected_query],
                selected_query
            )
            
            st.dataframe(df)
            if fig:
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Workflow

```python
def run_etl_pipeline(max_records=500):
    """Complete ETL pipeline execution"""
    print("Starting ETL Pipeline...")
    
    # Extract
    print("1. Extracting data from API...")
    artifacts = collect_all_artifacts(max_records)
    print(f"   Extracted {len(artifacts)} artifacts")
    
    # Transform
    print("2. Transforming data...")
    dataframes = transform_artifacts(artifacts)
    print(f"   Metadata: {len(dataframes['metadata'])} records")
    print(f"   Media: {len(dataframes['media'])} records")
    print(f"   Colors: {len(dataframes['colors'])} records")
    
    # Load
    print("3. Loading to database...")
    load_to_database(dataframes)
    print("ETL Pipeline completed successfully!")
    
    return dataframes
```

### Error Handling for API Requests

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(page, size, max_retries=3):
    """Fetch with exponential backoff retry logic"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page, size)
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Request failed, retrying in {wait_time}s...")
            time.sleep(wait_time)
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
```

### Database Connection Problems
```python
# Test database connection
try:
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT 1")
    print("Database connection successful")
    cursor.close()
    conn.close()
except Error as e:
    print(f"Database connection failed: {e}")
```

### Rate Limiting
```python
# Implement rate limiting
import time

def rate_limited_fetch(total_pages, delay=1.0):
    """Fetch with rate limiting"""
    for page in range(1, total_pages + 1):
        data = fetch_artifacts(page=page)
        # Process data
        time.sleep(delay)  # Respect API rate limits
```

### Large Dataset Handling
```python
# Process in batches for large datasets
def batch_load(dataframes, batch_size=100):
    """Load data in batches to avoid memory issues"""
    for table_name, df in dataframes.items():
        for i in range(0, len(df), batch_size):
            batch = df[i:i+batch_size]
            # Load batch to database
            print(f"Loading batch {i//batch_size + 1} for {table_name}")
```

This skill provides complete coverage for building ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit.
