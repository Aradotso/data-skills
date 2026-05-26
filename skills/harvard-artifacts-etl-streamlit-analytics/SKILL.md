---
name: harvard-artifacts-etl-streamlit-analytics
description: Build end-to-end ETL pipelines with Harvard Art Museums API, SQL analytics, and Streamlit visualization dashboards
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create a streamlit dashboard for art museum data
  - query Harvard artifacts API and store in SQL database
  - build data engineering pipeline for museum collections
  - visualize Harvard Art Museums data with plotly
  - extract transform load Harvard museum artifacts
  - create analytics dashboard for art collections data
  - setup ETL workflow for museum API data
---

# Harvard Artifacts ETL & Streamlit Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application that extracts artifact data from the Harvard Art Museums API, transforms it through ETL pipelines, stores it in SQL databases, and visualizes insights using Streamlit dashboards. It demonstrates real-world patterns for API integration, batch data processing, relational database design, and interactive analytics.

## What It Does

- **API Integration**: Fetches artifact metadata, media, and color data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables
- **SQL Storage**: Stores data in MySQL/TiDB with proper foreign key relationships
- **Analytics Engine**: Executes 20+ predefined SQL queries for data insights
- **Visualization**: Interactive Streamlit dashboard with Plotly charts

**Architecture**: `API → ETL → SQL → Analytics → Visualization`

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

Create a `.env` file or set environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

```python
import os
import mysql.connector
from dotenv import load_dotenv

load_dotenv()

# Connection configuration
db_config = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

# Create database connection
conn = mysql.connector.connect(**db_config)
cursor = conn.cursor()

# Create tables
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    accessionyear INT,
    totalpageviews INT,
    totaluniquepageviews INT
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmedia (
    mediaid INT PRIMARY KEY,
    artifactid INT,
    baseimageurl VARCHAR(1000),
    format VARCHAR(100),
    height INT,
    width INT,
    FOREIGN KEY (artifactid) REFERENCES artifactmetadata(id)
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    artifactid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(100),
    percent FLOAT,
    FOREIGN KEY (artifactid) REFERENCES artifactmetadata(id)
)
""")

conn.commit()
conn.close()
```

## API Integration

### Fetching Artifacts with Pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(size=100, page=1):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    params = {
        'apikey': API_KEY,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(BASE_URL, params=params)
    response.raise_for_status()
    
    data = response.json()
    return data['records'], data['info']

# Fetch multiple pages
def fetch_all_artifacts(max_records=500):
    """Fetch artifacts across multiple pages"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        records, info = fetch_artifacts(size=size, page=page)
        all_artifacts.extend(records)
        
        # Check if more pages available
        if info['next'] is None:
            break
        
        page += 1
        
        # Rate limiting (Harvard API allows ~2500 requests/day)
        import time
        time.sleep(0.5)
    
    return all_artifacts[:max_records]
```

### Extract Specific Artifact Details

```python
def get_artifact_by_id(artifact_id):
    """Fetch specific artifact by ID"""
    url = f"{BASE_URL}/{artifact_id}"
    params = {'apikey': API_KEY}
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    
    return response.json()
```

## ETL Pipeline

### Transform Nested JSON to Relational Format

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform artifact JSON into normalized dataframes"""
    
    # Metadata table
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'accessionyear': artifact.get('accessionyear'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        })
        
        # Extract media information
        if artifact.get('images'):
            for image in artifact['images']:
                media_records.append({
                    'mediaid': image.get('imageid'),
                    'artifactid': artifact.get('id'),
                    'baseimageurl': image.get('baseimageurl', ''),
                    'format': image.get('format', ''),
                    'height': image.get('height'),
                    'width': image.get('width')
                })
        
        # Extract color information
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_records.append({
                    'artifactid': artifact.get('id'),
                    'color': color.get('color', ''),
                    'spectrum': color.get('spectrum', ''),
                    'hue': color.get('hue', ''),
                    'percent': color.get('percent', 0.0)
                })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load Data into SQL

```python
def load_to_sql(df_metadata, df_media, df_colors, db_config):
    """Batch insert dataframes into SQL database"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    try:
        # Insert metadata (parent table first)
        for _, row in df_metadata.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (id, title, culture, century, classification, department, accessionyear, totalpageviews, totaluniquepageviews)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE
                    title=VALUES(title),
                    culture=VALUES(culture),
                    century=VALUES(century)
            """, tuple(row))
        
        # Insert media
        for _, row in df_media.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (mediaid, artifactid, baseimageurl, format, height, width)
                VALUES (%s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE
                    baseimageurl=VALUES(baseimageurl)
            """, tuple(row))
        
        # Insert colors
        for _, row in df_colors.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (artifactid, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
        
        conn.commit()
        print(f"✓ Loaded {len(df_metadata)} artifacts, {len(df_media)} media, {len(df_colors)} colors")
        
    except Exception as e:
        conn.rollback()
        raise e
    finally:
        cursor.close()
        conn.close()
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar for configuration
with st.sidebar:
    st.header("Configuration")
    
    # API section
    st.subheader("API Data Collection")
    num_artifacts = st.number_input("Number of artifacts to fetch", 10, 1000, 100)
    
    if st.button("Fetch & Load Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_all_artifacts(max_records=num_artifacts)
            df_metadata, df_media, df_colors = transform_artifacts(artifacts)
            load_to_sql(df_metadata, df_media, df_colors, db_config)
            st.success(f"✓ Loaded {len(artifacts)} artifacts")

# Main content area
tab1, tab2, tab3 = st.tabs(["📊 Analytics", "🔍 Query Explorer", "📈 Visualizations"])

with tab1:
    st.header("Key Metrics")
    
    # Execute summary query
    conn = mysql.connector.connect(**db_config)
    
    col1, col2, col3 = st.columns(3)
    
    with col1:
        total_artifacts = pd.read_sql("SELECT COUNT(*) as count FROM artifactmetadata", conn)
        st.metric("Total Artifacts", total_artifacts['count'][0])
    
    with col2:
        total_cultures = pd.read_sql("SELECT COUNT(DISTINCT culture) as count FROM artifactmetadata WHERE culture IS NOT NULL", conn)
        st.metric("Unique Cultures", total_cultures['count'][0])
    
    with col3:
        total_media = pd.read_sql("SELECT COUNT(*) as count FROM artifactmedia", conn)
        st.metric("Media Items", total_media['count'][0])
    
    conn.close()
```

