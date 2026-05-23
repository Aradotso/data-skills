---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - fetch and analyze Harvard artifacts collection
  - create data engineering pipeline with API integration
  - set up Harvard Art Museums analytics dashboard
  - design ETL workflow for museum artifact data
  - implement SQL analytics for art museum collections
  - visualize Harvard Art Museums data with Streamlit
  - extract transform load Harvard API data
---

# Harvard Artifacts Collection Data Engineering & Analytics App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides a complete data engineering solution for working with the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit. The application extracts artifact metadata, transforms nested JSON into relational tables, loads data into SQL databases, and provides analytics dashboards.

## Architecture

The pipeline follows: **API → ETL → SQL → Analytics → Visualization**

- **Data Source**: Harvard Art Museums API (paginated REST API)
- **ETL**: Python with Requests and Pandas
- **Storage**: MySQL / TiDB Cloud (relational database)
- **Analytics**: SQL queries
- **Visualization**: Streamlit + Plotly

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
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="your_db_name"
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

## Database Schema

The project uses three main tables with relational structure:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    url TEXT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(50),
    baseimageurl TEXT,
    iiifbaseuri TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## API Integration

### Fetching Data from Harvard Art Museums API

```python
import requests
import os
from typing import Dict, List

def fetch_artifacts(api_key: str, page: int = 1, size: int = 100) -> Dict:
    """
    Fetch artifacts from Harvard Art Museums API with pagination.
    
    Args:
        api_key: Harvard API key from environment
        page: Page number (default: 1)
        size: Number of records per page (max: 100)
    
    Returns:
        JSON response containing artifact records
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

# Usage
api_key = os.getenv("HARVARD_API_KEY")
data = fetch_artifacts(api_key, page=1, size=100)

print(f"Total records: {data['info']['totalrecords']}")
print(f"Pages: {data['info']['pages']}")
print(f"Records fetched: {len(data['records'])}")
```

### Handling Pagination

```python
def fetch_all_artifacts(api_key: str, max_records: int = 500) -> List[Dict]:
    """
    Fetch multiple pages of artifacts with rate limiting.
    
    Args:
        api_key: Harvard API key
        max_records: Maximum number of records to fetch
    
    Returns:
        List of all artifact records
    """
    import time
    
    all_records = []
    page = 1
    size = 100
    
    while len(all_records) < max_records:
        try:
            data = fetch_artifacts(api_key, page=page, size=size)
            records = data.get('records', [])
            
            if not records:
                break
            
            all_records.extend(records)
            page += 1
            
            # Rate limiting - be respectful to API
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_records[:max_records]
```

## ETL Pipeline

### Extract: Parse API Response

```python
import pandas as pd

def extract_metadata(records: List[Dict]) -> pd.DataFrame:
    """
    Extract artifact metadata from API records.
    
    Args:
        records: List of artifact records from API
    
    Returns:
        DataFrame with metadata fields
    """
    metadata = []
    
    for record in records:
        metadata.append({
            'id': record.get('id'),
            'title': record.get('title', 'Unknown'),
            'culture': record.get('culture', 'Unknown'),
            'period': record.get('period', 'Unknown'),
            'century': record.get('century', 'Unknown'),
            'classification': record.get('classification', 'Unknown'),
            'department': record.get('department', 'Unknown'),
            'division': record.get('division', 'Unknown'),
            'dated': record.get('dated', 'Unknown'),
            'url': record.get('url', '')
        })
    
    return pd.DataFrame(metadata)
```

### Transform: Process Nested Data

```python
def extract_media(records: List[Dict]) -> pd.DataFrame:
    """
    Extract and flatten media/image data from nested JSON.
    
    Args:
        records: List of artifact records
    
    Returns:
        DataFrame with media information
    """
    media_data = []
    
    for record in records:
        artifact_id = record.get('id')
        images = record.get('images', [])
        
        for image in images:
            media_data.append({
                'artifact_id': artifact_id,
                'media_type': 'image',
                'baseimageurl': image.get('baseimageurl', ''),
                'iiifbaseuri': image.get('iiifbaseuri', '')
            })
    
    return pd.DataFrame(media_data)

def extract_colors(records: List[Dict]) -> pd.DataFrame:
    """
    Extract color information from artifacts.
    
    Args:
        records: List of artifact records
    
    Returns:
        DataFrame with color analysis data
    """
    color_data = []
    
    for record in records:
        artifact_id = record.get('id')
        colors = record.get('colors', [])
        
        for color in colors:
            color_data.append({
                'artifact_id': artifact_id,
                'color': color.get('color', 'Unknown'),
                'spectrum': color.get('spectrum', 'Unknown'),
                'hue': color.get('hue', 'Unknown'),
                'percent': color.get('percent', 0.0)
            })
    
    return pd.DataFrame(color_data)
```

