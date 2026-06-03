---
name: harvard-artifacts-etl-analytics
description: ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and analyze Harvard art collection data
  - set up artifact data pipeline with SQL storage
  - visualize museum collection data with Streamlit
  - query Harvard Art Museums API and store results
  - build end-to-end data engineering pipeline for art data
  - analyze artifact metadata and color patterns
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project implements a complete data engineering pipeline for the Harvard Art Museums API. It demonstrates:

- **API Integration**: Fetching paginated artifact data from Harvard Art Museums
- **ETL Pipeline**: Extracting, transforming nested JSON, and loading into relational SQL tables
- **SQL Analytics**: 20+ predefined analytical queries for artifact insights
- **Visualization**: Interactive Streamlit dashboard with Plotly charts

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
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

### Environment Variables

Create a `.env` file in the project root:

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free)
3. Add to your `.env` file

### Database Setup

The project supports MySQL or TiDB Cloud. Create three tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    dated VARCHAR(200),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    division VARCHAR(200),
    objectnumber VARCHAR(100),
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    artifact_id INT,
    media_id INT PRIMARY KEY,
    baseimageurl VARCHAR(500),
    format VARCHAR(50),
    publiccaption TEXT,
    technique VARCHAR(200),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    hue VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

The app will launch at `http://localhost:8501`

## Key Components

### 1. API Data Collection

```python
import requests
import pandas as pd
from dotenv import load_dotenv
import os

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    url = f"https://api.harvardartmuseums.org/object"
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

def collect_artifacts_batch(num_pages=5):
    """Collect multiple pages of artifacts"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(page=page)
        artifacts = data.get('records', [])
        all_artifacts.extend(artifacts)
        print(f"Collected page {page}: {len(artifacts)} artifacts")
    
    return all_artifacts
```

### 2. ETL Transform Functions

```python
def transform_artifact_metadata(artifacts):
    """Transform artifact data into metadata table format"""
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'dated': artifact.get('dated'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'division': artifact.get('division'),
            'objectnumber': artifact.get('objectnumber'),
            'url': artifact.get('url')
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)

def transform_artifact_media(artifacts):
    """Extract media/images data"""
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            record = {
                'artifact_id': artifact_id,
                'media_id': img.get('imageid'),
                'baseimageurl': img.get('baseimageurl'),
                'format': img.get('format'),
                'publiccaption': img.get('publiccaption'),
                'technique': img.get('technique')
            }
            media_records.append(record)
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """Extract color analysis data"""
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            record = {
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'percentage': color.get('percent'),
                'hue': color.get('hue')
            }
            color_records.append(record)
    
    return pd.DataFrame(color_records)
```

### 3. SQL Load Operations

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

def load_to_sql(df, table_name, connection):
    """Batch insert DataFrame to SQL table"""
    cursor = connection.cursor()
    
    # Prepare INSERT statement
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    insert_query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Convert DataFrame to list of tuples
    data = [tuple(row) for row in df.values]
    
    # Execute batch insert
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} rows into {table_name}")
    cursor.close()

def execute_etl_pipeline(num_pages=5):
    """Complete ETL pipeline execution"""
    # Extract
    artifacts = collect_artifacts_batch(num_pages)
    
    # Transform
    metadata_df = transform_artifact_metadata(artifacts)
    media_df = transform_artifact_media(artifacts)
    colors_df = transform_artifact_colors(artifacts)
    
    # Load
    conn = get_db_connection()
    try:
        load_to_sql(metadata_df, 'artifactmetadata', conn)
        load_to_sql(media_df, 'artifactmedia', conn)
        load_to_sql(colors_df, 'artifactcolors', conn)
    finally:
        conn.close()
    
    return len(artifacts)
