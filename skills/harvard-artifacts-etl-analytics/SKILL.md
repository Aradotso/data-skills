---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Streamlit, MySQL, and Plotly
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - show me how to extract and load Harvard artifacts data
  - create a data pipeline for museum artifact analytics
  - build a Streamlit dashboard for Harvard Art Museums
  - how to transform Harvard API data into SQL tables
  - analyze Harvard artifacts collection with SQL queries
  - visualize museum data with Plotly and Streamlit
  - set up Harvard Art Museums data engineering project
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards with Plotly visualizations.

## What This Project Does

The Harvard Artifacts Collection application:
- **Extracts** artifact data from the Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables (metadata, media, colors)
- **Loads** data into MySQL/TiDB Cloud databases with proper schema design
- **Analyzes** using 20+ predefined SQL queries for insights
- **Visualizes** results through interactive Plotly charts in a Streamlit interface

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

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

### API Configuration

Get your Harvard Art Museums API key from: https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dOd0Z8MpY2rvQ/viewform

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

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'port': int(os.getenv('DB_PORT', 3306))
}

conn = mysql.connector.connect(**db_config)
```

### Database Schema

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    dated VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    creditline TEXT,
    copyright TEXT,
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
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

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'fields': 'id,title,culture,dated,century,classification,department,division,technique,medium,creditline,copyright,url,images,colors'
    }
    
    try:
        response = requests.get(BASE_URL, params=params, timeout=30)
        response.raise_for_status()
        data = response.json()
        
        # Rate limiting
        time.sleep(0.5)
        
        return data.get('records', []), data.get('info', {})
    except requests.exceptions.RequestException as e:
        print(f"Error fetching data: {e}")
        return [], {}

def collect_all_artifacts(api_key, max_pages=10):
    """
    Collect artifacts across multiple pages
    """
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        records, info = fetch_artifacts(api_key, page=page)
        
        if not records:
            break
            
        all_artifacts.extend(records)
        print(f"Collected page {page}: {len(records)} artifacts")
        
        if info.get('next') is None:
            break
    
    return all_artifacts
```

### Transform: Normalize JSON to Relational Format

```python
import pandas as pd

def transform_metadata(artifacts):
    """
    Extract artifact metadata into flat structure
    """
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'division': artifact.get('division', '')[:200],
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'creditline': artifact.get('creditline', ''),
            'copyright': artifact.get('copyright', ''),
            'url': artifact.get('url', '')[:500]
        })
    
    return pd.DataFrame(metadata)

def transform_media(artifacts):
    """
    Extract media/images into separate table
    """
    media = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media.append({
                'artifact_id': artifact_id,
                'media_type': 'image',
                'baseimageurl': image.get('baseimageurl', '')[:500],
                'iiifbaseuri': image.get('iiifbaseuri', '')[:500]
            })
    
    return pd.DataFrame(media)

def transform_colors(artifacts):
    """
    Extract color data into separate table
    """
    colors = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        color_list = artifact.get('colors', [])
        
        for color in color_list:
            colors.append({
                'artifact_id': artifact_id,
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'hue': color.get('hue', '')[:50],
                'percent': color.get('percent', 0.0)
            })
    
    return pd.DataFrame(colors)
```

### Load: Batch Insert into MySQL

```python
def load_metadata(df, connection):
    """
    Load metadata using batch insert
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, dated, century, classification, department, 
     division, technique, medium, creditline, copyright, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Loaded {cursor.rowcount} metadata records")
    cursor.close()

def load_media(df, connection):
    """
    Load media data
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, media_type, baseimageurl, iiifbaseuri)
    VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Loaded {cursor.rowcount} media records")
    cursor.close()

def load_colors(df, connection):
    """
    Load color data
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Loaded {cursor.rowcount} color records")
    cursor.close()
```

## Analytical SQL Queries

### Sample Analytics Queries

