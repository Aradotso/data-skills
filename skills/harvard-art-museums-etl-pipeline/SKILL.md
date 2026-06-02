---
name: harvard-art-museums-etl-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - set up a data engineering project with Harvard Art Museums API
  - create a Streamlit analytics dashboard for museum artifacts
  - extract and transform Harvard Art Museums API data
  - build a SQL database for art museum collections
  - analyze Harvard Art Museums data with Python
  - create visualizations for museum artifact data
  - implement paginated API data collection for museums
---

# Harvard Art Museums ETL Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an end-to-end data engineering and analytics application that demonstrates building ETL pipelines using the Harvard Art Museums API. It extracts artifact data, transforms it into relational tables, loads it into SQL databases, and provides interactive analytics dashboards using Streamlit.

## What It Does

- **API Integration**: Collects artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color data
- **SQL Database**: Stores structured data in MySQL/TiDB Cloud with proper relationships
- **Analytics**: Executes 20+ predefined SQL queries for insights
- **Visualization**: Creates interactive dashboards with Plotly charts

## Architecture

```
Harvard Art Museums API → ETL (Python) → SQL Database → Analytics → Streamlit Dashboard
```

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

## Configuration

### API Key Setup

Obtain an API key from [Harvard Art Museums API](https://harvardartmuseums.org/collections/api):

```python
import os

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Connection

```python
import mysql.connector
import os

def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```

## Database Schema

### Create Tables

```python
import mysql.connector

def create_tables(connection):
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            century VARCHAR(100),
            dated VARCHAR(255),
            classification VARCHAR(255),
            department VARCHAR(255),
            division VARCHAR(255),
            medium VARCHAR(500),
            technique VARCHAR(500),
            dimensions VARCHAR(500),
            url VARCHAR(500),
            creditline TEXT
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            media_type VARCHAR(100),
            baseimageurl VARCHAR(500),
            iiifbaseuri VARCHAR(500),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()
    cursor.close()
```

## ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts with pagination"""
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(BASE_URL, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        print(f"Error: {response.status_code}")
        return None

def fetch_all_artifacts(api_key, max_pages=10):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        
        if data and 'records' in data:
            all_artifacts.extend(data['records'])
            time.sleep(1)  # Rate limiting
        else:
            break
    
    return all_artifacts
```

### Transform: Process JSON Data

```python
import pandas as pd

def transform_metadata(artifacts):
    """Transform artifacts into metadata DataFrame"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique'),
            'dimensions': artifact.get('dimensions'),
            'url': artifact.get('url'),
            'creditline': artifact.get('creditline')
        })
    
    return pd.DataFrame(metadata)

def transform_media(artifacts):
    """Extract media information"""
    media = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media.append({
                'artifact_id': artifact_id,
                'media_type': 'image',
                'baseimageurl': image.get('baseimageurl'),
                'iiifbaseuri': image.get('iiifbaseuri')
            })
    
    return pd.DataFrame(media)

def transform_colors(artifacts):
    """Extract color information"""
    colors = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        color_data = artifact.get('colors', [])
        
        for color in color_data:
            colors.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(colors)
```

### Load: Insert Data into SQL

```python
def load_metadata(connection, df):
    """Batch insert metadata"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT IGNORE INTO artifactmetadata 
        (id, title, culture, century, dated, classification, department, 
         division, medium, technique, dimensions, url, creditline)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
    print(f"Inserted {cursor.rowcount} metadata records")

def load_media(connection, df):
    """Batch insert media"""
    if df.empty:
        return
    
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia (artifact_id, media_type, baseimageurl, iiifbaseuri)
        VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
    print(f"Inserted {cursor.rowcount} media records")

def load_colors(connection, df):
    """Batch insert colors"""
    if df.empty:
        return
    
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
    print(f"Inserted {cursor.rowcount} color records")
```

## Complete ETL Execution

```python
def run_etl_pipeline():
    """Execute complete ETL pipeline"""
    # Extract
    api_key = os.getenv('HARVARD_API_KEY')
    artifacts = fetch_all_artifacts(api_key, max_pages=5)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    metadata_df = transform_metadata(artifacts)
    media_df = transform_media(artifacts)
    colors_df = transform_colors(artifacts)
    print("Transformation complete")
    
    # Load
    connection = get_db_connection()
    create_tables(connection)
    load_metadata(connection, metadata_df)
    load_media(connection, media_df)
    load_colors(connection, colors_df)
    connection.close()
    print("ETL pipeline completed successfully")

if __name__ == "__main__":
    run_etl_pipeline()
```

## Analytics Queries

### Sample SQL Queries

```python
# Query 1: Artifacts by Culture
query_by_culture = """
    SELECT culture, COUNT(*) as count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY count DESC
    LIMIT 10
"""

# Query 2: Artifacts by Century
query_by_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY century
"""

# Query 3: Most Common Colors
query_colors = """
    SELECT color, COUNT(*) as count
    FROM artifactcolors
    GROUP BY color
    ORDER BY count DESC
    LIMIT 10
"""

# Query 4: Artifacts with Images
query_with_images = """
    SELECT 
        COUNT(DISTINCT a.id) as total_artifacts,
        COUNT(DISTINCT m.artifact_id) as with_images
    FROM artifactmetadata a
    LEFT JOIN artifactmedia m ON a.id = m.artifact_id
"""

# Query 5: Department Distribution
query_by_department = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY count DESC
"""

def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)
```

## Streamlit Dashboard

### Basic Dashboard Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Connect to database
    conn = get_db_connection()
    
    # Query selection
    query_options = {
        "Artifacts by Culture": query_by_culture,
        "Artifacts by Century": query_by_century,
        "Most Common Colors": query_colors,
        "Department Distribution": query_by_department
    }
    
    selected_query = st.sidebar.selectbox(
        "Select Analysis",
        list(query_options.keys())
    )
    
    # Execute query
    df = execute_query(conn, query_options[selected_query])
    
    # Display results
    st.subheader(f"Results: {selected_query}")
    st.dataframe(df)
    
    # Visualization
    if len(df.columns) == 2:
        fig = px.bar(
            df,
            x=df.columns[0],
            y=df.columns[1],
            title=selected_query
        )
        st.plotly_chart(fig)
    
    conn.close()

if __name__ == "__main__":
    main()
```

### Advanced Dashboard with Multiple Views

```python
def advanced_dashboard():
    st.set_page_config(page_title="Harvard Art Museums", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics")
    
    conn = get_db_connection()
    
    # Metrics row
    col1, col2, col3, col4 = st.columns(4)
    
    with col1:
        total = pd.read_sql("SELECT COUNT(*) as count FROM artifactmetadata", conn)
        st.metric("Total Artifacts", total['count'][0])
    
    with col2:
        cultures = pd.read_sql("SELECT COUNT(DISTINCT culture) as count FROM artifactmetadata", conn)
        st.metric("Unique Cultures", cultures['count'][0])
    
    with col3:
        with_images = pd.read_sql("SELECT COUNT(DISTINCT artifact_id) as count FROM artifactmedia", conn)
        st.metric("With Images", with_images['count'][0])
    
    with col4:
        colors_count = pd.read_sql("SELECT COUNT(*) as count FROM artifactcolors", conn)
        st.metric("Color Records", colors_count['count'][0])
    
    # Visualizations
    st.header("Visual Analytics")
    
    tab1, tab2, tab3 = st.tabs(["By Culture", "By Century", "By Colors"])
    
    with tab1:
        df_culture = execute_query(conn, query_by_culture)
        fig = px.bar(df_culture, x='culture', y='count', title='Top 10 Cultures')
        st.plotly_chart(fig, use_container_width=True)
    
    with tab2:
        df_century = execute_query(conn, query_by_century)
        fig = px.line(df_century, x='century', y='count', title='Artifacts by Century')
        st.plotly_chart(fig, use_container_width=True)
    
    with tab3:
        df_colors = execute_query(conn, query_colors)
        fig = px.pie(df_colors, values='count', names='color', title='Color Distribution')
        st.plotly_chart(fig, use_container_width=True)
    
    conn.close()
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_artifact_id(connection):
    """Get the highest artifact ID already loaded"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) as max_id FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_load(api_key, connection):
    """Load only new artifacts"""
    last_id = get_last_artifact_id(connection)
    
    # Fetch artifacts with ID filter
    params = {
        'apikey': api_key,
        'q': f'id:>{last_id}',
        'size': 100
    }
    
    response = requests.get(BASE_URL, params=params)
    if response.status_code == 200:
        artifacts = response.json().get('records', [])
        # Transform and load...
```

### Error Handling

```python
def safe_etl_pipeline():
    """ETL with error handling"""
    try:
        api_key = os.getenv('HARVARD_API_KEY')
        if not api_key:
            raise ValueError("HARVARD_API_KEY not set")
        
        connection = get_db_connection()
        
        try:
            artifacts = fetch_all_artifacts(api_key, max_pages=5)
            
            if not artifacts:
                print("No artifacts fetched")
                return
            
            metadata_df = transform_metadata(artifacts)
            load_metadata(connection, metadata_df)
            
            print("ETL completed successfully")
            
        finally:
            connection.close()
            
    except mysql.connector.Error as db_error:
        print(f"Database error: {db_error}")
    except requests.RequestException as api_error:
        print(f"API error: {api_error}")
    except Exception as e:
        print(f"Unexpected error: {e}")
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limiting errors:

```python
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session_with_retries():
    session = requests.Session()
    retry = Retry(
        total=5,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504]
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    return session

session = create_session_with_retries()
response = session.get(BASE_URL, params=params)
```

### Database Connection Issues

```python
def robust_db_connection(max_retries=3):
    """Connect with retry logic"""
    for attempt in range(max_retries):
        try:
            connection = mysql.connector.connect(
                host=os.getenv('DB_HOST'),
                user=os.getenv('DB_USER'),
                password=os.getenv('DB_PASSWORD'),
                database=os.getenv('DB_NAME'),
                connect_timeout=10
            )
            return connection
        except mysql.connector.Error as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
```

### Memory Management for Large Datasets

```python
def chunked_etl(api_key, chunk_size=100):
    """Process data in chunks"""
    connection = get_db_connection()
    page = 1
    
    while True:
        artifacts = fetch_artifacts(api_key, page=page, size=chunk_size)
        
        if not artifacts or 'records' not in artifacts:
            break
        
        records = artifacts['records']
        if not records:
            break
        
        # Process chunk
        metadata_df = transform_metadata(records)
        load_metadata(connection, metadata_df)
        
        page += 1
        time.sleep(1)
    
    connection.close()
```

## Running the Application

```bash
# Run ETL pipeline
python etl_pipeline.py

# Start Streamlit dashboard
streamlit run app.py

# Access dashboard at http://localhost:8501
```
