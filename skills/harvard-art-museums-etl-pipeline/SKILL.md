---
name: harvard-art-museums-etl-pipeline
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering app with the Harvard Art Museums API
  - set up artifact collection analytics with SQL and Streamlit
  - analyze Harvard Art Museums data with Python
  - build a museum data pipeline with visualization
  - extract and transform Harvard Art Museums API data
  - create analytics dashboard for museum artifacts
  - implement ETL for art collection data
---

# Harvard Art Museums ETL Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What It Does

The Harvard Art Museums ETL Pipeline:
- Extracts artifact data from the Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into relational database tables (metadata, media, colors)
- Loads structured data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries for insights
- Visualizes results through interactive Streamlit dashboards with Plotly charts

**Architecture:** API → ETL → SQL → Analytics → Visualization

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
DB_HOST=your_db_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

Register at: https://harvardartmuseums.org/collections/api

### Database Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    division VARCHAR(200),
    department VARCHAR(200),
    classification VARCHAR(200),
    medium VARCHAR(500),
    technique VARCHAR(500),
    period VARCHAR(200),
    accession_number VARCHAR(100),
    object_number VARCHAR(100),
    url TEXT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    artifact_id INT,
    image_url TEXT,
    media_type VARCHAR(100),
    is_primary BOOLEAN,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    hex_code VARCHAR(10),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

def collect_all_artifacts(total_records=500):
    """Collect multiple pages of artifacts"""
    artifacts = []
    size = 100
    pages_needed = (total_records + size - 1) // size
    
    for page in range(1, pages_needed + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page, size=size)
        artifacts.extend(data.get('records', []))
        
        if len(artifacts) >= total_records:
            break
    
    return artifacts[:total_records]
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifact_data(artifacts):
    """Transform raw API data into structured dataframes"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'century': artifact.get('century', '')[:100],
            'dated': artifact.get('dated', '')[:200],
            'division': artifact.get('division', '')[:200],
            'department': artifact.get('department', '')[:200],
            'classification': artifact.get('classification', '')[:200],
            'medium': artifact.get('medium', '')[:500],
            'technique': artifact.get('technique', '')[:500],
            'period': artifact.get('period', '')[:200],
            'accession_number': artifact.get('accessionyear', '')[:100],
            'object_number': artifact.get('objectnumber', '')[:100],
            'url': artifact.get('url', '')
        }
        metadata_list.append(metadata)
        
        # Extract media
        for img in artifact.get('images', []):
            media = {
                'artifact_id': artifact.get('id'),
                'image_url': img.get('baseimageurl', ''),
                'media_type': 'image',
                'is_primary': img.get('isprimary', False)
            }
            media_list.append(media)
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_data = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color', ''),
                'hex_code': color.get('hex', ''),
                'percentage': color.get('percent', 0)
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

def load_to_database(df_metadata, df_media, df_colors):
    """Load dataframes into MySQL database"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        if connection.is_connected():
            cursor = connection.cursor()
            
            # Load metadata
            for _, row in df_metadata.iterrows():
                query = """
                INSERT INTO artifactmetadata 
                (id, title, culture, century, dated, division, department, 
                 classification, medium, technique, period, accession_number, 
                 object_number, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
                """
                cursor.execute(query, tuple(row))
            
            # Load media
            for _, row in df_media.iterrows():
                query = """
                INSERT INTO artifactmedia 
                (artifact_id, image_url, media_type, is_primary)
                VALUES (%s, %s, %s, %s)
                """
                cursor.execute(query, tuple(row))
            
            # Load colors
            for _, row in df_colors.iterrows():
                query = """
                INSERT INTO artifactcolors 
                (artifact_id, color, hex_code, percentage)
                VALUES (%s, %s, %s, %s)
                """
                cursor.execute(query, tuple(row))
            
            connection.commit()
            print(f"Loaded {len(df_metadata)} artifacts successfully")
            
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 3. Analytical SQL Queries

