---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics application using Harvard Art Museums API with Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard for Harvard Art Museums data
  - extract and transform art collection data
  - visualize museum artifact metadata with Streamlit
  - set up SQL database for art collections
  - query Harvard Art Museums API
  - analyze artifact colors and media patterns
  - build data engineering pipeline for museum collections
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color data into relational databases
- **SQL Analytics**: Executes 20+ predefined analytical queries on structured artifact data
- **Interactive Visualization**: Streamlit dashboard with Plotly charts for real-time data exploration
- **Database Design**: Properly normalized tables with foreign key relationships

**Architecture Flow**: `Harvard API → Python ETL → MySQL/TiDB → SQL Analytics → Streamlit Dashboard`

## Installation

### Prerequisites

```bash
# Python 3.8+
# MySQL or TiDB Cloud account
# Harvard Art Museums API key (get from https://www.harvardartmuseums.org/collections/api)
```

### Setup Steps

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_db_username"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
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

## Configuration

### Database Connection

```python
import mysql.connector
import os

# Database configuration
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'port': int(os.getenv('DB_PORT', 3306))
}

# Establish connection
conn = mysql.connector.connect(**db_config)
cursor = conn.cursor()
```

### API Configuration

```python
import os
import requests

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

# API request with pagination
def fetch_artifacts(page=1, size=100):
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size
    }
    response = requests.get(BASE_URL, params=params)
    return response.json()
```

## Database Schema

### Create Tables

```sql
-- Artifact Metadata Table
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    technique VARCHAR(500),
    accessionyear INT,
    INDEX idx_culture (culture),
    INDEX idx_century (century),
    INDEX idx_department (department)
);

-- Artifact Media Table
CREATE TABLE IF NOT EXISTS artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    publiccaption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE,
    INDEX idx_artifact (artifact_id)
);

-- Artifact Colors Table
CREATE TABLE IF NOT EXISTS artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE,
    INDEX idx_artifact (artifact_id),
    INDEX idx_color (color)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os
import time

def extract_artifacts(num_pages=10):
    """Extract artifacts from Harvard Art Museums API"""
    API_KEY = os.getenv('HARVARD_API_KEY')
    BASE_URL = "https://api.harvardartmuseums.org/object"
    
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': API_KEY,
            'page': page,
            'size': 100
        }
        
        try:
            response = requests.get(BASE_URL, params=params)
            response.raise_for_status()
            data = response.json()
            
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            print(f"Fetched page {page}: {len(artifacts)} artifacts")
            
            # Rate limiting
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts
```

### Transform: Process JSON to Structured Data

```python
import pandas as pd

def transform_metadata(artifacts):
    """Transform artifacts to metadata dataframe"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'division': artifact.get('division', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'period': artifact.get('period', '')[:255],
            'technique': artifact.get('technique', '')[:500],
            'accessionyear': artifact.get('accessionyear')
        })
    
    return pd.DataFrame(metadata_list)

def transform_media(artifacts):
    """Transform media information"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media_list.append({
                'artifact_id': artifact_id,
                'iiifbaseuri': image.get('iiifbaseuri', '')[:500],
                'baseimageurl': image.get('baseimageurl', '')[:500],
                'publiccaption': image.get('publiccaption', '')
            })
    
    return pd.DataFrame(media_list)

def transform_colors(artifacts):
    """Transform color information"""
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_list.append({
                'artifact_id': artifact_id,
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'hue': color.get('hue', '')[:50],
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(color_list)
```

### Load: Insert into Database

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(df, table_name, conn):
    """Load dataframe to MySQL table"""
    cursor = conn.cursor()
    
    # Prepare batch insert
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    
    insert_query = f"""
        INSERT INTO {table_name} ({columns})
        VALUES ({placeholders})
        ON DUPLICATE KEY UPDATE
        {', '.join([f"{col}=VALUES({col})" for col in df.columns if col != 'id'])}
    """
    
    # Convert dataframe to list of tuples
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data)
        conn.commit()
        print(f"Successfully inserted {cursor.rowcount} rows into {table_name}")
    except Error as e:
        print(f"Error loading to {table_name}: {e}")
        conn.rollback()
    finally:
        cursor.close()

# Complete ETL flow
def run_etl_pipeline(num_pages=10):
    """Run complete ETL pipeline"""
    # Extract
    print("Extracting data from API...")
    artifacts = extract_artifacts(num_pages)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_metadata(artifacts)
    media_df = transform_media(artifacts)
    colors_df = transform_colors(artifacts)
    
    # Load
    print("Loading data to database...")
    conn = mysql.connector.connect(**db_config)
    
    load_to_database(metadata_df, 'artifactmetadata', conn)
    load_to_database(media_df, 'artifactmedia', conn)
    load_to_database(colors_df, 'artifactcolors', conn)
    
    conn.close()
    print("ETL pipeline completed!")
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Query 1: Artifacts by Culture
query_artifacts_by_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 20;
"""

# Query 2: Artifacts by Century
query_artifacts_by_century = """
SELECT century, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE century IS NOT NULL AND century != ''
GROUP BY century
ORDER BY artifact_count DESC;
"""

# Query 3: Media Availability Analysis
query_media_availability = """
SELECT 
    CASE 
        WHEN baseimageurl IS NOT NULL THEN 'Has Image'
        ELSE 'No Image'
    END as media_status,
    COUNT(*) as count
