---
name: harvard-art-museums-etl-pipeline
description: Build end-to-end ETL pipelines for museum artifact data using Harvard Art Museums API with SQL analytics and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering app with Harvard artifacts API
  - set up museum artifact collection analytics with Streamlit
  - process Harvard Art Museums API data into SQL database
  - visualize art museum collection data with interactive dashboards
  - implement artifact metadata ETL with color and media analysis
  - analyze museum artifacts using SQL queries and Python
  - create art collection data pipeline with API integration
---

# Harvard Art Museums ETL Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering solution for extracting, transforming, and analyzing artifact data from the Harvard Art Museums API. It demonstrates real-world ETL pipeline construction, SQL database design, analytical query execution, and interactive Streamlit dashboards for museum collection insights.

## What It Does

- **Data Collection**: Fetches artifact metadata, media files, and color palettes from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into relational database schema (artifacts, media, colors)
- **SQL Analytics**: Executes 20+ analytical queries for culture distribution, century analysis, media availability, and color patterns
- **Interactive Dashboards**: Visualizes query results using Plotly charts in Streamlit interface
- **Database Management**: Supports MySQL and TiDB Cloud for scalable data storage

## Installation

### Prerequisites

```bash
# Python 3.8+ required
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Database Setup

**Option 1: MySQL Local**
```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(255),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    dated VARCHAR(255),
    credit_line TEXT,
    accession_number VARCHAR(100),
    url TEXT
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_image_url TEXT,
    thumbnail_url TEXT,
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);
```

**Option 2: TiDB Cloud**
- Create free cluster at https://tidbcloud.com
- Use same SQL schema above
- Configure connection with provided credentials

### API Key Setup

1. Get your free API key from https://www.harvardartmuseums.org/collections/api
2. Create `.env` file:

```bash
# .env
HARVARD_API_KEY=your_api_key_here
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

## Project Structure

```
harvard-artifacts-app/
├── app.py                 # Main Streamlit application
├── etl_pipeline.py        # ETL logic for data extraction and loading
├── sql_queries.py         # Predefined analytical SQL queries
├── database_config.py     # Database connection management
├── requirements.txt       # Python dependencies
├── .env                   # Environment variables (not committed)
└── README.md
```

## Core Components