```python
# Sample analytical queries
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY artifact_count DESC
    """,
    
    "Artifacts with Images": """
        SELECT 
            CASE WHEN media.artifact_id IS NULL THEN 'No Image' ELSE 'Has Image' END as image_status,
            COUNT(DISTINCT meta.id) as artifact_count
        FROM artifactmetadata meta
        LEFT JOIN artifactmedia media ON meta.id = media.artifact_id
        GROUP BY image_status
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Department": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND department != ''
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Classification Distribution": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL AND classification != ''
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 20
    """
}

def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        df = pd.read_sql(query, connection)
        connection.close()
        return df
    except Error as e:
        print(f"Query error: {e}")
        return pd.DataFrame()
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("End-to-end ETL and Analytics Pipeline")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        st.header("📥 Data Collection from API")
        
        num_records = st.number_input(
            "Number of artifacts to collect",
            min_value=100,
            max_value=5000,
            value=500,
            step=100
        )
        
        if st.button("Start ETL Pipeline"):
            with st.spinner("Fetching artifacts..."):
                artifacts = collect_all_artifacts(num_records)
                st.success(f"Collected {len(artifacts)} artifacts")
            
            with st.spinner("Transforming data..."):
                df_meta, df_media, df_colors = transform_artifact_data(artifacts)
                st.success("Data transformed successfully")
            
            with st.spinner("Loading to database..."):
                load_to_database(df_meta, df_media, df_colors)
                st.success("Data loaded to database")
    
    elif page == "SQL Analytics":
        st.header("📊 SQL Analytics")
        
        query_name = st.selectbox(
            "Select an analytical query",
            list(ANALYTICAL_QUERIES.keys())
        )
        
        if st.button("Execute Query"):
            query = ANALYTICAL_QUERIES[query_name]
            st.code(query, language='sql')
            
            with st.spinner("Executing query..."):
                df_result = execute_query(query)
                
            if not df_result.empty:
                st.dataframe(df_result)
                
                # Auto-generate visualization
                if len(df_result.columns) >= 2:
                    fig = px.bar(
                        df_result,
                        x=df_result.columns[0],
                        y=df_result.columns[1],
                        title=query_name
                    )
                    st.plotly_chart(fig, use_container_width=True)
    
    elif page == "Visualizations":
        st.header("📈 Interactive Visualizations")
        
        # Culture distribution
        df_culture = execute_query(ANALYTICAL_QUERIES["Artifacts by Culture"])
        if not df_culture.empty:
            st.subheader("Artifacts by Culture")
            fig = px.pie(df_culture, values='artifact_count', names='culture')
            st.plotly_chart(fig, use_container_width=True)
        
        # Century timeline
        df_century = execute_query(ANALYTICAL_QUERIES["Artifacts by Century"])
        if not df_century.empty:
            st.subheader("Artifacts by Century")
            fig = px.bar(df_century, x='century', y='artifact_count')
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Batch Processing with Pagination

```python
def batch_collect_artifacts(total_records=1000, batch_size=100):
    """Collect artifacts in batches to manage memory"""
    all_artifacts = []
    
    for start in range(0, total_records, batch_size):
        page = (start // batch_size) + 1
        artifacts = fetch_artifacts(page=page, size=batch_size)
        all_artifacts.extend(artifacts.get('records', []))
        
        # Process batch immediately
        if len(all_artifacts) >= batch_size:
            df_meta, df_media, df_colors = transform_artifact_data(all_artifacts)
            load_to_database(df_meta, df_media, df_colors)
            all_artifacts = []  # Clear memory
    
    return True
```

### Error Handling and Retry Logic

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Retry {attempt + 1} after {wait_time}s...")
            time.sleep(wait_time)
```

### Data Quality Checks

```python
def validate_artifact_data(df_metadata):
    """Validate data quality before loading"""
    issues = []
    
    # Check for duplicate IDs
    if df_metadata['id'].duplicated().any():
        issues.append("Duplicate artifact IDs found")
    
    # Check for null required fields
    if df_metadata['id'].isnull().any():
        issues.append("Missing artifact IDs")
    
    # Check data types
    if not pd.api.types.is_numeric_dtype(df_metadata['id']):
        issues.append("Invalid ID data type")
    
    return issues
```

## Troubleshooting

### API Rate Limiting

```python
import time

def rate_limited_fetch(page, delay=0.5):
    """Add delay between API calls"""
    time.sleep(delay)
    return fetch_artifacts(page=page)
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD')
        )
        if connection.is_connected():
            print("✅ Database connection successful")
            connection.close()
            return True
    except Error as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Memory Management for Large Datasets

```python
def chunked_dataframe_insert(df, chunk_size=1000):
    """Insert large dataframes in chunks"""
    for i in range(0, len(df), chunk_size):
        chunk = df[i:i + chunk_size]
        # Process chunk
        yield chunk
```

### Missing API Key

```python
def validate_config():
    """Validate required environment variables"""
    required_vars = ['HARVARD_API_KEY', 'DB_HOST', 'DB_USER', 'DB_PASSWORD']
    missing = [var for var in required_vars if not os.getenv(var)]
    
    if missing:
        raise EnvironmentError(f"Missing environment variables: {', '.join(missing)}")
```

## Advanced Usage

### Custom Query Builder

```python
def build_custom_query(table, columns, where_clause=None, group_by=None, order_by=None, limit=None):
    """Build custom SQL queries dynamically"""
    query = f"SELECT {', '.join(columns)} FROM {table}"
    
    if where_clause:
        query += f" WHERE {where_clause}"
    if group_by:
        query += f" GROUP BY {group_by}"
    if order_by:
        query += f" ORDER BY {order_by}"
    if limit:
        query += f" LIMIT {limit}"
    
    return query
```

### Export Results

```python
def export_query_results(query, output_format='csv'):
    """Export query results to file"""
    df = execute_query(query)
    
    if output_format == 'csv':
        df.to_csv('query_results.csv', index=False)
    elif output_format == 'json':
        df.to_json('query_results.json', orient='records')
    elif output_format == 'excel':
        df.to_excel('query_results.xlsx', index=False)
```

This skill provides comprehensive coverage of building ETL pipelines with the Harvard Art Museums API, including data extraction, transformation, database loading, SQL analytics, and interactive visualization with Streamlit.
