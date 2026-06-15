---
name: harvard-art-museums-etl-analytics
description: Build end-to-end data engineering pipelines with Harvard Art Museums API, ETL processes, SQL analytics, and Streamlit visualization
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - show me how to create a data analytics dashboard with museum data
  - help me set up a Streamlit app for Harvard artifacts collection
  - how to extract and transform Harvard Art Museums API data
  - build a SQL analytics pipeline for museum artifacts
  - create interactive visualizations for museum collection data
  - set up a data engineering project with art museums API
  - show me how to query and visualize Harvard museum data
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is a complete data pipeline that:

- **Extracts** artifact data from the Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables (metadata, media, colors)
- **Loads** data into SQL databases (MySQL/TiDB Cloud) with batch operations
- **Analyzes** data using 20+ predefined SQL queries
- **Visualizes** insights through interactive Plotly charts in a Streamlit dashboard

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY=your_api_key_here
export DB_HOST=your_database_host
export DB_USER=your_database_user
export DB_PASSWORD=your_database_password
export DB_NAME=harvard_artifacts
```

Required packages:
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

Obtain an API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

Set up MySQL or TiDB Cloud connection:

```python
import mysql.connector
import os

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'port': int(os.getenv('DB_PORT', 3306))
}

connection = mysql.connector.connect(**db_config)
```

## Core Components

### 1. Data Extraction

Extract artifacts from the Harvard API with pagination:

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_pages=5, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    """
    all_records = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': size,
            'page': page
        }
        
        response = requests.get(BASE_URL, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            all_records.extend(records)
            print(f"Fetched page {page}: {len(records)} records")
        else:
            print(f"Error on page {page}: {response.status_code}")
            break
    
    return all_records
```

### 2. Data Transformation

Transform nested JSON into relational tables:

```python
def transform_artifact_metadata(records):
    """
    Extract and transform artifact metadata
    """
    metadata = []
    
    for record in records:
        metadata.append({
            'objectid': record.get('objectid'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'period': record.get('period'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'medium': record.get('medium'),
            'dated': record.get('dated'),
            'department': record.get('department'),
            'division': record.get('division')
        })
    
    return pd.DataFrame(metadata)

def transform_artifact_media(records):
    """
    Extract media/image data
    """
    media = []
    
    for record in records:
        objectid = record.get('objectid')
        images = record.get('images', [])
        
        for img in images:
            media.append({
                'objectid': objectid,
                'imageid': img.get('imageid'),
                'baseimageurl': img.get('baseimageurl'),
                'width': img.get('width'),
                'height': img.get('height'),
                'format': img.get('format')
            })
    
    return pd.DataFrame(media)

def transform_artifact_colors(records):
    """
    Extract color data
    """
    colors = []
    
    for record in records:
        objectid = record.get('objectid')
        color_list = record.get('colors', [])
        
        for color in color_list:
            colors.append({
                'objectid': objectid,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(colors)
```

### 3. Database Schema Setup

Create normalized tables with proper relationships:

```python
def create_tables(connection):
    """
    Create database schema for artifact data
    """
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            medium TEXT,
            dated VARCHAR(200),
            department VARCHAR(200),
            division VARCHAR(200)
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            imageid INT,
            baseimageurl VARCHAR(500),
            width INT,
            height INT,
            format VARCHAR(50),
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            hue VARCHAR(100),
            percent FLOAT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    connection.commit()
    cursor.close()
```

### 4. Data Loading

Batch insert data into SQL database:

```python
def load_metadata(df, connection):
    """
    Load artifact metadata into database
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, period, century, classification, medium, dated, department, division)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    # Convert DataFrame to list of tuples
    data = df.fillna('').values.tolist()
    
    # Batch insert
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
    
    print(f"Loaded {len(data)} metadata records")

def load_media(df, connection):
    """
    Load artifact media into database
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (objectid, imageid, baseimageurl, width, height, format)
        VALUES (%s, %s, %s, %s, %s, %s)
    """
    
    data = df.fillna('').values.tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
    
    print(f"Loaded {len(data)} media records")

def load_colors(df, connection):
    """
    Load artifact colors into database
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors 
        (objectid, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    data = df.fillna(0).values.tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
    
    print(f"Loaded {len(data)} color records")
```

### 5. SQL Analytics Queries

Example analytical queries:

