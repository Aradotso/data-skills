---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - analyze Harvard Art Museums collection
  - create a data engineering app with Streamlit
  - fetch and transform API data into SQL
  - visualize museum artifacts analytics
  - design a relational database for art collections
  - implement batch data ingestion pipeline
  - query art museum metadata with SQL
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It demonstrates:

- **ETL Pipeline**: Extract data from Harvard Art Museums API, transform nested JSON into relational format, load into SQL database
- **Database Design**: Three-table schema with proper foreign key relationships
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboard**: Streamlit app with Plotly visualizations

**Architecture**: API → ETL → SQL (MySQL/TiDB) → Analytics → Streamlit Visualization

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

**Required Dependencies**:
- `streamlit` - Web app framework
- `pandas` - Data manipulation
- `requests` - API calls
- `mysql-connector-python` or `pymysql` - Database connectivity
- `plotly` - Interactive visualizations

## Database Schema

The project uses a three-table relational schema:

```sql
-- Artifact Metadata (main table)
CREATE TABLE artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accession_number VARCHAR(100),
    object_number VARCHAR(100)
);

-- Artifact Media (images/files)
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    media_type VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);
```

## API Integration

### Fetching Data from Harvard Art Museums API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only get artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestError as e:
        print(f"API request failed: {e}")
        return None

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)

if data:
    total_records = data['info']['totalrecords']
    artifacts = data['records']
    print(f"Total artifacts available: {total_records}")
    print(f"Fetched: {len(artifacts)} artifacts")
```

### Handling Pagination

```python
def fetch_all_artifacts(api_key, max_pages=10):
    """
    Fetch multiple pages of artifacts with rate limiting
    """
    import time
    
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        
        if data and 'records' in data:
            all_artifacts.extend(data['records'])
            time.sleep(0.5)  # Rate limiting
        else:
            break
    
    return all_artifacts
```

## ETL Pipeline

### Extract and Transform

```python
import pandas as pd

def transform_artifact_data(raw_artifacts):
    """
    Transform nested JSON into flat DataFrames for SQL insertion
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata = {
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'accession_number': artifact.get('accessionyear', 'Unknown'),
            'object_number': artifact.get('objectnumber', 'Unknown')
        }
        metadata_records.append(metadata)
        
        # Extract media/images
        if 'images' in artifact and artifact['images']:
            for image in artifact['images']:
                media = {
                    'artifact_id': artifact.get('id'),
                    'image_url': image.get('baseimageurl', ''),
                    'media_type': 'image'
                }
                media_records.append(media)
        
        # Extract colors
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                color_record = {
                    'artifact_id': artifact.get('id'),
                    'color_hex': color.get('hex', ''),
                    'color_percent': color.get('percent', 0.0)
                }
                color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load to Database

```python
import mysql.connector
from mysql.connector import Error

def create_connection():
    """Create database connection using environment variables"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection failed: {e}")
        return None

def batch_insert_metadata(df_metadata):
    """
    Batch insert artifact metadata with error handling
    """
    connection = create_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (artifact_id, title, culture, century, classification, 
     department, dated, accession_number, object_number)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    try:
        data_tuples = [tuple(row) for row in df_metadata.values]
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} records into artifactmetadata")
        return True
    except Error as e:
        print(f"Insert failed: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()

def batch_insert_media(df_media):
    """Batch insert artifact media"""
    connection = create_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, image_url, media_type)
    VALUES (%s, %s, %s)
    """
    
    try:
        data_tuples = [tuple(row) for row in df_media.values]
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} records into artifactmedia")
        return True
    except Error as e:
        print(f"Insert failed: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

## SQL Analytics Queries

### Example Analytical Queries

```python
# Top 10 cultures by artifact count
QUERY_TOP_CULTURES = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture != 'Unknown'
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Artifacts by century distribution
QUERY_CENTURY_DISTRIBUTION = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL AND century != 'Unknown'
GROUP BY century
ORDER BY count DESC;
"""

# Media availability analysis
QUERY_MEDIA_STATS = """
SELECT 
    COUNT(DISTINCT am.artifact_id) as total_artifacts,
    COUNT(DISTINCT media.artifact_id) as artifacts_with_media,
    ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT am.artifact_id), 2) as media_percentage
FROM artifactmetadata am
LEFT JOIN artifactmedia media ON am.artifact_id = media.artifact_id;
"""

# Most common colors across artifacts
QUERY_TOP_COLORS = """
SELECT color_hex, COUNT(*) as usage_count, AVG(color_percent) as avg_percent
FROM artifactcolors
GROUP BY color_hex
ORDER BY usage_count DESC
LIMIT 15;
"""

# Department-wise classification breakdown
QUERY_DEPT_CLASSIFICATION = """
SELECT department, classification, COUNT(*) as count
FROM artifactmetadata
WHERE department != 'Unknown' AND classification != 'Unknown'
GROUP BY department, classification
ORDER BY count DESC
LIMIT 20;
"""
```

### Executing Queries

```python
def execute_query(query):
    """Execute SQL query and return DataFrame"""
    connection = create_connection()
    if not connection:
        return None
    
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query execution failed: {e}")
        return None
    finally:
        connection.close()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Collection Analytics")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    else:
        show_visualizations_page()

