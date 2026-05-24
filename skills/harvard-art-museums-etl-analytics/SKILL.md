---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build ETL pipeline for museum data
  - extract Harvard Art Museums API data
  - create data engineering project with Streamlit
  - set up SQL analytics for art collection
  - visualize museum artifacts with Plotly
  - implement batch data pipeline for API
  - analyze art museum metadata with SQL
  - create interactive dashboard for cultural data
---

# Harvard Art Museums ETL & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides a complete data engineering pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational tables, loads it into SQL databases (MySQL/TiDB), and visualizes analytics through an interactive Streamlit dashboard.

## What It Does

- **API Integration**: Collects artifact metadata, media, and color data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into structured relational tables (artifactmetadata, artifactmedia, artifactcolors)
- **SQL Storage**: Batch inserts data into MySQL/TiDB with proper foreign key relationships
- **Analytics**: Executes 20+ predefined SQL queries for insights on culture, century, media, departments, and colors
- **Visualization**: Interactive Plotly charts and Streamlit dashboard for real-time analytics

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
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

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Connection
DB_HOST=your_db_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Register for a free API key
3. Add to `.env` file

### Database Setup

```sql
CREATE DATABASE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    url TEXT
);

CREATE TABLE artifactmedia (
    artifact_id INT,
    media_id VARCHAR(100),
    media_url TEXT,
    media_type VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

Access the dashboard at `http://localhost:8501`

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
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

def collect_all_artifacts(api_key, max_pages=10):
    """Collect artifacts across multiple pages with rate limiting"""
    import time
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(data['records'])
        
        # Rate limiting
        time.sleep(1)
        
        if page >= data['info']['pages']:
            break
    
    return all_artifacts
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform nested JSON to relational dataframes"""
    
    # Metadata table
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url')
        })
        
        # Extract media
        if artifact.get('images'):
            for image in artifact['images']:
                media_records.append({
                    'artifact_id': artifact['id'],
                    'media_id': image.get('imageid'),
                    'media_url': image.get('baseimageurl'),
                    'media_type': 'image'
                })
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_records.append({
                    'artifact_id': artifact['id'],
                    'color': color.get('color'),
                    'color_percent': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def create_db_connection():
    """Create MySQL/TiDB connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def batch_insert_dataframe(connection, df, table_name, batch_size=1000):
    """Batch insert dataframe into SQL table"""
    cursor = connection.cursor()
    
    # Prepare INSERT statement
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Batch insert
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        values = [tuple(row) for row in batch.values]
        cursor.executemany(query, values)
        connection.commit()
        print(f"Inserted {len(values)} records into {table_name}")
    
    cursor.close()

def load_artifacts_to_db(metadata_df, media_df, colors_df):
    """Load all dataframes to database"""
    conn = create_db_connection()
    if not conn:
        return False
    
    try:
        batch_insert_dataframe(conn, metadata_df, 'artifactmetadata')
        batch_insert_dataframe(conn, media_df, 'artifactmedia')
        batch_insert_dataframe(conn, colors_df, 'artifactcolors')
        return True
    finally:
        conn.close()
```

### 4. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
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
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Most Common Colors": """
        SELECT color, COUNT(*) as usage_count, 
               AVG(color_percent) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Artifacts with Most Media": """
        SELECT am.id, am.title, COUNT(med.media_id) as media_count
        FROM artifactmetadata am
        JOIN artifactmedia med ON am.id = med.artifact_id
        GROUP BY am.id, am.title
        ORDER BY media_count DESC
        LIMIT 10
    """
}

def execute_query(query):
    """Execute SQL query and return results as dataframe"""
    conn = create_db_connection()
    if not conn:
        return None
    
    try:
        df = pd.read_sql(query, conn)
        return df
    finally:
        conn.close()
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    selected_query = st.sidebar.selectbox(
        "Choose an analysis:",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Execute selected query
    if st.sidebar.button("Run Query"):
        with st.spinner("Executing query..."):
            df = execute_query(ANALYTICS_QUERIES[selected_query])
            
            if df is not None and not df.empty:
                st.subheader(f"Results: {selected_query}")
                
                # Display data table
                st.dataframe(df, use_container_width=True)
                
                # Auto-generate visualization
                if len(df.columns) >= 2:
                    fig = px.bar(
                        df.head(15), 
                        x=df.columns[0], 
                        y=df.columns[1],
                        title=selected_query
                    )
                    st.plotly_chart(fig, use_container_width=True)
            else:
                st.warning("No data returned")
    
    # ETL Pipeline section
    st.sidebar.markdown("---")
    st.sidebar.header("ETL Pipeline")
    
    if st.sidebar.button("Run ETL"):
        with st.spinner("Running ETL pipeline..."):
            api_key = os.getenv('HARVARD_API_KEY')
            
            # Extract
            artifacts = collect_all_artifacts(api_key, max_pages=5)
            st.info(f"Extracted {len(artifacts)} artifacts")
            
            # Transform
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.info(f"Transformed into {len(metadata_df)} metadata, {len(media_df)} media, {len(colors_df)} color records")
            
            # Load
            success = load_artifacts_to_db(metadata_df, media_df, colors_df)
            if success:
                st.success("ETL pipeline completed successfully!")
            else:
                st.error("ETL pipeline failed")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_artifact_id(connection):
    """Get the most recent artifact ID in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_etl(api_key, last_id):
    """Only fetch artifacts newer than last_id"""
    # Modify API call to filter by ID or date
    pass
```

### Error Handling for API Calls

```python
def safe_api_call(url, params, max_retries=3):
    """API call with retry logic"""
    import time
    
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

### API Rate Limiting
**Issue**: HTTP 429 errors
**Solution**: Add delays between requests
```python
import time
time.sleep(1)  # 1 second between requests
```

### Database Connection Timeout
**Issue**: Lost connection to MySQL
**Solution**: Use connection pooling
```python
from mysql.connector import pooling

connection_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)
```

### Memory Issues with Large Datasets
**Issue**: Out of memory with large API responses
**Solution**: Process in smaller chunks
```python
def process_in_chunks(artifacts, chunk_size=500):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        metadata_df, media_df, colors_df = transform_artifacts(chunk)
        load_artifacts_to_db(metadata_df, media_df, colors_df)
```

### NULL Values in SQL Queries
**Issue**: Aggregations skewed by NULL values
**Solution**: Filter NULLs in WHERE clauses
```sql
SELECT culture, COUNT(*) 
FROM artifactmetadata 
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
```

This skill enables AI agents to help developers build complete data engineering pipelines using the Harvard Art Museums API with proper ETL practices, SQL analytics, and interactive visualization.
