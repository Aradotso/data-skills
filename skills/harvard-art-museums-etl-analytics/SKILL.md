---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build a data pipeline from Harvard Art Museums API
  - create ETL workflow for museum artifact data
  - set up analytics dashboard with Streamlit and SQL
  - extract and transform Harvard museum collection data
  - build data engineering app with Harvard API
  - visualize museum artifact data with Plotly
  - create SQL analytics for art collection data
  - implement batch ETL for museum artifacts
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables building end-to-end data engineering and analytics applications using the Harvard Art Museums API. It covers ETL pipeline development, SQL database design, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App demonstrates:
- Dynamic data collection from Harvard Art Museums API with pagination and rate limiting
- ETL pipeline for transforming nested JSON into relational SQL tables
- SQL analytics with 20+ predefined queries
- Interactive Streamlit dashboards with Plotly visualizations
- Real-world data engineering patterns for museum/collection datasets

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Run the Streamlit app
streamlit run app.py
```

**Required Dependencies:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Store credentials in environment variables or `.env` file:

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

Load environment variables in Python:

```python
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
DB_CONFIG = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}
```

## Core Components

### 1. API Data Collection

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    try:
        response = requests.get(url, params=params)
        response.raise_for_status()
        data = response.json()
        return data
    except requests.exceptions.RequestException as e:
        print(f"Error fetching data: {e}")
        return None

def collect_all_artifacts(api_key, max_pages=10):
    """
    Collect multiple pages of artifact data with rate limiting
    """
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        
        if data and 'records' in data:
            all_artifacts.extend(data['records'])
            time.sleep(1)  # Rate limiting
        else:
            break
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd

def extract_metadata(artifacts):
    """
    Extract artifact metadata into structured format
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'description': artifact.get('description'),
            'accession_year': artifact.get('accessionyear')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def extract_media(artifacts):
    """
    Extract media/image information from artifacts
    """
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media = {
                'artifact_id': artifact_id,
                'image_id': image.get('imageid'),
                'base_image_url': image.get('baseimageurl'),
                'width': image.get('width'),
                'height': image.get('height'),
                'format': image.get('format')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def extract_colors(artifacts):
    """
    Extract color information from artifacts
    """
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percentage': color.get('percent'),
                'hex': color.get('hex')
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### 3. SQL Database Schema

```python
import mysql.connector

def create_database_schema(db_config):
    """
    Create database tables for artifact data
    """
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    # Create artifactmetadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            description TEXT,
            accession_year INT
        )
    """)
    
    # Create artifactmedia table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_id INT,
            base_image_url VARCHAR(500),
            width INT,
            height INT,
            format VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    # Create artifactcolors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            percentage FLOAT,
            hex VARCHAR(10),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    conn.commit()
    cursor.close()
    conn.close()
```

### 4. Batch Data Loading

```python
def batch_insert_metadata(df, db_config, batch_size=100):
    """
    Batch insert artifact metadata into SQL database
    """
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    insert_query = """
        INSERT IGNORE INTO artifactmetadata 
        (artifact_id, title, culture, century, classification, department, dated, description, accession_year)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        values = [tuple(row) for row in batch.values]
        cursor.executemany(insert_query, values)
        conn.commit()
        print(f"Inserted batch {i//batch_size + 1}")
    
    cursor.close()
    conn.close()

def load_all_data(metadata_df, media_df, colors_df, db_config):
    """
    Load all dataframes into SQL database
    """
    batch_insert_metadata(metadata_df, db_config)
    batch_insert_metadata(media_df, db_config)  # Similar function for media
    batch_insert_metadata(colors_df, db_config)  # Similar function for colors
```

### 5. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Top Cultures": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
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
            COUNT(DISTINCT m.artifact_id) as with_images,
            (SELECT COUNT(*) FROM artifactmetadata) as total,
            ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
        FROM artifactmedia m
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 20
    """
}

def run_analytics_query(query, db_config):
    """
    Execute analytics query and return results as DataFrame
    """
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### 6. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Execute selected query
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            df = run_analytics_query(ANALYTICS_QUERIES[query_name], DB_CONFIG)
            
            # Display results
            st.subheader(f"Results: {query_name}")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df.head(15), 
                    x=df.columns[0], 
                    y=df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Complete ETL Workflow

```python
import os
from dotenv import load_dotenv

