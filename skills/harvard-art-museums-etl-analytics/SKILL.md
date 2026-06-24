---
name: harvard-art-museums-etl-analytics
description: Build end-to-end data pipelines from Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create a data engineering project with museum artifacts
  - set up Harvard Art Museums data analytics dashboard
  - extract and analyze art collection data from Harvard API
  - build a Streamlit app for museum data visualization
  - implement ETL for Harvard Art Museums collection
  - create SQL analytics for art artifacts data
  - design a museum data pipeline with Python
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates how to build a complete data engineering pipeline using the Harvard Art Museums API. It covers API integration, ETL operations, SQL database design, analytics queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Art Museums ETL Analytics App provides:
- **API Data Collection**: Fetch artifact data from Harvard Art Museums API with pagination
- **ETL Pipeline**: Extract, transform, and load nested JSON into relational tables
- **SQL Storage**: Structure data across `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **Analytics Queries**: 20+ predefined SQL queries for artifact insights
- **Visualization**: Interactive Plotly charts via Streamlit interface

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required Dependencies:**
```
streamlit
pandas
requests
pymysql
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

### Get Harvard API Key

1. Visit [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Register for a free API key
3. Add to your `.env` file

### Database Setup

**MySQL/TiDB Schema:**

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    copyright VARCHAR(500),
    INDEX idx_culture (culture),
    INDEX idx_century (century),
    INDEX idx_department (department)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    image_url VARCHAR(1000),
    caption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id),
    INDEX idx_artifact_id (artifact_id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    color_percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id),
    INDEX idx_artifact_id (artifact_id)
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
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        api_key: Harvard API key
        page: Page number (default 1)
        size: Number of records per page (max 100)
    
    Returns:
        dict: JSON response with artifact records
    """
    url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv("HARVARD_API_KEY")
data = fetch_artifacts(api_key, page=1, size=100)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Fetched: {len(data['records'])} artifacts")
```

### 2. ETL Pipeline

```python
import pandas as pd
import pymysql
from typing import List, Dict

def extract_metadata(records: List[Dict]) -> pd.DataFrame:
    """Extract artifact metadata into DataFrame"""
    metadata = []
    for record in records:
        metadata.append({
            'id': record.get('id'),
            'title': record.get('title', '')[:500],
            'culture': record.get('culture', '')[:255],
            'period': record.get('period', '')[:255],
            'century': record.get('century', '')[:100],
            'classification': record.get('classification', '')[:255],
            'department': record.get('department', '')[:255],
            'dated': record.get('dated', '')[:255],
            'technique': record.get('technique', '')[:500],
            'medium': record.get('medium', '')[:500],
            'dimensions': record.get('dimensions', '')[:500],
            'creditline': record.get('creditline', ''),
            'copyright': record.get('copyright', '')[:500]
        })
    return pd.DataFrame(metadata)

def extract_media(records: List[Dict]) -> pd.DataFrame:
    """Extract media/image data into DataFrame"""
    media_data = []
    for record in records:
        artifact_id = record.get('id')
        images = record.get('images', [])
        
        for img in images:
            media_data.append({
                'artifact_id': artifact_id,
                'media_type': 'image',
                'image_url': img.get('baseimageurl', ''),
                'caption': img.get('caption', '')
            })
    return pd.DataFrame(media_data)

def extract_colors(records: List[Dict]) -> pd.DataFrame:
    """Extract color data into DataFrame"""
    color_data = []
    for record in records:
        artifact_id = record.get('id')
        colors = record.get('colors', [])
        
        for color in colors:
            color_data.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', ''),
                'color_name': color.get('color', ''),
                'color_percent': color.get('percent', 0)
            })
    return pd.DataFrame(color_data)
```

### 3. Load Data to SQL

```python
def get_db_connection():
    """Create database connection"""
    return pymysql.connect(
        host=os.getenv("DB_HOST"),
        port=int(os.getenv("DB_PORT", 3306)),
        user=os.getenv("DB_USER"),
        password=os.getenv("DB_PASSWORD"),
        database=os.getenv("DB_NAME"),
        charset='utf8mb4'
    )

def load_metadata(df: pd.DataFrame):
    """Load metadata with INSERT IGNORE for duplicates"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT IGNORE INTO artifactmetadata 
        (id, title, culture, period, century, classification, 
         department, dated, technique, medium, dimensions, 
         creditline, copyright)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    
    for _, row in df.iterrows():
        cursor.execute(insert_query, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()

def load_media(df: pd.DataFrame):
    """Load media data"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia (artifact_id, media_type, image_url, caption)
        VALUES (%s, %s, %s, %s)
    """
    
    for _, row in df.iterrows():
        cursor.execute(insert_query, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()

def load_colors(df: pd.DataFrame):
    """Load color data"""
    conn = get_db_connection()
    cursor = cursor()
    
    insert_query = """
        INSERT INTO artifactcolors (artifact_id, color_hex, color_name, color_percent)
        VALUES (%s, %s, %s, %s)
    """
    
    for _, row in df.iterrows():
        cursor.execute(insert_query, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
```

### 4. Complete ETL Workflow

```python
def run_etl_pipeline(num_pages=5):
    """
    Complete ETL pipeline
    
    Args:
        num_pages: Number of API pages to fetch (100 records each)
    """
    api_key = os.getenv("HARVARD_API_KEY")
    
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}...")
        
        # Extract
        data = fetch_artifacts(api_key, page=page, size=100)
        records = data.get('records', [])
        
        # Transform
        metadata_df = extract_metadata(records)
        media_df = extract_media(records)
        colors_df = extract_colors(records)
        
        # Load
        load_metadata(metadata_df)
        load_media(media_df)
        load_colors(colors_df)
        
        print(f"Page {page} completed: {len(records)} artifacts processed")

# Execute pipeline
if __name__ == "__main__":
    run_etl_pipeline(num_pages=10)
```

## Analytics Queries

### Sample SQL Analytics

```python
ANALYTICS_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century Distribution": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department-wise Collection": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Most Common Color Palettes": """
        SELECT color_name, COUNT(*) as usage_count,
               AVG(color_percent) as avg_percent
        FROM artifactcolors
        WHERE color_name IS NOT NULL
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Artifacts with Most Images": """
        SELECT a.id, a.title, COUNT(m.id) as image_count
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY a.id, a.title
        ORDER BY image_count DESC
        LIMIT 10
    """
}

def execute_query(query: str) -> pd.DataFrame:
    """Execute SQL query and return DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")

st.title("🎨 Harvard Art Museums Data Analytics")

# Sidebar for query selection
query_name = st.sidebar.selectbox(
    "Select Analysis",
    list(ANALYTICS_QUERIES.keys())
)

# Execute query
if st.button("Run Analysis"):
    with st.spinner("Executing query..."):
        query = ANALYTICS_QUERIES[query_name]
        df = execute_query(query)
        
        # Display results
        st.subheader(f"Results: {query_name}")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(
                df.head(20),
                x=df.columns[0],
                y=df.columns[1],
                title=query_name
            )
            st.plotly_chart(fig, use_container_width=True)
```

### ETL Control Panel

```python
st.sidebar.header("ETL Pipeline")

num_pages = st.sidebar.number_input("Pages to Fetch", min_value=1, max_value=50, value=5)

if st.sidebar.button("Run ETL Pipeline"):
    progress_bar = st.progress(0)
    status_text = st.empty()
    
    for page in range(1, num_pages + 1):
        status_text.text(f"Processing page {page}/{num_pages}...")
        
        # Run ETL for this page
        data = fetch_artifacts(os.getenv("HARVARD_API_KEY"), page=page)
        records = data.get('records', [])
        
        metadata_df = extract_metadata(records)
        media_df = extract_media(records)
        colors_df = extract_colors(records)
        
        load_metadata(metadata_df)
        load_media(media_df)
        load_colors(colors_df)
        
        progress_bar.progress(page / num_pages)
    
    st.success(f"ETL Complete! Processed {num_pages * 100} artifacts")
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Batch Processing with Rate Limiting

```python
import time

def fetch_with_rate_limit(api_key, total_pages, delay=1):
    """Fetch data with rate limiting to avoid API throttling"""
    all_records = []
    
    for page in range(1, total_pages + 1):
        data = fetch_artifacts(api_key, page=page)
        all_records.extend(data.get('records', []))
        
        # Rate limit: 1 second between requests
        if page < total_pages:
            time.sleep(delay)
    
    return all_records
```

### Error Handling

```python
def safe_fetch_artifacts(api_key, page, retries=3):
    """Fetch with retry logic"""
    for attempt in range(retries):
        try:
            return fetch_artifacts(api_key, page)
        except requests.exceptions.RequestException as e:
            if attempt == retries - 1:
                raise
            print(f"Retry {attempt + 1}/{retries} after error: {e}")
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

**API Key Issues:**
- Verify `.env` file exists and contains `HARVARD_API_KEY`
- Check API key is active at Harvard Art Museums dashboard

**Database Connection Errors:**
- Confirm MySQL/TiDB is running and accessible
- Verify credentials in `.env` match your database
- Check firewall rules if using cloud database

**Data Loading Failures:**
- Ensure tables are created with correct schema
- Check character encoding (use `utf8mb4` for special characters)
- Monitor for PRIMARY KEY violations on duplicate records

**Streamlit Performance:**
- Cache query results: use `@st.cache_data` decorator
- Limit initial data loads to smaller page counts
- Add pagination for large result sets

**Memory Issues:**
- Process data in smaller batches
- Clear DataFrames after loading: `del df`
- Use database cursors for streaming large results
