---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL storage, and Streamlit analytics dashboards
triggers:
  - build an ETL pipeline for museum data
  - analyze Harvard Art Museums collection
  - create data engineering pipeline with Streamlit
  - extract and transform art museum API data
  - set up SQL analytics for cultural artifacts
  - visualize museum collection data with Plotly
  - design museum artifact data warehouse
  - implement art collection data pipeline
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

This project provides a complete data engineering and analytics solution for the Harvard Art Museums API. It demonstrates ETL pipeline development, relational database design, SQL analytics, and interactive visualization using Streamlit.

## What This Project Does

- **Data Collection**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Processing**: Transforms nested JSON into normalized relational tables (metadata, media, colors)
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper schema design and foreign keys
- **Analytics**: Executes 20+ predefined SQL queries for insights
- **Visualization**: Renders interactive dashboards with Plotly charts in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Install core dependencies if requirements.txt unavailable
pip install streamlit pandas requests mysql-connector-python plotly
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

```python
# Store in environment variable
export HARVARD_API_KEY="your_api_key_here"

# Or configure in Streamlit app
# The app typically has a sidebar input for API key configuration
```

### Database Configuration

```python
import os
import mysql.connector

# Database connection configuration
db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

# Connect to database
connection = mysql.connector.connect(**db_config)
cursor = connection.cursor()
```

## Key Components

### 1. API Data Extraction

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"Error fetching data: {e}")
        return None

def collect_multiple_pages(api_key, num_pages=5, delay=1):
    """
    Collect data from multiple pages with rate limiting
    """
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        
        if data and 'records' in data:
            all_artifacts.extend(data['records'])
            time.sleep(delay)  # Rate limiting
        else:
            break
    
    return all_artifacts
```

### 2. ETL Pipeline - Transform

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Extract and transform artifact metadata
    """
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'description': artifact.get('description'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline')
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)

def transform_artifact_media(artifacts):
    """
    Extract media/image information
    """
    media_records = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for image in images:
            record = {
                'objectid': objectid,
                'imageid': image.get('imageid'),
                'baseimageurl': image.get('baseimageurl'),
                'width': image.get('width'),
                'height': image.get('height'),
                'iiifbaseuri': image.get('iiifbaseuri')
            }
            media_records.append(record)
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """
    Extract color information
    """
    color_records = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            record = {
                'objectid': objectid,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(record)
    
    return pd.DataFrame(color_records)
```

### 3. Database Schema Setup

```python
def create_database_schema(cursor):
    """
    Create database tables with proper schema
    """
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            century VARCHAR(100),
            dated VARCHAR(255),
            classification VARCHAR(255),
            medium TEXT,
            department VARCHAR(255),
            division VARCHAR(255),
            description TEXT,
            dimensions VARCHAR(500),
            creditline TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            imageid INT,
            baseimageurl TEXT,
            width INT,
            height INT,
            iiifbaseuri TEXT,
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
```

### 4. Load Data to SQL

```python
def load_metadata_to_sql(df, connection, cursor):
    """
    Batch insert metadata into SQL database
    """
    insert_query = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, century, dated, classification, 
         medium, department, division, description, dimensions, creditline)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    # Convert DataFrame to list of tuples
    records = df.to_records(index=False).tolist()
    
    # Batch insert
    cursor.executemany(insert_query, records)
    connection.commit()
    print(f"Inserted {len(records)} metadata records")

def load_media_to_sql(df, connection, cursor):
    """
    Batch insert media data
    """
    insert_query = """
        INSERT INTO artifactmedia 
        (objectid, imageid, baseimageurl, width, height, iiifbaseuri)
        VALUES (%s, %s, %s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    print(f"Inserted {len(records)} media records")
```

