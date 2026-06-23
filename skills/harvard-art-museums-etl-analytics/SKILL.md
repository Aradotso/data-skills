---
name: harvard-art-museums-etl-analytics
description: ETL pipeline and analytics app using Harvard Art Museums API with Python, SQL, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - create analytics dashboard with Harvard Art Museums API
  - set up data engineering workflow with Streamlit
  - extract and analyze art collection metadata
  - build SQL analytics for museum artifacts
  - create interactive visualization for art data
  - develop end-to-end data pipeline for cultural heritage
  - implement batch processing for museum API data
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an end-to-end data engineering and analytics application that extracts artifact data from the Harvard Art Museums API, transforms it into relational database tables, and provides interactive SQL analytics through a Streamlit dashboard.

## What It Does

- **Data Collection**: Fetches artifact metadata, media details, and color information from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON responses into normalized relational tables
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper foreign key relationships
- **Analytics**: Executes 20+ predefined SQL queries for artifact analysis
- **Visualization**: Generates interactive Plotly charts based on query results

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
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Register for a free API key
3. Add to your `.env` file

### Database Setup

```sql
CREATE DATABASE harvard_artifacts;

USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    provenance TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl VARCHAR(500),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

The app will open at `http://localhost:8501`

## Key Code Patterns

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
    per_page = 100
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': per_page,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data['records'])
            
            if len(data['records']) == 0:
                break
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### Data Transformation

```python
import pandas as pd

def transform_artifact_data(artifacts):
    """Transform API response into dataframes for each table"""
    
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'provenance': artifact.get('provenance')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        if 'images' in artifact and artifact['images']:
            for image in artifact['images']:
                media = {
                    'artifact_id': artifact.get('id'),
                    'media_type': 'image',
                    'baseimageurl': image.get('baseimageurl'),
                    'height': image.get('height'),
                    'width': image.get('width')
                }
                media_list.append(media)
        
        # Extract color information
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                color_data = {
                    'artifact_id': artifact.get('id'),
                    'color_hex': color.get('hex'),
                    'color_percent': color.get('percent')
                }
                colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Database Loading

```python
import mysql.connector
from mysql.connector import Error

def create_db_connection():
    """Create database connection"""
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

def load_to_database(metadata_df, media_df, colors_df):
    """Batch insert data into SQL database"""
    connection = create_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Insert metadata (batch)
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, division, 
             dated, period, technique, medium, dimensions, creditline, provenance)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        if not media_df.empty:
            media_query = """
                INSERT INTO artifactmedia 
                (artifact_id, media_type, baseimageurl, height, width)
                VALUES (%s, %s, %s, %s, %s)
            """
            cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        if not colors_df.empty:
            colors_query = """
                INSERT INTO artifactcolors 
                (artifact_id, color_hex, color_percent)
                VALUES (%s, %s, %s)
            """
            cursor.executemany(colors_query, colors_df.values.tolist())
        
        connection.commit()
        return True
        
    except Error as e:
        print(f"Error loading data: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

### Streamlit Analytics Dashboard

```python
import streamlit as st
import plotly.express as px

def run_analytics_query(query_name, query_sql):
    """Execute SQL query and return results"""
    connection = create_db_connection()
    if not connection:
        return None
    
    try:
        df = pd.read_sql(query_sql, connection)
        return df
    except Error as e:
        st.error(f"Query error: {e}")
        return None
    finally:
        connection.close()

# Sample analytics queries
ANALYTICS_QUERIES = {
    "Artifacts by Department": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Top Cultures": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT am.id) as artifacts_with_media,
            (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts
        FROM artifactmetadata am
        INNER JOIN artifactmedia med ON am.id = med.artifact_id
    """,
    
    "Top Color Usage": """
        SELECT color_hex, COUNT(*) as usage_count, AVG(color_percent) as avg_percent
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY usage_count DESC
        LIMIT 10
    """
}

# Streamlit app
st.title("Harvard Art Museums Analytics Dashboard")

query_choice = st.selectbox("Select Analytics Query", list(ANALYTICS_QUERIES.keys()))

if st.button("Run Query"):
    with st.spinner("Executing query..."):
        results = run_analytics_query(query_choice, ANALYTICS_QUERIES[query_choice])
        
        if results is not None and not results.empty:
            st.dataframe(results)
            
            # Auto-generate visualization
            if len(results.columns) == 2:
                fig = px.bar(results, x=results.columns[0], y=results.columns[1],
                           title=query_choice)
                st.plotly_chart(fig)
```

## Common Workflows

### Complete ETL Pipeline

```python
def run_etl_pipeline(num_records=100):
    """Complete ETL pipeline execution"""
    
    # Extract
    st.info("Extracting data from API...")
    artifacts = fetch_artifacts(num_records)
    
    # Transform
    st.info("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifact_data(artifacts)
    
    # Load
    st.info("Loading to database...")
    success = load_to_database(metadata_df, media_df, colors_df)
    
    if success:
        st.success(f"Successfully loaded {len(metadata_df)} artifacts!")
    else:
        st.error("ETL pipeline failed")
    
    return success
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        response = requests.get(url, params=params)
        
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 429:  # Rate limited
            wait_time = 2 ** attempt
            time.sleep(wait_time)
        else:
            break
    
    return None
```

### Database Connection Issues

```python
# Test connection
def test_db_connection():
    try:
        conn = create_db_connection()
        if conn and conn.is_connected():
            print("Database connection successful")
            conn.close()
            return True
    except Exception as e:
        print(f"Connection failed: {e}")
    return False
```

### Handling Missing Data

```python
# Clean data before insert
def clean_dataframe(df):
    """Handle NULL values and data types"""
    df = df.fillna('')
    df = df.replace({pd.NA: '', None: ''})
    return df
```

## Performance Optimization

### Batch Processing

```python
def batch_insert(cursor, query, data, batch_size=1000):
    """Insert data in batches for better performance"""
    for i in range(0, len(data), batch_size):
        batch = data[i:i + batch_size]
        cursor.executemany(query, batch)
```

This skill enables AI coding agents to help developers build production-ready ETL pipelines for cultural heritage data with proper database design, API integration, and interactive analytics.
