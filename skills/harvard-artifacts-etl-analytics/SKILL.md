---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with SQL and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums data
  - show me how to use the Harvard artifacts analytics app
  - help me set up data engineering with Harvard API
  - create a Streamlit dashboard for museum artifact data
  - build SQL analytics for Harvard Art Museums collection
  - extract and transform Harvard museum API data
  - visualize artifact data with Plotly and Streamlit
  - implement museum data pipeline with Python
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

An end-to-end data engineering and analytics application that demonstrates real-world ETL pipelines using the Harvard Art Museums API. This project extracts artifact data, transforms it into relational structures, loads it into SQL databases, and provides interactive analytics dashboards via Streamlit.

## What It Does

This application creates a complete data pipeline:
- **Extract**: Fetches artifact data from Harvard Art Museums API with pagination
- **Transform**: Converts nested JSON into normalized relational tables
- **Load**: Batch inserts data into MySQL/TiDB Cloud databases
- **Analyze**: Executes 20+ predefined analytical SQL queries
- **Visualize**: Renders interactive dashboards with Plotly charts

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

## Configuration

### Database Setup

The application uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    classification VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    medium VARCHAR(500),
    technique VARCHAR(500),
    dimensions VARCHAR(500)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl VARCHAR(500),
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

### API Configuration

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

```python
import os
import requests

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(page=1, size=100):
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size
    }
    response = requests.get(BASE_URL, params=params)
    return response.json()
```

## ETL Pipeline Implementation

### Extract Phase

```python
import requests
import time

def extract_paginated_data(max_pages=10):
    """Extract artifact data with pagination handling"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            response = requests.get(
                BASE_URL,
                params={
                    'apikey': os.getenv('HARVARD_API_KEY'),
                    'page': page,
                    'size': 100
                }
            )
            
            if response.status_code == 200:
                data = response.json()
                artifacts = data.get('records', [])
                all_artifacts.extend(artifacts)
                
                # Rate limiting
                time.sleep(0.5)
            else:
                print(f"Error on page {page}: {response.status_code}")
                break
                
        except Exception as e:
            print(f"Exception on page {page}: {str(e)}")
            break
    
    return all_artifacts
```

### Transform Phase

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON into relational dataframes"""
    
    # Transform metadata
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'classification': artifact.get('classification'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique'),
            'dimensions': artifact.get('dimensions')
        })
        
        # Extract media
        images = artifact.get('images', [])
        for img in images:
            media_records.append({
                'artifact_id': artifact.get('id'),
                'media_type': 'image',
                'baseimageurl': img.get('baseimageurl'),
                'format': img.get('format')
            })
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return {
        'metadata': pd.DataFrame(metadata_records),
        'media': pd.DataFrame(media_records),
        'colors': pd.DataFrame(color_records)
    }
```

### Load Phase

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(dataframes):
    """Batch insert transformed data into SQL database"""
    
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        cursor = connection.cursor()
        
        # Load metadata
        metadata_query = """
            INSERT IGNORE INTO artifactmetadata 
            (id, title, culture, classification, century, dated, 
             department, division, medium, technique, dimensions)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """
        metadata_values = dataframes['metadata'].values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Load media
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, media_type, baseimageurl, format)
            VALUES (%s, %s, %s, %s)
        """
        media_values = dataframes['media'].values.tolist()
        cursor.executemany(media_query, media_values)
        
        # Load colors
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        colors_values = dataframes['colors'].values.tolist()
        cursor.executemany(colors_query, colors_values)
        
        connection.commit()
        print(f"Successfully loaded {len(metadata_values)} artifacts")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Top 10 cultures by artifact count
query_cultures = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Artifacts by century distribution
query_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Media availability analysis
query_media = """
    SELECT 
        CASE 
            WHEN COUNT(m.id) > 0 THEN 'Has Media'
            ELSE 'No Media'
        END as media_status,
        COUNT(DISTINCT a.id) as artifact_count
    FROM artifactmetadata a
    LEFT JOIN artifactmedia m ON a.id = m.artifact_id
    GROUP BY media_status
"""

# Color spectrum distribution
query_colors = """
    SELECT spectrum, COUNT(*) as usage_count, AVG(percent) as avg_percent
    FROM artifactcolors
    WHERE spectrum IS NOT NULL
    GROUP BY spectrum
    ORDER BY usage_count DESC
"""

# Department-wise classification
query_dept_class = """
    SELECT department, classification, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL AND classification IS NOT NULL
    GROUP BY department, classification
    ORDER BY department, count DESC
"""
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    else:
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection")
    
    num_pages = st.number_input("Number of pages to fetch", 1, 100, 10)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Extracting data..."):
            raw_data = extract_paginated_data(max_pages=num_pages)
            st.success(f"Extracted {len(raw_data)} artifacts")
        
        with st.spinner("Transforming data..."):
            transformed = transform_artifacts(raw_data)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            load_to_database(transformed)
            st.success("Data loaded to database")

def show_sql_analytics():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Top Cultures": query_cultures,
        "Century Distribution": query_century,
        "Media Availability": query_media,
        "Color Spectrum": query_colors,
        "Department Classification": query_dept_class
    }
    
    selected_query = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        df = execute_query(queries[selected_query])
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1])
            st.plotly_chart(fig)

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    df = pd.read_sql(query, connection)
    connection.close()
    return df

if __name__ == "__main__":
    main()
```

### Running the Dashboard

```bash
streamlit run app.py
```

## Common Patterns

### Error Handling for API Calls

```python
def safe_api_call(url, params, max_retries=3):
    """Robust API call with retry logic"""
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

### Data Validation

```python
def validate_artifact_data(artifact):
    """Validate required fields exist"""
    required_fields = ['id', 'title']
    return all(field in artifact for field in required_fields)

# Use in transform
valid_artifacts = [a for a in raw_data if validate_artifact_data(a)]
```

### Incremental Loading

```python
def get_max_artifact_id():
    """Get highest artifact ID already in database"""
    query = "SELECT MAX(id) FROM artifactmetadata"
    connection = mysql.connector.connect(...)
    cursor = connection.cursor()
    cursor.execute(query)
    result = cursor.fetchone()[0]
    connection.close()
    return result or 0

def incremental_load():
    """Only load new artifacts"""
    max_id = get_max_artifact_id()
    # Filter artifacts with id > max_id
    new_artifacts = [a for a in all_artifacts if a['id'] > max_id]
    return new_artifacts
```

## Troubleshooting

### API Rate Limiting
```python
# Add delays between requests
import time
time.sleep(0.5)  # 500ms between calls

# Or use rate limiter
from ratelimit import limits, sleep_and_retry

@sleep_and_retry
@limits(calls=10, period=1)  # 10 calls per second
def call_api(url, params):
    return requests.get(url, params=params)
```

### Database Connection Issues
```python
# Use connection pooling
from mysql.connector import pooling

connection_pool = pooling.MySQLConnectionPool(
    pool_name="artifacts_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_connection():
    return connection_pool.get_connection()
```

### Memory Management for Large Datasets
```python
# Process in chunks
def load_in_chunks(dataframe, chunk_size=1000):
    for start in range(0, len(dataframe), chunk_size):
        chunk = dataframe[start:start + chunk_size]
        load_to_database({'metadata': chunk, 'media': pd.DataFrame(), 'colors': pd.DataFrame()})
```

### Missing Data Handling
```python
# Fill NaN values before loading
df['culture'] = df['culture'].fillna('Unknown')
df['century'] = df['century'].fillna('Undated')

# Or use NULLIF in SQL
query = "SELECT NULLIF(culture, '') as culture FROM artifactmetadata"
```
