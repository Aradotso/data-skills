---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - show me how to extract and transform artifact data from Harvard API
  - help me create a data engineering pipeline with museum artifacts
  - how do I visualize Harvard Art Museums data with Streamlit
  - build an analytics dashboard for museum collection data
  - create SQL queries for Harvard artifacts analysis
  - set up a data pipeline for art museum collections
  - extract color and media data from Harvard Art Museums API
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates end-to-end data engineering using the Harvard Art Museums API. It covers ETL pipelines, SQL database design, analytics queries, and interactive Streamlit dashboards for museum artifact data.

## What It Does

The application:
- Extracts artifact data from Harvard Art Museums API with pagination
- Transforms nested JSON into relational database tables
- Loads data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries on artifact metadata, media, and colors
- Visualizes results through interactive Plotly charts in Streamlit

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages**:
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Connection
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

Create the required tables:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accessionyear INT,
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    mediacount INT,
    primaryimageurl TEXT,
    hasimage BOOLEAN,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(100),
    spectrum VARCHAR(100),
    hue VARCHAR(100),
    percent DECIMAL(5,2),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## API Integration

### Fetching Artifacts with Pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_pages=5):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': 100,  # Max results per page
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}: {len(data.get('records', []))} artifacts")
        else:
            print(f"Error on page {page}: {response.status_code}")
            break
    
    return artifacts
```

### Rate Limiting and Error Handling

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff retry logic"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            
            if response.status_code == 429:  # Rate limited
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
                continue
            
            response.raise_for_status()
            return response.json()
            
        except RequestException as e:
            print(f"Attempt {attempt + 1} failed: {e}")
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
    
    return None
```

## ETL Pipeline

### Extract and Transform

```python
import pandas as pd

def transform_artifact_data(artifacts):
    """Transform raw API data into relational tables"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        object_id = artifact.get('objectid')
        
        # Extract metadata
        metadata_records.append({
            'objectid': object_id,
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'accessionyear': artifact.get('accessionyear'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        })
        
        # Extract media info
        media_records.append({
            'objectid': object_id,
            'mediacount': artifact.get('mediacount', 0),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'hasimage': 1 if artifact.get('primaryimageurl') else 0
        })
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_records.append({
                'objectid': object_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load to Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Batch insert data into SQL tables"""
    connection = get_db_connection()
    cursor = connection.cursor()
    
    try:
        # Insert metadata
        metadata_query = """
            INSERT INTO artifactmetadata 
            (objectid, title, culture, period, century, classification, 
             department, dated, accessionyear, totalpageviews, totaluniquepageviews)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
            title=VALUES(title), culture=VALUES(culture)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        media_query = """
            INSERT INTO artifactmedia (objectid, mediacount, primaryimageurl, hasimage)
            VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        colors_query = """
            INSERT INTO artifactcolors (objectid, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(colors_query, colors_df.values.tolist())
        
        connection.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        connection.rollback()
    finally:
        cursor.close()
        connection.close()
```

## Analytics Queries

### Common Analytical Patterns

```python
# Top 10 cultures by artifact count
query_cultures = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Artifacts by century distribution
query_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Most popular artifacts by pageviews
query_popular = """
    SELECT title, culture, totalpageviews
    FROM artifactmetadata
    WHERE totalpageviews > 0
    ORDER BY totalpageviews DESC
    LIMIT 20
"""

# Color distribution analysis
query_colors = """
    SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
    FROM artifactcolors
    GROUP BY color
    ORDER BY usage_count DESC
    LIMIT 15
"""

# Department classification breakdown
query_dept_class = """
    SELECT department, classification, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL AND classification IS NOT NULL
    GROUP BY department, classification
    ORDER BY count DESC
    LIMIT 25
"""
```

### Execute Queries

```python
def execute_query(query):
    """Execute SQL query and return DataFrame"""
    connection = get_db_connection()
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return pd.DataFrame()
    finally:
        connection.close()
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for data collection
with st.sidebar:
    st.header("Data Collection")
    num_pages = st.number_input("Number of pages to fetch", 1, 10, 5)
    
    if st.button("Fetch & Load Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(num_pages)
            metadata_df, media_df, colors_df = transform_artifact_data(artifacts)
            load_to_database(metadata_df, media_df, colors_df)
            st.success(f"Loaded {len(metadata_df)} artifacts!")

# Analytics section
st.header("📊 Analytics Queries")

queries = {
    "Top Cultures": query_cultures,
    "Century Distribution": query_century,
    "Popular Artifacts": query_popular,
    "Color Usage": query_colors,
    "Department Classification": query_dept_class
}

selected_query = st.selectbox("Select Analysis", list(queries.keys()))

if st.button("Run Query"):
    result_df = execute_query(queries[selected_query])
    
    if not result_df.empty:
        st.dataframe(result_df, use_container_width=True)
        
        # Auto-generate chart
        if len(result_df.columns) >= 2:
            fig = px.bar(
                result_df.head(15),
                x=result_df.columns[0],
                y=result_df.columns[1],
                title=selected_query
            )
            st.plotly_chart(fig, use_container_width=True)
```

### Advanced Visualization

```python
def create_interactive_dashboard():
    """Create multi-chart dashboard"""
    col1, col2 = st.columns(2)
    
    with col1:
        st.subheader("Artifacts by Culture")
        culture_df = execute_query(query_cultures)
        fig1 = px.bar(culture_df, x='culture', y='artifact_count', 
                      color='artifact_count', color_continuous_scale='Viridis')
        st.plotly_chart(fig1, use_container_width=True)
    
    with col2:
        st.subheader("Color Distribution")
        color_df = execute_query(query_colors)
        fig2 = px.pie(color_df, values='usage_count', names='color')
        st.plotly_chart(fig2, use_container_width=True)
    
    st.subheader("Timeline Analysis")
    century_df = execute_query(query_century)
    fig3 = px.line(century_df, x='century', y='count', markers=True)
    st.plotly_chart(fig3, use_container_width=True)
```

## Common Patterns

### Complete ETL Workflow

```python
def run_etl_pipeline(num_pages=5):
    """Complete ETL pipeline execution"""
    print("Starting ETL pipeline...")
    
    # Extract
    artifacts = fetch_artifacts(num_pages)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    metadata_df, media_df, colors_df = transform_artifact_data(artifacts)
    print("Data transformation complete")
    
    # Load
    load_to_database(metadata_df, media_df, colors_df)
    print("ETL pipeline complete!")
    
    return len(artifacts)

# Run it
if __name__ == "__main__":
    total_loaded = run_etl_pipeline(num_pages=10)
    print(f"Successfully loaded {total_loaded} artifacts")
```

## Troubleshooting

**API Key Issues**: Ensure `HARVARD_API_KEY` is set in `.env`. Get a free key at https://harvardartmuseums.org/collections/api

**Database Connection Errors**: Verify all `DB_*` environment variables are correct. Test connection:
```python
connection = get_db_connection()
print("Connected!" if connection.is_connected() else "Failed")
```

**Rate Limiting**: Harvard API has rate limits. Add delays between requests:
```python
time.sleep(0.5)  # 500ms delay between requests
```

**Missing Data**: Some artifacts have null fields. Handle gracefully:
```python
artifact.get('culture', 'Unknown')
```

**Large Dataset Memory**: Process in batches:
```python
for i in range(0, len(artifacts), 100):
    batch = artifacts[i:i+100]
    # Process batch
```
