---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a museum artifact analytics dashboard
  - extract and analyze Harvard Art Museums API data
  - set up a data engineering pipeline with Streamlit visualization
  - connect to Harvard Art Museums API and store in SQL
  - analyze art museum collections with SQL queries
  - build a museum data warehouse with interactive analytics
  - create artifact visualizations from Harvard API
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards with Plotly visualizations.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON into normalized SQL tables
- **SQL Database**: Stores artifact metadata, media, and color information in relational tables
- **Analytics Queries**: 20+ predefined SQL queries for insights on artifacts, cultures, centuries, and media
- **Interactive Visualization**: Streamlit dashboard with Plotly charts for real-time analytics

**Architecture Flow**: API → ETL → SQL (MySQL/TiDB) → Analytics → Streamlit Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required Dependencies**:
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

1. Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Store it as an environment variable:

```bash
export HARVARD_API_KEY="your_api_key_here"
```

Or create a `.env` file:
```env
HARVARD_API_KEY=your_api_key_here
```

### Database Configuration

Set up MySQL or TiDB Cloud connection:

```python
import os
import mysql.connector

# Database connection parameters
DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

# Create connection
conn = mysql.connector.connect(**DB_CONFIG)
```

Environment variables for database:
```env
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

## Running the Application

### Start Streamlit Dashboard

```bash
streamlit run app.py
```

The app will be available at `http://localhost:8501`

## Database Schema

The ETL pipeline creates three normalized tables:

### artifactmetadata
```sql
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    division VARCHAR(255)
);
```

### artifactmedia
```sql
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    totalimages INT,
    totalimagesinagedouturl INT,
    primaryimageurl TEXT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

### artifactcolors
```sql
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## ETL Pipeline Code Examples

### Extract: Fetch Data from API

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, max_records=100, page_size=100):
    """
    Fetch artifact data from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key
        max_records: Maximum number of records to fetch
        page_size: Records per API call (max 100)
    
    Returns:
        List of artifact records
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_records = []
    page = 1
    
    while len(all_records) < max_records:
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code != 200:
            print(f"Error: {response.status_code}")
            break
            
        data = response.json()
        records = data.get('records', [])
        
        if not records:
            break
            
        all_records.extend(records)
        page += 1
        
        # Rate limiting
        if page % 10 == 0:
            import time
            time.sleep(1)
    
    return all_records[:max_records]
```

### Transform: Process JSON to DataFrames

```python
def transform_artifacts(raw_data):
    """
    Transform raw API data into normalized DataFrames
    
    Args:
        raw_data: List of artifact records from API
    
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
            'title': record.get('title', '')[:500],  # Truncate
            'culture': record.get('culture', ''),
            'period': record.get('period', ''),
            'century': record.get('century', ''),
            'classification': record.get('classification', ''),
            'department': record.get('department', ''),
            'dated': record.get('dated', ''),
            'division': record.get('division', '')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        media = {
            'objectid': record.get('objectid'),
            'totalimages': record.get('totalimages', 0),
            'totalimagesinagedouturl': record.get('totalimagesinagedouturl', 0),
            'primaryimageurl': record.get('primaryimageurl', '')
        }
        media_list.append(media)
        
        # Extract color information
        colors = record.get('colors', [])
        for color in colors:
            color_record = {
                'objectid': record.get('objectid'),
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_record)
    
    metadata_df = pd.DataFrame(metadata_list)
    media_df = pd.DataFrame(media_list)
    colors_df = pd.DataFrame(colors_list)
    
    return metadata_df, media_df, colors_df
```

### Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """
    Load transformed data into SQL database
    
    Args:
        metadata_df: DataFrame with artifact metadata
        media_df: DataFrame with media information
        colors_df: DataFrame with color data
        db_config: Database connection configuration
    """
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        
        # Insert metadata (batch insert)
        metadata_query = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, period, century, classification, department, dated, division)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
        """
        metadata_values = metadata_df.values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        media_query = """
        INSERT INTO artifactmedia 
        (objectid, totalimages, totalimagesinagedouturl, primaryimageurl)
        VALUES (%s, %s, %s, %s)
        """
        media_values = media_df.values.tolist()
        cursor.executemany(media_query, media_values)
        
        # Insert colors
        if not colors_df.empty:
            colors_query = """
            INSERT INTO artifactcolors 
            (objectid, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
            """
            colors_values = colors_df.values.tolist()
            cursor.executemany(colors_query, colors_values)
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        conn.rollback()
    finally:
        if conn.is_connected():
            cursor.close()
            conn.close()
```

## Analytics Queries

### Sample Analytical Queries

```python
# Top 10 cultures by artifact count
query_1 = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Artifacts by century
query_2 = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL AND century != ''
GROUP BY century
ORDER BY count DESC;
"""

# Media availability analysis
query_3 = """
SELECT 
    CASE 
        WHEN totalimages > 0 THEN 'Has Images'
        ELSE 'No Images'
    END as image_status,
    COUNT(*) as count
FROM artifactmedia
GROUP BY image_status;
"""

# Top colors across artifacts
query_4 = """
SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
FROM artifactcolors
WHERE color IS NOT NULL
GROUP BY color
ORDER BY usage_count DESC
LIMIT 10;
"""

# Department distribution
query_5 = """
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE department IS NOT NULL AND department != ''
GROUP BY department
ORDER BY artifact_count DESC;
"""

# Classification analysis
query_6 = """
SELECT classification, COUNT(*) as count
FROM artifactmetadata
WHERE classification IS NOT NULL
GROUP BY classification
ORDER BY count DESC
LIMIT 15;
"""
```

### Execute Queries in Streamlit

