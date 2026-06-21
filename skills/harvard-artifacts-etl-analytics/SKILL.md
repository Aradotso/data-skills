---
name: harvard-artifacts-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - set up Harvard artifacts data engineering pipeline
  - create analytics dashboard for museum collection data
  - extract and transform Harvard API artifact data
  - build Streamlit app with Harvard Art Museums API
  - analyze Harvard museum artifacts with SQL
  - visualize Harvard art collection data
  - implement museum data ETL workflow
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI coding agents to help developers build complete data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON into relational SQL tables
- **Database Design**: Structures data into `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Visualization**: Interactive Plotly dashboards built with Streamlit

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Environment Setup

Create a `.env` file in the project root:

```bash
# .env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

Get your Harvard API key at: https://docs.harvardartmuseums.org/

### Database Setup

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE IF NOT EXISTS artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    medium VARCHAR(500),
    technique VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    totalpageviews INT DEFAULT 0,
    totaluniquepageviews INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact media table
CREATE TABLE IF NOT EXISTS artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    mediacount INT DEFAULT 0,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid) ON DELETE CASCADE
);

-- Artifact colors table
CREATE TABLE IF NOT EXISTS artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid) ON DELETE CASCADE
);
```

## Key API Patterns

### Fetching Data from Harvard API

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
        page: Page number (default 1)
        size: Results per page (max 100)
    
    Returns:
        JSON response with artifact records
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=50)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Fetched: {len(data['records'])} artifacts")
```

### Batch Data Collection with Rate Limiting

```python
import time
from typing import List, Dict

def collect_multiple_pages(api_key: str, num_pages: int = 5, delay: float = 0.5) -> List[Dict]:
    """
    Collect artifacts across multiple pages with rate limiting
    
    Args:
        api_key: Harvard API key
        num_pages: Number of pages to fetch
        delay: Seconds to wait between requests
    
    Returns:
        List of all artifact records
    """
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page, size=100)
            all_artifacts.extend(data['records'])
            print(f"✓ Page {page}: {len(data['records'])} artifacts")
            
            if page < num_pages:
                time.sleep(delay)  # Rate limiting
                
        except requests.exceptions.RequestException as e:
            print(f"✗ Error on page {page}: {e}")
            break
    
    return all_artifacts
```

## ETL Pipeline Implementation

### Extract and Transform

```python
import pandas as pd

def transform_artifacts(raw_data: List[Dict]) -> tuple:
    """
    Transform raw API data into three normalized dataframes
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in raw_data:
        # Extract metadata
        metadata = {
            'objectid': record.get('objectid'),
            'title': record.get('title', 'Untitled')[:500],
            'culture': record.get('culture'),
            'century': record.get('century'),
            'dated': record.get('dated'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'division': record.get('division'),
            'medium': record.get('medium', '')[:500],
            'technique': record.get('technique', '')[:500],
            'dimensions': record.get('dimensions', '')[:500],
            'creditline': record.get('creditline'),
            'totalpageviews': record.get('totalpageviews', 0),
            'totaluniquepageviews': record.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
        
        # Extract media data
        media = {
            'objectid': record.get('objectid'),
            'baseimageurl': record.get('baseimageurl'),
            'primaryimageurl': record.get('primaryimageurl'),
            'iiifbaseuri': record.get('images', [{}])[0].get('iiifbaseuri') if record.get('images') else None,
            'mediacount': len(record.get('images', []))
        }
        media_list.append(media)
        
        # Extract color data
        for color_data in record.get('colors', []):
            color = {
                'objectid': record.get('objectid'),
                'color': color_data.get('color'),
                'spectrum': color_data.get('spectrum'),
                'hue': color_data.get('hue'),
                'percent': color_data.get('percent')
            }
            colors_list.append(color)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load to Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection from environment variables"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_database(metadata_df, media_df, colors_df):
    """
    Load transformed dataframes into SQL database
    Uses batch inserts for performance
    """
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        
        # Insert metadata
        metadata_query = """
            INSERT INTO artifactmetadata 
            (objectid, title, culture, century, dated, classification, 
             department, division, medium, technique, dimensions, 
             creditline, totalpageviews, totaluniquepageviews)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        media_query = """
            INSERT INTO artifactmedia 
            (objectid, baseimageurl, primaryimageurl, iiifbaseuri, mediacount)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        if not colors_df.empty:
            colors_query = """
                INSERT INTO artifactcolors 
                (objectid, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """
            cursor.executemany(colors_query, colors_df.values.tolist())
        
        conn.commit()
        print(f"✓ Loaded {len(metadata_df)} artifacts to database")
        
    except Error as e:
        print(f"✗ Database error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

## Streamlit Dashboard Implementation

### Main App Structure

```python
import streamlit as st

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a section",
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
    
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=os.getenv('HARVARD_API_KEY', ''))
    
    num_pages = st.slider("Number of pages to collect", 1, 10, 3)
    
    if st.button("Start Collection"):
        if not api_key:
            st.error("Please provide an API key")
            return
        
        with st.spinner("Collecting data..."):
            # Collect data
            raw_data = collect_multiple_pages(api_key, num_pages)
            
            # Transform
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            
            # Load
            load_to_database(metadata_df, media_df, colors_df)
            
            st.success(f"✓ Collected and loaded {len(metadata_df)} artifacts!")
            
            # Show preview
            st.subheader("Data Preview")
            st.dataframe(metadata_df.head(10))
