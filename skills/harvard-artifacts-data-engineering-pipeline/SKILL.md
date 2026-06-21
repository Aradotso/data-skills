---
name: harvard-artifacts-data-engineering-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Streamlit, SQL, and Plotly
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - create an ETL workflow for museum artifact data
  - set up analytics dashboard for Harvard artifacts
  - build a Streamlit app with museum collection data
  - query and visualize Harvard Art Museums data
  - implement SQL analytics for artifact collections
  - extract and transform museum API data
  - create interactive visualizations for artifact metadata
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application that demonstrates ETL pipelines, SQL analytics, and interactive visualization using the Harvard Art Museums API. It's designed to simulate real-world data pipeline workflows commonly used in analytics and data engineering roles.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App:
- Fetches artifact data from the Harvard Art Museums API with pagination and rate limiting
- Performs ETL operations to transform nested JSON into relational database structures
- Stores data in MySQL/TiDB Cloud with proper schema design
- Executes analytical SQL queries on the stored data
- Visualizes results through interactive Streamlit dashboards with Plotly charts

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

### API Key Setup

Obtain a Harvard Art Museums API key from [https://harvardartmuseums.org/collections/api](https://harvardartmuseums.org/collections/api)

Create a `.env` file or configure through the Streamlit app:
```bash
HARVARD_API_KEY=your_api_key_here
```

### Database Configuration

Configure MySQL/TiDB Cloud connection:
```python
# Database credentials (use environment variables)
DB_HOST=os.getenv('DB_HOST')
DB_USER=os.getenv('DB_USER')
DB_PASSWORD=os.getenv('DB_PASSWORD')
DB_NAME=os.getenv('DB_NAME')
DB_PORT=os.getenv('DB_PORT', 3306)
```

## Running the Application

```bash
streamlit run app.py
```

The app will open in your browser at `http://localhost:8501`

## Core Components

### 1. API Data Extraction

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    url = f"https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Collect multiple pages
def collect_artifacts(api_key, num_pages=5):
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        records, info = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(records)
        print(f"Collected page {page}/{num_pages}")
    
    return all_artifacts
```

### 2. ETL Pipeline - Transform

```python
def transform_artifacts(raw_data):
    """
    Transform nested JSON into relational tables
    """
    # Metadata table
    metadata = []
    media = []
    colors = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata.append({
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'accession_year': artifact.get('accessionyear')
        })
        
        # Extract media information
        if artifact.get('images'):
            for img in artifact['images']:
                media.append({
                    'artifact_id': artifact.get('id'),
                    'image_id': img.get('imageid'),
                    'base_url': img.get('baseimageurl'),
                    'width': img.get('width'),
                    'height': img.get('height')
                })
        
        # Extract color data
        if artifact.get('colors'):
            for color in artifact['colors']:
                colors.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'percent': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata),
        pd.DataFrame(media),
        pd.DataFrame(colors)
    )
```

### 3. Database Schema Creation

```python
import mysql.connector

def create_database_schema(connection):
    """
    Create tables for artifacts data
    """
    cursor = connection.cursor()
    
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
            accession_year INT
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_id INT,
            base_url TEXT,
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
            color VARCHAR(50),
            spectrum VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    connection.commit()
    cursor.close()
```

### 4. Load Data to Database

```python
def load_to_database(df_metadata, df_media, df_colors, connection):
    """
    Batch insert dataframes into SQL tables
    """
    cursor = connection.cursor()
    
    # Insert metadata
    for _, row in df_metadata.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (artifact_id, title, culture, century, classification, department, dated, accession_year)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """, tuple(row))
    
    # Insert media
    for _, row in df_media.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (artifact_id, image_id, base_url, width, height)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in df_colors.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, percent)
            VALUES (%s, %s, %s, %s)
        """, tuple(row))
    
    connection.commit()
    cursor.close()