FROM artifactmedia
GROUP BY media_status;
"""

# Query 4: Top Colors in Collection
query_top_colors = """
SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
FROM artifactcolors
WHERE color IS NOT NULL
GROUP BY color
ORDER BY frequency DESC
LIMIT 15;
"""

# Query 5: Artifacts by Department
query_by_department = """
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE department IS NOT NULL AND department != ''
GROUP BY department
ORDER BY artifact_count DESC;
"""

# Query 6: Accession Year Distribution
query_accession_years = """
SELECT accessionyear, COUNT(*) as count
FROM artifactmetadata
WHERE accessionyear IS NOT NULL
GROUP BY accessionyear
ORDER BY accessionyear DESC
LIMIT 20;
"""

# Query 7: Culture and Century Analysis
query_culture_century = """
SELECT culture, century, COUNT(*) as count
FROM artifactmetadata
WHERE culture IS NOT NULL AND century IS NOT NULL
GROUP BY culture, century
ORDER BY count DESC
LIMIT 20;
"""
```

## Streamlit Dashboard Implementation

### Basic App Structure

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

# Database connection
@st.cache_resource
def get_database_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Execute query and return dataframe
def execute_query(query):
    conn = get_database_connection()
    df = pd.read_sql(query, conn)
    return df

# Main app
st.title("🎨 Harvard Art Museums Analytics Dashboard")
st.markdown("---")

# Sidebar for query selection
st.sidebar.header("Analytics Queries")

queries = {
    "Artifacts by Culture": query_artifacts_by_culture,
    "Artifacts by Century": query_artifacts_by_century,
    "Media Availability": query_media_availability,
    "Top Colors": query_top_colors,
    "By Department": query_by_department,
    "Accession Years": query_accession_years
}

selected_query = st.sidebar.selectbox("Select Analysis", list(queries.keys()))

# Execute and display
if st.sidebar.button("Run Analysis"):
    with st.spinner("Running query..."):
        df = execute_query(queries[selected_query])
        
        # Display table
        st.subheader(f"Results: {selected_query}")
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

### ETL Control Panel

```python
# ETL section in Streamlit
st.sidebar.markdown("---")
st.sidebar.header("ETL Pipeline")

num_pages = st.sidebar.slider("Number of Pages to Fetch", 1, 50, 10)

if st.sidebar.button("Run ETL Pipeline"):
    with st.spinner("Running ETL pipeline..."):
        try:
            run_etl_pipeline(num_pages)
            st.sidebar.success(f"ETL completed! Fetched {num_pages} pages.")
        except Exception as e:
            st.sidebar.error(f"ETL failed: {e}")
```

## Common Patterns

### Pagination Handler

```python
def fetch_all_artifacts_with_pagination(max_records=1000):
    """Fetch artifacts with automatic pagination"""
    API_KEY = os.getenv('HARVARD_API_KEY')
    BASE_URL = "https://api.harvardartmuseums.org/object"
    
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        params = {
            'apikey': API_KEY,
            'page': page,
            'size': size
        }
        
        response = requests.get(BASE_URL, params=params)
        data = response.json()
        
        records = data.get('records', [])
        if not records:
            break
        
        all_artifacts.extend(records)
        page += 1
        time.sleep(0.5)  # Rate limiting
    
    return all_artifacts[:max_records]
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_api_call(url, params, retries=3):
    """API call with retry logic"""
    for attempt in range(retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            logger.warning(f"Attempt {attempt + 1} failed: {e}")
            if attempt == retries - 1:
                logger.error("All retries failed")
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Running the Application

### Local Development

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

### Environment Variables

Create a `.env` file:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

Load in Python:

```python
from dotenv import load_dotenv
load_dotenv()
```

## Troubleshooting

### API Rate Limiting

```python
# Add delay between requests
import time
time.sleep(0.5)  # 500ms delay

# Use exponential backoff for retries
def exponential_backoff(attempt):
    return min(2 ** attempt, 60)  # Max 60 seconds
```

### Database Connection Errors

```python
# Test connection
try:
    conn = mysql.connector.connect(**db_config)
    print("Connection successful!")
    conn.close()
except mysql.connector.Error as e:
    print(f"Connection failed: {e}")
    
# Use connection pooling for better performance
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    **db_config
)
```

### Empty or Null Data Handling

```python
# Safe data extraction
def safe_get(dictionary, key, default=''):
    value = dictionary.get(key, default)
    return value if value is not None else default

# Clean dataframe before loading
df = df.where(pd.notnull(df), None)
df = df.fillna('')  # or use None for NULL in database
```

### Memory Issues with Large Datasets

```python
# Use chunking for large datasets
def load_in_chunks(df, table_name, conn, chunk_size=1000):
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i+chunk_size]
        load_to_database(chunk, table_name, conn)
        print(f"Loaded chunk {i//chunk_size + 1}")
```

This skill provides complete coverage for building ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit.
