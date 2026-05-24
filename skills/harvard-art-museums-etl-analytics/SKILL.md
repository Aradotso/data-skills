---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifact data
  - fetch and analyze Harvard Art Museums collection
  - create a data engineering pipeline with museum API
  - set up artifact analytics with SQL and Streamlit
  - extract Harvard museum data into a database
  - build analytics dashboard for art collection data
  - implement ETL for cultural heritage datasets
  - process Harvard Art Museums API with Python
---

# Harvard Art Museums ETL & Analytics Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It covers ETL pipeline development, SQL database design, analytical queries, and interactive Streamlit dashboards for cultural heritage data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App demonstrates a complete data pipeline:

1. **Extract**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
2. **Transform**: Convert nested JSON into relational table structures
3. **Load**: Batch insert into MySQL/TiDB Cloud databases
4. **Analyze**: Execute predefined SQL queries for insights
5. **Visualize**: Display results in interactive Streamlit dashboards with Plotly charts

The pipeline manages three core data entities:
- **Artifact Metadata**: Core object information (title, culture, century, classification)
- **Artifact Media**: Associated images and media files
- **Artifact Colors**: Dominant color palettes extracted from images

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install requirements
pip install -r requirements.txt

# Set up environment variables
cp .env.example .env
# Edit .env with your credentials
```

### Environment Configuration

Create a `.env` file:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

## Database Schema

### Tables Structure

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accession_number VARCHAR(100),
    url TEXT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_url TEXT,
    image_width INT,
    image_height INT,
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(7),
    color_percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetching API Data

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifact data from Harvard Art Museums API with pagination
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch objects with images
    }
    
    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status()
        data = response.json()
        return data.get('records', []), data.get('info', {})
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return [], {}

def fetch_all_artifacts(max_records=1000):
    """
    Fetch multiple pages of artifacts with rate limiting
    """
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        records, info = fetch_artifacts(page=page, size=size)
        if not records:
            break
        
        all_artifacts.extend(records)
        print(f"Fetched {len(all_artifacts)} artifacts...")
        
        # Check if more pages exist
        if info.get('next') is None:
            break
        
        page += 1
        
    return all_artifacts[:max_records]
```

### Transform: Processing JSON to Relational Format

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Extract core metadata from artifact records
    """
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Untitled'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accession_number': artifact.get('accessionyear'),
            'url': artifact.get('url')
        })
    
    return pd.DataFrame(metadata)

def transform_artifact_media(artifacts):
    """
    Extract media/image information from artifacts
    """
    media_data = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media_data.append({
                'artifact_id': artifact_id,
                'base_url': image.get('baseimageurl'),
                'image_width': image.get('width'),
                'image_height': image.get('height'),
                'format': image.get('format')
            })
    
    return pd.DataFrame(media_data)

def transform_artifact_colors(artifacts):
    """
    Extract color palette information from artifacts
    """
    color_data = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_percentage': color.get('percent')
            })
    
    return pd.DataFrame(color_data)
```

### Load: Inserting into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """
    Create MySQL database connection using environment variables
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata_to_db(df, connection):
    """
    Batch insert artifact metadata into database
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, dated, classification, department, 
     division, technique, medium, dimensions, creditline, accession_number, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    data = df.values.tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    print(f"Inserted {cursor.rowcount} metadata records")

def load_media_to_db(df, connection):
    """
    Batch insert media data into database
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, base_url, image_width, image_height, format)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data = df.values.tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    print(f"Inserted {cursor.rowcount} media records")

def load_colors_to_db(df, connection):
    """
    Batch insert color data into database
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color_hex, color_percentage)
    VALUES (%s, %s, %s)
    """
    
    data = df.values.tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    print(f"Inserted {cursor.rowcount} color records")
```

## Complete ETL Pipeline

```python
def run_etl_pipeline(num_records=500):
    """
    Execute complete ETL pipeline: Extract → Transform → Load
    """
    print("Starting ETL Pipeline...")
    
    # Extract
    print("Step 1: Extracting data from API...")
    artifacts = fetch_all_artifacts(max_records=num_records)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    print("Step 2: Transforming data...")
    df_metadata = transform_artifact_metadata(artifacts)
    df_media = transform_artifact_media(artifacts)
    df_colors = transform_artifact_colors(artifacts)
    print(f"Transformed: {len(df_metadata)} metadata, {len(df_media)} media, {len(df_colors)} colors")
    
    # Load
    print("Step 3: Loading data into database...")
    connection = create_database_connection()
    
    if connection and connection.is_connected():
        load_metadata_to_db(df_metadata, connection)
        load_media_to_db(df_media, connection)
        load_colors_to_db(df_colors, connection)
        connection.close()
        print("ETL Pipeline completed successfully!")
    else:
        print("Failed to connect to database")

# Execute pipeline
if __name__ == "__main__":
    run_etl_pipeline(num_records=1000)