```

### 5. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
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
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN EXISTS (SELECT 1 FROM artifactmedia WHERE artifactmedia.artifact_id = artifactmetadata.artifact_id)
            THEN 'Has Media' ELSE 'No Media' END as media_status,
            COUNT(*) as count
        FROM artifactmetadata
        GROUP BY media_status
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """
}

def execute_query(connection, query):
    """Execute SQL query and return results as DataFrame"""
    return pd.read_sql(query, connection)
```

### 6. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    # Database connection
    if st.sidebar.button("Connect to Database"):
        try:
            conn = mysql.connector.connect(
                host=os.getenv('DB_HOST'),
                user=os.getenv('DB_USER'),
                password=os.getenv('DB_PASSWORD'),
                database=os.getenv('DB_NAME')
            )
            st.sidebar.success("Connected!")
            st.session_state['conn'] = conn
        except Exception as e:
            st.sidebar.error(f"Connection failed: {e}")
    
    # ETL Section
    st.header("ETL Pipeline")
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data..."):
            artifacts = collect_artifacts(api_key, num_pages=5)
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
            
            if 'conn' in st.session_state:
                load_to_database(df_meta, df_media, df_colors, st.session_state['conn'])
                st.success(f"Loaded {len(df_meta)} artifacts!")
    
    # Analytics Section
    st.header("SQL Analytics")
    query_name = st.selectbox("Select Query", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Execute Query"):
        if 'conn' in st.session_state:
            query = ANALYTICS_QUERIES[query_name]
            results = execute_query(st.session_state['conn'], query)
            
            st.dataframe(results)
            
            # Auto-generate visualization
            if len(results.columns) >= 2:
                fig = px.bar(results, x=results.columns[0], y=results.columns[1],
                           title=query_name)
                st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def get_last_loaded_artifact_id(connection):
    """Get the highest artifact_id already in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(artifact_id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(api_key, connection):
    """Load only new artifacts since last ETL run"""
    last_id = get_last_loaded_artifact_id(connection)
    
    # Fetch artifacts with ID filter
    artifacts = fetch_artifacts_after_id(api_key, last_id)
    # Transform and load...
```

### Pattern 2: Error Handling and Retry Logic

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
```

### Pattern 3: Data Quality Checks

```python
def validate_artifacts(df_metadata):
    """Run data quality checks"""
    checks = {
        'null_titles': df_metadata['title'].isnull().sum(),
        'duplicate_ids': df_metadata['artifact_id'].duplicated().sum(),
        'missing_departments': df_metadata['department'].isnull().sum()
    }
    
    return checks
```

## Troubleshooting

### API Rate Limiting
If you encounter rate limit errors:
```python
import time

def rate_limited_fetch(api_key, pages):
    for page in range(1, pages + 1):
        fetch_artifacts(api_key, page)
        time.sleep(1)  # Add 1 second delay between requests
```

### Database Connection Issues
```python
# Use connection pooling for better performance
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

connection = db_pool.get_connection()
```

### Memory Issues with Large Datasets
```python
def batch_process(artifacts, batch_size=1000):
    """Process artifacts in batches to manage memory"""
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i + batch_size]
        df_meta, df_media, df_colors = transform_artifacts(batch)
        load_to_database(df_meta, df_media, df_colors, connection)
```

### Handling Missing Data
```python
def safe_get(obj, key, default=None):
    """Safely extract nested values"""
    try:
        return obj.get(key, default)
    except AttributeError:
        return default

# Use in transformation
metadata.append({
    'artifact_id': safe_get(artifact, 'id'),
    'title': safe_get(artifact, 'title', 'Unknown'),
    'culture': safe_get(artifact, 'culture', 'Unknown')
})
```

## Best Practices

1. **Always use environment variables** for sensitive data (API keys, DB credentials)
2. **Implement pagination** when fetching large datasets from the API
3. **Use batch inserts** for better database performance
4. **Add logging** to track ETL pipeline progress
5. **Validate data** before loading to database
6. **Handle API rate limits** with delays and retry logic
7. **Create indexes** on frequently queried columns for better performance

```python
# Add index for better query performance
cursor.execute("CREATE INDEX idx_culture ON artifactmetadata(culture)")
cursor.execute("CREATE INDEX idx_century ON artifactmetadata(century)")
```
