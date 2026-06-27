---
name: harvard-art-museums-data-pipeline
description: Build end-to-end data pipelines using the Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - how do I extract data from the Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create a data engineering project with Streamlit
  - analyze Harvard Art Museums collection data
  - set up SQL analytics for art museum data
  - visualize artifact metadata with Plotly
  - implement pagination for Harvard API requests
  - design relational schema for museum artifacts
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App provides:

- **API Integration**: Extract artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transform nested JSON into normalized relational tables
- **SQL Database**: Store artifact metadata, media, and color data in MySQL/TiDB
- **Analytics**: Execute 20+ predefined SQL queries for insights
- **Visualization**: Interactive Plotly charts in Streamlit dashboards

Architecture: `API → ETL → SQL → Analytics → Visualization`

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

## Key Dependencies

```python
import streamlit as st
import pandas as pd
import requests
import mysql.connector
import plotly.express as px
from typing import List, Dict, Optional
```

## Configuration

### API Setup

Obtain an API key from Harvard Art Museums:
- Visit: https://www.harvardartmuseums.org/collections/api
- Register for free API access
- Store key in environment variable: `HARVARD_API_KEY`

### Database Setup

```python
# Database connection configuration
db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

# Create connection
def get_db_connection():
    return mysql.connector.connect(**db_config)
```

### Database Schema

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    object_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    medium VARCHAR(500),
    technique VARCHAR(500),
    period VARCHAR(200),
    provenance TEXT,
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    object_id INT,
    image_url VARCHAR(1000),
    image_width INT,
    image_height INT,
    format VARCHAR(50),
    FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    object_id INT,
    color_hex VARCHAR(10),
    color_percent DECIMAL(5,2),
    FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os
from typing import List, Dict

def fetch_artifacts(api_key: str, page: int = 1, size: int = 100) -> Dict:
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

def extract_all_artifacts(api_key: str, max_pages: int = 10) -> List[Dict]:
    """
    Extract artifacts across multiple pages
    """
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page)
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            # Check if more pages exist
            if page >= data.get('info', {}).get('pages', 0):
                break
                
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts
```

### Transform: Process JSON to Relational Format

```python
import pandas as pd
from typing import List, Dict, Tuple

def transform_artifact_metadata(artifacts: List[Dict]) -> pd.DataFrame:
    """
    Transform artifact data into metadata DataFrame
    """
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'object_id': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique'),
            'period': artifact.get('period'),
            'provenance': artifact.get('provenance'),
            'url': artifact.get('url')
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)

def transform_artifact_media(artifacts: List[Dict]) -> pd.DataFrame:
    """
    Extract media/image data from artifacts
    """
    media_records = []
    
    for artifact in artifacts:
        object_id = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for image in images:
            record = {
                'object_id': object_id,
                'image_url': image.get('baseimageurl'),
                'image_width': image.get('width'),
                'image_height': image.get('height'),
                'format': image.get('format')
            }
            media_records.append(record)
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts: List[Dict]) -> pd.DataFrame:
    """
    Extract color palette data from artifacts
    """
    color_records = []
    
    for artifact in artifacts:
        object_id = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            record = {
                'object_id': object_id,
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent')
            }
            color_records.append(record)
    
    return pd.DataFrame(color_records)
```

### Load: Batch Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error
import pandas as pd

def batch_insert_metadata(df: pd.DataFrame, connection):
    """
    Batch insert artifact metadata into SQL database
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (object_id, title, culture, century, dated, classification, 
         department, division, medium, technique, period, provenance, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    # Convert DataFrame to list of tuples
    records = df.to_records(index=False).tolist()
    
    try:
        cursor.executemany(insert_query, records)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error inserting metadata: {e}")
        connection.rollback()
    finally:
        cursor.close()

def batch_insert_media(df: pd.DataFrame, connection):
    """
    Batch insert media data
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (object_id, image_url, image_width, image_height, format)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    
    try:
        cursor.executemany(insert_query, records)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
    finally:
        cursor.close()

def batch_insert_colors(df: pd.DataFrame, connection):
    """
    Batch insert color data
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors 
        (object_id, color_hex, color_percent)
        VALUES (%s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    
    try:
        cursor.executemany(insert_query, records)
        connection.commit()
        print(f"Inserted {cursor.rowcount} color records")
    except Error as e:
        print(f"Error inserting colors: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

## Complete ETL Pipeline

```python
import os
import mysql.connector

def run_etl_pipeline(max_pages: int = 5):
    """
    Execute complete ETL pipeline
    """
    # Extract
    api_key = os.getenv('HARVARD_API_KEY')
    print("Extracting data from Harvard Art Museums API...")
    artifacts = extract_all_artifacts(api_key, max_pages=max_pages)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(artifacts)
    media_df = transform_artifact_media(artifacts)
    colors_df = transform_artifact_colors(artifacts)
    
    # Load
    print("Loading data into database...")
    connection = get_db_connection()
    
    try:
        batch_insert_metadata(metadata_df, connection)
        batch_insert_media(media_df, connection)
        batch_insert_colors(colors_df, connection)
        print("ETL pipeline completed successfully!")
    finally:
        connection.close()
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Top 10 cultures by artifact count
query_1 = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10;
"""

# Artifacts by century
query_2 = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC;
"""

# Media availability analysis
query_3 = """
    SELECT 
        am.department,
        COUNT(DISTINCT am.object_id) as total_artifacts,
        COUNT(DISTINCT media.object_id) as artifacts_with_images,
        ROUND(COUNT(DISTINCT media.object_id) * 100.0 / COUNT(DISTINCT am.object_id), 2) as image_percentage
    FROM artifactmetadata am
    LEFT JOIN artifactmedia media ON am.object_id = media.object_id
    GROUP BY am.department
    ORDER BY image_percentage DESC;
"""