```python
import streamlit as st
import pandas as pd
import plotly.express as px

def run_query(query, db_config):
    """Execute SQL query and return DataFrame"""
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# In Streamlit app
st.title("Harvard Art Museums Analytics")

query_options = {
    "Top 10 Cultures": query_1,
    "Artifacts by Century": query_2,
    "Media Availability": query_3,
    "Top Colors": query_4,
    "Department Distribution": query_5
}

selected_query = st.selectbox("Select Analysis", list(query_options.keys()))

if st.button("Run Query"):
    df = run_query(query_options[selected_query], DB_CONFIG)
    
    # Display table
    st.dataframe(df)
    
    # Auto-generate visualization
    if len(df.columns) >= 2:
        fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                     title=selected_query)
        st.plotly_chart(fig)
```

## Streamlit Dashboard Components

### Complete App Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px
import mysql.connector
import os

# Page configuration
st.set_page_config(
    page_title="Harvard Art Museums Analytics",
    page_icon="🎨",
    layout="wide"
)

# Database configuration
DB_CONFIG = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'port': int(os.getenv('DB_PORT', 3306))
}

# Sidebar for data collection
st.sidebar.header("Data Collection")
api_key = st.sidebar.text_input("Harvard API Key", type="password")
num_records = st.sidebar.slider("Records to Fetch", 10, 500, 100)

if st.sidebar.button("Fetch & Load Data"):
    with st.spinner("Fetching data from API..."):
        raw_data = fetch_artifacts(api_key, num_records)
        metadata_df, media_df, colors_df = transform_artifacts(raw_data)
        load_to_database(metadata_df, media_df, colors_df, DB_CONFIG)
        st.sidebar.success(f"Loaded {len(metadata_df)} artifacts!")

# Main analytics section
st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Query selector
queries = {
    "Culture Distribution": "SELECT culture, COUNT(*) as count FROM artifactmetadata WHERE culture != '' GROUP BY culture ORDER BY count DESC LIMIT 10",
    "Century Timeline": "SELECT century, COUNT(*) as count FROM artifactmetadata WHERE century != '' GROUP BY century ORDER BY count DESC",
    "Color Analysis": "SELECT color, COUNT(*) as count FROM artifactcolors GROUP BY color ORDER BY count DESC LIMIT 10"
}

selected = st.selectbox("Choose Analysis", list(queries.keys()))

if st.button("Run Analysis"):
    conn = mysql.connector.connect(**DB_CONFIG)
    df = pd.read_sql(queries[selected], conn)
    conn.close()
    
    col1, col2 = st.columns([1, 2])
    
    with col1:
        st.subheader("Results")
        st.dataframe(df, use_container_width=True)
    
    with col2:
        st.subheader("Visualization")
        fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                     title=selected, color=df.columns[1])
        st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Full ETL Pipeline Execution

```python
import os
from dotenv import load_dotenv

load_dotenv()

# Configuration
API_KEY = os.getenv('HARVARD_API_KEY')
DB_CONFIG = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

# Execute ETL
def run_etl_pipeline(api_key, db_config, num_records=100):
    """Complete ETL pipeline execution"""
    
    # Extract
    print("Extracting data from API...")
    raw_data = fetch_artifacts(api_key, num_records)
    
    # Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_data)
    
    # Load
    print("Loading data to database...")
    load_to_database(metadata_df, media_df, colors_df, db_config)
    
    print("ETL pipeline completed successfully!")
    return metadata_df, media_df, colors_df

# Run
if __name__ == "__main__":
    run_etl_pipeline(API_KEY, DB_CONFIG, num_records=100)
```

### Incremental Data Loading

```python
def get_existing_object_ids(db_config):
    """Fetch already loaded object IDs to avoid duplicates"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    cursor.execute("SELECT objectid FROM artifactmetadata")
    existing_ids = set(row[0] for row in cursor.fetchall())
    conn.close()
    return existing_ids

def incremental_load(api_key, db_config, num_records=100):
    """Load only new artifacts not already in database"""
    existing_ids = get_existing_object_ids(db_config)
    raw_data = fetch_artifacts(api_key, num_records * 2)  # Fetch extra
    
    # Filter out existing records
    new_data = [r for r in raw_data if r.get('objectid') not in existing_ids]
    
    if new_data:
        metadata_df, media_df, colors_df = transform_artifacts(new_data)
        load_to_database(metadata_df, media_df, colors_df, db_config)
        print(f"Loaded {len(new_data)} new artifacts")
    else:
        print("No new artifacts to load")
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
import time

def fetch_with_retry(api_key, max_retries=3):
    """Fetch data with exponential backoff"""
    for attempt in range(max_retries):
        try:
            data = fetch_artifacts(api_key, 100)
            return data
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Too many requests
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_database_connection(db_config):
    """Test database connectivity"""
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        conn.close()
        print("✓ Database connection successful")
        return True
    except Error as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Missing Data Handling

```python
def safe_transform(record, field, default=''):
    """Safely extract field with default value"""
    value = record.get(field, default)
    return value if value is not None else default

# Use in transformation
metadata = {
    'objectid': record.get('objectid'),
    'title': safe_transform(record, 'title', 'Untitled'),
    'culture': safe_transform(record, 'culture', 'Unknown'),
    'century': safe_transform(record, 'century', 'Unknown')
}
```

### Empty Query Results

```python
def safe_query_execution(query, db_config):
    """Execute query with error handling"""
    try:
        conn = mysql.connector.connect(**db_config)
        df = pd.read_sql(query, conn)
        conn.close()
        
        if df.empty:
            st.warning("No data found for this query")
            return None
        return df
    except Exception as e:
        st.error(f"Query error: {e}")
        return None
```

This skill provides comprehensive coverage for building ETL pipelines and analytics dashboards using the Harvard Art Museums API with SQL storage and Streamlit visualization.
