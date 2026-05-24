---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering project with museum artifacts
  - fetch and analyze Harvard Art Museums API data
  - set up a Streamlit analytics dashboard for art collections
  - design SQL schema for artifact metadata
  - implement batch data loading from Harvard API
  - visualize museum collection analytics with Plotly
  - create a data pipeline for art museum artifacts
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering solution for collecting, transforming, and analyzing artifact data from the Harvard Art Museums API. It demonstrates production-grade ETL patterns, relational database design, SQL analytics, and interactive visualization using Streamlit.

## What It Does

- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables
- **Loads** data into MySQL/TiDB Cloud with batch optimization
- **Analyzes** using 20+ predefined SQL queries
- **Visualizes** results through interactive Plotly charts in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY=your_api_key_here
export DB_HOST=your_db_host
export DB_USER=your_db_user
export DB_PASSWORD=your_db_password
export DB_NAME=harvard_artifacts
```

Required packages:
- `streamlit` - Web application framework
- `pandas` - Data manipulation
- `requests` - API calls
- `mysql-connector-python` or `pymysql` - Database connectivity
- `plotly` - Interactive visualizations

## Database Setup

### Schema Design

The project uses three main tables with proper foreign key relationships:

```sql
-- Artifact Metadata (main table)
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    accession_number VARCHAR(100),
    dated VARCHAR(200),
    url TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact Media (images and media files)
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_url TEXT,
    alt_text TEXT,
    caption TEXT,
    iiif_base_url TEXT,
    media_type VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors (color analysis data)
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(7),
    color_name VARCHAR(100),
    percentage FLOAT,
    hue VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## API Integration

### Fetching Data from Harvard Art Museums

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key from environment
        page: Page number (pagination)
        size: Number of records per page (max 100)
    
    Returns:
        dict: JSON response with artifact data
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)

print(f"Total records: {data['info']['totalrecords']}")
print(f"Pages: {data['info']['pages']}")
```

### Pagination Handler

```python
import time

def fetch_all_artifacts(api_key, max_pages=10, delay=1):
    """
    Fetch multiple pages of artifacts with rate limiting
    
    Args:
        api_key: Harvard API key
        max_pages: Maximum number of pages to fetch
        delay: Delay between requests (seconds)
    
    Returns:
        list: All artifact records
    """
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        
        data = fetch_artifacts(api_key, page=page, size=100)
        artifacts = data.get('records', [])
        all_artifacts.extend(artifacts)
        
        # Rate limiting
        if page < max_pages:
            time.sleep(delay)
    
    return all_artifacts
```

## ETL Pipeline Implementation

### Extract and Transform

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform raw API data into structured metadata
    
    Args:
        artifacts: List of artifact records from API
    
    Returns:
        pd.DataFrame: Normalized metadata
    """
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'accession_number': artifact.get('accessionyear'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url')
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)

def transform_artifact_media(artifacts):
    """
    Extract media information from nested JSON
    
    Args:
        artifacts: List of artifact records
    
    Returns:
        pd.DataFrame: Media records with artifact_id foreign key
    """
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            record = {
                'artifact_id': artifact_id,
                'base_url': image.get('baseimageurl'),
                'alt_text': image.get('alttext'),
                'caption': image.get('caption'),
                'iiif_base_url': image.get('iiifbaseuri'),
                'media_type': 'image'
            }
            media_records.append(record)
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """
    Extract color analysis data
    
    Args:
        artifacts: List of artifact records
    
    Returns:
        pd.DataFrame: Color records with artifact_id foreign key
    """
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            record = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_name': color.get('color'),
                'percentage': color.get('percent'),
                'hue': color.get('hue')
            }
            color_records.append(record)
    
    return pd.DataFrame(color_records)
```

### Load to Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """
    Create database connection using environment variables
    
    Returns:
        mysql.connector.connection: Database connection
    """
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def batch_insert_metadata(df, batch_size=100):
    """
    Insert artifact metadata in batches for performance
    
    Args:
        df: DataFrame with metadata
        batch_size: Number of records per batch
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, 
         department, accession_number, dated, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        values = [tuple(row) for row in batch.values]
        
        cursor.executemany(insert_query, values)
        conn.commit()
        print(f"Inserted batch {i//batch_size + 1}")
    
    cursor.close()
    conn.close()

