---
name: harvard-artifacts-data-engineering-pipeline
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API with SQL and Streamlit
triggers:
  - how do I fetch artifacts from Harvard Art Museums API
  - help me build an ETL pipeline for museum data
  - create a data engineering project with Harvard API
  - set up SQL analytics for art museum collections
  - build a Streamlit dashboard for artifact data
  - extract and transform Harvard museum artifact data
  - analyze art collection data with SQL queries
  - visualize museum artifact metadata with Plotly
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a production-grade ETL pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational tables, loads it into SQL databases (MySQL/TiDB), and provides interactive analytics dashboards using Streamlit.

## What This Project Does

The Harvard Artifacts Collection app provides:
- **API Integration**: Paginated data collection from Harvard Art Museums API with rate limiting
- **ETL Pipeline**: Extract nested JSON, transform to relational format, batch load to SQL
- **SQL Analytics**: 20+ predefined analytical queries for artifact insights
- **Interactive Dashboards**: Streamlit-based visualization with Plotly charts
- **Database Design**: Normalized schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables

## Installation

### Prerequisites
- Python 3.8+
- MySQL or TiDB Cloud database
- Harvard Art Museums API key (get from https://harvardartmuseums.org/collections/api)

### Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Configure environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_db_host"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

### Required Dependencies

```python
# requirements.txt
streamlit>=1.28.0
pandas>=2.0.0
requests>=2.31.0
mysql-connector-python>=8.0.0
plotly>=5.17.0
python-dotenv>=1.0.0
```

## Database Schema Setup

```sql
-- Create artifactmetadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(500),
    dimensions VARCHAR(500),
    dated VARCHAR(200),
    creditline TEXT,
    accessionyear INT,
    totalpageviews INT DEFAULT 0,
    totaluniquepageviews INT DEFAULT 0
);

-- Create artifactmedia table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    renditionnumber VARCHAR(50),
    format VARCHAR(100),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Create artifactcolors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core ETL Pipeline

### 1. Extract: Fetch Data from API

```python
import requests
import os
from typing import List, Dict

def fetch_artifacts(api_key: str, size: int = 100, pages: int = 5) -> List[Dict]:
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        api_key: Harvard API key from environment
        size: Number of records per page (max 100)
        pages: Number of pages to fetch
    
    Returns:
        List of artifact dictionaries
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, pages + 1):
        params = {
            'apikey': api_key,
            'size': size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}: {len(data.get('records', []))} records")
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return all_artifacts
```

### 2. Transform: Convert to Relational Format

```python
import pandas as pd
from typing import Tuple

def transform_artifacts(artifacts: List[Dict]) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
    """
    Transform nested JSON to relational DataFrames
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'period': artifact.get('period', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'division': artifact.get('division', '')[:200],
            'technique': artifact.get('technique', '')[:500],
            'dimensions': artifact.get('dimensions', '')[:500],
            'dated': artifact.get('dated', '')[:200],
            'creditline': artifact.get('creditline', ''),
            'accessionyear': artifact.get('accessionyear'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_records.append(metadata)
        
        # Extract media (images)
        for image in artifact.get('images', []):
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': image.get('baseimageurl', '')[:500],
                'iiifbaseuri': image.get('iiifbaseuri', '')[:500],
                'renditionnumber': image.get('renditionnumber', '')[:50],
                'format': image.get('format', '')[:100],
                'height': image.get('height'),
                'width': image.get('width')
            }
            media_records.append(media)
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_data = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'hue': color.get('hue', '')[:50],
                'percent': color.get('percent', 0.0)
            }
            color_records.append(color_data)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### 3. Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df: pd.DataFrame, 
                     media_df: pd.DataFrame, 
                     colors_df: pd.DataFrame,
                     db_config: Dict[str, str]) -> bool:
    """
    Batch insert DataFrames into MySQL database
    
    Args:
        metadata_df: Artifact metadata DataFrame
        media_df: Media/images DataFrame
        colors_df: Colors DataFrame
        db_config: Database connection config
    
    Returns:
        True if successful
    """
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Insert metadata (batch insert for performance)
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, classification, department, 
             division, technique, dimensions, dated, creditline, accessionyear, 
             totalpageviews, totaluniquepageviews)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, iiifbaseuri, renditionnumber, format, height, width)
            VALUES (%s, %s, %s, %s, %s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(colors_query, colors_df.values.tolist())
        
        connection.commit()
        print(f"Inserted {len(metadata_df)} artifacts, {len(media_df)} media, {len(colors_df)} colors")
        return True
        
    except Error as e:
        print(f"Database error: {e}")
        return False
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

## Streamlit Analytics Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px
import os

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Database configuration from environment
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page(db_config)
    elif page == "SQL Analytics":
        show_analytics_page(db_config)
    else:
        show_visualizations_page(db_config)

if __name__ == "__main__":
    main()
```

### ETL Pipeline Page

```python
def show_etl_page(db_config: Dict[str, str]):
    st.header("📥 ETL Pipeline - Data Collection")
    
    api_key = os.getenv('HARVARD_API_KEY')
    
    col1, col2 = st.columns(2)
    with col1:
        num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=20, value=5)
    with col2:
        page_size = st.number_input("Records per page", min_value=10, max_value=100, value=100)
    
    if st.button("🚀 Start ETL Pipeline"):
        with st.spinner("Fetching artifacts from API..."):
            artifacts = fetch_artifacts(api_key, size=page_size, pages=num_pages)
            st.success(f"✅ Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success(f"✅ Transformed to {len(metadata_df)} metadata, {len(media_df)} media, {len(colors_df)} color records")
        
        with st.spinner("Loading to database..."):
            success = load_to_database(metadata_df, media_df, colors_df, db_config)
            if success:
                st.success("✅ Data successfully loaded to database!")
            else:
                st.error("❌ Error loading data to database")
```

### SQL Analytics Page

```python
def show_analytics_page(db_config: Dict[str, str]):
    st.header("📊 SQL Analytics Dashboard")
    
    # Predefined analytical queries
    queries = {
        "Top 10 Cultures by Artifact Count": """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        "Artifacts Distribution by Century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        "Most Popular Artifacts (by Page Views)": """
            SELECT title, culture, totalpageviews
            FROM artifactmetadata
            ORDER BY totalpageviews DESC
            LIMIT 15
        """,
        "Department-wise Artifact Count": """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """,
        "Top Colors Across All Artifacts": """
            SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY color
            ORDER BY usage_count DESC
            LIMIT 10
        """,
        "Artifacts with Most Images": """
            SELECT m.title, m.culture, COUNT(i.media_id) as image_count
            FROM artifactmetadata m
            JOIN artifactmedia i ON m.id = i.artifact_id
            GROUP BY m.id, m.title, m.culture
            ORDER BY image_count DESC
            LIMIT 10
        """,
        "Accession Year Distribution": """
            SELECT accessionyear, COUNT(*) as count
            FROM artifactmetadata
            WHERE accessionyear IS NOT NULL
            GROUP BY accessionyear
            ORDER BY accessionyear DESC
            LIMIT 20
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        query_results = execute_query(db_config, queries[selected_query])
        
        if query_results is not None and not query_results.empty:
            st.dataframe(query_results, use_container_width=True)
            
            # Auto-generate visualization
            if len(query_results.columns) >= 2:
                fig = px.bar(
                    query_results, 
                    x=query_results.columns[0], 
                    y=query_results.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)
        else:
            st.warning("No results found")