def complete_etl_pipeline():
    """
    Full ETL pipeline from API to database
    """
    # Load configuration
    load_dotenv()
    api_key = os.getenv('HARVARD_API_KEY')
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    # Step 1: Extract data from API
    print("Extracting data from API...")
    artifacts = collect_all_artifacts(api_key, max_pages=10)
    
    # Step 2: Transform data
    print("Transforming data...")
    metadata_df = extract_metadata(artifacts)
    media_df = extract_media(artifacts)
    colors_df = extract_colors(artifacts)
    
    # Step 3: Load into database
    print("Creating database schema...")
    create_database_schema(db_config)
    
    print("Loading data into database...")
    load_all_data(metadata_df, media_df, colors_df, db_config)
    
    print("ETL pipeline completed successfully!")

# Run the pipeline
if __name__ == "__main__":
    complete_etl_pipeline()
```

## Troubleshooting

### API Rate Limiting
```python
import time
from functools import wraps

def rate_limit(calls_per_second=1):
    """Decorator to rate limit API calls"""
    def decorator(func):
        last_called = [0.0]
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            left_to_wait = 1.0 / calls_per_second - elapsed
            if left_to_wait > 0:
                time.sleep(left_to_wait)
            ret = func(*args, **kwargs)
            last_called[0] = time.time()
            return ret
        return wrapper
    return decorator

@rate_limit(calls_per_second=2)
def fetch_artifacts_safe(api_key, page=1):
    return fetch_artifacts(api_key, page)
```

### Database Connection Errors
```python
def safe_db_connection(db_config, retries=3):
    """Retry database connection with exponential backoff"""
    for attempt in range(retries):
        try:
            conn = mysql.connector.connect(**db_config)
            return conn
        except mysql.connector.Error as e:
            wait_time = 2 ** attempt
            print(f"Connection failed, retrying in {wait_time}s...")
            time.sleep(wait_time)
    raise Exception("Could not connect to database after retries")
```

### Handling Missing Data
```python
def clean_artifact_data(df):
    """Clean and validate artifact data"""
    # Fill missing values
    df['culture'] = df['culture'].fillna('Unknown')
    df['century'] = df['century'].fillna('Unknown')
    
    # Remove duplicates
    df = df.drop_duplicates(subset=['artifact_id'])
    
    # Validate data types
    df['artifact_id'] = pd.to_numeric(df['artifact_id'], errors='coerce')
    df = df.dropna(subset=['artifact_id'])
    
    return df
```

## Common Patterns

### Incremental Data Loading
```python
def get_last_synced_id(db_config):
    """Get the last synced artifact ID"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(artifact_id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return result[0] if result[0] else 0

def incremental_sync(api_key, db_config):
    """Sync only new artifacts"""
    last_id = get_last_synced_id(db_config)
    # Fetch artifacts with ID > last_id
    # Process and load only new data
```

### Data Quality Validation
```python
def validate_etl_results(db_config):
    """Validate data quality after ETL"""
    conn = mysql.connector.connect(**db_config)
    
    checks = {
        'Total artifacts': "SELECT COUNT(*) FROM artifactmetadata",
        'Artifacts with images': "SELECT COUNT(DISTINCT artifact_id) FROM artifactmedia",
        'Artifacts with colors': "SELECT COUNT(DISTINCT artifact_id) FROM artifactcolors",
        'Null titles': "SELECT COUNT(*) FROM artifactmetadata WHERE title IS NULL"
    }
    
    for check_name, query in checks.items():
        cursor = conn.cursor()
        cursor.execute(query)
        result = cursor.fetchone()[0]
        print(f"{check_name}: {result}")
        cursor.close()
    
    conn.close()
```
