---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - show me how to extract and load museum artifact data into SQL
  - create a data engineering pipeline with Harvard API
  - build analytics dashboard for art museum collections
  - extract Harvard Art Museums API data to database
  - analyze museum artifacts with SQL queries
  - set up Streamlit app for art collection analytics
  - pipeline Harvard museum data to TiDB or MySQL
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming nested JSON into relational structures, loading into SQL databases (MySQL/TiDB Cloud), and visualizing insights through Streamlit dashboards. It includes 20+ analytical SQL queries and interactive Plotly visualizations.

**Architecture**: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### API Key Setup

1. Get a free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Create a `.env` file or use Streamlit secrets:

```python
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_db_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_art_db
```

### Streamlit Secrets Configuration

```toml
# .streamlit/secrets.toml
[api]
harvard_api_key = "your_api_key_here"

[database]
host = "your_db_host"
user = "your_db_user"
password = "your_db_password"
database = "harvard_art_db"
port = 3306
```

## Database Schema

The ETL pipeline creates three main tables:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    description TEXT,
    creditline TEXT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);
```

## Core ETL Pipeline

### 1. Extract Data from API

```python
import requests
import pandas as pd
from typing import List, Dict

def extract_artifacts(api_key: str, num_records: int = 100) -> List[Dict]:
    """
    Extract artifact data from Harvard Art Museums API with pagination.
    """
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'page': page,
            'size': min(size, num_records - len(artifacts))
        }
        
        try:
            response = requests.get(base_url, params=params, timeout=30)
            response.raise_for_status()
            data = response.json()
            
            records = data.get('records', [])
            if not records:
                break
                
            artifacts.extend(records)
            page += 1
            
            # Rate limiting
            import time
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return artifacts[:num_records]
```

### 2. Transform Data

```python
def transform_metadata(artifacts: List[Dict]) -> pd.DataFrame:
    """
    Transform artifact data into metadata DataFrame.
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'period': artifact.get('period', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'division': artifact.get('division', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'description': artifact.get('description', ''),
            'creditline': artifact.get('creditline', '')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_media(artifacts: List[Dict]) -> pd.DataFrame:
    """
    Transform media/image data into DataFrame.
    """
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        media = {
            'artifact_id': artifact_id,
            'baseimageurl': artifact.get('baseimageurl', ''),
            'primaryimageurl': artifact.get('primaryimageurl', ''),
            'iiifbaseuri': images[0].get('iiifbaseuri', '') if images else ''
        }
        media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_colors(artifacts: List[Dict]) -> pd.DataFrame:
    """
    Transform color data into normalized DataFrame.
    """
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', ''),
                'color_percent': color.get('percent', 0.0)
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### 3. Load Data into SQL

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection(config: Dict) -> mysql.connector.connection:
    """
    Create connection to MySQL/TiDB database.
    """
    try:
        connection = mysql.connector.connect(
            host=config['host'],
            user=config['user'],
            password=config['password'],
            database=config['database'],
            port=config.get('port', 3306)
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata(df: pd.DataFrame, connection) -> int:
    """
    Batch insert metadata into database.
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (artifact_id, title, culture, period, century, classification, 
     department, division, dated, description, creditline)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE 
    title=VALUES(title), culture=VALUES(culture)
    """
    
    records = df.to_records(index=False)
    data_tuples = [tuple(record) for record in records]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        return cursor.rowcount
    except Error as e:
        print(f"Insert error: {e}")
        connection.rollback()
        return 0
    finally:
        cursor.close()

def load_media(df: pd.DataFrame, connection) -> int:
    """Load media data."""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, baseimageurl, primaryimageurl, iiifbaseuri)
    VALUES (%s, %s, %s, %s)
    """
    
    records = df.to_records(index=False)
    data_tuples = [tuple(record) for record in records]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        return cursor.rowcount
    except Error as e:
        print(f"Insert error: {e}")
        connection.rollback()
        return 0
    finally:
        cursor.close()

