---
name: harvard-artifacts-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create a data engineering project with Harvard API
  - set up analytics dashboard for museum artifacts
  - extract and transform Harvard Art Museums API data
  - build SQL analytics for art collection data
  - visualize Harvard museum artifacts with Streamlit
  - implement ETL workflow for museum API
  - analyze art museum data with Python and SQL
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build complete ETL (Extract, Transform, Load) pipelines using the Harvard Art Museums API. The project demonstrates production-grade data engineering patterns: API integration with pagination and rate limiting, transformation of nested JSON to relational schemas, SQL database design, analytical query execution, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Connects to Harvard Art Museums API with secure key management and pagination
- **ETL Pipeline**: Extracts artifact metadata, transforms nested JSON into normalized tables, loads into SQL databases
- **Database Design**: Relational schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **SQL Analytics**: 20+ predefined analytical queries for insights on cultures, time periods, media, and colors
- **Interactive Dashboards**: Streamlit-based UI with Plotly visualizations for real-time analytics

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
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Obtain a free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Store credentials in `.env` file:
```bash
HARVARD_API_KEY=your_key_here
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Setup

The project supports MySQL or TiDB Cloud. Create the database schema:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    dated VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    technique VARCHAR(500),
    division VARCHAR(255),
    department VARCHAR(255),
    creditline TEXT,
    accession_number VARCHAR(255),
    object_number VARCHAR(255),
    url TEXT
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_image_url TEXT,
    image_count INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Usage Patterns

### Running the Streamlit App

```bash
streamlit run app.py
```

The app provides three main sections:
1. **Data Collection**: Fetch artifacts from Harvard API
2. **ETL Pipeline**: Transform and load data into SQL
3. **Analytics Dashboard**: Run queries and visualize results

### API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'page': page,
            'size': min(size, num_records - len(artifacts)),
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code != 200:
            raise Exception(f"API Error: {response.status_code}")
        
        data = response.json()
        artifacts.extend(data.get('records', []))
        
        if len(data.get('records', [])) < size:
            break
            
        page += 1
    
    return artifacts[:num_records]
```