### 5. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
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
    
    "Artifacts by Department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN m.objectid IS NOT NULL THEN 'Has Images' ELSE 'No Images' END as status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.objectid = m.objectid
        GROUP BY status
    """,
    
    "Top 10 Color Distributions": """
        SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Average Image Dimensions": """
        SELECT 
            AVG(width) as avg_width,
            AVG(height) as avg_height,
            MAX(width) as max_width,
            MAX(height) as max_height
        FROM artifactmedia
        WHERE width IS NOT NULL AND height IS NOT NULL
    """
}

def execute_analytics_query(cursor, query_name):
    """
    Execute a specific analytics query and return results
    """
    query = ANALYTICS_QUERIES[query_name]
    cursor.execute(query)
    
    # Fetch results and column names
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    
    # Convert to DataFrame
    df = pd.DataFrame(results, columns=columns)
    return df
```

### 6. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_streamlit_app():
    """
    Main Streamlit application
    """
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.write("End-to-End Data Engineering & Analytics Platform")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    db_host = st.sidebar.text_input("Database Host", value="localhost")
    db_user = st.sidebar.text_input("Database User", value="root")
    db_password = st.sidebar.text_input("Database Password", type="password")
    db_name = st.sidebar.text_input("Database Name", value="harvard_artifacts")
    
    # ETL Section
    st.header("1. ETL Pipeline")
    num_pages = st.number_input("Number of Pages to Fetch", min_value=1, max_value=20, value=5)
    
    if st.button("Run ETL Pipeline"):
        if not api_key:
            st.error("Please provide API key")
            return
        
        with st.spinner("Fetching data from API..."):
            artifacts = collect_multiple_pages(api_key, num_pages=num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata = transform_artifact_metadata(artifacts)
            df_media = transform_artifact_media(artifacts)
            df_colors = transform_artifact_colors(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            # Database connection and loading logic here
            st.success("Data loaded to SQL database")
    
    # Analytics Section
    st.header("2. SQL Analytics")
    
    query_name = st.selectbox("Select Analytics Query", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Execute Query"):
        # Execute query and display results
        # df_results = execute_analytics_query(cursor, query_name)
        
        # Display table
        # st.dataframe(df_results)
        
        # Auto-generate visualization
        # if len(df_results.columns) >= 2:
        #     fig = px.bar(df_results, x=df_results.columns[0], y=df_results.columns[1])
        #     st.plotly_chart(fig)
        pass

if __name__ == "__main__":
    create_streamlit_app()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Complete ETL Workflow

```python
import os
import mysql.connector
from datetime import datetime

def run_complete_etl():
    """
    Complete ETL workflow from API to database
    """
    # Configuration
    api_key = os.getenv('HARVARD_API_KEY')
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    # Extract
    print(f"[{datetime.now()}] Starting extraction...")
    artifacts = collect_multiple_pages(api_key, num_pages=10)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    print(f"[{datetime.now()}] Starting transformation...")
    df_metadata = transform_artifact_metadata(artifacts)
    df_media = transform_artifact_media(artifacts)
    df_colors = transform_artifact_colors(artifacts)
    print(f"Transformed: {len(df_metadata)} metadata, {len(df_media)} media, {len(df_colors)} colors")
    
    # Load
    print(f"[{datetime.now()}] Starting load...")
    connection = mysql.connector.connect(**db_config)
    cursor = connection.cursor()
    
    create_database_schema(cursor)
    load_metadata_to_sql(df_metadata, connection, cursor)
    load_media_to_sql(df_media, connection, cursor)
    # Similar for colors
    
    cursor.close()
    connection.close()
    print(f"[{datetime.now()}] ETL complete!")
```

## Troubleshooting

### API Rate Limiting

```python
# Add exponential backoff for rate limits
import time
from requests.exceptions import HTTPError

def fetch_with_retry(api_key, page, max_retries=3):
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except HTTPError as e:
            if e.response.status_code == 429:  # Too Many Requests
                wait_time = (2 ** attempt) * 2
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    return None
```

### Database Connection Issues

```python
# Test database connection
def test_db_connection(db_config):
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        connection.close()
        return True
    except Exception as e:
        print(f"Database connection failed: {e}")
        return False
```

### Data Quality Checks

```python
def validate_etl_data(df_metadata, df_media, df_colors):
    """
    Perform data quality checks
    """
    checks = []
    
    # Check for duplicates
    duplicates = df_metadata['objectid'].duplicated().sum()
    checks.append(f"Duplicate objectids: {duplicates}")
    
    # Check for nulls in critical fields
    null_titles = df_metadata['title'].isnull().sum()
    checks.append(f"Null titles: {null_titles}")
    
    # Check foreign key integrity
    media_orphans = set(df_media['objectid']) - set(df_metadata['objectid'])
    checks.append(f"Orphaned media records: {len(media_orphans)}")
    
    return checks
```

## Best Practices

1. **Always use environment variables** for API keys and database credentials
2. **Implement rate limiting** when fetching from the API
3. **Use batch inserts** for better SQL performance
4. **Validate data** before loading to database
5. **Create indexes** on frequently queried columns (objectid, culture, department)
6. **Log ETL operations** for debugging and monitoring
7. **Handle API errors gracefully** with retries and fallbacks