def load_colors(df: pd.DataFrame, connection) -> int:
    """Load color data."""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color_hex, color_percent)
    VALUES (%s, %s, %s)
    """
    
    records = df.to_records(index=False)
    data_tuples = [tuple(record) for record in records]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        return cursor.rowcount
    except Error as e:
        print(f"Insert error: {e}")
        connection.rollback()
        return 0
    finally:
        cursor.close()
```

## Complete ETL Pipeline Example

```python
import os
from dotenv import load_dotenv

def run_etl_pipeline(num_records: int = 100):
    """
    Execute full ETL pipeline.
    """
    load_dotenv()
    
    # 1. EXTRACT
    print("Extracting data from API...")
    api_key = os.getenv('HARVARD_API_KEY')
    artifacts = extract_artifacts(api_key, num_records)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # 2. TRANSFORM
    print("Transforming data...")
    metadata_df = transform_metadata(artifacts)
    media_df = transform_media(artifacts)
    colors_df = transform_colors(artifacts)
    
    print(f"Metadata records: {len(metadata_df)}")
    print(f"Media records: {len(media_df)}")
    print(f"Color records: {len(colors_df)}")
    
    # 3. LOAD
    print("Loading data into database...")
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME'),
        'port': int(os.getenv('DB_PORT', 3306))
    }
    
    connection = create_database_connection(db_config)
    if connection:
        metadata_rows = load_metadata(metadata_df, connection)
        media_rows = load_media(media_df, connection)
        color_rows = load_colors(colors_df, connection)
        
        print(f"Loaded {metadata_rows} metadata rows")
        print(f"Loaded {media_rows} media rows")
        print(f"Loaded {color_rows} color rows")
        
        connection.close()
        print("ETL pipeline completed successfully!")
    else:
        print("Failed to connect to database")

# Run the pipeline
if __name__ == "__main__":
    run_etl_pipeline(num_records=500)
```

## Analytics SQL Queries

### Common Analytical Queries

```python
# Sample analytical queries used in the dashboard
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY artifact_count DESC
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
            SUM(CASE WHEN primaryimageurl != '' THEN 1 ELSE 0 END) as with_image,
            SUM(CASE WHEN primaryimageurl = '' THEN 1 ELSE 0 END) as without_image
        FROM artifactmedia
    """,
    
    "Top Colors Used": """
        SELECT color_hex, COUNT(*) as usage_count, AVG(color_percent) as avg_percent
        FROM artifactcolors
        WHERE color_hex IS NOT NULL
        GROUP BY color_hex
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts with Most Colors": """
        SELECT am.artifact_id, am.title, COUNT(ac.color_id) as color_count
        FROM artifactmetadata am
        JOIN artifactcolors ac ON am.artifact_id = ac.artifact_id
        GROUP BY am.artifact_id, am.title
        ORDER BY color_count DESC
        LIMIT 20
    """,
    
    "Classification Summary": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 20
    """
}

def execute_query(connection, query: str) -> pd.DataFrame:
    """
    Execute SQL query and return results as DataFrame.
    """
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query execution error: {e}")
        return pd.DataFrame()
```

## Streamlit Dashboard Application

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection from secrets
    db_config = {
        'host': st.secrets["database"]["host"],
        'user': st.secrets["database"]["user"],
        'password': st.secrets["database"]["password"],
        'database': st.secrets["database"]["database"],
        'port': st.secrets["database"].get("port", 3306)
    }
    
    connection = create_database_connection(db_config)
    
    if not connection:
        st.error("Failed to connect to database")
        return
    
    # Tabs for different functionalities
    tab1, tab2, tab3 = st.tabs(["📊 Analytics", "🔄 ETL Pipeline", "📈 Custom Query"])
    
    with tab1:
        st.header("Pre-built Analytics Queries")
        
        query_name = st.selectbox(
            "Select Analysis",
            list(ANALYTICAL_QUERIES.keys())
        )
        
        if st.button("Run Query"):
            with st.spinner("Executing query..."):
                df = execute_query(connection, ANALYTICAL_QUERIES[query_name])
                
                if not df.empty:
                    st.subheader("Results")
                    st.dataframe(df, use_container_width=True)
                    
                    # Auto-generate visualization
                    if len(df.columns) == 2:
                        fig = px.bar(
                            df,
                            x=df.columns[0],
                            y=df.columns[1],
                            title=query_name
                        )
                        st.plotly_chart(fig, use_container_width=True)
    
    with tab2:
        st.header("ETL Pipeline Execution")
        
        num_records = st.number_input(
            "Number of records to fetch",
            min_value=10,
            max_value=1000,
            value=100,
            step=10
        )
        
        if st.button("Run ETL Pipeline"):
            api_key = st.secrets["api"]["harvard_api_key"]
            
            progress_bar = st.progress(0)
            status_text = st.empty()
            
            # Extract
            status_text.text("Extracting data from API...")
            progress_bar.progress(33)
            artifacts = extract_artifacts(api_key, num_records)
            
            # Transform
            status_text.text("Transforming data...")
            progress_bar.progress(66)
            metadata_df = transform_metadata(artifacts)
            media_df = transform_media(artifacts)
            colors_df = transform_colors(artifacts)
            
            # Load
            status_text.text("Loading data into database...")
            load_metadata(metadata_df, connection)
            load_media(media_df, connection)
            load_colors(colors_df, connection)
            
            progress_bar.progress(100)
            status_text.text("ETL completed!")
            st.success(f"Successfully processed {len(artifacts)} artifacts")
    
    with tab3:
        st.header("Custom SQL Query")
        
        custom_query = st.text_area(
            "Enter SQL Query",
            height=150,
            placeholder="SELECT * FROM artifactmetadata LIMIT 10"
        )
        
        if st.button("Execute Custom Query"):
            if custom_query.strip():
                df = execute_query(connection, custom_query)
                if not df.empty:
                    st.dataframe(df, use_container_width=True)
                    
                    # Download option
                    csv = df.to_csv(index=False)
                    st.download_button(
                        "Download Results",
                        csv,
                        "query_results.csv",
                        "text/csv"
                    )
    
    connection.close()

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Run Streamlit dashboard
streamlit run app.py

# Run ETL pipeline standalone
python etl_pipeline.py

# Access dashboard
# Open browser to http://localhost:8501
```