### ETL Transformation

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON into normalized DataFrames"""
    
    # Extract metadata
    metadata = []
    media_data = []
    color_data = []
    
    for artifact in raw_data:
        # Metadata table
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'dated': artifact.get('dated'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'technique': artifact.get('technique'),
            'division': artifact.get('division'),
            'department': artifact.get('department'),
            'creditline': artifact.get('creditline'),
            'accession_number': artifact.get('accessionNumber'),
            'object_number': artifact.get('objectNumber'),
            'url': artifact.get('url')
        })
        
        # Media table
        if artifact.get('primaryimageurl'):
            media_data.append({
                'artifact_id': artifact.get('id'),
                'base_image_url': artifact.get('primaryimageurl'),
                'image_count': len(artifact.get('images', []))
            })
        
        # Colors table
        for color in artifact.get('colors', []):
            color_data.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata),
        pd.DataFrame(media_data),
        pd.DataFrame(color_data)
    )
```

### Loading Data to SQL

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df):
    """Batch insert DataFrames into SQL database"""
    
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT', 3306),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        cursor = connection.cursor()
        
        # Insert metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, dated, century, classification, technique, 
         division, department, creditline, accession_number, object_number, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        media_query = """
        INSERT INTO artifactmedia (artifact_id, base_image_url, image_count)
        VALUES (%s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        colors_query = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(colors_query, colors_df.values.tolist())
        
        connection.commit()
        print(f"Inserted {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database Error: {e}")
        connection.rollback()
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### Analytics Queries

```python
def execute_analytics_query(query_name):
    """Execute predefined analytics queries"""
    
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 10
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'top_departments': """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        
        'color_distribution': """
            SELECT c.spectrum, COUNT(*) as count, AVG(c.percent) as avg_percent
            FROM artifactcolors c
            GROUP BY c.spectrum
            ORDER BY count DESC
        """,
        
        'artifacts_with_images': """
            SELECT 
                CASE WHEN m.base_image_url IS NOT NULL THEN 'With Images' 
                     ELSE 'Without Images' END as status,
                COUNT(*) as count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY status
        """
    }
    
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    result_df = pd.read_sql(queries[query_name], connection)
    connection.close()
    
    return result_df
```

### Streamlit Visualization

```python
import streamlit as st
import plotly.express as px

def create_analytics_dashboard():
    """Build interactive analytics dashboard"""
    
    st.title("Harvard Art Museums Analytics")
    
    # Query selector
    query_options = {
        'Artifacts by Culture': 'artifacts_by_culture',
        'Artifacts by Century': 'artifacts_by_century',
        'Top Departments': 'top_departments',
        'Color Distribution': 'color_distribution'
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            result = execute_analytics_query(query_options[selected_query])
            
            # Display table
            st.dataframe(result)
            
            # Auto-generate chart
            if len(result.columns) == 2:
                fig = px.bar(
                    result,
                    x=result.columns[0],
                    y=result.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig)
```

## Common Workflows

### Complete ETL Pipeline

```python
def run_full_etl_pipeline(num_artifacts=500):
    """Execute complete ETL workflow"""
    
    # Extract
    st.info("Step 1: Extracting data from API...")
    raw_data = fetch_artifacts(num_artifacts)
    
    # Transform
    st.info("Step 2: Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_data)
    
    # Load
    st.info("Step 3: Loading to database...")
    load_to_database(metadata_df, media_df, colors_df)
    
    st.success(f"ETL Complete! Processed {len(metadata_df)} artifacts")
    
    return {
        'metadata_records': len(metadata_df),
        'media_records': len(media_df),
        'color_records': len(colors_df)
    }
```

### Incremental Data Updates

```python
def incremental_update():
    """Fetch only new artifacts since last update"""
    
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    last_id = cursor.fetchone()[0] or 0
    
    # Fetch artifacts with ID > last_id
    api_key = os.getenv('HARVARD_API_KEY')
    url = f"https://api.harvardartmuseums.org/object?apikey={api_key}&id={last_id+1}-999999"
    
    response = requests.get(url)
    new_artifacts = response.json().get('records', [])
    
    if new_artifacts:
        metadata_df, media_df, colors_df = transform_artifacts(new_artifacts)
        load_to_database(metadata_df, media_df, colors_df)
    
    connection.close()
    return len(new_artifacts)
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_rate_limit(num_records, delay=1):
    """Add delay between API calls to avoid rate limits"""
    artifacts = []
    page = 1
    
    while len(artifacts) < num_records:
        response = requests.get(base_url, params={'page': page, ...})
        artifacts.extend(response.json().get('records', []))
        
        time.sleep(delay)  # Wait 1 second between requests
        page += 1
    
    return artifacts
```

### Database Connection Issues

```python
def get_db_connection(retries=3):
    """Retry database connection with exponential backoff"""
    for attempt in range(retries):
        try:
            return mysql.connector.connect(
                host=os.getenv('DB_HOST'),
                user=os.getenv('DB_USER'),
                password=os.getenv('DB_PASSWORD'),
                database=os.getenv('DB_NAME')
            )
        except Error as e:
            if attempt < retries - 1:
                time.sleep(2 ** attempt)
            else:
                raise e
```

### Handling Null Values

```python
def clean_dataframe(df):
    """Handle NULL values before database insert"""
    # Replace NaN with None for SQL NULL
    df = df.where(pd.notnull(df), None)
    
    # Truncate long strings
    for col in df.select_dtypes(include=['object']).columns:
        df[col] = df[col].apply(lambda x: str(x)[:500] if x else None)
    
    return df
```

This skill provides everything needed to build production-ready ETL pipelines and analytics dashboards using museum API data, SQL databases, and modern Python visualization tools.