```

## SQL Analytics Queries

### Example Analytical Queries

```python
# Query 1: Top 10 cultures by artifact count
query_cultures = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Query 2: Artifacts by century distribution
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC;
"""

# Query 3: Media availability analysis
query_media = """
SELECT 
    COUNT(DISTINCT am.artifact_id) as artifacts_with_media,
    COUNT(DISTINCT a.id) - COUNT(DISTINCT am.artifact_id) as artifacts_without_media
FROM artifactmetadata a
LEFT JOIN artifactmedia am ON a.id = am.artifact_id;
"""

# Query 4: Most common color palettes
query_colors = """
SELECT color_hex, COUNT(*) as frequency, 
       ROUND(AVG(color_percentage), 2) as avg_percentage
FROM artifactcolors
GROUP BY color_hex
ORDER BY frequency DESC
LIMIT 20;
"""

# Query 5: Department-wise artifact classification
query_dept_class = """
SELECT department, classification, COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL AND classification IS NOT NULL
GROUP BY department, classification
ORDER BY department, count DESC;
"""

def execute_query(connection, query):
    """
    Execute SQL query and return results as DataFrame
    """
    cursor = connection.cursor()
    cursor.execute(query)
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("Interactive analysis of artifact collection data")
    
    # Database connection
    connection = create_database_connection()
    
    if not connection:
        st.error("Failed to connect to database")
        return
    
    # Sidebar for query selection
    st.sidebar.header("Select Analysis")
    
    queries = {
        "Top Cultures": query_cultures,
        "Century Distribution": query_century,
        "Media Availability": query_media,
        "Color Palettes": query_colors,
        "Department Classification": query_dept_class
    }
    
    selected_query = st.sidebar.selectbox("Choose Query", list(queries.keys()))
    
    # Execute and display results
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            df_result = execute_query(connection, queries[selected_query])
            
            st.subheader(f"Results: {selected_query}")
            st.dataframe(df_result)
            
            # Generate visualization
            if len(df_result.columns) >= 2:
                fig = px.bar(
                    df_result, 
                    x=df_result.columns[0], 
                    y=df_result.columns[1],
                    title=f"{selected_query} Visualization"
                )
                st.plotly_chart(fig, use_container_width=True)
    
    connection.close()

if __name__ == "__main__":
    main()
```

### Running the Streamlit App

```bash
# Start the dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_artifact_id(connection):
    """Get the highest artifact ID currently in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(start_id):
    """Load only new artifacts since last ETL run"""
    artifacts = fetch_all_artifacts()
    new_artifacts = [a for a in artifacts if a.get('id', 0) > start_id]
    # Process only new artifacts...
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_pipeline():
    try:
        artifacts = fetch_all_artifacts()
        logger.info(f"Fetched {len(artifacts)} artifacts")
        
        # Process with error handling
        df_metadata = transform_artifact_metadata(artifacts)
        
        connection = create_database_connection()
        load_metadata_to_db(df_metadata, connection)
        
    except Exception as e:
        logger.error(f"ETL pipeline failed: {e}")
        raise
    finally:
        if connection and connection.is_connected():
            connection.close()
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(page, max_retries=3):
    for attempt in range(max_retries):
        try:
            records, info = fetch_artifacts(page=page)
            return records, info
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Rate limited
                wait_time = 2 ** attempt  # Exponential backoff
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    return [], {}
```

### Database Connection Issues

```python
def verify_database_connection():
    """Test database connectivity and schema"""
    connection = create_database_connection()
    
    if not connection:
        print("❌ Cannot connect to database")
        return False
    
    cursor = connection.cursor()
    
    # Check if tables exist
    cursor.execute("SHOW TABLES")
    tables = [table[0] for table in cursor.fetchall()]
    
    required_tables = ['artifactmetadata', 'artifactmedia', 'artifactcolors']
    missing = [t for t in required_tables if t not in tables]
    
    if missing:
        print(f"❌ Missing tables: {missing}")
        return False
    
    print("✅ Database connection and schema verified")
    connection.close()
    return True
```

### Memory Management for Large Datasets

```python
def batch_process_artifacts(batch_size=100):
    """Process artifacts in batches to manage memory"""
    page = 1
    connection = create_database_connection()
    
    while True:
        artifacts, info = fetch_artifacts(page=page, size=batch_size)
        
        if not artifacts:
            break
        
        # Transform and load batch
        df_metadata = transform_artifact_metadata(artifacts)
        load_metadata_to_db(df_metadata, connection)
        
        print(f"Processed batch {page}")
        page += 1
        
        if not info.get('next'):
            break
    
    connection.close()
```

## Key Commands Summary

```bash
# Run ETL pipeline
python etl_pipeline.py

# Start analytics dashboard
streamlit run app.py

# Run with custom record count
python etl_pipeline.py --records 5000

# Database verification
python verify_db.py
```

This skill provides a complete reference for building production-grade ETL pipelines for cultural heritage data with modern data engineering practices.
