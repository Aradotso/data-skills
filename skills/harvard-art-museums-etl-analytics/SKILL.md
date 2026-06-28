---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for museum artifact data
  - create a Harvard Art Museums data analytics app
  - extract and analyze art collection data from Harvard API
  - set up a Streamlit dashboard for artifact analytics
  - build a data engineering pipeline with museum API
  - create SQL analytics for art museum collections
  - develop an artifacts collection data pipeline
  - visualize Harvard Art Museums data with Plotly
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a production-grade ETL pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational database tables, and provides SQL-based analytics with interactive Streamlit visualizations.

## What It Does

- **Extracts** artifact metadata, media details, and color information from Harvard Art Museums API
- **Transforms** nested JSON responses into normalized relational tables
- **Loads** data into MySQL/TiDB Cloud with proper foreign key relationships
- **Analyzes** collection data using 20+ predefined SQL queries
- **Visualizes** insights through interactive Plotly charts in a Streamlit dashboard

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

### API Key Setup

Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file or configure in Streamlit:

```python
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Schema

The application creates three main tables:

```sql
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(255),
    dated VARCHAR(255),
    department VARCHAR(255),
    classification VARCHAR(255),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    imageurl VARCHAR(1000),
    baseimageurl VARCHAR(1000),
    iiifbaseuri VARCHAR(1000),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Running the Application

```bash
streamlit run app.py
```

The application launches on `http://localhost:8501` with three main sections:
1. **API Configuration** - Set API key and database credentials
2. **ETL Pipeline** - Collect and load artifact data
3. **SQL Analytics Dashboard** - Run queries and visualize results

## Core Components

### 1. API Data Collection

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    url = f"https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Collect multiple pages
def collect_artifacts(api_key, num_pages=5):
    all_artifacts = []
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(data.get('records', []))
    return all_artifacts
```

### 2. Data Transformation

```python
def transform_metadata(artifacts):
    """Transform artifacts into metadata table format"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_media(artifacts):
    """Extract media/image data"""
    media_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for image in images:
            media = {
                'objectid': objectid,
                'imageurl': image.get('imageurl'),
                'baseimageurl': image.get('baseimageurl'),
                'iiifbaseuri': image.get('iiifbaseuri')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_colors(artifacts):
    """Extract color data"""
    colors_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'objectid': objectid,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            colors_list.append(color_data)
    
    return pd.DataFrame(colors_list)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def create_connection(host, user, password, database):
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(
            host=host,
            user=user,
            password=password,
            database=database
        )
        return connection
    except Error as e:
        raise Exception(f"Database connection failed: {e}")

def batch_insert_metadata(connection, df):
    """Batch insert metadata with conflict handling"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (objectid, title, culture, century, dated, department, 
     classification, medium, dimensions, creditline, accessionyear)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()

def load_all_data(connection, artifacts):
    """Complete ETL load process"""
    # Transform
    metadata_df = transform_metadata(artifacts)
    media_df = transform_media(artifacts)
    colors_df = transform_colors(artifacts)
    
    # Load
    batch_insert_metadata(connection, metadata_df)
    batch_insert_media(connection, media_df)
    batch_insert_colors(connection, colors_df)
    
    return len(artifacts)
```

### 4. SQL Analytics Queries

```python
# Example analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Century Distribution": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY artifact_count DESC
    """,
    
    "Department Statistics": """
        SELECT department, 
               COUNT(*) as total_artifacts,
               COUNT(DISTINCT classification) as classifications
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Color Analysis": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT m.objectid) as artifacts_with_images,
            COUNT(a.objectid) as total_artifacts,
            ROUND(COUNT(DISTINCT m.objectid) * 100.0 / COUNT(a.objectid), 2) as percentage
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.objectid = m.objectid
    """
}

def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    cursor = connection.cursor(dictionary=True)
    cursor.execute(query)
    results = cursor.fetchall()
    cursor.close()
    return pd.DataFrame(results)
```

### 5. Streamlit Dashboard Integration

```python
import streamlit as st
import plotly.express as px

def create_visualization(df, query_name):
    """Auto-generate appropriate visualization"""
    if len(df.columns) == 2:
        # Bar chart for categorical + numeric
        fig = px.bar(
            df, 
            x=df.columns[0], 
            y=df.columns[1],
            title=query_name,
            labels={df.columns[0]: df.columns[0].title(), 
                   df.columns[1]: df.columns[1].title()}
        )
        return fig
    return None

# Streamlit app structure
st.title("Harvard Art Museums Analytics Dashboard")

# Sidebar for query selection
query_name = st.sidebar.selectbox("Select Analytics Query", list(ANALYTICS_QUERIES.keys()))

if st.sidebar.button("Run Query"):
    conn = create_connection(
        st.session_state.db_host,
        st.session_state.db_user,
        st.session_state.db_password,
        st.session_state.db_name
    )
    
    query = ANALYTICS_QUERIES[query_name]
    results_df = execute_query(conn, query)
    
    st.subheader(f"Results: {query_name}")
    st.dataframe(results_df)
    
    fig = create_visualization(results_df, query_name)
    if fig:
        st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_objectid(connection):
    """Get the highest objectid already in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(objectid) as max_id FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(api_key, connection, num_artifacts=100):
    """Load only new artifacts"""
    last_id = get_last_objectid(connection)
    # Fetch artifacts with objectid > last_id
    # Implementation depends on API filtering capabilities
```

### Error Handling & Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_pipeline(api_key, connection):
    """ETL with comprehensive error handling"""
    try:
        logger.info("Starting data collection...")
        artifacts = collect_artifacts(api_key)
        logger.info(f"Collected {len(artifacts)} artifacts")
        
        logger.info("Transforming data...")
        metadata_df = transform_metadata(artifacts)
        
        logger.info("Loading to database...")
        batch_insert_metadata(connection, metadata_df)
        
        logger.info("ETL completed successfully")
        return True
    except Exception as e:
        logger.error(f"ETL pipeline failed: {e}")
        return False
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff"""
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

### Database Connection Issues

- Ensure firewall allows connections to MySQL port (3306)
- For TiDB Cloud, whitelist your IP address
- Verify credentials in environment variables
- Test connection before running ETL

### Missing Data Handling

```python
def safe_get(artifact, key, default=None):
    """Safely extract nested data"""
    try:
        return artifact.get(key, default)
    except AttributeError:
        return default
```

## Performance Optimization

```python
# Use batch inserts instead of individual inserts
# Recommended batch size: 100-1000 records

def optimized_batch_insert(connection, df, batch_size=500):
    """Insert in batches for better performance"""
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        batch_insert_metadata(connection, batch)
```
