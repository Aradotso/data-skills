---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics application for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - connect to Harvard Art Museums API and store in SQL
  - create analytics dashboard for museum artifacts
  - extract and transform Harvard museum collection data
  - set up data pipeline with Streamlit visualization
  - query and analyze Harvard Art Museums metadata
  - implement artifact data engineering workflow
  - visualize museum collection insights with Plotly
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is an end-to-end data engineering solution that:

- Extracts artifact data from the Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into relational database schema
- Loads structured data into MySQL/TiDB Cloud databases
- Provides 20+ predefined SQL analytics queries
- Visualizes insights through interactive Streamlit dashboards with Plotly charts

The application demonstrates production-grade ETL pipelines with proper error handling, batch processing, and foreign key relationships.

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# MySQL or TiDB Cloud account
# Harvard Art Museums API key from https://www.harvardartmuseums.org/collections/api
```

### Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages include:
# - streamlit
# - pandas
# - mysql-connector-python (or pymysql)
# - requests
# - plotly
```

### Environment Configuration

Create a `.env` file or configure environment variables:

```bash
# API Configuration
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

## Database Schema

The application uses three main tables with relational structure:

```sql
-- Artifact Metadata (main table)
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    period VARCHAR(255),
    dated VARCHAR(255),
    url TEXT,
    credit_line TEXT
);

-- Artifact Media (one-to-many)
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    image_width INT,
    image_height INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors (one-to-many)
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from typing import List, Dict

def fetch_artifacts(api_key: str, size: int = 100, page: int = 1) -> Dict:
    """
    Fetch artifacts from Harvard Art Museums API with pagination.
    
    Args:
        api_key: Harvard API key from environment
        size: Number of records per page (max 100)
        page: Page number for pagination
    
    Returns:
        JSON response with artifact data
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "size": size,
        "page": page,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

def collect_artifacts_batch(api_key: str, total_records: int = 1000) -> List[Dict]:
    """
    Collect multiple pages of artifact data with rate limiting.
    """
    import time
    
    artifacts = []
    page = 1
    size = 100
    
    while len(artifacts) < total_records:
        try:
            data = fetch_artifacts(api_key, size=size, page=page)
            records = data.get('records', [])
            
            if not records:
                break
                
            artifacts.extend(records)
            page += 1
            
            # Rate limiting - be respectful to API
            time.sleep(0.5)
            
        except Exception as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return artifacts[:total_records]
```

### 2. Data Transformation

```python
import pandas as pd
from typing import List, Dict, Tuple

def transform_artifacts(raw_data: List[Dict]) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
    """
    Transform nested JSON into three normalized dataframes.
    
    Returns:
        (metadata_df, media_df, colors_df)
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        artifact_id = artifact.get('id')
        
        # Extract metadata
        metadata_records.append({
            'id': artifact_id,
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'technique': artifact.get('technique', '')[:500],
            'period': artifact.get('period', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'url': artifact.get('url', ''),
            'credit_line': artifact.get('creditline', '')
        })
        
        # Extract media information
        images = artifact.get('images', [])
        for img in images:
            media_records.append({
                'artifact_id': artifact_id,
                'image_url': img.get('baseimageurl', ''),
                'image_width': img.get('width'),
                'image_height': img.get('height')
            })
        
        # Extract color information
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', ''),
                'color_percent': color.get('percent', 0)
            })
    
    metadata_df = pd.DataFrame(metadata_records)
    media_df = pd.DataFrame(media_records)
    colors_df = pd.DataFrame(color_records)
    
    return metadata_df, media_df, colors_df
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error
import pandas as pd

def create_database_connection(host: str, user: str, password: str, database: str):
    """Create MySQL connection from environment variables."""
    try:
        connection = mysql.connector.connect(
            host=host,
            user=user,
            password=password,
            database=database
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def batch_insert_metadata(connection, df: pd.DataFrame):
    """Batch insert artifact metadata with proper error handling."""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, technique, period, dated, url, credit_line)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture), century=VALUES(century)
    """
    
    data_tuples = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error inserting metadata: {e}")
        connection.rollback()
    finally:
        cursor.close()

def batch_insert_media(connection, df: pd.DataFrame):
    """Batch insert artifact media data."""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, image_url, image_width, image_height)
    VALUES (%s, %s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
    finally:
        cursor.close()

def batch_insert_colors(connection, df: pd.DataFrame):
    """Batch insert artifact color data."""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
    VALUES (%s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} color records")
    except Error as e:
        print(f"Error inserting colors: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

### 4. Analytics Queries

```python
# Sample analytical queries included in the application

ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN image_url IS NOT NULL THEN 'Has Media' ELSE 'No Media' END as media_status,
            COUNT(DISTINCT artifact_id) as artifact_count
        FROM artifactmedia
        GROUP BY media_status
    """,
    
    "Top Color Usage": """
        SELECT color_hex, COUNT(*) as usage_count, AVG(color_percent) as avg_percent
        FROM artifactcolors
        WHERE color_hex IS NOT NULL
        GROUP BY color_hex
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Classification Analysis": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL AND classification != ''
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 20
    """
}

def execute_analytics_query(connection, query_name: str) -> pd.DataFrame:
    """Execute predefined analytics query and return results."""
    query = ANALYTICS_QUERIES.get(query_name)
    if not query:
        raise ValueError(f"Query '{query_name}' not found")
    
    return pd.read_sql(query, connection)