def batch_insert_media(df):
    """Insert media records"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, base_url, alt_text, caption, iiif_base_url, media_type)
        VALUES (%s, %s, %s, %s, %s, %s)
    """
    
    values = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, values)
    conn.commit()
    
    cursor.close()
    conn.close()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics")
    st.markdown("End-to-end ETL Pipeline and Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        data_collection_page()
    elif page == "SQL Analytics":
        analytics_page()
    elif page == "Visualizations":
        visualization_page()

def data_collection_page():
    st.header("📥 Data Collection")
    
    api_key = st.text_input("Harvard API Key", type="password")
    num_pages = st.number_input("Number of pages", min_value=1, max_value=50, value=5)
    
    if st.button("Fetch and Load Data"):
        with st.spinner("Fetching data..."):
            artifacts = fetch_all_artifacts(api_key, max_pages=num_pages)
            
            # Transform
            metadata_df = transform_artifact_metadata(artifacts)
            media_df = transform_artifact_media(artifacts)
            colors_df = transform_artifact_colors(artifacts)
            
            # Load
            batch_insert_metadata(metadata_df)
            batch_insert_media(media_df)
            
            st.success(f"Loaded {len(metadata_df)} artifacts!")
            st.dataframe(metadata_df.head())

if __name__ == "__main__":
    main()
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
def get_artifacts_by_culture():
    """Count artifacts by culture"""
    query = """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """
    return execute_query(query)

def get_media_availability():
    """Analyze media availability per artifact"""
    query = """
        SELECT 
            am.classification,
            COUNT(DISTINCT am.id) as total_artifacts,
            COUNT(DISTINCT med.artifact_id) as artifacts_with_media,
            ROUND(COUNT(DISTINCT med.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as media_percentage
        FROM artifactmetadata am
        LEFT JOIN artifactmedia med ON am.id = med.artifact_id
        GROUP BY am.classification
        ORDER BY media_percentage DESC
    """
    return execute_query(query)

def get_color_distribution():
    """Analyze color usage across artifacts"""
    query = """
        SELECT 
            color_name,
            COUNT(*) as usage_count,
            AVG(percentage) as avg_percentage
        FROM artifactcolors
        WHERE color_name IS NOT NULL
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 10
    """
    return execute_query(query)

def get_artifacts_by_century():
    """Distribution of artifacts by century"""
    query = """
        SELECT 
            century,
            COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY century
    """
    return execute_query(query)

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### Analytics Dashboard

```python
def analytics_page():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Artifacts by Culture": get_artifacts_by_culture,
        "Media Availability": get_media_availability,
        "Color Distribution": get_color_distribution,
        "Artifacts by Century": get_artifacts_by_century
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        df = queries[selected_query]()
        
        st.subheader("Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(
                df, 
                x=df.columns[0], 
                y=df.columns[1],
                title=selected_query
            )
            st.plotly_chart(fig, use_container_width=True)
```

## Configuration

### Environment Variables

Create a `.env` file:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306

# Optional: Rate Limiting
API_DELAY_SECONDS=1
MAX_PAGES_PER_REQUEST=10
```

Load in Python:

```python
from dotenv import load_dotenv
load_dotenv()
```

## Common Patterns

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, max_pages):
    """ETL with error handling and logging"""
    try:
        # Extract
        artifacts = fetch_all_artifacts(api_key, max_pages)
        
        if not artifacts:
            raise ValueError("No artifacts fetched")
        
        # Transform
        metadata_df = transform_artifact_metadata(artifacts)
        media_df = transform_artifact_media(artifacts)
        
        # Validate
        if metadata_df.empty:
            raise ValueError("No metadata to load")
        
        # Load
        batch_insert_metadata(metadata_df)
        batch_insert_media(media_df)
        
        return {
            'success': True,
            'records_loaded': len(metadata_df)
        }
        
    except Exception as e:
        return {
            'success': False,
            'error': str(e)
        }
```

### Incremental Loading

```python
def get_last_artifact_id():
    """Get the last loaded artifact ID"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return result[0] if result[0] else 0

def incremental_load(api_key):
    """Only load new artifacts"""
    last_id = get_last_artifact_id()
    
    # Fetch all artifacts
    all_artifacts = fetch_all_artifacts(api_key, max_pages=5)
    
    # Filter new ones
    new_artifacts = [a for a in all_artifacts if a['id'] > last_id]
    
    if new_artifacts:
        metadata_df = transform_artifact_metadata(new_artifacts)
        batch_insert_metadata(metadata_df)
        print(f"Loaded {len(new_artifacts)} new artifacts")
    else:
        print("No new artifacts to load")
```

## Troubleshooting

### API Rate Limiting

If you encounter `429 Too Many Requests`:

```python
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def get_session_with_retries():
    """Create session with automatic retries"""
    session = requests.Session()
    retry = Retry(
        total=5,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504]
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    return session
```

### Database Connection Issues

```python
def test_db_connection():
    """Verify database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        print("✓ Database connection successful")
        return True
    except Error as e:
        print(f"✗ Database error: {e}")
        return False
```

### Memory Issues with Large Datasets

```python
def chunked_etl(api_key, total_pages, chunk_size=5):
    """Process data in chunks to avoid memory issues"""
    for start_page in range(1, total_pages, chunk_size):
        end_page = min(start_page + chunk_size, total_pages)
        
        print(f"Processing pages {start_page} to {end_page}")
        
        artifacts = fetch_all_artifacts(
            api_key, 
            max_pages=chunk_size
        )
        
        # Process chunk
        metadata_df = transform_artifact_metadata(artifacts)
        batch_insert_metadata(metadata_df)
        
        # Clear memory
        del artifacts
        del metadata_df
```

### Missing Data Handling

```python
def safe_transform(artifacts):
    """Handle missing or malformed data gracefully"""
    metadata_records = []
    
    for artifact in artifacts:
        try:
            record = {
                'id': artifact.get('id', 0),
                'title': artifact.get('title') or 'Untitled',
                'culture': artifact.get('culture') or 'Unknown',
                'century': artifact.get('century') or 'Unknown',
                'classification': artifact.get('classification') or 'Unclassified'
            }
            metadata_records.append(record)
        except Exception as e:
            print(f"Skipping artifact due to error: {e}")
            continue
    
    return pd.DataFrame(metadata_records)
```

This skill provides everything needed to build, deploy, and extend the Harvard Art Museums ETL analytics application with production-ready patterns.
