---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifact collections
  - fetch and store Harvard museum API data in SQL
  - visualize Harvard Art Museums data with Streamlit
  - build data engineering pipeline for museum artifacts
  - analyze Harvard artifacts collection with SQL queries
  - create museum data warehouse with Python
  - implement ETL for art museum collections API
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It demonstrates ETL pipeline construction, SQL database design, analytical query execution, and interactive visualization using Streamlit. The application extracts artifact metadata, media, and color information, transforms it into relational tables, loads it into MySQL/TiDB Cloud, and provides a dashboard for running 20+ analytical queries with auto-generated visualizations.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

### Prerequisites

```bash
# Python 3.8+ required
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Environment Configuration

Create a `.env` file in the project root:

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

```sql
CREATE DATABASE harvard_artifacts;

USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    classification VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    period VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    creditline TEXT,
    accession_number VARCHAR(100),
    url TEXT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    has_image BOOLEAN,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE
);
```

## Running the Application

```bash
streamlit run app.py
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
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        api_key: Harvard API key from environment
        page: Page number for pagination
        size: Number of records per page (max 100)
    
    Returns:
        JSON response with artifact records
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Pages available: {data['info']['pages']}")
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

def extract_artifact_metadata(records: List[Dict]) -> pd.DataFrame:
    """
    Extract and transform artifact metadata from API response
    """
    metadata = []
    
    for record in records:
        metadata.append({
            'id': record.get('id'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'classification': record.get('classification'),
            'century': record.get('century'),
            'dated': record.get('dated'),
            'department': record.get('department'),
            'division': record.get('division'),
            'period': record.get('period'),
            'technique': record.get('technique'),
            'medium': record.get('medium'),
            'creditline': record.get('creditline'),
            'accession_number': record.get('accessionyear'),
            'url': record.get('url')
        })
    
    return pd.DataFrame(metadata)

def extract_artifact_media(records: List[Dict]) -> pd.DataFrame:
    """
    Extract media/image data from artifacts
    """
    media_data = []
    
    for record in records:
        artifact_id = record.get('id')
        primary_image = record.get('primaryimageurl')
        has_image = 1 if primary_image else 0
        
        media_data.append({
            'artifact_id': artifact_id,
            'image_url': primary_image,
            'has_image': has_image
        })
    
    return pd.DataFrame(media_data)

def extract_artifact_colors(records: List[Dict]) -> pd.DataFrame:
    """
    Extract color information from artifacts
    """
    color_data = []
    
    for record in records:
        artifact_id = record.get('id')
        colors = record.get('colors', [])
        
        for color in colors:
            color_data.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'percentage': color.get('percent')
            })
    
    return pd.DataFrame(color_data)

# Full ETL workflow
def run_etl(api_key: str, num_pages: int = 5):
    """
    Complete ETL process: Extract from API, Transform, Load to database
    """
    all_records = []
    
    # Extract
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        all_records.extend(data.get('records', []))
    
    # Transform
    metadata_df = extract_artifact_metadata(all_records)
    media_df = extract_artifact_media(all_records)
    colors_df = extract_artifact_colors(all_records)
    
    # Load
    load_to_database(metadata_df, media_df, colors_df)
    
    return len(all_records)
