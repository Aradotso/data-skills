---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with SQL and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract data from Harvard Art Museums API
  - set up SQL database for artifact collections
  - visualize museum data with Streamlit
  - analyze Harvard Art Museums collection data
  - build data engineering pipeline for cultural artifacts
  - create museum artifact analytics application
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is an end-to-end data engineering solution that:

- Extracts artifact data from the Harvard Art Museums API
- Transforms nested JSON into relational database structures
- Loads data into MySQL/TiDB Cloud databases
- Provides SQL-based analytics queries
- Visualizes insights through interactive Streamlit dashboards

This project demonstrates production-ready ETL patterns, API pagination handling, batch processing, and data visualization for cultural heritage datasets.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
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

### Database Setup

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

# Establish database connection
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=os.getenv('DB_PORT', 3306),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Create tables
def create_tables():
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            dated VARCHAR(200),
            classification VARCHAR(200),
            department VARCHAR(200),
            division VARCHAR(200),
            accession_year INT,
            technique VARCHAR(500),
            medium VARCHAR(500),
            dimensions VARCHAR(500),
            total_page_views INT,
            total_unique_page_views INT
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            image_url VARCHAR(1000),
            base_image_url VARCHAR(1000),
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    conn.commit()
    cursor.close()
    conn.close()

create_tables()
```

## Key API Patterns

### Extract Data from Harvard Art Museums API

```python
import requests
import time
from typing import List, Dict