### Predefined Analytics Queries

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
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century IS NOT NULL 
        GROUP BY century 
        ORDER BY count DESC
    """,
    
    "Most Viewed Artifacts": """
        SELECT title, culture, totalpageviews 
        FROM artifactmetadata 
        WHERE totalpageviews > 0 
        ORDER BY totalpageviews DESC 
        LIMIT 20
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent 
        FROM artifactcolors 
        GROUP BY color 
        ORDER BY frequency DESC 
        LIMIT 15
    """,
    
    "Department Analysis": """
        SELECT department, COUNT(*) as count, AVG(totalpageviews) as avg_views
        FROM artifactmetadata 
        WHERE department IS NOT NULL 
        GROUP BY department 
        ORDER BY count DESC
    """,
    
    "Artifacts with Multiple Images": """
        SELECT a.id, a.title, COUNT(m.mediaid) as image_count
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.id = m.artifactid
        GROUP BY a.id, a.title
        HAVING image_count > 1
        ORDER BY image_count DESC
        LIMIT 20
    """
}

# Query executor in Streamlit
with tab2:
    st.header("SQL Query Explorer")
    
    query_name = st.selectbox("Select Query", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Execute Query"):
        conn = mysql.connector.connect(**db_config)
        
        query = ANALYTICS_QUERIES[query_name]
        df_result = pd.read_sql(query, conn)
        
        st.subheader("Query Results")
        st.dataframe(df_result, use_container_width=True)
        
        # Auto-generate visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)
        
        conn.close()
```

## Common Patterns

### ETL Workflow Orchestration

```python
def run_etl_pipeline(num_records=100):
    """Complete ETL workflow"""
    
    # Extract
    st.write("📥 Extracting data from API...")
    artifacts = fetch_all_artifacts(max_records=num_records)
    
    # Transform
    st.write("🔄 Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(artifacts)
    
    # Load
    st.write("💾 Loading to database...")
    load_to_sql(df_metadata, df_media, df_colors, db_config)
    
    st.success("✅ ETL Pipeline Complete!")
    
    return {
        'artifacts': len(df_metadata),
        'media': len(df_media),
        'colors': len(df_colors)
    }
```

### Incremental Data Updates

```python
def get_latest_artifact_id(db_config):
    """Get the most recent artifact ID in database"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    conn.close()
    return result[0] if result[0] else 0

def incremental_load(db_config):
    """Load only new artifacts"""
    latest_id = get_latest_artifact_id(db_config)
    
    # Fetch artifacts with ID > latest_id
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'size': 100,
        'sort': 'id',
        'sortorder': 'asc',
        'id': f'>{latest_id}'
    }
    
    response = requests.get(BASE_URL, params=params)
    new_artifacts = response.json()['records']
    
    if new_artifacts:
        df_metadata, df_media, df_colors = transform_artifacts(new_artifacts)
        load_to_sql(df_metadata, df_media, df_colors, db_config)
        return len(new_artifacts)
    
    return 0
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            response.raise_for_status()
            return response.json()
        except HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_db_connection(db_config):
    """Verify database connectivity"""
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.fetchone()
        conn.close()
        return True
    except Exception as e:
        st.error(f"Database connection failed: {e}")
        return False

# Use in Streamlit
if not test_db_connection(db_config):
    st.stop()
```

### Handle Missing Data

```python
def safe_transform(artifact):
    """Transform with null handling"""
    return {
        'id': artifact.get('id'),
        'title': artifact.get('title', 'Untitled')[:500],
        'culture': artifact.get('culture') or 'Unknown',
        'century': artifact.get('century') or 'Unknown',
        'accessionyear': artifact.get('accessionyear') or None,
        'totalpageviews': artifact.get('totalpageviews') or 0
    }
```

### Memory Management for Large Datasets

```python
def batch_process_artifacts(artifact_ids, batch_size=50):
    """Process artifacts in batches to avoid memory issues"""
    for i in range(0, len(artifact_ids), batch_size):
        batch_ids = artifact_ids[i:i+batch_size]
        
        # Fetch batch
        artifacts = [get_artifact_by_id(aid) for aid in batch_ids]
        
        # Transform and load
        df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        load_to_sql(df_metadata, df_media, df_colors, db_config)
        
        # Clear memory
        del artifacts, df_metadata, df_media, df_colors
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

This skill enables AI coding agents to help developers build complete data engineering pipelines using the Harvard Art Museums API with ETL processes, SQL analytics, and interactive dashboards.