```

### 3. Database Loading

```python
def get_db_connection():
    """
    Create database connection using environment variables
    """
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_database(metadata_df, media_df, colors_df):
    """
    Batch insert data into SQL tables with error handling
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Load metadata (parent table first)
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, classification, century, dated, department, 
         division, period, technique, medium, creditline, accession_number, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Load media
        media_query = """
        INSERT INTO artifactmedia (artifact_id, image_url, has_image)
        VALUES (%s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
        
        # Load colors
        colors_query = """
        INSERT INTO artifactcolors (artifact_id, color, percentage)
        VALUES (%s, %s, %s)
        """
        if not colors_df.empty:
            cursor.executemany(colors_query, colors_df.values.tolist())
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Exception as e:
        conn.rollback()
        print(f"Error loading data: {e}")
        raise
    finally:
        cursor.close()
        conn.close()
```

### 4. SQL Analytics Queries

```python
# Sample analytical queries
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
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency, 
               ROUND(AVG(percentage), 2) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Artifacts with Images": """
        SELECT has_image, COUNT(*) as count
        FROM artifactmedia
        GROUP BY has_image
    """,
    
    "Artifacts by Department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Classification Distribution": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Artifacts by Period and Culture": """
        SELECT period, culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE period IS NOT NULL AND culture IS NOT NULL
        GROUP BY period, culture
        ORDER BY count DESC
        LIMIT 20
    """
}

def execute_query(query: str) -> pd.DataFrame:
    """
    Execute SQL query and return results as DataFrame
    """
    conn = get_db_connection()
    try:
        df = pd.read_sql(query, conn)
        return df
    finally:
        conn.close()
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for ETL operations
    with st.sidebar:
        st.header("ETL Operations")
        
        api_key = st.text_input("Harvard API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))
        num_pages = st.slider("Number of pages to fetch", 1, 10, 5)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Running ETL..."):
                try:
                    records_loaded = run_etl(api_key, num_pages)
                    st.success(f"✅ Loaded {records_loaded} artifacts")
                except Exception as e:
                    st.error(f"❌ Error: {e}")
    
    # Main analytics section
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Query", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Execute Query"):
        query = ANALYTICS_QUERIES[query_name]
        
        with st.spinner("Executing query..."):
            df = execute_query(query)
            
            # Display results
            st.subheader("Query Results")
            st.dataframe(df, use_container_width=True)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                st.subheader("Visualization")
                
                # Determine chart type based on data
                x_col = df.columns[0]
                y_col = df.columns[1]
                
                fig = px.bar(df, x=x_col, y=y_col, 
                            title=query_name,
                            labels={x_col: x_col.title(), y_col: y_col.title()})
                
                st.plotly_chart(fig, use_container_width=True)
            
            # Download option
            csv = df.to_csv(index=False)
            st.download_button("Download CSV", csv, 
                             f"{query_name.replace(' ', '_')}.csv")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_artifact_id():
    """Get the highest artifact ID already in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    return result or 0

def incremental_etl(api_key: str):
    """Load only new artifacts since last ETL run"""
    last_id = get_last_artifact_id()
    
    # Fetch only artifacts with ID > last_id
    # Implementation depends on API filtering capabilities
    pass
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_run(api_key: str, num_pages: int):
    """ETL with comprehensive error handling"""
    try:
        logger.info(f"Starting ETL for {num_pages} pages")
        records = run_etl(api_key, num_pages)
        logger.info(f"Successfully loaded {records} records")
        return records
    except requests.RequestException as e:
        logger.error(f"API request failed: {e}")
        raise
    except mysql.connector.Error as e:
        logger.error(f"Database error: {e}")
        raise
    except Exception as e:
        logger.error(f"Unexpected error: {e}")
        raise
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff on rate limit"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            response.raise_for_status()
            return response.json()
        except requests.HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt
                logger.warning(f"Rate limited. Waiting {wait_time}s")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        print("✅ Database connection successful")
        return True
    except Exception as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Missing or Null Data

```python
def clean_artifact_data(df: pd.DataFrame) -> pd.DataFrame:
    """Handle missing values in artifact data"""
    # Replace None with appropriate defaults
    df['culture'] = df['culture'].fillna('Unknown')
    df['century'] = df['century'].fillna('Undated')
    df['title'] = df['title'].fillna('Untitled')
    
    # Truncate long text fields
    df['creditline'] = df['creditline'].str[:1000]
    
    return df
```

## Best Practices

1. **Always use environment variables** for credentials
2. **Implement batch inserts** for better performance (100-1000 records at a time)
3. **Use transactions** when loading related data across multiple tables
4. **Add indexes** on frequently queried columns (culture, century, department)
5. **Cache query results** in Streamlit to avoid redundant database calls
6. **Monitor API usage** to stay within rate limits
7. **Validate data** before insertion to prevent constraint violations
