---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create a data engineering pipeline with Harvard artifacts API
  - build analytics dashboard for museum collection data
  - extract and analyze Harvard art museum artifacts
  - set up SQL database for art collection metadata
  - visualize museum artifact data with Streamlit
  - query Harvard Art Museums API with pagination
  - create relational database from artifact JSON data
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering pipelines using the Harvard Art Museums API. Extract artifact metadata, transform nested JSON into relational structures, load into SQL databases, and create interactive analytics dashboards with Streamlit and Plotly.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App demonstrates:
- **API Integration**: Fetch paginated artifact data from Harvard Art Museums API
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data
- **SQL Database Design**: Relational schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **Analytics**: Pre-built SQL queries for insights on culture, century, media availability, and color patterns
- **Visualization**: Interactive Plotly charts in a Streamlit dashboard

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

## Key Dependencies

```python
import streamlit as st
import pandas as pd
import requests
import mysql.connector
import plotly.express as px
from typing import List, Dict, Optional
```

## Configuration

### API Setup

Obtain a free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Store credentials securely:

```python
# Use environment variables
import os

API_KEY = os.getenv('HARVARD_API_KEY')
API_BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

```python
# MySQL/TiDB Cloud connection
DB_CONFIG = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'port': int(os.getenv('DB_PORT', 3306))
}

def get_db_connection():
    """Create database connection with error handling"""
    try:
        conn = mysql.connector.connect(**DB_CONFIG)
        return conn
    except mysql.connector.Error as err:
        st.error(f"Database connection failed: {err}")
        return None
```

## Database Schema

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accessionyear INT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    division VARCHAR(200),
    period VARCHAR(200)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(1000),
    iiifbaseuri VARCHAR(1000),
    imagetype VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
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

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
def fetch_artifacts(api_key: str, page: int = 1, size: int = 100) -> Dict:
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        api_key: Your Harvard API key
        page: Page number (default: 1)
        size: Results per page (max: 100)
    
    Returns:
        JSON response with artifact records
    """
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(API_BASE_URL, params=params, timeout=30)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        st.error(f"API request failed: {e}")
        return None

def fetch_all_artifacts(api_key: str, max_records: int = 500) -> List[Dict]:
    """
    Fetch multiple pages of artifacts
    
    Args:
        api_key: Your Harvard API key
        max_records: Maximum number of records to fetch
    
    Returns:
        List of artifact dictionaries
    """
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        data = fetch_artifacts(api_key, page, size)
        if not data or 'records' not in data:
            break
        
        records = data['records']
        if not records:
            break
        
        all_artifacts.extend(records)
        page += 1
        
        # Respect API rate limits
        import time
        time.sleep(0.5)
    
    return all_artifacts[:max_records]
```

### Transform: Clean and Structure Data

```python
def transform_artifact_metadata(artifacts: List[Dict]) -> pd.DataFrame:
    """
    Transform artifact JSON into structured metadata DataFrame
    
    Args:
        artifacts: List of artifact dictionaries from API
    
    Returns:
        Pandas DataFrame with artifact metadata
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'accessionyear': artifact.get('accessionyear'),
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'division': artifact.get('division', '')[:200],
            'period': artifact.get('period', '')[:200]
        })
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts: List[Dict]) -> pd.DataFrame:
    """
    Extract media/image data from artifacts
    
    Args:
        artifacts: List of artifact dictionaries from API
    
    Returns:
        Pandas DataFrame with media URLs
    """
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media_list.append({
                'artifact_id': artifact_id,
                'baseimageurl': img.get('baseimageurl', '')[:1000],
                'iiifbaseuri': img.get('iiifbaseuri', '')[:1000],
                'imagetype': img.get('imagetype', '')[:100]
            })
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts: List[Dict]) -> pd.DataFrame:
    """
    Extract color data from artifacts
    
    Args:
        artifacts: List of artifact dictionaries from API
    
    Returns:
        Pandas DataFrame with color information
    """
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color_obj in colors:
            color_list.append({
                'artifact_id': artifact_id,
                'color': color_obj.get('color', '')[:50],
                'spectrum': color_obj.get('spectrum', '')[:50],
                'hue': color_obj.get('hue', '')[:50],
                'percent': color_obj.get('percent', 0.0)
            })
    
    return pd.DataFrame(color_list)
```

### Load: Insert into Database

