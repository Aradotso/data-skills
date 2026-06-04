---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifact data
  - create a data engineering project with Harvard API
  - set up artifact collection analytics dashboard
  - implement museum data pipeline with SQL
  - visualize Harvard Art Museums data
  - extract and analyze art collection metadata
  - build a Streamlit analytics app for museum data
  - design SQL schema for artifact collections
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

This is an end-to-end data engineering application that demonstrates professional ETL pipelines using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational database structures, loads data into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics dashboards via Streamlit.

**Core Architecture:** API → ETL → SQL → Analytics → Visualization

The project creates three normalized tables:
- `artifactmetadata` - Core artifact information
- `artifactmedia` - Media files and images
- `artifactcolors` - Color palette data

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required Dependencies:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Get your free API key from: https://harvardartmuseums.org/collections/api

### 2. Database Setup

Set up MySQL or TiDB Cloud database with connection credentials.

### 3. Environment Variables

Create a `.env` file or configure through Streamlit secrets:

```python
# .env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

**Streamlit Secrets Configuration** (`.streamlit/secrets.toml`):
```toml
HARVARD_API_KEY = "your_api_key_here"
DB_HOST = "your_database_host"
DB_PORT = 3306
DB_USER = "your_username"
DB_PASSWORD = "your_password"
DB_NAME = "harvard_artifacts"
```

## Running the Application

```bash
streamlit run app.py
```

Access the dashboard at `http://localhost:8501`

## Core ETL Pipeline Pattern

### Extract: API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Extract artifact data from Harvard Art Museums API
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "page": page,
        "size": size
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv("HARVARD_API_KEY")
data = fetch_artifacts(api_key, page=1, size=100)
artifacts = data.get('records', [])
```

### Transform: JSON to Relational Structure

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform artifact JSON into metadata DataFrame
    """
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'technique': artifact.get('technique'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'creditline': artifact.get('creditline'),
            'description': artifact.get('description')
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)

def transform_artifact_media(artifacts):
    """
    Transform media/image data into separate table
    """
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            record = {
                'artifact_id': artifact_id,
                'image_id': img.get('imageid'),
                'base_url': img.get('baseimageurl'),
                'format': img.get('format'),
                'width': img.get('width'),
                'height': img.get('height'),
                'renditionnumber': img.get('renditionnumber')
            }
            media_records.append(record)
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """
    Transform color data into normalized table
    """
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            record = {
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(record)
    
    return pd.DataFrame(color_records)
```

### Load: SQL Database Operations

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """
    Create MySQL database connection
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def create_tables(connection):
    """
    Create normalized database schema
    """
    cursor = connection.cursor()
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            period VARCHAR(255),
            century VARCHAR(255),
            classification VARCHAR(255),
            technique TEXT,
            dated VARCHAR(255),
            department VARCHAR(255),
            division VARCHAR(255),
            creditline TEXT,
            description TEXT
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_id INT,
            base_url TEXT,
            format VARCHAR(50),
            width INT,
            height INT,
            renditionnumber VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            hue VARCHAR(100),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    connection.commit()
    cursor.close()

def load_dataframe_to_sql(df, table_name, connection):
    """
    Batch insert DataFrame into SQL table
    """
    cursor = connection.cursor()
    
    # Generate INSERT statement
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    insert_query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Batch insert
    data_tuples = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data_tuples)
    
    connection.commit()
    cursor.close()
    
    return cursor.rowcount
```

## Complete ETL Workflow

```python
def run_etl_pipeline(api_key, num_pages=5):
    """
    Complete ETL pipeline execution
    """
    connection = create_database_connection()
    create_tables(connection)
    
    all_metadata = []
    all_media = []
    all_colors = []
    
    # Extract and Transform
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page, size=100)
        artifacts = data.get('records', [])
        
        # Transform
        metadata_df = transform_artifact_metadata(artifacts)
        media_df = transform_artifact_media(artifacts)
        colors_df = transform_artifact_colors(artifacts)
        
        all_metadata.append(metadata_df)
        all_media.append(media_df)
        all_colors.append(colors_df)
    
    # Combine all pages
    final_metadata = pd.concat(all_metadata, ignore_index=True)
    final_media = pd.concat(all_media, ignore_index=True)
    final_colors = pd.concat(all_colors, ignore_index=True)
    
    # Load
    load_dataframe_to_sql(final_metadata, 'artifactmetadata', connection)
    load_dataframe_to_sql(final_media, 'artifactmedia', connection)
    load_dataframe_to_sql(final_colors, 'artifactcolors', connection)
    
    connection.close()
    print("ETL pipeline completed successfully!")
```

## SQL Analytics Queries

### Example Analytical Queries

```python
def execute_sql_query(connection, query):
    """
    Execute SQL query and return results as DataFrame
    """
    return pd.read_sql(query, connection)

# Query 1: Artifacts by Century
query_by_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
LIMIT 10
"""

