---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with SQL and Streamlit
triggers:
  - how do I fetch data from the Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create a Streamlit dashboard for art collection analytics
  - query Harvard museum artifacts with SQL
  - set up TiDB or MySQL for artifact metadata storage
  - visualize museum collection data with Plotly
  - implement pagination for Harvard Art Museums API
  - analyze artifact colors and media with SQL queries
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering pipelines using the Harvard Art Museums API. It covers extracting artifact data via REST API, transforming JSON into relational structures, loading into SQL databases (MySQL/TiDB), running analytical queries, and visualizing insights with Streamlit and Plotly.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App demonstrates a complete data pipeline:

1. **Extract**: Fetch artifact metadata, media, and color data from Harvard Art Museums API
2. **Transform**: Parse nested JSON and normalize into relational tables
3. **Load**: Batch insert into MySQL/TiDB Cloud with proper schema design
4. **Analyze**: Run 20+ predefined SQL analytics queries
5. **Visualize**: Display results in interactive Streamlit dashboards with Plotly charts

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages typically include:
pip install streamlit pandas requests mysql-connector-python plotly
```

## Configuration

### API Key Setup

Get your API key from [Harvard Art Museums API](https://docs.harvardartmuseums.org/):

```python
# Store in environment variable
export HARVARD_API_KEY='your_api_key_here'

# Or configure in Streamlit app
import os
api_key = os.getenv('HARVARD_API_KEY')
```

### Database Configuration

```python
import mysql.connector

# MySQL/TiDB connection
db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

conn = mysql.connector.connect(**db_config)
```

## Database Schema

Create the following tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
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
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accession_number VARCHAR(100),
    INDEX idx_culture (culture),
    INDEX idx_century (century),
    INDEX idx_classification (classification)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_url VARCHAR(1000),
    media_type VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id),
    INDEX idx_artifact (artifact_id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id),
    INDEX idx_artifact (artifact_id)
);
```

## ETL Pipeline Implementation

### Extract: API Data Fetching

```python
import requests
import time

def fetch_artifacts(api_key, total_records=100, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    
    params = {
        'apikey': api_key,
        'size': page_size,
        'page': 1
    }
    
    while len(artifacts) < total_records:
        try:
            response = requests.get(base_url, params=params, timeout=30)
            response.raise_for_status()
            data = response.json()
            
            records = data.get('records', [])
            if not records:
                break
                
            artifacts.extend(records)
            
            # Check if we've reached the end
            if len(records) < page_size:
                break
                
            params['page'] += 1
            time.sleep(0.5)  # Rate limiting
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {params['page']}: {e}")
            break
    
    return artifacts[:total_records]
```

### Transform: Data Normalization

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform nested JSON into relational dataframes
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
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
            'medium': artifact.get('medium', '')[:500],
            'dimensions': artifact.get('dimensions', '')[:500],
            'creditline': artifact.get('creditline', ''),
            'accession_number': artifact.get('accessionyear', '')[:100]
        }
        metadata_records.append(metadata)
        
        # Extract media (images)
        if 'images' in artifact and artifact['images']:
            for img in artifact['images']:
                media_records.append({
                    'artifact_id': artifact.get('id'),
                    'media_url': img.get('baseimageurl', '')[:1000],
                    'media_type': 'image'
                })
        
        # Extract colors
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                color_records.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color', '')[:50],
                    'percentage': color.get('percent', 0.0)
                })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load: Batch SQL Insert

```python
def load_to_database(conn, metadata_df, media_df, colors_df):
    """
    Load transformed data into SQL database with batch inserts
    """
    cursor = conn.cursor()
    
    # Insert metadata
    metadata_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, division, 
     dated, period, technique, medium, dimensions, creditline, accession_number)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    cursor.executemany(metadata_query, metadata_df.values.tolist())
    
    # Insert media
    if not media_df.empty:
        media_query = """
        INSERT INTO artifactmedia (artifact_id, media_url, media_type)
        VALUES (%s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
    
    # Insert colors
    if not colors_df.empty:
        colors_query = """
        INSERT INTO artifactcolors (artifact_id, color, percentage)
        VALUES (%s, %s, %s)
        """
        cursor.executemany(colors_query, colors_df.values.tolist())
    
    conn.commit()
    cursor.close()
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("API Key", type="password", 
                                     value=os.getenv('HARVARD_API_KEY', ''))
    
    # Database connection
    if st.sidebar.button("Connect to Database"):
        try:
            conn = mysql.connector.connect(**db_config)
            st.sidebar.success("✅ Connected to database")
            st.session_state['db_conn'] = conn
        except Exception as e:
            st.sidebar.error(f"❌ Connection failed: {e}")
    
    # ETL Pipeline
    st.header("📥 Data Collection")
    num_records = st.number_input("Number of records to fetch", 
                                   min_value=10, max_value=1000, value=100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data..."):
            artifacts = fetch_artifacts(api_key, num_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Data transformed")
        
        if 'db_conn' in st.session_state:
            with st.spinner("Loading to database..."):
                load_to_database(st.session_state['db_conn'], 
                                metadata_df, media_df, colors_df)
                st.success("✅ ETL Pipeline completed!")

if __name__ == "__main__":
    main()
```

