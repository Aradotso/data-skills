---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - fetch and analyze Harvard Art Museums API data
  - implement museum collection data engineering workflow
  - set up artifact metadata analytics with Streamlit
  - build SQL analytics for art museum collections
  - create interactive visualization for museum data
  - develop end-to-end data pipeline for cultural artifacts
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build and work with end-to-end data engineering pipelines for the Harvard Art Museums API. The project demonstrates real-world ETL operations, SQL database design, analytical queries, and interactive Streamlit dashboards for cultural artifact data.

## What This Project Does

The Harvard-Artifacts-Collection application:
- Extracts artifact data from Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into relational database schema (metadata, media, colors)
- Loads data into MySQL/TiDB Cloud with optimized batch inserts
- Provides 20+ predefined SQL analytical queries
- Visualizes results through interactive Plotly charts in Streamlit

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly
```

## Configuration

### API Key Setup

Get a free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Store credentials in environment variables or Streamlit secrets:

```python
# .streamlit/secrets.toml
[api]
key = "your-harvard-api-key"

[database]
host = "your-database-host"
user = "your-db-user"
password = "your-db-password"
database = "harvard_artifacts"
```

### Database Configuration

```python
import mysql.connector

# Connection setup
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

connection = mysql.connector.connect(**db_config)
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Core Components

### 1. API Data Collection

```python
import requests
import time

def fetch_artifacts(api_key, size=100, page=1):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

def collect_multiple_pages(api_key, total_records=1000, batch_size=100):
    """Collect data with rate limiting"""
    all_artifacts = []
    pages = (total_records // batch_size) + 1
    
    for page in range(1, pages + 1):
        data = fetch_artifacts(api_key, size=batch_size, page=page)
        all_artifacts.extend(data.get('records', []))
        time.sleep(1)  # Rate limiting
        
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd

def extract_metadata(artifacts):
    """Extract artifact metadata"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata_list.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dated': artifact.get('dated')
        })
    
    return pd.DataFrame(metadata_list)

def extract_media(artifacts):
    """Extract media/image data"""
    media_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for img in images:
            media_list.append({
                'objectid': objectid,
                'imageid': img.get('imageid'),
                'baseimageurl': img.get('baseimageurl'),
                'width': img.get('width'),
                'height': img.get('height'),
                'format': img.get('format')
            })
    
    return pd.DataFrame(media_list)

def extract_colors(artifacts):
    """Extract color palette data"""
    color_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_list.append({
                'objectid': objectid,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(color_list)
```

### 3. Database Schema Creation

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dated VARCHAR(100)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    imageid VARCHAR(100),
    baseimageurl VARCHAR(500),
    width INT,
    height INT,
    format VARCHAR(50),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

### 4. Data Loading

```python
def load_to_database(df, table_name, connection):
    """Batch insert data into SQL database"""
    cursor = connection.cursor()
    
    # Generate column names and placeholders
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    
    insert_query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Convert DataFrame to list of tuples
    data_tuples = [tuple(row) for row in df.values]
    
    # Batch insert
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    
    return cursor.rowcount

# Usage
connection = mysql.connector.connect(**db_config)

metadata_df = extract_metadata(artifacts)
media_df = extract_media(artifacts)
colors_df = extract_colors(artifacts)

load_to_database(metadata_df, 'artifactmetadata', connection)
load_to_database(media_df, 'artifactmedia', connection)
load_to_database(colors_df, 'artifactcolors', connection)
```

### 5. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10;
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC;
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10;
    """,
    
    "Artifacts with Most Images": """
        SELECT m.objectid, m.title, COUNT(med.imageid) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia med ON m.objectid = med.objectid
        GROUP BY m.objectid, m.title
        ORDER BY image_count DESC
        LIMIT 10;
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC;
    """
}

def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)
```

### 6. Streamlit Visualization

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Query selector
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        # Execute query
        query = ANALYTICS_QUERIES[query_name]
        result_df = execute_query(connection, query)
        
        # Display results
        st.dataframe(result_df)
        
        # Auto-generate visualization
        if len(result_df.columns) >= 2:
            fig = px.bar(
                result_df, 
                x=result_df.columns[0], 
                y=result_df.columns[1],
                title=query_name
            )
            st.plotly_chart(fig)

if __name__ == "__main__":
    create_dashboard()
```

## Common Patterns

### Complete ETL Workflow

```python
def run_etl_pipeline(api_key, db_config, num_records=1000):
    """Complete ETL pipeline execution"""
    
    # Extract
    st.info("Extracting data from API...")
    artifacts = collect_multiple_pages(api_key, total_records=num_records)
    
    # Transform
    st.info("Transforming data...")
    metadata_df = extract_metadata(artifacts)
    media_df = extract_media(artifacts)
    colors_df = extract_colors(artifacts)
    
    # Load
    st.info("Loading to database...")
    connection = mysql.connector.connect(**db_config)
    
    load_to_database(metadata_df, 'artifactmetadata', connection)
    load_to_database(media_df, 'artifactmedia', connection)
    load_to_database(colors_df, 'artifactcolors', connection)
    
    connection.close()
    
    st.success(f"ETL Complete! Loaded {len(metadata_df)} artifacts")
```

### Error Handling for API Calls

```python
def safe_api_call(api_key, page, max_retries=3):
    """API call with retry logic"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page=page)
        except requests.exceptions.RequestException as e:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
                continue
            else:
                st.error(f"Failed after {max_retries} attempts: {e}")
                return None
```

## Troubleshooting

**API Rate Limiting**: Add `time.sleep(1)` between requests. Harvard API has rate limits.

**Database Connection Issues**: Verify credentials and network access. For TiDB Cloud, ensure IP whitelist is configured.

**Missing Data Fields**: Not all artifacts have all fields. Use `.get()` with defaults:
```python
culture = artifact.get('culture', 'Unknown')
```

**Large Dataset Memory**: Process in batches:
```python
for i in range(0, len(artifacts), 100):
    batch = artifacts[i:i+100]
    process_batch(batch)
```

**Streamlit Session State**: Cache database connections:
```python
@st.cache_resource
def get_connection():
    return mysql.connector.connect(**db_config)
```