```

### 5. Streamlit Application

```python
import streamlit as st
import plotly.express as px
import os

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    connection = create_database_connection(**db_config)
    
    if not connection:
        st.error("Failed to connect to database. Check your credentials.")
        return
    
    # Tab layout
    tab1, tab2, tab3 = st.tabs(["📥 Data Collection", "📊 Analytics", "📈 Visualizations"])
    
    with tab1:
        st.header("Collect Artifact Data")
        
        api_key = os.getenv('HARVARD_API_KEY')
        if not api_key:
            api_key = st.text_input("Enter Harvard API Key", type="password")
        
        num_records = st.number_input("Number of records to collect", 
                                       min_value=100, max_value=10000, value=1000)
        
        if st.button("Start ETL Pipeline"):
            with st.spinner("Extracting data from API..."):
                raw_data = collect_artifacts_batch(api_key, num_records)
                st.success(f"Extracted {len(raw_data)} artifacts")
            
            with st.spinner("Transforming data..."):
                metadata_df, media_df, colors_df = transform_artifacts(raw_data)
                st.success("Data transformed successfully")
            
            with st.spinner("Loading to database..."):
                batch_insert_metadata(connection, metadata_df)
                batch_insert_media(connection, media_df)
                batch_insert_colors(connection, colors_df)
                st.success("Data loaded to database!")
    
    with tab2:
        st.header("SQL Analytics")
        
        query_selection = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
        
        if st.button("Run Query"):
            with st.spinner("Executing query..."):
                results = execute_analytics_query(connection, query_selection)
                st.dataframe(results, use_container_width=True)
                
                # Auto-generate visualization
                if len(results.columns) >= 2:
                    fig = px.bar(results, 
                                x=results.columns[0], 
                                y=results.columns[1],
                                title=query_selection)
                    st.plotly_chart(fig, use_container_width=True)
    
    with tab3:
        st.header("Custom Visualizations")
        
        viz_type = st.selectbox("Visualization Type", 
                                ["Culture Distribution", "Century Timeline", "Color Analysis"])
        
        if viz_type == "Culture Distribution":
            query = ANALYTICS_QUERIES["Artifacts by Culture"]
            df = pd.read_sql(query, connection)
            fig = px.pie(df, values='artifact_count', names='culture', 
                         title='Artifact Distribution by Culture')
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

### Pattern 1: Incremental Data Loading

```python
def get_max_artifact_id(connection):
    """Get the highest artifact ID already in database."""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_etl(connection, api_key: str):
    """Only fetch artifacts newer than what's in database."""
    max_id = get_max_artifact_id(connection)
    
    # Fetch with filter
    params = {
        "apikey": api_key,
        "size": 100,
        "sort": "id",
        "sortorder": "desc"
    }
    
    # Continue fetching until hitting max_id
    # Implementation depends on API capabilities
```

### Pattern 2: Error Recovery

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def resilient_etl(connection, api_key: str, batch_size: int = 100):
    """ETL with checkpointing and error recovery."""
    checkpoint_file = "etl_checkpoint.txt"
    
    # Load checkpoint
    start_page = 1
    if os.path.exists(checkpoint_file):
        with open(checkpoint_file, 'r') as f:
            start_page = int(f.read())
    
    page = start_page
    
    try:
        while True:
            artifacts = fetch_artifacts(api_key, size=batch_size, page=page)
            if not artifacts.get('records'):
                break
            
            # Transform and load
            metadata_df, media_df, colors_df = transform_artifacts(artifacts['records'])
            batch_insert_metadata(connection, metadata_df)
            batch_insert_media(connection, media_df)
            batch_insert_colors(connection, colors_df)
            
            # Save checkpoint
            with open(checkpoint_file, 'w') as f:
                f.write(str(page))
            
            logger.info(f"Completed page {page}")
            page += 1
            
    except Exception as e:
        logger.error(f"ETL failed at page {page}: {e}")
        raise
```

## Troubleshooting

### API Rate Limiting

```python
# Add exponential backoff
import time
from requests.exceptions import HTTPError

def fetch_with_retry(api_key, size, page, max_retries=3):
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, size, page)
        except HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Pooling

```python
from mysql.connector import pooling

connection_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_connection():
    return connection_pool.get_connection()
```

### Large Dataset Performance

```python
# Use chunks for large dataframes
def batch_insert_chunks(connection, df: pd.DataFrame, table_query: str, chunk_size: int = 1000):
    """Insert large dataframes in chunks."""
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i+chunk_size]
        data_tuples = [tuple(row) for row in chunk.values]
        
        cursor = connection.cursor()
        cursor.executemany(table_query, data_tuples)
        connection.commit()
        cursor.close()
        
        print(f"Processed {i+len(chunk)}/{len(df)} records")
```

## Configuration Best Practices

```python
# config.py - Centralized configuration
import os
from dataclasses import dataclass

@dataclass
class Config:
    # API settings
    HARVARD_API_KEY: str = os.getenv('HARVARD_API_KEY', '')
    API_BASE_URL: str = "https://api.harvardartmuseums.org/object"
    API_RATE_LIMIT: float = 0.5  # seconds between requests
    
    # Database settings
    DB_HOST: str = os.getenv('DB_HOST', 'localhost')
    DB_PORT: int = int(os.getenv('DB_PORT', '3306'))
    DB_USER: str = os.getenv('DB_USER', '')
    DB_PASSWORD: str = os.getenv('DB_PASSWORD', '')
    DB_NAME: str = os.getenv('DB_NAME', 'harvard_artifacts')
    
    # ETL settings
    BATCH_SIZE: int = 100
    MAX_RECORDS_PER_RUN: int = 1000
    CHECKPOINT_ENABLED: bool = True

config = Config()
```

This skill enables AI agents to help developers build complete ETL pipelines with the Harvard Art Museums API, implementing proper data engineering practices including extraction, transformation, loading, analytics, and visualization.
