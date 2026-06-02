---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifact data
  - integrate Harvard Art Museums API with SQL database
  - create a data engineering pipeline with Streamlit dashboard
  - analyze art collection data with SQL queries
  - set up artifact metadata extraction and visualization
  - implement museum data analytics with interactive charts
  - build an art collection data pipeline
  - create a cultural heritage data engineering application
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build, configure, and extend end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL operations, SQL database design, analytical queries, and interactive Streamlit dashboards for cultural artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Collect artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract nested JSON, transform to relational format, load into SQL databases
- **SQL Storage**: Three-table schema (artifactmetadata, artifactmedia, artifactcolors) with foreign keys
- **Analytics**: 20+ predefined SQL queries for cultural data insights
- **Visualization**: Interactive Plotly charts rendered in Streamlit dashboard

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
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

### API Key Setup

Obtain a free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

```python
import mysql.connector
import os

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

connection = mysql.connector.connect(**db_config)
cursor = connection.cursor()
```

## Database Schema

### Create Tables

```sql
-- Artifact Metadata Table
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    accession_number VARCHAR(100),
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE IF NOT EXISTS artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    image_width INT,
    image_height INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE IF NOT EXISTS artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key API Patterns

### Fetch Artifacts with Pagination

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=5, page_size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(BASE_URL, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}: {len(data.get('records', []))} artifacts")
        else:
            print(f"Error on page {page}: {response.status_code}")
            break
        
        time.sleep(0.5)  # Rate limiting
    
    return all_artifacts
```

