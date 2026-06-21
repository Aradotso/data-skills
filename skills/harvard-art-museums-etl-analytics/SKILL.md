---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - show me how to query Harvard Art Museums API and store in SQL
  - create analytics dashboard for museum artifact data
  - extract and transform Harvard museum collection data
  - build data engineering pipeline with Streamlit visualization
  - set up SQL database for Harvard Art Museums artifacts
  - analyze museum artifact data with Python and Plotly
  - create end-to-end data pipeline for art collection
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates building production-ready ETL pipelines using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational tables, loads data into SQL databases (MySQL/TiDB), and creates interactive analytics dashboards with Streamlit and Plotly.

## What It Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables (artifactmetadata, artifactmedia, artifactcolors)
- **SQL Storage**: Batch inserts with foreign key relationships for optimal performance
- **Analytics Queries**: 20+ predefined SQL queries for culture, century, media, and color analysis
- **Interactive Dashboards**: Real-time visualization using Streamlit and Plotly

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
export DB_NAME="your_database_name"
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

### API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = 'https://api.harvardartmuseums.org/object'
```

### Database Configuration

```python
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

conn = mysql.connector.connect(**db_config)
cursor = conn.cursor()
```

## Key Components

### 1. Extract: Fetch Data from API

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_records=100):
    """Fetch artifacts with pagination handling"""
    artifacts = []
    page = 1
    per_page = 100
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'page': page,
            'size': per_page
        }
        
        response = requests.get(BASE_URL, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            artifacts.extend(records)
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### 2. Transform: Normalize JSON to Relational Tables

```python
def transform_artifacts(artifacts):
    """Transform nested JSON into relational dataframes"""
    
    # Metadata table
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        
        # Extract metadata
        metadata = {
            'artifact_id': artifact_id,
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions')
        }
        metadata_list.append(metadata)
        
        # Extract media (images)
        images = artifact.get('images', [])
        for img in images:
            media = {
                'artifact_id': artifact_id,
                'image_id': img.get('imageid'),
                'base_url': img.get('baseimageurl'),
                'format': img.get('format'),
                'width': img.get('width'),
                'height': img.get('height')
            }
            media_list.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            colors_list.append(color_data)
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

### 3. Load: Create Tables and Insert Data

```python
def create_tables(cursor):
    """Create database schema"""
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            dated VARCHAR(255),
            medium TEXT,
            dimensions TEXT
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

def batch_insert(cursor, df, table_name):
    """Batch insert dataframe into SQL table"""
    if df.empty:
        return
    
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    
    sql = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Convert dataframe to list of tuples
    data = [tuple(row) for row in df.values]
    
    cursor.executemany(sql, data)
```

### 4. Analytics: SQL Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN image_id IS NOT NULL THEN 'Has Image' ELSE 'No Image' END as media_status,
            COUNT(DISTINCT artifact_id) as count
        FROM artifactmetadata m
        LEFT JOIN artifactmedia med ON m.artifact_id = med.artifact_id
        GROUP BY media_status
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
}