### Load: Batch Insert to SQL

```python
import mysql.connector
from typing import List, Tuple

def get_db_connection():
    """Create database connection using environment variables."""
    return mysql.connector.connect(
        host=os.getenv("DB_HOST"),
        user=os.getenv("DB_USER"),
        password=os.getenv("DB_PASSWORD"),
        database=os.getenv("DB_NAME")
    )

def batch_insert_metadata(df: pd.DataFrame, batch_size: int = 100):
    """
    Batch insert metadata into SQL database.
    
    Args:
        df: DataFrame with metadata
        batch_size: Number of rows per batch
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, division, dated, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        values = [tuple(row) for row in batch.values]
        cursor.executemany(insert_query, values)
        conn.commit()
    
    cursor.close()
    conn.close()

def batch_insert_media(df: pd.DataFrame):
    """Insert media data with foreign key relationships."""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, media_type, baseimageurl, iiifbaseuri)
        VALUES (%s, %s, %s, %s)
    """
    
    values = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, values)
    conn.commit()
    
    cursor.close()
    conn.close()
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Top 10 cultures by artifact count
query_cultures = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture != 'Unknown'
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Artifacts by century
query_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL AND century != 'Unknown'
    GROUP BY century
    ORDER BY century
"""

# Media availability analysis
query_media = """
    SELECT 
        am.department,
        COUNT(DISTINCT am.id) as total_artifacts,
        COUNT(DISTINCT med.artifact_id) as artifacts_with_media,
        ROUND(COUNT(DISTINCT med.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as media_percentage
    FROM artifactmetadata am
    LEFT JOIN artifactmedia med ON am.id = med.artifact_id
    GROUP BY am.department
    ORDER BY media_percentage DESC
"""

# Color distribution analysis
query_colors = """
    SELECT 
        spectrum,
        COUNT(*) as occurrence,
        AVG(percent) as avg_percent
    FROM artifactcolors
    GROUP BY spectrum
    ORDER BY occurrence DESC
"""

def execute_query(query: str) -> pd.DataFrame:
    """Execute SQL query and return results as DataFrame."""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Application

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_analytics()
    else:
        show_visualizations()

def show_data_collection():
    """Page for fetching and loading data."""
    st.header("📥 Data Collection from Harvard API")
    
    api_key = st.text_input("Enter Harvard API Key", type="password", 
                            value=os.getenv("HARVARD_API_KEY", ""))
    
    num_records = st.number_input("Number of records to fetch", 
                                  min_value=10, max_value=1000, value=100)
    
    if st.button("Fetch & Load Data"):
        with st.spinner("Fetching data from API..."):
            records = fetch_all_artifacts(api_key, max_records=num_records)
            
            # ETL process
            metadata_df = extract_metadata(records)
            media_df = extract_media(records)
            colors_df = extract_colors(records)
            
            # Load to database
            batch_insert_metadata(metadata_df)
            batch_insert_media(media_df)
            # Similar for colors
            
            st.success(f"✅ Loaded {len(metadata_df)} artifacts successfully!")
            st.dataframe(metadata_df.head())

def show_analytics():
    """Page for running SQL queries."""
    st.header("📊 SQL Analytics")
    
    queries = {
        "Top Cultures": query_cultures,
        "Artifacts by Century": query_century,
        "Media Availability": query_media,
        "Color Distribution": query_colors
    }
    
    selected = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        df = execute_query(queries[selected])
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1])
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(api_key: str, page: int, max_retries: int = 3):
    """Fetch with exponential backoff retry logic."""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connection and print diagnostics."""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT VERSION()")
        version = cursor.fetchone()
        print(f"✅ Connected to MySQL version: {version[0]}")
        cursor.close()
        conn.close()
        return True
    except Exception as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Memory Optimization for Large Datasets

```python
def chunked_insert(df: pd.DataFrame, chunk_size: int = 1000):
    """Process large DataFrames in chunks to avoid memory issues."""
    for start in range(0, len(df), chunk_size):
        chunk = df.iloc[start:start+chunk_size]
        batch_insert_metadata(chunk)
        print(f"Processed {start + len(chunk)}/{len(df)} records")
```

## Configuration

Create a `.env` file in the project root:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

Load environment variables in Python:

```python
from dotenv import load_dotenv
load_dotenv()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```