```python
ANALYTICAL_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century Distribution": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department-wise Classification": """
        SELECT department, classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND classification IS NOT NULL
        GROUP BY department, classification
        ORDER BY department, count DESC
    """,
    
    "Media Availability Analysis": """
        SELECT 
            CASE WHEN m.artifact_id IS NULL THEN 'No Media' ELSE 'Has Media' END as media_status,
            COUNT(DISTINCT a.id) as artifact_count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY media_status
    """,
    
    "Top 10 Most Common Colors": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Artifacts with Color Diversity": """
        SELECT a.id, a.title, COUNT(DISTINCT c.color) as color_count
        FROM artifactmetadata a
        JOIN artifactcolors c ON a.id = c.artifact_id
        GROUP BY a.id, a.title
        HAVING color_count >= 5
        ORDER BY color_count DESC
        LIMIT 20
    """
}

def execute_query(connection, query):
    """
    Execute analytical query and return DataFrame
    """
    cursor = connection.cursor()
    cursor.execute(query)
    
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    
    cursor.close()
    
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    # Database connection
    try:
        conn = mysql.connector.connect(**db_config)
        st.sidebar.success("✅ Database Connected")
    except Exception as e:
        st.sidebar.error(f"❌ Database Error: {e}")
        return
    
    if page == "ETL Pipeline":
        show_etl_page(conn)
    elif page == "SQL Analytics":
        show_analytics_page(conn)
    elif page == "Visualizations":
        show_visualizations_page(conn)
    
    conn.close()

def show_etl_page(conn):
    """
    ETL Pipeline execution page
    """
    st.header("ETL Pipeline Execution")
    
    col1, col2 = st.columns(2)
    
    with col1:
        max_pages = st.number_input("Max Pages to Fetch", 1, 100, 10)
    
    with col2:
        if st.button("Run ETL Pipeline"):
            with st.spinner("Extracting data from API..."):
                api_key = os.getenv('HARVARD_API_KEY')
                artifacts = collect_all_artifacts(api_key, max_pages)
                st.success(f"✅ Extracted {len(artifacts)} artifacts")
            
            with st.spinner("Transforming data..."):
                metadata_df = transform_metadata(artifacts)
                media_df = transform_media(artifacts)
                colors_df = transform_colors(artifacts)
                st.success("✅ Data transformed")
            
            with st.spinner("Loading to database..."):
                load_metadata(metadata_df, conn)
                load_media(media_df, conn)
                load_colors(colors_df, conn)
                st.success("✅ Data loaded successfully")

def show_analytics_page(conn):
    """
    SQL Analytics page with query execution
    """
    st.header("SQL Analytics Dashboard")
    
    query_name = st.selectbox(
        "Select Analysis Query",
        list(ANALYTICAL_QUERIES.keys())
    )
    
    if st.button("Execute Query"):
        query = ANALYTICAL_QUERIES[query_name]
        
        # Show SQL query
        with st.expander("View SQL Query"):
            st.code(query, language="sql")
        
        # Execute and display results
        try:
            df = execute_query(conn, query)
            
            st.subheader("Query Results")
            st.dataframe(df, use_container_width=True)
            
            # Auto-generate visualization if applicable
            if len(df.columns) >= 2:
                fig = px.bar(
                    df.head(15),
                    x=df.columns[0],
                    y=df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)
        
        except Exception as e:
            st.error(f"Query execution error: {e}")

def show_visualizations_page(conn):
    """
    Interactive visualizations page
    """
    st.header("Data Visualizations")
    
    viz_type = st.radio(
        "Select Visualization",
        ["Culture Distribution", "Century Timeline", "Color Analysis"]
    )
    
    if viz_type == "Culture Distribution":
        df = execute_query(conn, ANALYTICAL_QUERIES["Top 10 Cultures by Artifact Count"])
        fig = px.bar(df, x='culture', y='artifact_count', 
                     title="Top 10 Cultures by Artifact Count")
        st.plotly_chart(fig, use_container_width=True)
    
    elif viz_type == "Century Timeline":
        df = execute_query(conn, ANALYTICAL_QUERIES["Artifacts by Century Distribution"])
        fig = px.line(df, x='century', y='count',
                      title="Artifact Distribution Across Centuries")
        st.plotly_chart(fig, use_container_width=True)
    
    elif viz_type == "Color Analysis":
        df = execute_query(conn, ANALYTICAL_QUERIES["Top 10 Most Common Colors"])
        fig = px.pie(df, names='color', values='frequency',
                     title="Color Distribution in Artifacts")
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Pattern 1: Incremental ETL Updates

```python
def get_latest_artifact_id(connection):
    """
    Get the highest artifact ID already in database
    """
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    
    return result[0] if result[0] else 0

def incremental_etl(api_key, connection):
    """
    Only fetch and load new artifacts
    """
    latest_id = get_latest_artifact_id(connection)
    
    # Fetch only artifacts with ID > latest_id
    params = {
        'apikey': api_key,
        'q': f'id:>{latest_id}',
        'size': 100
    }
    
    # Continue with ETL pipeline...
```

### Pattern 2: Error Handling and Logging

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    filename='etl_pipeline.log'
)

def safe_etl_execution(api_key, connection):
    """
    ETL with comprehensive error handling
    """
    try:
        logging.info("Starting ETL pipeline")
        
        artifacts = collect_all_artifacts(api_key, max_pages=10)
        logging.info(f"Extracted {len(artifacts)} artifacts")
        
        metadata_df = transform_metadata(artifacts)
        load_metadata(metadata_df, connection)
        
        logging.info("ETL completed successfully")
        
    except requests.exceptions.RequestException as e:
        logging.error(f"API request failed: {e}")
        raise
    
    except mysql.connector.Error as e:
        logging.error(f"Database error: {e}")
        connection.rollback()
        raise
    
    except Exception as e:
        logging.error(f"Unexpected error: {e}")
        raise
```

### Pattern 3: Data Quality Validation

```python
def validate_data_quality(df, table_name):
    """
    Validate data before loading
    """
    issues = []
    
    # Check for nulls in critical fields
    if table_name == "metadata":
        if df['id'].isnull().any():
            issues.append("NULL values found in ID field")
    
    # Check for duplicates
    if df.duplicated(subset=['id']).any():
        issues.append(f"Duplicate IDs found: {df[df.duplicated(subset=['id'])]['id'].tolist()}")
    
    # Check data types
    if not pd.api.types.is_numeric_dtype(df['id']):
        issues.append("ID field is not numeric")
    
    if issues:
        for issue in issues:
            logging.warning(f"Data quality issue in {table_name}: {issue}")
        return False
    
    return True
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff for rate limits
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_retry_session():
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
```

### Database Connection Pooling
```python
from mysql.connector import pooling

connection_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    **db_config
)

def get_connection():
    return connection_pool.get_connection()
```

### Memory Management for Large Datasets
```python
def batch_etl(api_key, connection, batch_size=1000):
    """
    Process artifacts in batches to manage memory
    """
    page = 1
    
    while True:
        artifacts = fetch_artifacts(api_key, page=page, size=batch_size)
        
        if not artifacts:
            break
        
        # Process batch
        metadata_df = transform_metadata(artifacts)
        load_metadata(metadata_df, connection)
        
        # Clear memory
        del artifacts, metadata_df
        
        page += 1
```

This skill provides comprehensive guidance for building production-ready ETL pipelines and analytics dashboards using the Harvard Art Museums API, with real-world patterns for data engineering workflows.