def execute_query(cursor, query):
    """Execute SQL query and return results as dataframe"""
    cursor.execute(query)
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    return pd.DataFrame(results, columns=columns)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("API Key", type="password", value=os.getenv('HARVARD_API_KEY'))
    num_records = st.sidebar.slider("Number of Records", 10, 1000, 100)
    
    # ETL Pipeline
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Fetching data..."):
            artifacts = fetch_artifacts(api_key, num_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata, df_media, df_colors = transform_artifacts(artifacts)
            st.success("Data transformed")
        
        with st.spinner("Loading to database..."):
            conn = mysql.connector.connect(**db_config)
            cursor = conn.cursor()
            
            create_tables(cursor)
            batch_insert(cursor, df_metadata, 'artifactmetadata')
            batch_insert(cursor, df_media, 'artifactmedia')
            batch_insert(cursor, df_colors, 'artifactcolors')
            
            conn.commit()
            cursor.close()
            conn.close()
            st.success("Data loaded to SQL")
    
    # Analytics Section
    st.header("📊 Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        
        df_result = execute_query(cursor, ANALYTICS_QUERIES[query_name])
        
        # Display table
        st.dataframe(df_result)
        
        # Visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, 
                        x=df_result.columns[0], 
                        y=df_result.columns[1],
                        title=query_name)
            st.plotly_chart(fig)
        
        cursor.close()
        conn.close()

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def get_max_artifact_id(cursor):
    """Get the highest artifact_id already in database"""
    cursor.execute("SELECT MAX(artifact_id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def fetch_new_artifacts(api_key, last_id):
    """Fetch only artifacts with ID greater than last_id"""
    params = {
        'apikey': api_key,
        'q': f'id:>{last_id}',
        'size': 100
    }
    response = requests.get(BASE_URL, params=params)
    return response.json().get('records', [])
```

### Pattern 2: Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_pipeline(api_key, num_records):
    """ETL pipeline with error handling"""
    try:
        logger.info(f"Starting ETL for {num_records} records")
        artifacts = fetch_artifacts(api_key, num_records)
        
        logger.info("Transforming data...")
        df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        
        logger.info("Loading to database...")
        # Load operations here
        
        logger.info("ETL completed successfully")
        return True
        
    except requests.RequestException as e:
        logger.error(f"API Error: {e}")
        return False
    except mysql.connector.Error as e:
        logger.error(f"Database Error: {e}")
        return False
    except Exception as e:
        logger.error(f"Unexpected Error: {e}")
        return False
```

### Pattern 3: Data Validation

```python
def validate_artifacts(artifacts):
    """Validate artifact data before transformation"""
    valid_artifacts = []
    
    for artifact in artifacts:
        # Check required fields
        if not artifact.get('id'):
            logger.warning(f"Skipping artifact without ID")
            continue
            
        # Validate data types
        if artifact.get('id') and not isinstance(artifact['id'], int):
            logger.warning(f"Invalid ID type: {artifact['id']}")
            continue
        
        valid_artifacts.append(artifact)
    
    return valid_artifacts
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
        elif response.status_code == 429:  # Rate limit
            wait_time = 2 ** attempt
            logger.warning(f"Rate limited. Waiting {wait_time}s...")
            time.sleep(wait_time)
        else:
            logger.error(f"Error {response.status_code}")
            break
    
    return None
```

### Database Connection Issues

```python
def get_db_connection(max_retries=3):
    """Get database connection with retry logic"""
    for attempt in range(max_retries):
        try:
            conn = mysql.connector.connect(**db_config)
            return conn
        except mysql.connector.Error as e:
            logger.error(f"Connection attempt {attempt + 1} failed: {e}")
            time.sleep(2)
    
    raise Exception("Could not connect to database")
```

### Memory Optimization for Large Datasets

```python
def process_in_chunks(artifacts, chunk_size=100):
    """Process large datasets in chunks"""
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        df_metadata, df_media, df_colors = transform_artifacts(chunk)
        
        # Load chunk to database
        conn = get_db_connection()
        cursor = conn.cursor()
        batch_insert(cursor, df_metadata, 'artifactmetadata')
        batch_insert(cursor, df_media, 'artifactmedia')
        batch_insert(cursor, df_colors, 'artifactcolors')
        conn.commit()
        cursor.close()
        conn.close()
        
        logger.info(f"Processed chunk {i//chunk_size + 1}")
```

## Best Practices

1. **Always use environment variables** for API keys and database credentials
2. **Implement pagination** when fetching large datasets from the API
3. **Use batch inserts** instead of row-by-row inserts for performance
4. **Validate data** before transformation and loading
5. **Add logging** at each ETL stage for debugging
6. **Handle API rate limits** with exponential backoff
7. **Use transactions** for database operations to ensure data integrity
8. **Create indexes** on frequently queried columns (culture, century, department)