def show_data_collection_page():
    """Page for ETL operations"""
    st.header("📥 Data Collection & ETL")
    
    api_key = st.text_input("Harvard API Key", type="password", 
                             value=os.getenv('HARVARD_API_KEY', ''))
    num_pages = st.slider("Number of pages to fetch", 1, 20, 5)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_all_artifacts(api_key, max_pages=num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_meta, df_media, df_colors = transform_artifact_data(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            batch_insert_metadata(df_meta)
            batch_insert_media(df_media)
            batch_insert_colors(df_colors)
            st.success("Data loaded to database")

def show_analytics_page():
    """Page for SQL query execution"""
    st.header("📊 SQL Analytics Dashboard")
    
    queries = {
        "Top 10 Cultures": QUERY_TOP_CULTURES,
        "Century Distribution": QUERY_CENTURY_DISTRIBUTION,
        "Media Statistics": QUERY_MEDIA_STATS,
        "Top Colors": QUERY_TOP_COLORS,
        "Department Classification": QUERY_DEPT_CLASSIFICATION
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        df = execute_query(queries[selected_query])
        
        if df is not None and not df.empty:
            st.dataframe(df, use_container_width=True)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                            title=selected_query)
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

### Visualization Examples

```python
def create_culture_chart(df):
    """Create interactive bar chart for culture distribution"""
    fig = px.bar(
        df, 
        x='culture', 
        y='artifact_count',
        title='Top Cultures by Artifact Count',
        labels={'culture': 'Culture', 'artifact_count': 'Number of Artifacts'},
        color='artifact_count',
        color_continuous_scale='viridis'
    )
    fig.update_layout(xaxis_tickangle=-45)
    return fig

def create_color_chart(df):
    """Create color distribution visualization"""
    df['color_display'] = df['color_hex'].apply(
        lambda x: f'<span style="color:{x}">■</span> {x}'
    )
    
    fig = px.bar(
        df,
        x='color_hex',
        y='usage_count',
        title='Most Common Colors in Collection',
        color='color_hex',
        color_discrete_map={row['color_hex']: row['color_hex'] 
                           for _, row in df.iterrows()}
    )
    return fig
```

## Common Patterns

### Complete ETL Workflow

```python
def run_complete_etl_pipeline(api_key, num_pages=5):
    """
    Complete ETL workflow from API to database
    """
    print("Step 1: Extract data from API")
    raw_artifacts = fetch_all_artifacts(api_key, max_pages=num_pages)
    
    print(f"Step 2: Transform {len(raw_artifacts)} artifacts")
    df_metadata, df_media, df_colors = transform_artifact_data(raw_artifacts)
    
    print("Step 3: Load to database")
    success = (
        batch_insert_metadata(df_metadata) and
        batch_insert_media(df_media) and
        batch_insert_colors(df_colors)
    )
    
    if success:
        print("ETL pipeline completed successfully")
        return True
    else:
        print("ETL pipeline failed")
        return False
```

## Troubleshooting

**API Rate Limiting**:
```python
import time
from functools import wraps

def rate_limited(max_per_second=2):
    """Decorator for rate limiting API calls"""
    min_interval = 1.0 / max_per_second
    
    def decorator(func):
        last_called = [0.0]
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            left_to_wait = min_interval - elapsed
            if left_to_wait > 0:
                time.sleep(left_to_wait)
            ret = func(*args, **kwargs)
            last_called[0] = time.time()
            return ret
        return wrapper
    return decorator

@rate_limited(max_per_second=2)
def fetch_artifacts_rate_limited(api_key, page):
    return fetch_artifacts(api_key, page)
```

**Database Connection Issues**:
```python
def test_database_connection():
    """Test database connectivity"""
    try:
        connection = create_connection()
        if connection and connection.is_connected():
            print("✓ Database connection successful")
            cursor = connection.cursor()
            cursor.execute("SELECT DATABASE();")
            db_name = cursor.fetchone()[0]
            print(f"✓ Connected to database: {db_name}")
            cursor.close()
            connection.close()
            return True
        else:
            print("✗ Database connection failed")
            return False
    except Error as e:
        print(f"✗ Connection error: {e}")
        return False
```

**Handling Missing Data**:
```python
def clean_artifact_data(df):
    """Clean and standardize artifact data"""
    # Replace None/NaN with 'Unknown'
    df = df.fillna('Unknown')
    
    # Truncate long strings
    string_columns = df.select_dtypes(include=['object']).columns
    for col in string_columns:
        df[col] = df[col].astype(str).str[:500]
    
    # Remove duplicates
    df = df.drop_duplicates(subset=['artifact_id'], keep='first')
    
    return df
```