## Analytics Queries

### Sample SQL Queries

```python
# Query definitions
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Top Colors": """
        SELECT color, COUNT(*) as artifact_count, 
               AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
            COUNT(DISTINCT a.id) - COUNT(DISTINCT m.artifact_id) as artifacts_without_media
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND department != ''
        GROUP BY department
        ORDER BY count DESC
    """
}

def run_analytics_query(conn, query_name):
    """Execute analytics query and return results"""
    cursor = conn.cursor(dictionary=True)
    cursor.execute(ANALYTICS_QUERIES[query_name])
    results = cursor.fetchall()
    cursor.close()
    return pd.DataFrame(results)
```

### Visualization Integration

```python
def visualize_results(df, query_name):
    """Generate Plotly charts from query results"""
    if df.empty:
        st.warning("No data to visualize")
        return
    
    # Auto-detect x and y columns
    columns = df.columns.tolist()
    x_col = columns[0]
    y_col = columns[1] if len(columns) > 1 else columns[0]
    
    fig = px.bar(df, x=x_col, y=y_col, title=query_name)
    fig.update_layout(xaxis_tickangle=-45)
    st.plotly_chart(fig, use_container_width=True)

# Usage in Streamlit
st.header("📊 Analytics Dashboard")
query_choice = st.selectbox("Select Query", list(ANALYTICS_QUERIES.keys()))

if st.button("Run Query"):
    if 'db_conn' in st.session_state:
        results_df = run_analytics_query(st.session_state['db_conn'], query_choice)
        st.dataframe(results_df)
        visualize_results(results_df, query_choice)
```

## Common Patterns

### Error Handling for API Requests

```python
def safe_api_fetch(api_key, page, size=100):
    """Fetch with retry logic and error handling"""
    max_retries = 3
    
    for attempt in range(max_retries):
        try:
            response = requests.get(
                "https://api.harvardartmuseums.org/object",
                params={'apikey': api_key, 'page': page, 'size': size},
                timeout=30
            )
            response.raise_for_status()
            return response.json()
        except requests.exceptions.Timeout:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                time.sleep(5)
                continue
            raise
```

### Database Connection Pooling

```python
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    **db_config
)

def get_connection():
    """Get connection from pool"""
    return db_pool.get_connection()
```

## Troubleshooting

### API Rate Limiting
- Harvard API has rate limits; implement exponential backoff
- Add `time.sleep(0.5)` between requests
- Monitor HTTP 429 responses

### Database Connection Issues
- Verify firewall rules for TiDB Cloud/MySQL port
- Check SSL requirements for cloud databases
- Use connection pooling for concurrent requests

### Memory Issues with Large Datasets
```python
# Process in chunks
def fetch_in_chunks(api_key, total_records, chunk_size=100):
    for offset in range(0, total_records, chunk_size):
        artifacts = fetch_artifacts(api_key, chunk_size)
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        load_to_database(conn, metadata_df, media_df, colors_df)
```

### Streamlit Session State
```python
# Initialize session state
if 'db_conn' not in st.session_state:
    st.session_state['db_conn'] = None

# Clean up on app restart
def cleanup():
    if st.session_state.get('db_conn'):
        st.session_state['db_conn'].close()

st.on_session_end = cleanup
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY='your_api_key'
export DB_HOST='your_db_host'
export DB_USER='your_db_user'
export DB_PASSWORD='your_db_password'
export DB_NAME='harvard_artifacts'

# Launch Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```