def fetch_artifacts(api_key: str, size: int = 100, max_pages: int = 10) -> List[Dict]:
    """
    Fetch artifacts with pagination handling
    
    Args:
        api_key: Harvard Art Museums API key
        size: Number of records per page (max 100)
        max_pages: Maximum number of pages to fetch
    
    Returns:
        List of artifact dictionaries
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        params = {
            'apikey': api_key,
            'size': size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            records = data.get('records', [])
            if not records:
                break
                
            all_artifacts.extend(records)
            print(f"Fetched page {page}: {len(records)} artifacts")
            
            # Rate limiting - respect API limits
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts
```

### Transform JSON to Relational Format

```python
import pandas as pd
from typing import List, Dict, Tuple

def transform_artifacts(artifacts: List[Dict]) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
    """
    Transform nested JSON artifacts into relational dataframes
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        
        # Extract metadata
        metadata = {
            'objectid': objectid,
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'period': artifact.get('period', ''),
            'century': artifact.get('century', ''),
            'dated': artifact.get('dated', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'division': artifact.get('division', ''),
            'accession_year': artifact.get('accessioned', 0),
            'technique': artifact.get('technique', ''),
            'medium': artifact.get('medium', ''),
            'dimensions': artifact.get('dimensions', ''),
            'total_page_views': artifact.get('totalpageviews', 0),
            'total_unique_page_views': artifact.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
        
        # Extract media/images
        images = artifact.get('images', [])
        for img in images:
            media = {
                'objectid': objectid,
                'image_url': img.get('imageurl', ''),
                'base_image_url': img.get('baseimageurl', '')
            }
            media_list.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'objectid': objectid,
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load Data into SQL Database

```python
def load_to_database(metadata_df: pd.DataFrame, media_df: pd.DataFrame, colors_df: pd.DataFrame):
    """
    Batch insert dataframes into SQL database
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Insert metadata (batch)
    metadata_values = metadata_df.values.tolist()
    metadata_query = """
        INSERT IGNORE INTO artifactmetadata 
        (objectid, title, culture, period, century, dated, classification, 
         department, division, accession_year, technique, medium, dimensions, 
         total_page_views, total_unique_page_views)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    cursor.executemany(metadata_query, metadata_values)
    
    # Insert media
    if not media_df.empty:
        media_values = media_df.values.tolist()
        media_query = """
            INSERT INTO artifactmedia (objectid, image_url, base_image_url)
            VALUES (%s, %s, %s)
        """
        cursor.executemany(media_query, media_values)
    
    # Insert colors
    if not colors_df.empty:
        colors_values = colors_df.values.tolist()
        colors_query = """
            INSERT INTO artifactcolors (objectid, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(colors_query, colors_values)
    
    conn.commit()
    cursor.close()
    conn.close()
    print(f"Loaded {len(metadata_df)} artifacts to database")
```

## Streamlit Application Structure

### Main App Entry Point

```python
import streamlit as st
from dotenv import load_dotenv
import os

load_dotenv()

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Module",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    else:
        show_visualizations_page()

if __name__ == "__main__":
    main()
```

### ETL Pipeline Page

```python
def show_etl_page():
    st.header("📥 ETL Pipeline")
    
    api_key = os.getenv('HARVARD_API_KEY')
    
    col1, col2 = st.columns(2)
    with col1:
        num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    with col2:
        page_size = st.number_input("Records per page", min_value=10, max_value=100, value=100)
    
    if st.button("🚀 Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            artifacts = fetch_artifacts(api_key, size=page_size, max_pages=num_pages)
            st.success(f"Extracted {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success(f"Transformed into {len(metadata_df)} metadata, {len(media_df)} media, {len(colors_df)} color records")
        
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
            st.success("✅ ETL Pipeline completed successfully!")
        
        # Show sample data
        st.subheader("Sample Metadata")
        st.dataframe(metadata_df.head(10))
```

### Analytics Queries Page

```python
def show_analytics_page():
    st.header("📊 SQL Analytics Dashboard")
    
    queries = {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 15
        """,
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY century
        """,
        "Most Popular Artifacts": """
            SELECT title, culture, total_page_views
            FROM artifactmetadata
            ORDER BY total_page_views DESC
            LIMIT 20
        """,
        "Color Distribution": """
            SELECT color, COUNT(*) as frequency
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
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        conn = get_db_connection()
        df = pd.read_sql(queries[selected_query], conn)
        conn.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            import plotly.express as px
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], title=selected_query)
            st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Full ETL Workflow

```python
from dotenv import load_dotenv
import os

def run_full_etl_pipeline():
    """Complete ETL workflow from API to database"""
    load_dotenv()
    
    # Step 1: Extract
    print("Step 1: Extracting data from Harvard API...")
    api_key = os.getenv('HARVARD_API_KEY')
    artifacts = fetch_artifacts(api_key, size=100, max_pages=10)
    
    # Step 2: Transform
    print("Step 2: Transforming JSON to relational format...")
    metadata_df, media_df, colors_df = transform_artifacts(artifacts)
    
    # Step 3: Load
    print("Step 3: Loading data to SQL database...")
    load_to_database(metadata_df, media_df, colors_df)
    
    print(f"✅ Pipeline complete: {len(metadata_df)} artifacts processed")
    
    return metadata_df, media_df, colors_df
```

### Incremental Data Loading

```python
def get_latest_objectid():
    """Get the highest objectid already in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return result[0] if result[0] else 0

def fetch_new_artifacts_only(api_key: str):
    """Fetch only artifacts newer than what's in database"""
    latest_id = get_latest_objectid()
    
    all_artifacts = []
    page = 1
    
    while True:
        params = {
            'apikey': api_key,
            'size': 100,
            'page': page,
            'sort': 'objectid',
            'sortorder': 'desc'
        }
        
        response = requests.get("https://api.harvardartmuseums.org/object", params=params)
        records = response.json().get('records', [])
        
        new_records = [r for r in records if r.get('objectid', 0) > latest_id]
        if not new_records:
            break
            
        all_artifacts.extend(new_records)
        page += 1
        time.sleep(0.5)
    
    return all_artifacts
```

## Troubleshooting

### API Rate Limiting

```python
import time
from functools import wraps

def rate_limited(max_per_second=2):
    """Decorator to rate limit API calls"""
    min_interval = 1.0 / max_per_second
    
    def decorator(func):
        last_called = [0.0]
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            left_to_wait = min_interval - elapsed
            
            if left_to_wait > 0:
                time.sleep(left_to_wait)
            
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        
        return wrapper
    return decorator

@rate_limited(max_per_second=2)
def fetch_single_artifact(api_key: str, object_id: int):
    """Fetch a single artifact with rate limiting"""
    url = f"https://api.harvardartmuseums.org/object/{object_id}"
    response = requests.get(url, params={'apikey': api_key})
    return response.json()
```

### Database Connection Pooling

```python
from mysql.connector import pooling

# Create connection pool
db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_pooled_connection():
    """Get connection from pool"""
    return db_pool.get_connection()
```

### Handle Missing Data Gracefully

```python
def safe_extract(artifact: Dict, key: str, default=''):
    """Safely extract nested values from artifact JSON"""
    try:
        value = artifact.get(key, default)
        return value if value is not None else default
    except Exception:
        return default

def clean_metadata(metadata_df: pd.DataFrame) -> pd.DataFrame:
    """Clean and validate metadata before loading"""
    # Remove duplicates
    metadata_df = metadata_df.drop_duplicates(subset=['objectid'])
    
    # Handle null values
    metadata_df = metadata_df.fillna('')
    
    # Truncate long strings to fit database constraints
    metadata_df['title'] = metadata_df['title'].str[:500]
    metadata_df['culture'] = metadata_df['culture'].str[:200]
    
    return metadata_df
```

This skill enables AI agents to build complete ETL pipelines for cultural heritage data using the Harvard Art Museums API, implement SQL analytics, and create interactive dashboards with Streamlit.
