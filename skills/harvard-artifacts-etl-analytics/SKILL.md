---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create a data engineering app with Harvard artifacts API
  - set up analytics dashboard for museum collection data
  - implement SQL analytics on Harvard art collection
  - build Streamlit app for artifact visualization
  - extract and transform Harvard museum API data
  - create end-to-end data pipeline for art collection
  - analyze Harvard artifacts with Python and SQL
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualizations using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color data into relational SQL databases
- **SQL Analytics**: Executes 20+ predefined analytical queries for insights
- **Interactive Dashboards**: Visualizes query results using Streamlit and Plotly
- **Database Design**: Implements proper relational schema with foreign keys

**Architecture Flow**: API → ETL → SQL (MySQL/TiDB) → Analytics → Visualization (Streamlit)

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

**Required packages**:
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access the app at http://localhost:8501
```

## Configuration

### API Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

```python
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

```python
import mysql.connector
import os

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

connection = mysql.connector.connect(**db_config)
cursor = connection.cursor()
```

## Database Schema

### Create Tables

```python
def create_tables(cursor):
    """Create the relational database schema"""
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            division VARCHAR(200),
            dated VARCHAR(200),
            medium VARCHAR(500),
            technique VARCHAR(500),
            period VARCHAR(200),
            accession_year INT,
            primary_image_url TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            media_url TEXT,
            media_type VARCHAR(100),
            height INT,
            width INT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_hex VARCHAR(10),
            color_percent FLOAT,
            css3_color VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    cursor.connection.commit()
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_records=100):
    """Extract artifact data from Harvard API with pagination"""
    
    artifacts = []
    page = 1
    per_page = 100  # API limit
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'page': page,
            'size': per_page,
            'hasimage': 1  # Only get artifacts with images
        }
        
        response = requests.get(BASE_URL, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            artifacts.extend(records)
            
            if len(records) < per_page:
                break  # No more data
                
            page += 1
            time.sleep(0.5)  # Rate limiting
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### Transform: Process JSON Data

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform raw API data into structured metadata"""
    
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown')[:500],
            'culture': artifact.get('culture', 'Unknown')[:200],
            'century': artifact.get('century', 'Unknown')[:100],
            'classification': artifact.get('classification', 'Unknown')[:200],
            'department': artifact.get('department', 'Unknown')[:200],
            'division': artifact.get('division', 'Unknown')[:200],
            'dated': artifact.get('dated', 'Unknown')[:200],
            'medium': artifact.get('medium', 'Unknown')[:500],
            'technique': artifact.get('technique', 'Unknown')[:500],
            'period': artifact.get('period', 'Unknown')[:200],
            'accession_year': artifact.get('accessionyear'),
            'primary_image_url': artifact.get('primaryimageurl', '')
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)

def transform_artifact_media(artifacts):
    """Extract media information from artifacts"""
    
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            record = {
                'artifact_id': artifact_id,
                'media_url': img.get('baseimageurl'),
                'media_type': img.get('format'),
                'height': img.get('height'),
                'width': img.get('width')
            }
            media_records.append(record)
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """Extract color palette information"""
    
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            record = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent'),
                'css3_color': color.get('css3')
            }
            color_records.append(record)
    
    return pd.DataFrame(color_records)
```

### Load: Batch Insert into SQL

```python
def load_metadata(cursor, df):
    """Batch insert metadata into database"""
    
    query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, division, 
         dated, medium, technique, period, accession_year, primary_image_url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(query, records)
    cursor.connection.commit()

def load_media(cursor, df):
    """Batch insert media data"""
    
    query = """
        INSERT INTO artifactmedia (artifact_id, media_url, media_type, height, width)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(query, records)
    cursor.connection.commit()

def load_colors(cursor, df):
    """Batch insert color data"""
    
    query = """
        INSERT INTO artifactcolors (artifact_id, color_hex, color_percent, css3_color)
        VALUES (%s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(query, records)
    cursor.connection.commit()
```

## Complete ETL Pipeline

```python
def run_etl_pipeline(api_key, db_config, num_records=100):
    """Execute the complete ETL pipeline"""
    
    # Extract
    print(f"Extracting {num_records} artifacts from Harvard API...")
    artifacts = fetch_artifacts(api_key, num_records)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(artifacts)
    media_df = transform_artifact_media(artifacts)
    colors_df = transform_artifact_colors(artifacts)
    
    # Load
    print("Loading data into database...")
    connection = mysql.connector.connect(**db_config)
    cursor = connection.cursor()
    
    create_tables(cursor)
    load_metadata(cursor, metadata_df)
    load_media(cursor, media_df)
    load_colors(cursor, colors_df)
    
    cursor.close()
    connection.close()
    
    print(f"ETL complete! Loaded {len(metadata_df)} artifacts.")
    return metadata_df, media_df, colors_df
```

## SQL Analytics Queries

### Example Analytical Queries