## Common Patterns

### Incremental ETL

```python
def get_last_artifact_id(connection) -> int:
    """Get the last processed artifact ID."""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(artifact_id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_etl(api_key: str, connection):
    """Run incremental ETL to fetch only new artifacts."""
    last_id = get_last_artifact_id(connection)
    
    # Fetch artifacts with ID > last_id
    # (Note: API may need different filtering approach)
    artifacts = extract_artifacts(api_key, num_records=100)
    new_artifacts = [a for a in artifacts if a['id'] > last_id]
    
    # Process only new artifacts
    if new_artifacts:
        metadata_df = transform_metadata(new_artifacts)
        load_metadata(metadata_df, connection)
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    filename='etl_pipeline.log'
)

def safe_etl_execution(num_records: int):
    """ETL with comprehensive error handling."""
    try:
        logging.info(f"Starting ETL for {num_records} records")
        run_etl_pipeline(num_records)
        logging.info("ETL completed successfully")
    except Exception as e:
        logging.error(f"ETL failed: {str(e)}", exc_info=True)
        raise
```

## Troubleshooting

### API Rate Limiting

```python
# Add exponential backoff for rate limiting
import time
from requests.exceptions import HTTPError

def extract_with_retry(api_key: str, max_retries: int = 3):
    """Extract with retry logic for rate limits."""
    for attempt in range(max_retries):
        try:
            artifacts = extract_artifacts(api_key, num_records=100)
            return artifacts
        except HTTPError as e:
            if e.response.status_code == 429:  # Too Many Requests
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
# Connection pooling for better performance
from mysql.connector import pooling

def create_connection_pool(config: Dict):
    """Create a connection pool for multiple operations."""
    pool = pooling.MySQLConnectionPool(
        pool_name="harvard_pool",
        pool_size=5,
        pool_reset_session=True,
        **config
    )
    return pool

# Use pool
pool = create_connection_pool(db_config)
connection = pool.get_connection()
```

### Memory Management for Large Datasets

```python
def chunked_etl(api_key: str, total_records: int, chunk_size: int = 100):
    """Process large datasets in chunks."""
    connection = create_database_connection(db_config)
    
    for offset in range(0, total_records, chunk_size):
        print(f"Processing chunk {offset}-{offset+chunk_size}")
        
        artifacts = extract_artifacts(api_key, num_records=chunk_size)
        metadata_df = transform_metadata(artifacts)
        load_metadata(metadata_df, connection)
        
        # Clear memory
        del artifacts, metadata_df
    
    connection.close()
```

## Key Features Summary

- **Full ETL Pipeline**: Extract from API, transform JSON to relational, load to SQL
- **Multi-table Schema**: Normalized design with proper foreign keys
- **Batch Processing**: Efficient bulk inserts with ON DUPLICATE KEY UPDATE
- **Analytics Dashboard**: 20+ pre-built queries with Plotly visualizations
- **Interactive UI**: Streamlit-based dashboard with real-time query execution
- **Error Handling**: Retry logic, connection pooling, comprehensive logging
- **Scalability**: Chunked processing for large datasets, incremental updates