```python
def load_metadata_to_db(df: pd.DataFrame, conn) -> int:
    """
    Batch insert artifact metadata into database
    
    Args:
        df: DataFrame with artifact metadata
        conn: MySQL connection object
    
    Returns:
        Number of rows inserted
    """
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, 
     dated, accessionyear, technique, medium, division, period)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    conn.commit()
    
    rows_inserted = cursor.rowcount
    cursor.close()
    return rows_inserted

def load_media_to_db(df: pd.DataFrame, conn) -> int:
    """Insert artifact media records"""
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, baseimageurl, iiifbaseuri, imagetype)
    VALUES (%s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    conn.commit()
    
    rows_inserted = cursor.rowcount
    cursor.close()
    return rows_inserted

def load_colors_to_db(df: pd.DataFrame, conn) -> int:
    """Insert artifact color records"""
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    conn.commit()
    
    rows_inserted = cursor.rowcount
    cursor.close()
    return rows_inserted
```

## Analytics Queries

### Sample SQL Queries

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN m.artifact_id IS NOT NULL THEN 'Has Media' ELSE 'No Media' END as media_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY media_status
    """,
    
    "Top Colors": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Artifacts by Period and Classification": """
        SELECT period, classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE period IS NOT NULL AND classification IS NOT NULL
        GROUP BY period, classification
        ORDER BY count DESC
        LIMIT 20
    """
}

def execute_query(conn, query: str) -> pd.DataFrame:
    """
    Execute SQL query and return results as DataFrame
    
    Args:
        conn: MySQL connection
        query: SQL query string
    
    Returns:
        Query results as DataFrame
    """
    try:
        df = pd.read_sql(query, conn)
        return df
    except Exception as e:
        st.error(f"Query execution failed: {e}")
        return pd.DataFrame()
```

## Streamlit Dashboard

```python
def main():
    """Main Streamlit application"""
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    with st.sidebar:
        st.header("Configuration")
        api_key = st.text_input("Harvard API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))
        
        if st.button("Run ETL Pipeline"):
            if not api_key:
                st.error("Please provide API key")
                return
            
            with st.spinner("Fetching artifacts..."):
                artifacts = fetch_all_artifacts(api_key, max_records=500)
                st.success(f"Fetched {len(artifacts)} artifacts")
            
            with st.spinner("Transforming data..."):
                metadata_df = transform_artifact_metadata(artifacts)
                media_df = transform_artifact_media(artifacts)
                colors_df = transform_artifact_colors(artifacts)
                st.success("Data transformed")
            
            with st.spinner("Loading to database..."):
                conn = get_db_connection()
                if conn:
                    load_metadata_to_db(metadata_df, conn)
                    load_media_to_db(media_df, conn)
                    load_colors_to_db(colors_df, conn)
                    conn.close()
                    st.success("ETL pipeline completed")
    
    # Analytics section
    st.header("📊 Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        conn = get_db_connection()
        if conn:
            query = ANALYTICS_QUERIES[query_name]
            results = execute_query(conn, query)
            conn.close()
            
            st.subheader("Results")
            st.dataframe(results)
            
            # Auto-generate visualization
            if len(results) > 0 and len(results.columns) >= 2:
                fig = px.bar(results, x=results.columns[0], y=results.columns[1],
                           title=query_name)
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Incremental Data Loading

```python
def get_max_artifact_id(conn) -> int:
    """Get the highest artifact ID already in database"""
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def fetch_new_artifacts(api_key: str, last_id: int) -> List[Dict]:
    """Fetch only artifacts newer than last_id"""
    # Implement pagination with filtering logic
    pass
```

### Error Handling and Retry Logic

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(api_key: str, page: int, max_retries: int = 3) -> Dict:
    """Fetch with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            st.warning(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
            time.sleep(wait_time)
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:
```python
# Add delays between requests
import time
time.sleep(1)  # Wait 1 second between API calls

# Reduce batch size
artifacts = fetch_all_artifacts(api_key, max_records=100)
```

### Database Connection Issues

```python
# Test connection
def test_db_connection():
    try:
        conn = mysql.connector.connect(**DB_CONFIG)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        return True
    except mysql.connector.Error as err:
        st.error(f"Connection test failed: {err}")
        return False
```

### Memory Issues with Large Datasets

```python
# Process in chunks
def process_in_chunks(artifacts: List[Dict], chunk_size: int = 100):
    """Process large datasets in smaller batches"""
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        metadata_df = transform_artifact_metadata(chunk)
        # Process chunk...
```

### Missing or Null Data

```python
# Handle missing fields gracefully
def safe_get(dictionary: Dict, key: str, default='', max_len: int = None):
    """Safely extract value with default and length limit"""
    value = dictionary.get(key, default)
    if max_len and value:
        return str(value)[:max_len]
    return value
```

This skill equips AI coding agents to build production-ready ETL pipelines and analytics dashboards using the Harvard Art Museums API, SQL databases, and modern Python data tools.
