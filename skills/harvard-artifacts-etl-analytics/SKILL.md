---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Streamlit, SQL, and Plotly
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create a data engineering app with Harvard API
  - set up artifact collection analytics with Streamlit
  - build SQL analytics dashboard for museum artifacts
  - extract and visualize Harvard art museum data
  - create interactive analytics app for artifact collections
  - implement ETL workflow with Harvard Art Museums API
  - design relational database schema for museum artifacts
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App is a complete data pipeline that:

- **Extracts** artifact data from the Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into structured relational tables
- **Loads** data into MySQL/TiDB Cloud databases
- **Analyzes** data using SQL queries
- **Visualizes** results through interactive Streamlit dashboards with Plotly charts

It simulates production-grade data pipelines used in analytics and data engineering roles.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
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

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

Create a `.env` file or configure environment variables:

```bash
HARVARD_API_KEY=your_api_key_here
```

### Database Configuration

For MySQL/TiDB Cloud connection, configure in your application:

```python
import os
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

connection = mysql.connector.connect(**db_config)
```

## Database Schema

The project uses three main tables with foreign key relationships:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    medium VARCHAR(500),
    technique VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    imageid INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key API Patterns

### Extract Data from Harvard API

```python
import requests
import os
from typing import Dict, List

def fetch_artifacts(api_key: str, page: int = 1, size: int = 100) -> Dict:
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
print(f"Pages: {data['info']['pages']}")
```

### Paginated Data Collection

```python
def collect_all_artifacts(api_key: str, max_records: int = 500) -> List[Dict]:
    """
    Collect artifacts across multiple pages with rate limiting
    """
    import time
    
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        try:
            data = fetch_artifacts(api_key, page=page, size=size)
            records = data.get('records', [])
            
            if not records:
                break
                
            all_artifacts.extend(records)
            print(f"Collected {len(all_artifacts)} artifacts...")
            
            page += 1
            time.sleep(0.5)  # Rate limiting
            
        except requests.exceptions.RequestException as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts[:max_records]
```

## ETL Pipeline Implementation

### Transform Nested JSON to Relational Format

```python
import pandas as pd

def transform_artifacts_to_dataframes(artifacts: List[Dict]) -> tuple:
    """
    Transform nested JSON artifacts into three normalized DataFrames
    
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
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear'),
            'url': artifact.get('url')
        }
        metadata_records.append(metadata)
        
        # Extract media (images)
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        for img in images:
            media = {
                'artifact_id': artifact_id,
                'iiifbaseuri': img.get('iiifbaseuri'),
                'baseimageurl': img.get('baseimageurl'),
                'imageid': img.get('imageid')
            }
            media_records.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_record = {
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percent': color.get('percent')
            }
            color_records.append(color_record)
    
    metadata_df = pd.DataFrame(metadata_records)
    media_df = pd.DataFrame(media_records)
    colors_df = pd.DataFrame(color_records)
    
    return metadata_df, media_df, colors_df
```

### Load Data to SQL Database

```python
def load_to_database(metadata_df: pd.DataFrame, 
                     media_df: pd.DataFrame, 
                     colors_df: pd.DataFrame,
                     db_config: Dict):
    """
    Batch insert DataFrames into SQL database
    """
    import mysql.connector
    
    connection = mysql.connector.connect(**db_config)
    cursor = connection.cursor()
    
    try:
        # Insert metadata (parent table first)
        for _, row in metadata_df.iterrows():
            query = """
                INSERT INTO artifactmetadata 
                (id, title, culture, century, dated, classification, 
                 department, division, medium, technique, dimensions, 
                 creditline, accessionyear, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """
            cursor.execute(query, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            query = """
                INSERT INTO artifactmedia 
                (artifact_id, iiifbaseuri, baseimageurl, imageid)
                VALUES (%s, %s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            query = """
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, percent)
                VALUES (%s, %s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        connection.commit()
        print(f"Loaded {len(metadata_df)} artifacts to database")
        
    except Exception as e:
        connection.rollback()
        print(f"Error loading data: {e}")
        raise
    finally:
        cursor.close()
        connection.close()
```

## Streamlit Dashboard Implementation

### Main Application Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px
import mysql.connector
import os

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar configuration
st.sidebar.header("Configuration")

# Database connection
@st.cache_resource
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Execute query
def run_query(query: str) -> pd.DataFrame:
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### Analytics Query Examples

