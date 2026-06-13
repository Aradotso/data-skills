---
name: harvard-art-museums-etl-analytics
description: ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering project with museum artifacts
  - extract and analyze Harvard Art Museums API data
  - set up SQL analytics for art collection data
  - build a Streamlit dashboard for museum artifacts
  - create a data pipeline from Harvard API to database
  - analyze art museum data with SQL queries
  - build an end-to-end data analytics application
---

# Harvard Art Museums ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, and interactive data visualization with Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into relational databases
- **SQL Analytics**: Store structured data in MySQL/TiDB Cloud with proper schema design
- **Interactive Visualization**: Streamlit dashboard with 20+ predefined SQL queries and Plotly charts

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

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Obtaining Harvard API Key

1. Visit [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Register for a free API key
3. Add to `.env` file

## Database Schema

### Table: artifactmetadata

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(255),
    dated VARCHAR(255),
    division VARCHAR(255)
);
```

### Table: artifactmedia

```sql
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(1000),
    iiifbaseuri VARCHAR(1000),
    primaryimageurl VARCHAR(1000),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

### Table: artifactcolors

```sql
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
    
    Args:
        api_key: Harvard API key from environment
        page: Page number for pagination
        size: Number of records per page
    
    Returns:
        JSON response with artifact data
    """
    url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
artifacts = data['records']
total_records = data['info']['totalrecords']
```

### 2. ETL Pipeline with Pagination

```python
import pandas as pd
import time

def extract_all_artifacts(api_key, max_records=1000):
    """
    Extract artifacts with pagination handling
    """
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        try:
            data = fetch_artifacts(api_key, page=page, size=size)
            records = data['records']
            
            if not records:
                break
                
            all_artifacts.extend(records)
            print(f"Fetched page {page}: {len(records)} records")
            
            page += 1
            time.sleep(0.5)  # Rate limiting
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts[:max_records]

def transform_metadata(artifacts):
    """
    Transform artifact data into metadata DataFrame
    """
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'period': artifact.get('period', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'technique': artifact.get('technique', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'division': artifact.get('division', '')[:255]
        })
    
    return pd.DataFrame(metadata)

def transform_media(artifacts):
    """
    Transform media information into DataFrame
    """
    media = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        media.append({
            'artifact_id': artifact_id,
            'baseimageurl': artifact.get('baseimageurl', '')[:1000],
            'iiifbaseuri': artifact.get('iiifbaseuri', '')[:1000],
            'primaryimageurl': artifact.get('primaryimageurl', '')[:1000]
        })
    
    return pd.DataFrame(media)

def transform_colors(artifacts):
    """
    Transform color data from nested JSON
    """
    colors = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        color_list = artifact.get('colors', [])
        
        for color in color_list:
            colors.append({
                'artifact_id': artifact_id,
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'hue': color.get('hue', '')[:50],
                'percent': color.get('percent', 0.0)
            })
    
    return pd.DataFrame(colors)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """
    Create database connection using environment variables
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata(df_metadata, connection):
    """
    Batch insert metadata into database
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, 
         department, technique, dated, division)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    data = df_metadata.to_records(index=False).tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} metadata records")
    cursor.close()

def load_media(df_media, connection):
    """
    Batch insert media data
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, iiifbaseuri, primaryimageurl)
        VALUES (%s, %s, %s, %s)
    """
    
    data = df_media.to_records(index=False).tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} media records")
    cursor.close()

def load_colors(df_colors, connection):
    """
    Batch insert color data
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    data = df_colors.to_records(index=False).tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} color records")
    cursor.close()
```

### 4. Complete ETL Pipeline

```python
def run_etl_pipeline(max_records=1000):
    """
    Execute complete ETL pipeline
    """
    api_key = os.getenv('HARVARD_API_KEY')
    
    # Extract
    print("Extracting data from API...")
    artifacts = extract_all_artifacts(api_key, max_records)
    
    # Transform
    print("Transforming data...")
    df_metadata = transform_metadata(artifacts)
    df_media = transform_media(artifacts)
    df_colors = transform_colors(artifacts)
    
    # Load
    print("Loading data to database...")
    connection = get_db_connection()
    
    if connection:
        load_metadata(df_metadata, connection)
        load_media(df_media, connection)
        load_colors(df_colors, connection)
        connection.close()
        print("ETL pipeline completed successfully!")
    else:
        print("Failed to connect to database")

# Execute pipeline
if __name__ == "__main__":
    run_etl_pipeline(max_records=500)
```

### 5. SQL Analytics Queries

```python
def execute_query(query):
    """
    Execute SQL query and return results as DataFrame
    """
    connection = get_db_connection()
    if connection:
        df = pd.read_sql(query, connection)
        connection.close()
        return df
    return pd.DataFrame()

# Sample Analytics Queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Media Availability": """
        SELECT 
            SUM(CASE WHEN primaryimageurl != '' THEN 1 ELSE 0 END) as with_image,
            SUM(CASE WHEN primaryimageurl = '' THEN 1 ELSE 0 END) as without_image
        FROM artifactmedia
    """,
    
    "Top Colors Used": """
        SELECT color, SUM(percent) as total_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY total_percent DESC
        LIMIT 10
    """,
    
    "Artifacts with Most Colors": """
        SELECT a.id, a.title, COUNT(c.color) as color_count
        FROM artifactmetadata a
        JOIN artifactcolors c ON a.id = c.artifact_id
        GROUP BY a.id, a.title
        ORDER BY color_count DESC
        LIMIT 10
    """
}
```