```

### 4. SQL Analytics Queries

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Top 10 Colors": """
        SELECT color, COUNT(*) as occurrences, 
               AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY occurrences DESC
        LIMIT 10
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Artifacts with Most Images": """
        SELECT a.id, a.title, COUNT(m.media_id) as image_count
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY a.id, a.title
        ORDER BY image_count DESC
        LIMIT 10
    """,
    
    "Classification Analysis": """
        SELECT classification, COUNT(*) as count,
               COUNT(DISTINCT culture) as cultures
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """
}

def execute_query(query_name):
    """Execute analytical query and return results"""
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    
    try:
        cursor.execute(ANALYTICS_QUERIES[query_name])
        results = cursor.fetchall()
        return pd.DataFrame(results)
    finally:
        cursor.close()
        conn.close()
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar - ETL Controls
    st.sidebar.header("Data Collection")
    num_pages = st.sidebar.slider("Pages to Collect", 1, 20, 5)
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            count = execute_etl_pipeline(num_pages)
            st.sidebar.success(f"Loaded {count} artifacts!")
    
    # Main area - Analytics
    st.header("📊 Analytics Queries")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Execute Query"):
        df = execute_query(query_name)
        
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(
                df.head(15), 
                x=df.columns[0], 
                y=df.columns[1],
                title=query_name
            )
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(page, delay=0.5):
    """Add delay between API calls"""
    data = fetch_artifacts(page)
    time.sleep(delay)
    return data
```

### Error Handling

```python
def safe_etl_pipeline(num_pages):
    """ETL with error handling"""
    try:
        artifacts = collect_artifacts_batch(num_pages)
        conn = get_db_connection()
        
        metadata_df = transform_artifact_metadata(artifacts)
        load_to_sql(metadata_df, 'artifactmetadata', conn)
        
        conn.close()
        return True
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return False
    except Error as e:
        print(f"Database Error: {e}")
        return False
```

### Incremental Loading

```python
def get_max_artifact_id(conn):
    """Get the highest artifact ID already in database"""
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_load(artifacts):
    """Only load new artifacts"""
    conn = get_db_connection()
    max_id = get_max_artifact_id(conn)
    
    new_artifacts = [a for a in artifacts if a['id'] > max_id]
    
    if new_artifacts:
        df = transform_artifact_metadata(new_artifacts)
        load_to_sql(df, 'artifactmetadata', conn)
    
    conn.close()
    return len(new_artifacts)
```

## Troubleshooting

### API Key Issues

```python
# Verify API key is loaded
if not os.getenv('HARVARD_API_KEY'):
    raise ValueError("HARVARD_API_KEY not found in environment variables")
```

### Database Connection Failures

```python
# Test connection
try:
    conn = get_db_connection()
    conn.ping(reconnect=True)
    print("Database connection successful")
    conn.close()
except Error as e:
    print(f"Cannot connect to database: {e}")
```

### Handling Missing Data

```python
def safe_get(artifact, key, default=None):
    """Safely extract nested data"""
    return artifact.get(key, default)

# Use in transforms
record = {
    'id': safe_get(artifact, 'id'),
    'title': safe_get(artifact, 'title', 'Unknown'),
    'culture': safe_get(artifact, 'culture', 'Not Specified')
}
```

### Memory Management for Large Datasets

```python
def batch_process_artifacts(total_pages, batch_size=5):
    """Process in smaller batches"""
    for start_page in range(1, total_pages, batch_size):
        end_page = min(start_page + batch_size, total_pages + 1)
        artifacts = collect_artifacts_batch_range(start_page, end_page)
        
        # Process and load batch
        execute_etl_pipeline_for_batch(artifacts)
        
        # Clear memory
        del artifacts
```

## Advanced Usage

### Custom Query Execution

```python
def run_custom_query(sql_query):
    """Execute custom SQL query"""
    conn = get_db_connection()
    df = pd.read_sql(sql_query, conn)
    conn.close()
    return df

# Example
query = """
    SELECT a.culture, c.color, AVG(c.percentage) as avg_pct
    FROM artifactmetadata a
    JOIN artifactcolors c ON a.id = c.artifact_id
    WHERE a.culture IS NOT NULL
    GROUP BY a.culture, c.color
"""
result = run_custom_query(query)
```

### Export Results

```python
def export_query_results(query_name, format='csv'):
    """Export query results to file"""
    df = execute_query(query_name)
    
    if format == 'csv':
        df.to_csv(f"{query_name.replace(' ', '_')}.csv", index=False)
    elif format == 'json':
        df.to_json(f"{query_name.replace(' ', '_')}.json", orient='records')
```

This skill enables AI agents to help developers build complete data engineering pipelines using the Harvard Art Museums API, from data collection through analytics and visualization.