### Extract and Transform Data

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform raw API data into relational format"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown')[:500],
            'culture': artifact.get('culture', 'Unknown')[:200],
            'century': artifact.get('century', 'Unknown')[:100],
            'classification': artifact.get('classification', 'Unknown')[:200],
            'department': artifact.get('department', 'Unknown')[:200],
            'division': artifact.get('division', 'Unknown')[:200],
            'dated': artifact.get('dated', 'Unknown')[:200],
            'accession_number': artifact.get('accessionNumber', 'Unknown')[:100],
            'url': artifact.get('url', '')[:500]
        }
        metadata_list.append(metadata)
        
        # Extract media
        for image in artifact.get('images', []):
            media = {
                'artifact_id': artifact.get('id'),
                'image_url': image.get('baseimageurl', '')[:1000],
                'image_width': image.get('width'),
                'image_height': image.get('height')
            }
            media_list.append(media)
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_data = {
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex', '')[:10],
                'color_percent': color.get('percent')
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load Data into SQL

```python
def load_to_sql(df_metadata, df_media, df_colors, connection):
    """Batch insert data into SQL database"""
    cursor = connection.cursor()
    
    # Insert metadata
    metadata_query = """
    INSERT IGNORE INTO artifactmetadata 
    (id, title, culture, century, classification, department, division, dated, accession_number, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    cursor.executemany(metadata_query, df_metadata.values.tolist())
    
    # Insert media
    if not df_media.empty:
        media_query = """
        INSERT INTO artifactmedia (artifact_id, image_url, image_width, image_height)
        VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(media_query, df_media.values.tolist())
    
    # Insert colors
    if not df_colors.empty:
        colors_query = """
        INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
        VALUES (%s, %s, %s)
        """
        cursor.executemany(colors_query, df_colors.values.tolist())
    
    connection.commit()
    print(f"Loaded {len(df_metadata)} artifacts, {len(df_media)} images, {len(df_colors)} colors")
```

## Streamlit Application Patterns

### Main App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    else:
        show_visualizations()

if __name__ == "__main__":
    main()
```

### Data Collection Interface

```python
def show_data_collection():
    st.header("📥 Collect Artifact Data")
    
    num_pages = st.slider("Number of pages to fetch", 1, 20, 5)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(API_KEY, num_pages=num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            connection = mysql.connector.connect(**db_config)
            load_to_sql(df_meta, df_media, df_colors, connection)
            connection.close()
            st.success("Data loaded to SQL database")
```

### SQL Analytics Interface

```python
def show_sql_analytics():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Top 10 Cultures by Artifact Count": """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture != 'Unknown'
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century != 'Unknown'
            GROUP BY century
            ORDER BY count DESC
        """,
        "Top Colors Across Collection": """
            SELECT color_hex, COUNT(*) as usage_count
            FROM artifactcolors
            GROUP BY color_hex
            ORDER BY usage_count DESC
            LIMIT 15
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        connection = mysql.connector.connect(**db_config)
        df_result = pd.read_sql(queries[selected_query], connection)
        connection.close()
        
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) == 2:
            fig = px.bar(
                df_result,
                x=df_result.columns[0],
                y=df_result.columns[1],
                title=selected_query
            )
            st.plotly_chart(fig, use_container_width=True)
```

## Advanced Analytics Queries

### Department Analysis

```sql
-- Artifacts with images by department
SELECT 
    department,
    COUNT(DISTINCT am.id) as total_artifacts,
    COUNT(DISTINCT media.artifact_id) as artifacts_with_images,
    ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as image_percentage
FROM artifactmetadata am
LEFT JOIN artifactmedia media ON am.id = media.artifact_id
GROUP BY department
HAVING total_artifacts > 10
ORDER BY total_artifacts DESC;
```

### Color Pattern Analysis

```sql
-- Average color diversity per artifact
SELECT 
    am.culture,
    AVG(color_count) as avg_colors_per_artifact
FROM artifactmetadata am
JOIN (
    SELECT artifact_id, COUNT(*) as color_count
    FROM artifactcolors
    GROUP BY artifact_id
) color_stats ON am.id = color_stats.artifact_id
WHERE am.culture != 'Unknown'
GROUP BY am.culture
HAVING COUNT(*) > 5
ORDER BY avg_colors_per_artifact DESC;
```

## Common Patterns

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, num_pages, db_config):
    """ETL pipeline with comprehensive error handling"""
    try:
        # Extract
        artifacts = fetch_artifacts(api_key, num_pages)
        if not artifacts:
            raise ValueError("No artifacts fetched from API")
        
        # Transform
        df_meta, df_media, df_colors = transform_artifacts(artifacts)
        
        # Load
        connection = mysql.connector.connect(**db_config)
        try:
            load_to_sql(df_meta, df_media, df_colors, connection)
        finally:
            connection.close()
        
        return True, f"Successfully processed {len(artifacts)} artifacts"
    
    except requests.RequestException as e:
        return False, f"API Error: {str(e)}"
    except mysql.connector.Error as e:
        return False, f"Database Error: {str(e)}"
    except Exception as e:
        return False, f"Unexpected Error: {str(e)}"
```

### Incremental Data Loading

```python
def get_max_artifact_id(connection):
    """Get highest artifact ID already in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT COALESCE(MAX(id), 0) FROM artifactmetadata")
    return cursor.fetchone()[0]

def fetch_new_artifacts_only(api_key, connection):
    """Fetch only artifacts newer than what's in database"""
    max_id = get_max_artifact_id(connection)
    
    params = {
        'apikey': api_key,
        'size': 100,
        'sort': 'id',
        'sortorder': 'asc',
        'after': max_id
    }
    
    response = requests.get(BASE_URL, params=params)
    return response.json().get('records', [])
```

## Troubleshooting

### API Rate Limiting

```python
import time
from functools import wraps

def rate_limited(max_per_second=2):
    """Decorator to enforce rate limiting"""
    min_interval = 1.0 / max_per_second
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

@rate_limited(max_per_second=2)
def fetch_single_page(api_key, page):
    # API call logic
    pass
```

### Database Connection Pooling

```python
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    **db_config
)

def get_connection():
    return db_pool.get_connection()
```

### Handling Missing Data

```python
def safe_get(dictionary, key, default='Unknown', max_length=None):
    """Safely extract value from dictionary with type/length validation"""
    value = dictionary.get(key, default)
    if value is None:
        return default
    value = str(value)
    if max_length:
        return value[:max_length]
    return value
```

This skill enables agents to implement robust museum data pipelines with proper ETL practices, SQL optimization, and interactive analytics dashboards.