### 6. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_streamlit_app():
    """
    Create interactive Streamlit dashboard
    """
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    query_name = st.sidebar.selectbox(
        "Select Query",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.sidebar.button("Run Query"):
        with st.spinner("Executing query..."):
            query = ANALYTICS_QUERIES[query_name]
            df = execute_query(query)
            
            if not df.empty:
                st.subheader(f"Results: {query_name}")
                
                # Display table
                st.dataframe(df, use_container_width=True)
                
                # Generate visualization
                if len(df.columns) >= 2:
                    fig = px.bar(
                        df,
                        x=df.columns[0],
                        y=df.columns[1],
                        title=query_name
                    )
                    st.plotly_chart(fig, use_container_width=True)
                
                # Download button
                csv = df.to_csv(index=False)
                st.download_button(
                    label="Download CSV",
                    data=csv,
                    file_name=f"{query_name.replace(' ', '_')}.csv",
                    mime="text/csv"
                )
            else:
                st.warning("No data returned from query")
    
    # ETL Pipeline Section
    st.sidebar.markdown("---")
    st.sidebar.header("ETL Pipeline")
    
    max_records = st.sidebar.number_input(
        "Max Records to Fetch",
        min_value=100,
        max_value=5000,
        value=500,
        step=100
    )
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            run_etl_pipeline(max_records=max_records)
            st.success("ETL pipeline completed!")

# Run the app
if __name__ == "__main__":
    create_streamlit_app()
```

## Running the Application

```bash
# Run Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def get_max_artifact_id(connection):
    """Get the highest artifact ID already in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_etl():
    """Only fetch artifacts newer than what's in database"""
    connection = get_db_connection()
    max_id = get_max_artifact_id(connection)
    
    # Fetch only new artifacts
    url = f"https://api.harvardartmuseums.org/object?apikey={api_key}&id_gt={max_id}"
    # Continue with ETL...
```

### Pattern 2: Error Handling and Logging

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    filename='etl_pipeline.log'
)

def safe_etl_pipeline():
    """ETL with comprehensive error handling"""
    try:
        artifacts = extract_all_artifacts(api_key, max_records=1000)
        logging.info(f"Extracted {len(artifacts)} artifacts")
        
        df_metadata = transform_metadata(artifacts)
        logging.info(f"Transformed {len(df_metadata)} metadata records")
        
        connection = get_db_connection()
        load_metadata(df_metadata, connection)
        logging.info("Successfully loaded data to database")
        
    except requests.RequestException as e:
        logging.error(f"API request failed: {e}")
    except mysql.connector.Error as e:
        logging.error(f"Database error: {e}")
    except Exception as e:
        logging.error(f"Unexpected error: {e}")
    finally:
        if connection and connection.is_connected():
            connection.close()
```

### Pattern 3: Data Quality Checks

```python
def validate_artifacts(df):
    """
    Validate artifact data before loading
    """
    issues = []
    
    # Check for null IDs
    if df['id'].isnull().any():
        issues.append("Found null IDs")
    
    # Check for duplicates
    if df['id'].duplicated().any():
        issues.append("Found duplicate IDs")
    
    # Check title length
    if (df['title'].str.len() > 500).any():
        issues.append("Some titles exceed 500 characters")
    
    if issues:
        print("Data validation issues:")
        for issue in issues:
            print(f"  - {issue}")
        return False
    
    return True
```

## Troubleshooting

### API Rate Limiting

```python
import time
from functools import wraps

def rate_limit(calls_per_second=2):
    """Decorator to limit API calls"""
    min_interval = 1.0 / calls_per_second
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            wait_time = min_interval - elapsed
            if wait_time > 0:
                time.sleep(wait_time)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_second=2)
def fetch_artifacts_limited(api_key, page, size):
    return fetch_artifacts(api_key, page, size)
```

### Database Connection Issues

```python
def get_db_connection_with_retry(max_retries=3):
    """Retry database connection on failure"""
    for attempt in range(max_retries):
        try:
            connection = get_db_connection()
            if connection and connection.is_connected():
                return connection
        except Error as e:
            print(f"Connection attempt {attempt + 1} failed: {e}")
            time.sleep(2 ** attempt)  # Exponential backoff
    
    raise Exception("Failed to connect to database after retries")
```

### Memory Management for Large Datasets

```python
def process_in_chunks(artifacts, chunk_size=100):
    """Process large datasets in chunks"""
    connection = get_db_connection()
    
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        
        df_metadata = transform_metadata(chunk)
        df_media = transform_media(chunk)
        df_colors = transform_colors(chunk)
        
        load_metadata(df_metadata, connection)
        load_media(df_media, connection)
        load_colors(df_colors, connection)
        
        print(f"Processed chunk {i // chunk_size + 1}")
    
    connection.close()
```

This skill provides comprehensive guidance for building ETL pipelines and analytics applications using the Harvard Art Museums API with SQL databases and Streamlit visualization.