```python
# Query 1: Artifacts by Culture
query_culture = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL AND culture != ''
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query 2: Artifacts by Century
query_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL AND century != ''
    GROUP BY century
    ORDER BY count DESC
"""

# Query 3: Media Availability
query_media = """
    SELECT 
        CASE WHEN COUNT(m.objectid) > 0 THEN 'Has Images' ELSE 'No Images' END as media_status,
        COUNT(DISTINCT a.objectid) as artifact_count
    FROM artifactmetadata a
    LEFT JOIN artifactmedia m ON a.objectid = m.objectid
    GROUP BY media_status
"""

# Query 4: Top Colors Used
query_colors = """
    SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
    FROM artifactcolors
    WHERE color IS NOT NULL
    GROUP BY color
    ORDER BY usage_count DESC
    LIMIT 15
"""

# Query 5: Department Distribution
query_department = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL AND department != ''
    GROUP BY department
    ORDER BY count DESC
"""

def execute_query(connection, query):
    """
    Execute SQL query and return DataFrame
    """
    cursor = connection.cursor()
    cursor.execute(query)
    
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results, columns=columns)
```

### 6. Streamlit Dashboard

Build an interactive analytics dashboard:

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    connection = mysql.connector.connect(**db_config)
    
    # Query selection
    queries = {
        "Artifacts by Culture": query_culture,
        "Artifacts by Century": query_century,
        "Media Availability": query_media,
        "Top Colors Used": query_colors,
        "Department Distribution": query_department
    }
    
    selected_query = st.sidebar.selectbox("Select Analysis", list(queries.keys()))
    
    # Execute query
    if st.sidebar.button("Run Analysis"):
        with st.spinner("Executing query..."):
            df = execute_query(connection, queries[selected_query])
            
            st.subheader(selected_query)
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df,
                    x=df.columns[0],
                    y=df.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig)
    
    connection.close()

if __name__ == "__main__":
    main()
```

## Complete ETL Pipeline

Full pipeline orchestration:

```python
import os
from dotenv import load_dotenv
import mysql.connector
import requests
import pandas as pd

def run_etl_pipeline():
    """
    Complete ETL pipeline execution
    """
    # Load configuration
    load_dotenv()
    api_key = os.getenv('HARVARD_API_KEY')
    
    # Database connection
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    connection = mysql.connector.connect(**db_config)
    
    # Step 1: Create tables
    print("Creating database schema...")
    create_tables(connection)
    
    # Step 2: Extract
    print("Extracting data from API...")
    records = fetch_artifacts(api_key, num_pages=5, size=100)
    
    # Step 3: Transform
    print("Transforming data...")
    df_metadata = transform_artifact_metadata(records)
    df_media = transform_artifact_media(records)
    df_colors = transform_artifact_colors(records)
    
    # Step 4: Load
    print("Loading data into database...")
    load_metadata(df_metadata, connection)
    load_media(df_media, connection)
    load_colors(df_colors, connection)
    
    # Close connection
    connection.close()
    print("ETL pipeline completed successfully!")

if __name__ == "__main__":
    run_etl_pipeline()
```

## Running the Application

```bash
# Run the complete ETL pipeline
python etl_pipeline.py

# Launch the Streamlit dashboard
streamlit run app.py
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(api_key, num_pages=5, delay=1):
    """
    Fetch data with rate limiting
    """
    all_records = []
    
    for page in range(1, num_pages + 1):
        response = requests.get(BASE_URL, params={'apikey': api_key, 'page': page})
        
        if response.status_code == 200:
            all_records.extend(response.json().get('records', []))
        
        # Rate limiting
        time.sleep(delay)
    
    return all_records
```

### Error Handling

```python
def safe_api_call(url, params, max_retries=3):
    """
    API call with retry logic
    """
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            print(f"Attempt {attempt + 1} failed: {e}")
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
```

## Troubleshooting

### API Key Issues
- Verify API key is set: `echo $HARVARD_API_KEY`
- Request a new key from Harvard Art Museums API portal
- Check `.env` file is in project root and properly formatted

### Database Connection Errors
- Verify database credentials in environment variables
- Check firewall rules allow connection to database host
- Ensure database exists: `CREATE DATABASE harvard_artifacts;`
- Test connection: `mysql -h $DB_HOST -u $DB_USER -p`

### Data Loading Failures
- Check for duplicate `objectid` values in metadata
- Verify foreign key constraints are not violated
- Use `ON DUPLICATE KEY UPDATE` for upsert operations
- Check data types match schema definitions

### Streamlit Issues
- Clear cache: `streamlit cache clear`
- Check port availability: `lsof -i :8501`
- Run with verbose logging: `streamlit run app.py --logger.level=debug`

This skill provides comprehensive guidance for building production-ready data engineering pipelines with museum collection data, SQL analytics, and interactive visualizations.