def execute_query(db_config: Dict[str, str], query: str) -> pd.DataFrame:
    """Execute SQL query and return DataFrame"""
    try:
        connection = mysql.connector.connect(**db_config)
        df = pd.read_sql(query, connection)
        connection.close()
        return df
    except Error as e:
        st.error(f"Query error: {e}")
        return None
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(api_key: str, delay: float = 0.5):
    """Fetch artifacts with rate limiting"""
    artifacts = []
    for page in range(1, 11):
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'page': page, 'size': 100}
        )
        artifacts.extend(response.json().get('records', []))
        time.sleep(delay)  # Respect API rate limits
    return artifacts
```

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key: str, db_config: Dict):
    """ETL with comprehensive error handling"""
    try:
        # Extract
        artifacts = fetch_artifacts(api_key)
        if not artifacts:
            raise ValueError("No artifacts fetched")
        
        # Transform
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        # Validate
        if metadata_df.empty:
            raise ValueError("Metadata transformation failed")
        
        # Load
        success = load_to_database(metadata_df, media_df, colors_df, db_config)
        return success
        
    except requests.RequestException as e:
        print(f"API Error: {e}")
        return False
    except Error as e:
        print(f"Database Error: {e}")
        return False
    except Exception as e:
        print(f"Unexpected Error: {e}")
        return False
```

## Configuration

### Environment Variables Setup

Create a `.env` file:

```bash
# Harvard API
HARVARD_API_KEY=your_harvard_api_key

# Database Configuration
DB_HOST=your_mysql_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts

# Optional: TiDB Cloud Configuration
TIDB_HOST=gateway01.us-west-2.prod.aws.tidbcloud.com
TIDB_PORT=4000
TIDB_USER=your_tidb_user
TIDB_PASSWORD=your_tidb_password
```

Load in Python:

```python
from dotenv import load_dotenv
import os

load_dotenv()

api_key = os.getenv('HARVARD_API_KEY')
db_config = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}
```

## Troubleshooting

### API Connection Issues

```python
# Test API connectivity
def test_api_connection(api_key: str) -> bool:
    """Verify Harvard API is accessible"""
    try:
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'size': 1},
            timeout=10
        )
        return response.status_code == 200
    except requests.RequestException as e:
        print(f"API connection failed: {e}")
        return False
```

### Database Connection Issues

```python
def test_db_connection(db_config: Dict[str, str]) -> bool:
    """Test database connectivity"""
    try:
        connection = mysql.connector.connect(**db_config)
        if connection.is_connected():
            print("Database connection successful")
            connection.close()
            return True
    except Error as e:
        print(f"Database connection failed: {e}")
        return False
```

### Handle Missing Data Gracefully

```python
def safe_get(dictionary: Dict, key: str, default: str = '', max_length: int = None):
    """Safely extract and truncate dictionary values"""
    value = dictionary.get(key, default)
    if value is None:
        return default
    value = str(value)
    if max_length:
        return value[:max_length]
    return value
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Run ETL pipeline programmatically
python etl_pipeline.py

# Run specific analytics queries
python analytics_queries.py --query "top_cultures"
```

This skill enables AI agents to help developers build complete ETL pipelines, design normalized database schemas, execute analytical SQL queries, and create interactive data dashboards using the Harvard Art Museums collection.
