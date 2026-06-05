---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for the Harvard Art Museums API
  - show me how to create a data engineering app with Streamlit
  - help me extract and transform Harvard museum artifact data
  - how do I set up SQL analytics for museum collection data
  - build a data pipeline from Harvard Art Museums to SQL database
  - create interactive analytics dashboard for museum artifacts
  - how to implement ETL with Harvard Art Museums API
  - visualize Harvard museum data with Plotly and Streamlit
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualizations using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into relational SQL databases
- **SQL Analytics**: Execute predefined analytical queries on structured museum data
- **Interactive Dashboards**: Visualize query results with Plotly charts in a Streamlit interface
- **Database Design**: Properly normalized tables with foreign key relationships

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

### Required Dependencies

```txt
streamlit>=1.28.0
pandas>=2.0.0
requests>=2.31.0
mysql-connector-python>=8.1.0
plotly>=5.17.0
python-dotenv>=1.0.0
```

## Getting Harvard API Key

1. Visit [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Request an API key (free registration required)
3. Store key in environment variable: `HARVARD_API_KEY`

## Database Setup

### SQL Schema

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(300),
    medium VARCHAR(300),
    dated VARCHAR(200),
    url TEXT,
    imagecount INT,
    colorcount INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
    idsid VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core ETL Pipeline

### Extract: Fetching Data from API

```python
import requests
import os
from typing import Dict, List

def fetch_artifacts(api_key: str, page: int = 1, size: int = 100) -> Dict:
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

def collect_all_artifacts(api_key: str, max_records: int = 1000) -> List[Dict]:
    """
    Collect multiple pages of artifacts
    """
    artifacts = []
    page = 1
    size = 100
    
    while len(artifacts) < max_records:
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page, size=size)
        
        if not data.get('records'):
            break
            
        artifacts.extend(data['records'])
        page += 1
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return artifacts[:max_records]
```

### Transform: Processing JSON Data

```python
import pandas as pd

def transform_artifact_metadata(artifacts: List[Dict]) -> pd.DataFrame:
    """
    Transform artifact data into metadata dataframe
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'period': artifact.get('period', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'technique': artifact.get('technique', 'Unknown'),
            'medium': artifact.get('medium', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'url': artifact.get('url', ''),
            'imagecount': artifact.get('imagecount', 0),
            'colorcount': artifact.get('colorcount', 0)
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts: List[Dict]) -> pd.DataFrame:
    """
    Extract media/image data from artifacts
    """
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media = {
                'artifact_id': artifact_id,
                'baseimageurl': image.get('baseimageurl', ''),
                'format': image.get('format', ''),
                'height': image.get('height', 0),
                'width': image.get('width', 0),
                'idsid': image.get('idsid', '')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts: List[Dict]) -> pd.DataFrame:
    """
    Extract color data from artifacts
    """
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### Load: Inserting into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def create_db_connection(host: str, user: str, password: str, database: str):
    """
    Create MySQL database connection
    """
    try:
        connection = mysql.connector.connect(
            host=host,
            user=user,
            password=password,
            database=database
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def load_metadata(df: pd.DataFrame, connection):
    """
    Load metadata into artifactmetadata table
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, classification, department, 
     technique, medium, dated, url, imagecount, colorcount)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    print(f"Inserted {cursor.rowcount} metadata records")

def load_media(df: pd.DataFrame, connection):
    """
    Load media data into artifactmedia table
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, baseimageurl, format, height, width, idsid)
    VALUES (%s, %s, %s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    print(f"Inserted {cursor.rowcount} media records")

def load_colors(df: pd.DataFrame, connection):
    """
    Load color data into artifactcolors table
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    print(f"Inserted {cursor.rowcount} color records")
```

## Complete ETL Pipeline Example

```python
import os
from dotenv import load_dotenv

def run_etl_pipeline():
    """
    Execute complete ETL pipeline
    """
    # Load environment variables
    load_dotenv()
    api_key = os.getenv('HARVARD_API_KEY')
    
    # Extract
    print("Starting extraction...")
    artifacts = collect_all_artifacts(api_key, max_records=500)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(artifacts)
    media_df = transform_artifact_media(artifacts)
    colors_df = transform_artifact_colors(artifacts)
    
    # Load
    print("Loading into database...")
    connection = create_db_connection(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    if connection:
        load_metadata(metadata_df, connection)
        load_media(media_df, connection)
        load_colors(colors_df, connection)
        connection.close()
        print("ETL pipeline completed successfully!")
    else:
        print("Failed to connect to database")

if __name__ == "__main__":
    run_etl_pipeline()
```

## Analytical SQL Queries

### Common Analytical Patterns

```python
# Query 1: Artifacts by culture
query_by_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture != 'Unknown'
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Query 2: Artifacts by century
query_by_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC;
"""

# Query 3: Department distribution
query_by_department = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
GROUP BY department
ORDER BY total_artifacts DESC;
"""

# Query 4: Media availability
query_media_stats = """
SELECT 
    CASE WHEN imagecount > 0 THEN 'With Images' ELSE 'Without Images' END as image_status,
    COUNT(*) as count
FROM artifactmetadata
GROUP BY image_status;
"""

# Query 5: Color analysis
query_color_distribution = """
SELECT spectrum, COUNT(*) as count
FROM artifactcolors
GROUP BY spectrum
ORDER BY count DESC;
"""

# Query 6: Top artifacts with most images
query_top_imaged = """
SELECT title, culture, imagecount
FROM artifactmetadata
ORDER BY imagecount DESC
LIMIT 10;
"""

# Query 7: Classification breakdown
query_classification = """
SELECT classification, COUNT(*) as count
FROM artifactmetadata
WHERE classification != 'Unknown'
GROUP BY classification
ORDER BY count DESC
LIMIT 15;
"""
```

### Executing Queries

```python
def execute_query(connection, query: str) -> pd.DataFrame:
    """
    Execute SQL query and return results as DataFrame
    """
    cursor = connection.cursor()
    cursor.execute(query)
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Data Analytics")
    st.markdown("Interactive ETL Pipeline and Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.title("Configuration")
    
    # Database connection
    connection = create_db_connection(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    if not connection:
        st.error("Failed to connect to database")
        return
    
    # Tabs for different sections
    tab1, tab2, tab3 = st.tabs(["📊 Analytics", "🔄 ETL Pipeline", "📈 Visualizations"])
    
    with tab1:
        show_analytics(connection)
    
    with tab2:
        show_etl_controls()
    
    with tab3:
        show_visualizations(connection)

def show_analytics(connection):
    """
    Display analytical queries and results
    """
    st.subheader("SQL Analytics Dashboard")
    
    queries = {
        "Artifacts by Culture": query_by_culture,
        "Artifacts by Century": query_by_century,
        "Department Distribution": query_by_department,
        "Color Analysis": query_color_distribution
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            df = execute_query(connection, queries[selected_query])
            st.dataframe(df, use_container_width=True)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1])
                st.plotly_chart(fig, use_container_width=True)

def show_etl_controls():
    """
    Display ETL pipeline controls
    """
    st.subheader("ETL Pipeline Control")
    
    num_records = st.number_input("Number of records to fetch", 
                                   min_value=10, max_value=5000, value=100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            try:
                run_etl_pipeline()
                st.success(f"Successfully processed {num_records} records!")
            except Exception as e:
                st.error(f"ETL failed: {str(e)}")

if __name__ == "__main__":
    main()
```

### Advanced Visualization

```python
def show_visualizations(connection):
    """
    Display advanced visualizations
    """
    st.subheader("Data Visualizations")
    
    # Culture distribution
    df_culture = execute_query(connection, query_by_culture)
    fig1 = px.bar(df_culture, x='culture', y='artifact_count',
                  title="Top 10 Cultures by Artifact Count")
    st.plotly_chart(fig1, use_container_width=True)
    
    # Century timeline
    df_century = execute_query(connection, query_by_century)
    fig2 = px.line(df_century, x='century', y='count',
                   title="Artifacts by Century")
    st.plotly_chart(fig2, use_container_width=True)
    
    # Color spectrum pie chart
    df_colors = execute_query(connection, query_color_distribution)
    fig3 = px.pie(df_colors, names='spectrum', values='count',
                  title="Color Spectrum Distribution")
    st.plotly_chart(fig3, use_container_width=True)
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access in browser at http://localhost:8501
```

## Troubleshooting

### API Rate Limiting
```python
# Add delays between requests
import time

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
```

### Database Connection Issues
```python
# Test database connection
def test_db_connection():
    try:
        connection = create_db_connection(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        if connection.is_connected():
            print("Database connection successful!")
            connection.close()
            return True
    except Error as e:
        print(f"Database connection failed: {e}")
        return False
```

### Handling Missing Data
```python
# Safe data extraction with defaults
def safe_get(dictionary, key, default='Unknown'):
    """Safely get dictionary value with default"""
    value = dictionary.get(key, default)
    return value if value else default

# Apply to transformations
metadata = {
    'culture': safe_get(artifact, 'culture'),
    'period': safe_get(artifact, 'period'),
    'century': safe_get(artifact, 'century')
}
```

## Best Practices

1. **Environment Variables**: Always use `.env` file for sensitive credentials
2. **Batch Processing**: Use batch inserts for better database performance
3. **Error Handling**: Implement try-except blocks around API calls and database operations
4. **Data Validation**: Validate data types before inserting into database
5. **Logging**: Add logging for debugging ETL pipeline issues
6. **Incremental Loads**: Track processed records to avoid duplicates

```python
# Example with logging
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def etl_with_logging():
    logger.info("Starting ETL pipeline")
    try:
        artifacts = collect_all_artifacts(api_key, max_records=500)
        logger.info(f"Extracted {len(artifacts)} artifacts")
        # Continue pipeline...
    except Exception as e:
        logger.error(f"ETL failed: {e}")
        raise
```