```python
# Sample analytical queries for the dashboard

ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Top Color Usage": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN m.artifact_id IS NOT NULL THEN 'Has Media' 
                 ELSE 'No Media' END as media_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY media_status
    """,
    
    "Artifacts by Classification": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Accession Year Trends": """
        SELECT accessionyear, COUNT(*) as acquisitions
        FROM artifactmetadata
        WHERE accessionyear IS NOT NULL
        GROUP BY accessionyear
        ORDER BY accessionyear DESC
        LIMIT 50
    """,
    
    "Color Spectrum Analysis": """
        SELECT spectrum, COUNT(DISTINCT artifact_id) as artifact_count
        FROM artifactcolors
        WHERE spectrum IS NOT NULL
        GROUP BY spectrum
        ORDER BY artifact_count DESC
    """
}

# Dashboard implementation
st.sidebar.subheader("Select Analysis")
query_name = st.sidebar.selectbox("Choose Query", list(ANALYTICS_QUERIES.keys()))

if st.sidebar.button("Run Analysis"):
    with st.spinner("Executing query..."):
        query = ANALYTICS_QUERIES[query_name]
        df = run_query(query)
        
        st.subheader(f"Results: {query_name}")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)
```

### Interactive Data Collection Module

```python
# Add to Streamlit app
st.sidebar.header("Data Collection")

api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                 value=os.getenv('HARVARD_API_KEY', ''))
num_records = st.sidebar.slider("Records to Collect", 10, 500, 100)

if st.sidebar.button("Collect & Load Data"):
    if not api_key:
        st.error("Please provide API key")
    else:
        with st.spinner(f"Collecting {num_records} artifacts..."):
            # Collect data
            artifacts = collect_all_artifacts(api_key, num_records)
            
            # Transform
            metadata_df, media_df, colors_df = transform_artifacts_to_dataframes(artifacts)
            
            # Load
            db_config = {
                'host': os.getenv('DB_HOST'),
                'user': os.getenv('DB_USER'),
                'password': os.getenv('DB_PASSWORD'),
                'database': os.getenv('DB_NAME')
            }
            load_to_database(metadata_df, media_df, colors_df, db_config)
            
            st.success(f"Successfully loaded {len(artifacts)} artifacts!")
            st.balloons()
```

## Common Patterns

### Complete ETL Workflow

```python
def complete_etl_pipeline(api_key: str, num_records: int, db_config: Dict):
    """
    Execute full ETL pipeline from API to database
    """
    # Extract
    print("Extracting data from API...")
    artifacts = collect_all_artifacts(api_key, num_records)
    
    # Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts_to_dataframes(artifacts)
    
    # Clean data
    metadata_df = metadata_df.fillna('')
    media_df = media_df.fillna('')
    colors_df = colors_df.fillna(0)
    
    # Load
    print("Loading data to database...")
    load_to_database(metadata_df, media_df, colors_df, db_config)
    
    return {
        'artifacts': len(metadata_df),
        'media_records': len(media_df),
        'color_records': len(colors_df)
    }
```

### Error Handling for API Calls

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(api_key: str, page: int, max_retries: int = 3) -> Dict:
    """
    Fetch artifacts with exponential backoff retry logic
    """
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except RequestException as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt  # Exponential backoff
                print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
                time.sleep(wait_time)
            else:
                raise
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY=your_api_key
export DB_HOST=localhost
export DB_USER=root
export DB_PASSWORD=your_password
export DB_NAME=harvard_artifacts

# Run Streamlit app
streamlit run app.py
```

## Troubleshooting

**API Rate Limiting:**
- Add `time.sleep(0.5)` between requests
- Reduce batch size from 100 to 50

**Database Connection Errors:**
- Verify credentials in environment variables
- Check if MySQL/TiDB Cloud is accessible
- Ensure database exists before running migrations

**Data Type Mismatches:**
- Use `.fillna()` for NULL handling
- Convert types explicitly: `pd.to_numeric()`, `str()`

**Memory Issues with Large Datasets:**
- Process in smaller batches (100-200 records)
- Use database cursors instead of loading all into memory

**Streamlit Caching Issues:**
- Clear cache: `st.cache_data.clear()`
- Use `@st.cache_resource` for database connections

This skill enables building production-grade ETL pipelines and analytics dashboards using real-world museum data.