```python
# Query 1: Top 10 cultures by artifact count
query_top_cultures = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture != 'Unknown'
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query 2: Artifacts by century
query_by_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century != 'Unknown'
    GROUP BY century
    ORDER BY count DESC
"""

# Query 3: Media availability analysis
query_media_stats = """
    SELECT 
        am.classification,
        COUNT(DISTINCT am.id) as total_artifacts,
        COUNT(DISTINCT media.artifact_id) as artifacts_with_media,
        ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as media_percentage
    FROM artifactmetadata am
    LEFT JOIN artifactmedia media ON am.id = media.artifact_id
    GROUP BY am.classification
    ORDER BY total_artifacts DESC
    LIMIT 10
"""

# Query 4: Most common colors across artifacts
query_top_colors = """
    SELECT css3_color, COUNT(*) as usage_count, AVG(color_percent) as avg_percentage
    FROM artifactcolors
    WHERE css3_color IS NOT NULL
    GROUP BY css3_color
    ORDER BY usage_count DESC
    LIMIT 10
"""

# Query 5: Artifacts by department and division
query_department_division = """
    SELECT department, division, COUNT(*) as count
    FROM artifactmetadata
    WHERE department != 'Unknown' AND division != 'Unknown'
    GROUP BY department, division
    ORDER BY count DESC
"""

# Query 6: Accession trends over time
query_accession_trends = """
    SELECT accession_year, COUNT(*) as artifacts_acquired
    FROM artifactmetadata
    WHERE accession_year IS NOT NULL
    GROUP BY accession_year
    ORDER BY accession_year DESC
"""
```

### Execute Query and Return DataFrame

```python
def execute_query(cursor, query):
    """Execute SQL query and return results as DataFrame"""
    
    cursor.execute(query)
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar for configuration
    st.sidebar.header("Configuration")
    
    # ETL Section
    st.header("1️⃣ ETL Pipeline")
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            api_key = os.getenv('HARVARD_API_KEY')
            db_config = {
                'host': os.getenv('DB_HOST'),
                'user': os.getenv('DB_USER'),
                'password': os.getenv('DB_PASSWORD'),
                'database': os.getenv('DB_NAME')
            }
            
            metadata_df, media_df, colors_df = run_etl_pipeline(api_key, db_config, 100)
            st.success(f"Loaded {len(metadata_df)} artifacts!")
    
    # Analytics Section
    st.header("2️⃣ SQL Analytics")
    
    connection = mysql.connector.connect(**db_config)
    cursor = connection.cursor()
    
    # Query selector
    queries = {
        "Top Cultures": query_top_cultures,
        "Artifacts by Century": query_by_century,
        "Media Statistics": query_media_stats,
        "Top Colors": query_top_colors,
        "Department Analysis": query_department_division,
        "Accession Trends": query_accession_trends
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Execute Query"):
        query = queries[selected_query]
        df = execute_query(cursor, query)
        
        # Display results
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Visualization
        if len(df) > 0:
            st.subheader("Visualization")
            
            # Auto-generate chart based on data
            if len(df.columns) >= 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                           title=f"{selected_query} Analysis")
                st.plotly_chart(fig, use_container_width=True)
    
    cursor.close()
    connection.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Error Handling in ETL

```python
def safe_extract(api_key, num_records):
    """Extract with error handling and retry logic"""
    
    max_retries = 3
    retry_count = 0
    
    while retry_count < max_retries:
        try:
            artifacts = fetch_artifacts(api_key, num_records)
            return artifacts
        except requests.exceptions.RequestException as e:
            retry_count += 1
            print(f"Retry {retry_count}/{max_retries}: {e}")
            time.sleep(2 ** retry_count)  # Exponential backoff
    
    raise Exception("Failed to extract data after retries")
```

### Data Quality Checks

```python
def validate_data(df):
    """Perform data quality checks"""
    
    checks = {
        'null_ids': df['id'].isnull().sum(),
        'duplicate_ids': df['id'].duplicated().sum(),
        'missing_titles': (df['title'] == 'Unknown').sum(),
        'total_records': len(df)
    }
    
    print(f"Data Quality Report: {checks}")
    return checks
```

### Incremental Load Pattern

```python
def get_max_artifact_id(cursor):
    """Get the highest artifact ID already loaded"""
    
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_etl(api_key, db_config):
    """Load only new artifacts"""
    
    connection = mysql.connector.connect(**db_config)
    cursor = connection.cursor()
    
    max_id = get_max_artifact_id(cursor)
    
    # Fetch artifacts with ID > max_id
    params = {'apikey': api_key, 'after': max_id}
    # ... rest of ETL logic
```

## Troubleshooting

### API Rate Limiting

```python
# Add delays between requests
time.sleep(0.5)  # 500ms between calls

# Use session for connection pooling
session = requests.Session()
session.headers.update({'User-Agent': 'Harvard-ETL-App/1.0'})
```

### Database Connection Issues

```python
# Test connection
try:
    connection = mysql.connector.connect(**db_config)
    connection.ping(reconnect=True)
    print("Database connected successfully")
except mysql.connector.Error as e:
    print(f"Database error: {e}")
```

### Memory Issues with Large Datasets

```python
# Process in chunks
def chunked_load(df, chunk_size=1000):
    """Load data in chunks to avoid memory issues"""
    
    for start in range(0, len(df), chunk_size):
        chunk = df.iloc[start:start + chunk_size]
        load_metadata(cursor, chunk)
        print(f"Loaded chunk {start//chunk_size + 1}")
```

### Missing or Null Values

```python
# Handle nulls during transformation
def safe_get(data, key, default='Unknown'):
    """Safely extract data with fallback"""
    value = data.get(key, default)
    return value if value else default
```

This skill provides AI coding agents with comprehensive knowledge to help developers build, customize, and troubleshoot Harvard Art Museums ETL and analytics applications.