```

### SQL Analytics Interface

```python
def show_sql_analytics():
    st.header("📊 SQL Analytics")
    
    # Predefined queries
    queries = {
        "Top 10 Cultures by Artifact Count": """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        "Most Popular Colors": """
            SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY color
            ORDER BY frequency DESC
            LIMIT 10
        """,
        "Artifacts by Department": """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """,
        "Media Availability Stats": """
            SELECT 
                CASE WHEN mediacount > 0 THEN 'Has Media' ELSE 'No Media' END as media_status,
                COUNT(*) as count
            FROM artifactmedia
            GROUP BY media_status
        """
    }
    
    query_name = st.selectbox("Select a query", list(queries.keys()))
    
    if st.button("Execute Query"):
        result_df = execute_query(queries[query_name])
        
        if not result_df.empty:
            st.dataframe(result_df)
            
            # Auto-generate visualization
            if len(result_df.columns) == 2:
                import plotly.express as px
                fig = px.bar(result_df, x=result_df.columns[0], y=result_df.columns[1],
                            title=query_name)
                st.plotly_chart(fig, use_container_width=True)

def execute_query(query: str) -> pd.DataFrame:
    """Execute SQL query and return results as DataFrame"""
    try:
        conn = get_db_connection()
        df = pd.read_sql(query, conn)
        conn.close()
        return df
    except Error as e:
        st.error(f"Query error: {e}")
        return pd.DataFrame()
```

## Common Analytical Queries

### Artifact Distribution Analysis

```python
# Top departments with most artifacts
query_departments = """
SELECT department, COUNT(*) as total_artifacts,
       AVG(totalpageviews) as avg_pageviews
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY total_artifacts DESC
LIMIT 15
"""

# Century distribution
query_centuries = """
SELECT century, COUNT(*) as count,
       GROUP_CONCAT(DISTINCT culture SEPARATOR ', ') as cultures
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY century
"""
```

### Color Analysis

```python
# Dominant colors across collections
query_colors = """
SELECT c.spectrum, c.color, 
       COUNT(DISTINCT c.objectid) as artifact_count,
       AVG(c.percent) as avg_coverage,
       m.culture
FROM artifactcolors c
JOIN artifactmetadata m ON c.objectid = m.objectid
WHERE c.percent > 10
GROUP BY c.spectrum, c.color, m.culture
ORDER BY artifact_count DESC
LIMIT 20
"""
```

### Media and Image Analysis

```python
# Artifacts with most images
query_media = """
SELECT m.objectid, a.title, a.culture, m.mediacount
FROM artifactmedia m
JOIN artifactmetadata a ON m.objectid = a.objectid
WHERE m.mediacount > 0
ORDER BY m.mediacount DESC
LIMIT 10
"""
```

## Visualization Patterns

### Interactive Bar Charts

```python
import plotly.express as px

def create_bar_chart(df, x_col, y_col, title):
    """Create interactive Plotly bar chart"""
    fig = px.bar(
        df, 
        x=x_col, 
        y=y_col,
        title=title,
        labels={x_col: x_col.replace('_', ' ').title(), 
                y_col: y_col.replace('_', ' ').title()},
        color=y_col,
        color_continuous_scale='viridis'
    )
    fig.update_layout(xaxis_tickangle=-45)
    return fig
```

### Pie Charts for Distribution

```python
def create_pie_chart(df, names_col, values_col, title):
    """Create pie chart for categorical distribution"""
    fig = px.pie(
        df,
        names=names_col,
        values=values_col,
        title=title,
        hole=0.3  # Donut chart
    )
    return fig
```

## Troubleshooting

### API Rate Limiting

```python
# If you hit rate limits, increase delay between requests
def fetch_with_retry(api_key, page, max_retries=3):
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Too Many Requests
                wait_time = 2 ** attempt  # Exponential backoff
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
# Test database connection
def test_db_connection():
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        print("✓ Database connection successful")
        return True
    except Error as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Handling Missing Data

```python
# Clean and validate data before loading
def clean_artifact_data(df):
    """Handle null values and data type issues"""
    # Fill numeric columns with 0
    numeric_cols = ['totalpageviews', 'totaluniquepageviews']
    df[numeric_cols] = df[numeric_cols].fillna(0)
    
    # Truncate long strings
    string_cols = ['title', 'medium', 'technique', 'dimensions']
    for col in string_cols:
        if col in df.columns:
            df[col] = df[col].astype(str).str[:500]
    
    return df
```

### Memory Optimization for Large Datasets

```python
def process_large_dataset(api_key, total_pages, batch_size=5):
    """Process data in batches to avoid memory issues"""
    for batch_start in range(1, total_pages + 1, batch_size):
        batch_end = min(batch_start + batch_size - 1, total_pages)
        
        # Collect batch
        batch_data = collect_multiple_pages(api_key, batch_end - batch_start + 1)
        
        # Transform and load
        metadata_df, media_df, colors_df = transform_artifacts(batch_data)
        load_to_database(metadata_df, media_df, colors_df)
        
        print(f"✓ Processed batch {batch_start}-{batch_end}")
        
        # Clear memory
        del batch_data, metadata_df, media_df, colors_df
```

This skill provides comprehensive guidance for building complete ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit.