# Most common color palettes
query_4 = """
    SELECT color_hex, COUNT(*) as usage_count
    FROM artifactcolors
    GROUP BY color_hex
    ORDER BY usage_count DESC
    LIMIT 20;
"""

# Artifacts by classification and period
query_5 = """
    SELECT 
        classification,
        period,
        COUNT(*) as count
    FROM artifactmetadata
    WHERE classification IS NOT NULL AND period IS NOT NULL
    GROUP BY classification, period
    ORDER BY count DESC
    LIMIT 15;
"""

def execute_query(query: str, connection) -> pd.DataFrame:
    """
    Execute SQL query and return results as DataFrame
    """
    return pd.read_sql(query, connection)
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px
import pandas as pd

def create_streamlit_app():
    """
    Main Streamlit application
    """
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Data Analytics")
    st.markdown("---")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    # Database connection
    connection = get_db_connection()
    
    if page == "ETL Pipeline":
        show_etl_page(connection)
    elif page == "SQL Analytics":
        show_analytics_page(connection)
    elif page == "Visualizations":
        show_visualization_page(connection)
    
    connection.close()

def show_etl_page(connection):
    """
    ETL Pipeline execution page
    """
    st.header("📊 ETL Pipeline")
    
    max_pages = st.number_input("Number of pages to extract", min_value=1, max_value=50, value=5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            try:
                run_etl_pipeline(max_pages=max_pages)
                st.success("ETL pipeline completed successfully!")
            except Exception as e:
                st.error(f"Error: {e}")

def show_analytics_page(connection):
    """
    SQL Analytics query execution page
    """
    st.header("📈 SQL Analytics")
    
    queries = {
        "Top 10 Cultures": query_1,
        "Artifacts by Century": query_2,
        "Media Availability": query_3,
        "Common Colors": query_4,
        "Classification & Period": query_5
    }
    
    selected_query = st.selectbox("Select Analytics Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        query = queries[selected_query]
        st.code(query, language="sql")
        
        df = execute_query(query, connection)
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

def show_visualization_page(connection):
    """
    Interactive visualizations page
    """
    st.header("📊 Data Visualizations")
    
    col1, col2 = st.columns(2)
    
    with col1:
        # Culture distribution
        df = execute_query(query_1, connection)
        fig = px.pie(df, names='culture', values='artifact_count', title="Artifact Distribution by Culture")
        st.plotly_chart(fig, use_container_width=True)
    
    with col2:
        # Department analysis
        query = "SELECT department, COUNT(*) as count FROM artifactmetadata GROUP BY department"
        df = execute_query(query, connection)
        fig = px.bar(df, x='department', y='count', title="Artifacts by Department")
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    create_streamlit_app()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Handling API Rate Limits

```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(url: str, params: dict, max_retries: int = 3):
    """
    Fetch data with exponential backoff retry
    """
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            response.raise_for_status()
            return response.json()
        except HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    
    raise Exception("Max retries exceeded")
```

### Data Validation Before Loading

```python
def validate_metadata_df(df: pd.DataFrame) -> pd.DataFrame:
    """
    Validate and clean metadata DataFrame
    """
    # Remove duplicates
    df = df.drop_duplicates(subset=['object_id'])
    
    # Handle NULL object_ids
    df = df[df['object_id'].notna()]
    
    # Trim text fields
    text_columns = ['title', 'culture', 'medium']
    for col in text_columns:
        if col in df.columns:
            df[col] = df[col].str.strip()
    
    return df
```

### Incremental Data Loading

```python
def get_max_object_id(connection) -> int:
    """
    Get the maximum object_id already in database
    """
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(object_id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    return result if result else 0

def fetch_new_artifacts_only(api_key: str, last_object_id: int):
    """
    Fetch only artifacts newer than last_object_id
    """
    # Implementation depends on API filtering capabilities
    pass
```

## Troubleshooting

### Database Connection Issues

```python
# Test database connection
try:
    connection = mysql.connector.connect(**db_config)
    print("✓ Database connection successful")
    connection.close()
except Error as e:
    print(f"✗ Connection failed: {e}")
    print("Check: DB_HOST, DB_USER, DB_PASSWORD, DB_NAME environment variables")
```

### API Authentication Errors

```python
# Verify API key
api_key = os.getenv('HARVARD_API_KEY')
if not api_key:
    raise ValueError("HARVARD_API_KEY environment variable not set")

# Test API access
response = requests.get(
    "https://api.harvardartmuseums.org/object",
    params={'apikey': api_key, 'size': 1}
)
if response.status_code == 401:
    print("Invalid API key")
elif response.status_code == 200:
    print("✓ API key valid")
```

### Memory Issues with Large Datasets

```python
# Process in chunks
def batch_etl_pipeline(total_pages: int, batch_size: int = 10):
    """
    Process ETL in batches to manage memory
    """
    connection = get_db_connection()
    
    for start_page in range(1, total_pages + 1, batch_size):
        end_page = min(start_page + batch_size - 1, total_pages)
        print(f"Processing pages {start_page} to {end_page}")
        
        # Extract batch
        artifacts = []
        for page in range(start_page, end_page + 1):
            data = fetch_artifacts(api_key, page=page)
            artifacts.extend(data.get('records', []))
        
        # Transform and load batch
        metadata_df = transform_artifact_metadata(artifacts)
        batch_insert_metadata(metadata_df, connection)
        
        # Clear memory
        del artifacts, metadata_df
    
    connection.close()
```

This skill provides comprehensive guidance for building data engineering pipelines with the Harvard Art Museums API, covering ETL processes, SQL analytics, and interactive visualization with Streamlit.