### 1. API Data Extraction

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
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Fetch first 100 artifacts
artifacts, info = fetch_artifacts(page=1, size=100)
print(f"Total artifacts available: {info['totalrecords']}")
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform API response to metadata dataframe"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata_list.append({
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'technique': artifact.get('technique', 'Unknown'),
            'medium': artifact.get('medium', 'Unknown'),
            'dimensions': artifact.get('dimensions', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'credit_line': artifact.get('creditline', 'Unknown'),
            'accession_number': artifact.get('accessionyear', 'Unknown'),
            'url': artifact.get('url', 'Unknown')
        })
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """Extract media/image data"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        primary_image = artifact.get('primaryimageurl')
        
        if primary_image:
            media_list.append({
                'artifact_id': artifact_id,
                'base_image_url': primary_image,
                'thumbnail_url': artifact.get('baseimageurl', ''),
                'height': artifact.get('height', 0),
                'width': artifact.get('width', 0)
            })
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """Extract color palette data"""
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_list.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', ''),
                'color_name': color.get('color', ''),
                'percentage': color.get('percent', 0.0)
            })
    
    return pd.DataFrame(color_list)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_database_connection():
    """Create database connection"""
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

def load_metadata(df_metadata):
    """Batch insert artifact metadata"""
    conn = get_database_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (artifact_id, title, culture, century, classification, department, 
         technique, medium, dimensions, dated, credit_line, accession_number, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    data_tuples = [tuple(x) for x in df_metadata.to_numpy()]
    cursor.executemany(insert_query, data_tuples)
    conn.commit()
    
    print(f"Inserted {cursor.rowcount} metadata records")
    cursor.close()
    conn.close()

def load_media(df_media):
    """Insert media records"""
    conn = get_database_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, base_image_url, thumbnail_url, height, width)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    data_tuples = [tuple(x) for x in df_media.to_numpy()]
    cursor.executemany(insert_query, data_tuples)
    conn.commit()
    
    cursor.close()
    conn.close()
```

### 4. Analytical SQL Queries

```python
# sql_queries.py

ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture != 'Unknown'
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century != 'Unknown'
        GROUP BY century
        ORDER BY century
    """,
    
    "Media Availability Analysis": """
        SELECT 
            CASE WHEN m.artifact_id IS NOT NULL THEN 'Has Image' ELSE 'No Image' END as media_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.artifact_id = m.artifact_id
        GROUP BY media_status
    """,
    
    "Top Colors in Collection": """
        SELECT color_name, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        WHERE color_name != ''
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Department": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department != 'Unknown'
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Classification Distribution": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification != 'Unknown'
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Color Diversity by Artifact": """
        SELECT a.title, COUNT(c.color_id) as color_count
        FROM artifactmetadata a
        JOIN artifactcolors c ON a.artifact_id = c.artifact_id
        GROUP BY a.artifact_id, a.title
        ORDER BY color_count DESC
        LIMIT 10
    """
}

def execute_query(query_name):
    """Execute analytical query and return results"""
    conn = get_database_connection()
    cursor = conn.cursor(dictionary=True)
    
    query = ANALYTICAL_QUERIES[query_name]
    cursor.execute(query)
    results = cursor.fetchall()
    
    cursor.close()
    conn.close()
    
    return pd.DataFrame(results)
```

### 5. Streamlit Dashboard

```python
# app.py
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Collection Analytics")

# Sidebar navigation
page = st.sidebar.selectbox(
    "Select Page",
    ["ETL Pipeline", "SQL Analytics", "Data Visualization"]
)

if page == "ETL Pipeline":
    st.header("Extract-Transform-Load Pipeline")
    
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            all_artifacts = []
            for page_num in range(1, num_pages + 1):
                artifacts, info = fetch_artifacts(page=page_num, size=100)
                all_artifacts.extend(artifacts)
                st.write(f"Fetched page {page_num} - Total: {len(all_artifacts)} artifacts")
            
            # Transform
            df_metadata = transform_artifact_metadata(all_artifacts)
            df_media = transform_artifact_media(all_artifacts)
            df_colors = transform_artifact_colors(all_artifacts)
            
            # Load
            load_metadata(df_metadata)
            load_media(df_media)
            load_colors(df_colors)
            
            st.success(f"Successfully loaded {len(all_artifacts)} artifacts!")

elif page == "SQL Analytics":
    st.header("SQL Query Analytics")
    
    query_name = st.selectbox("Select Query", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Execute Query"):
        df_results = execute_query(query_name)
        
        st.subheader("Query Results")
        st.dataframe(df_results)
        
        # Auto-generate chart
        if len(df_results.columns) >= 2:
            x_col = df_results.columns[0]
            y_col = df_results.columns[1]
            
            fig = px.bar(df_results, x=x_col, y=y_col, title=query_name)
            st.plotly_chart(fig, use_container_width=True)

elif page == "Data Visualization":
    st.header("Interactive Dashboards")
    
    col1, col2 = st.columns(2)
    
    with col1:
        df_culture = execute_query("Artifacts by Culture")
        fig1 = px.pie(df_culture, names='culture', values='artifact_count', 
                      title='Collection by Culture')
        st.plotly_chart(fig1, use_container_width=True)
    
    with col2:
        df_colors = execute_query("Top Colors in Collection")
        fig2 = px.bar(df_colors, x='color_name', y='usage_count',
                      title='Most Common Colors')
        st.plotly_chart(fig2, use_container_width=True)
```

## Common Patterns

### Full ETL Workflow

```python
def run_full_etl_pipeline(num_pages=10):
    """Complete ETL pipeline execution"""
    print("Starting ETL pipeline...")
    
    # Extract
    all_artifacts = []
    for page in range(1, num_pages + 1):
        artifacts, _ = fetch_artifacts(page=page, size=100)
        all_artifacts.extend(artifacts)
        time.sleep(0.5)  # Rate limiting
    
    # Transform
    df_metadata = transform_artifact_metadata(all_artifacts)
    df_media = transform_artifact_media(all_artifacts)
    df_colors = transform_artifact_colors(all_artifacts)
    
    # Load
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    
    print(f"ETL complete: {len(all_artifacts)} artifacts processed")
    
    return {
        'metadata_count': len(df_metadata),
        'media_count': len(df_media),
        'color_count': len(df_colors)
    }
```

### Incremental Data Updates

```python
def get_latest_artifact_id():
    """Get the most recent artifact ID in database"""
    conn = get_database_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(artifact_id) FROM artifactmetadata")
    max_id = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    return max_id or 0

def incremental_update():
    """Fetch only new artifacts since last update"""
    last_id = get_latest_artifact_id()
    
    # Fetch new artifacts with filtering
    # Implementation depends on API capabilities
    pass
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
                print(f"Retry {attempt + 1} after {wait_time}s")
                time.sleep(wait_time)
            else:
                raise e
```

### Database Connection Issues
```python
def test_database_connection():
    """Verify database connectivity"""
    try:
        conn = get_database_connection()
        if conn and conn.is_connected():
            print("✓ Database connection successful")
            conn.close()
            return True
    except Error as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Missing Environment Variables
```python
def validate_env_vars():
    """Check required environment variables"""
    required = ['HARVARD_API_KEY', 'DB_HOST', 'DB_USER', 'DB_PASSWORD', 'DB_NAME']
    missing = [var for var in required if not os.getenv(var)]
    
    if missing:
        raise EnvironmentError(f"Missing required environment variables: {', '.join(missing)}")
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Run ETL pipeline standalone
python etl_pipeline.py

# Execute specific analytical query
python -c "from sql_queries import execute_query; print(execute_query('Artifacts by Culture'))"
```

## Key Configuration Options

- **API pagination**: Adjust `size` parameter (max 100 per page)
- **Rate limiting**: Add `time.sleep()` between requests
- **Database batch size**: Modify `executemany()` chunk size for performance
- **Query timeout**: Set `connection_timeout` in MySQL config
- **Image filtering**: Use `hasimage=1` parameter in API calls