# Query 2: Top Cultures
query_top_cultures = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 15
"""

# Query 3: Media Availability
query_media_availability = """
SELECT 
    CASE 
        WHEN EXISTS (SELECT 1 FROM artifactmedia WHERE artifactmedia.artifact_id = artifactmetadata.artifact_id)
        THEN 'Has Media'
        ELSE 'No Media'
    END as media_status,
    COUNT(*) as count
FROM artifactmetadata
GROUP BY media_status
"""

# Query 4: Color Distribution
query_color_distribution = """
SELECT color, COUNT(*) as frequency
FROM artifactcolors
GROUP BY color
ORDER BY frequency DESC
LIMIT 10
"""

# Query 5: Department Analysis
query_department_stats = """
SELECT 
    department,
    COUNT(*) as total_artifacts,
    COUNT(DISTINCT classification) as unique_classifications
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY total_artifacts DESC
"""
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password", value=os.getenv("HARVARD_API_KEY"))
    
    # ETL Section
    st.header("📥 ETL Pipeline")
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            run_etl_pipeline(api_key, num_pages)
            st.success("ETL completed successfully!")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    
    connection = create_database_connection()
    
    queries = {
        "Artifacts by Century": query_by_century,
        "Top Cultures": query_top_cultures,
        "Media Availability": query_media_availability,
        "Color Distribution": query_color_distribution,
        "Department Statistics": query_department_stats
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Execute Query"):
        df = execute_sql_query(connection, queries[selected_query])
        
        # Display table
        st.dataframe(df)
        
        # Visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                         title=selected_query)
            st.plotly_chart(fig)
    
    connection.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting and Pagination

```python
import time

def fetch_all_artifacts_with_rate_limiting(api_key, max_pages=10, delay=1):
    """
    Fetch artifacts with rate limiting to avoid API throttling
    """
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page)
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            print(f"Fetched page {page}: {len(artifacts)} artifacts")
            time.sleep(delay)  # Rate limiting
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

### Error Handling in ETL

```python
def safe_etl_execution(api_key, num_pages):
    """
    ETL with comprehensive error handling
    """
    try:
        connection = create_database_connection()
        if not connection:
            raise Exception("Failed to connect to database")
        
        create_tables(connection)
        
        for page in range(1, num_pages + 1):
            try:
                data = fetch_artifacts(api_key, page=page)
                artifacts = data.get('records', [])
                
                if not artifacts:
                    print(f"No artifacts on page {page}")
                    continue
                
                # Transform and load each page independently
                metadata_df = transform_artifact_metadata(artifacts)
                load_dataframe_to_sql(metadata_df, 'artifactmetadata', connection)
                
            except Exception as page_error:
                print(f"Error processing page {page}: {page_error}")
                continue
        
        connection.close()
        return True
        
    except Exception as e:
        print(f"ETL pipeline failed: {e}")
        return False
```

## Troubleshooting

### API Key Issues
- Ensure `HARVARD_API_KEY` is set in environment variables or Streamlit secrets
- Verify API key is valid at https://harvardartmuseums.org/collections/api
- Check for rate limiting (free tier has limits)

### Database Connection Errors
- Verify database credentials in `.env` or secrets.toml
- Ensure database server is running and accessible
- Check firewall rules for remote database connections (TiDB Cloud)
- Test connection with: `mysql -h HOST -u USER -p`

### Data Loading Issues
- Check for NULL values in primary key fields (artifact_id)
- Use `INSERT IGNORE` to skip duplicates
- Verify foreign key constraints match parent table data

### Streamlit Performance
- Limit number of pages fetched (start with 5-10)
- Use `@st.cache_data` for expensive operations
- Batch insert data rather than row-by-row

### Empty Results
- Verify tables exist: `SHOW TABLES;`
- Check table contents: `SELECT COUNT(*) FROM artifactmetadata;`
- Ensure ETL pipeline completed without errors
- Check API response structure hasn't changed
