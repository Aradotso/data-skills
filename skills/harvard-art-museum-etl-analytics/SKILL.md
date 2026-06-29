---
name: harvard-art-museum-etl-analytics
description: Build end-to-end data pipelines with Harvard Art Museums API, SQL databases, and Streamlit analytics dashboards
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums data
  - create a data pipeline using Harvard Art Museums API
  - set up analytics dashboard for museum artifact data
  - extract and analyze Harvard Art Museums collection
  - build Streamlit app with museum API data
  - query Harvard Art Museums artifacts with SQL
  - create museum data engineering pipeline
  - visualize Harvard Art Museums collection data
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application for the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations for museum artifact collections.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App enables:

- **API Integration**: Fetches artifact data from Harvard Art Museums with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON into normalized SQL tables
- **Database Design**: Stores data in relational schema (artifacts, media, colors)
- **SQL Analytics**: Runs 20+ predefined analytical queries on the collection
- **Interactive Dashboards**: Visualizes insights using Streamlit and Plotly

**Architecture**: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Get your free API key from [Harvard Art Museums API](https://harvardartmuseums.org/collections/api).

Store it securely using environment variables:

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

The project supports MySQL or TiDB Cloud. Configure connection details:

```python
# config.py or environment variables
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### 3. Database Schema Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    url VARCHAR(500)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
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

## Key Components

### ETL Pipeline

#### Extract: Fetch Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    all_artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(all_artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': min(size, num_records - len(all_artifacts)),
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            all_artifacts.extend(records)
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_artifacts[:num_records]
```

#### Transform: Normalize JSON to DataFrames

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform nested JSON into relational dataframes"""
    
    # Metadata extraction
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media info
        if 'primaryimageurl' in artifact or 'baseimageurl' in artifact:
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl'),
                'primaryimageurl': artifact.get('primaryimageurl'),
                'iiifbaseuri': artifact.get('iiifbaseuri')
            }
            media_list.append(media)
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_entry = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            colors_list.append(color_entry)
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

#### Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(df_metadata, df_media, df_colors):
    """Load dataframes into MySQL database"""
    
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        if connection.is_connected():
            cursor = connection.cursor()
            
            # Insert metadata (batch insert)
            metadata_query = """
                INSERT INTO artifactmetadata 
                (id, title, culture, period, century, dated, classification, 
                 department, technique, medium, dimensions, creditline, 
                 accessionyear, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """
            cursor.executemany(metadata_query, df_metadata.values.tolist())
            
            # Insert media
            media_query = """
                INSERT INTO artifactmedia 
                (artifact_id, baseimageurl, primaryimageurl, iiifbaseuri)
                VALUES (%s, %s, %s, %s)
            """
            cursor.executemany(media_query, df_media.values.tolist())
            
            # Insert colors
            colors_query = """
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """
            cursor.executemany(colors_query, df_colors.values.tolist())
            
            connection.commit()
            print(f"Loaded {len(df_metadata)} artifacts successfully")
            
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### Complete ETL Workflow

```python
def run_etl_pipeline(num_records=100):
    """Execute complete ETL pipeline"""
    print("Starting ETL pipeline...")
    
    # Extract
    print(f"Extracting {num_records} artifacts from API...")
    artifacts = fetch_artifacts(num_records)
    
    # Transform
    print("Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(artifacts)
    
    # Load
    print("Loading to database...")
    load_to_database(df_metadata, df_media, df_colors)
    
    print("ETL pipeline completed!")
    
    return df_metadata, df_media, df_colors

# Run the pipeline
if __name__ == "__main__":
    run_etl_pipeline(num_records=500)
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Query 1: Artifact count by culture
query_by_culture = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query 2: Artifacts by century
query_by_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY century
"""

# Query 3: Most common colors
query_top_colors = """
    SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
    FROM artifactcolors
    GROUP BY color
    ORDER BY frequency DESC
    LIMIT 15
"""

# Query 4: Department distribution
query_by_department = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY count DESC
"""

# Query 5: Media availability
query_media_availability = """
    SELECT 
        COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
        (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
        ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / 
              (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
    FROM artifactmedia m
"""

# Query 6: Artifacts by accession year
query_by_accession = """
    SELECT accessionyear, COUNT(*) as count
    FROM artifactmetadata
    WHERE accessionyear IS NOT NULL AND accessionyear > 1800
    GROUP BY accessionyear
    ORDER BY accessionyear DESC
    LIMIT 20
"""
```

### Execute Queries

```python
def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        df = pd.read_sql(query, connection)
        return df
        
    except Error as e:
        print(f"Query error: {e}")
        return None
    finally:
        if connection.is_connected():
            connection.close()
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Collection Analytics")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a section",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    else:
        show_visualizations_page()

def show_etl_page():
    """ETL pipeline interface"""
    st.header("📊 ETL Pipeline")
    
    num_records = st.number_input(
        "Number of records to fetch",
        min_value=10,
        max_value=1000,
        value=100,
        step=10
    )
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            df_metadata, df_media, df_colors = run_etl_pipeline(num_records)
            
            st.success(f"✅ Loaded {len(df_metadata)} artifacts!")
            
            # Show sample data
            st.subheader("Sample Metadata")
            st.dataframe(df_metadata.head(10))
            
            st.subheader("Sample Media Data")
            st.dataframe(df_media.head(10))
            
            st.subheader("Sample Color Data")
            st.dataframe(df_colors.head(10))

def show_analytics_page():
    """SQL analytics interface"""
    st.header("🔍 SQL Analytics Dashboard")
    
    queries = {
        "Artifacts by Culture": query_by_culture,
        "Artifacts by Century": query_by_century,
        "Top Colors": query_top_colors,
        "Department Distribution": query_by_department,
        "Media Availability": query_media_availability,
        "Recent Accessions": query_by_accession
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Execute Query"):
        with st.spinner("Running query..."):
            result_df = execute_query(queries[selected_query])
            
            if result_df is not None and not result_df.empty:
                st.subheader("Query Results")
                st.dataframe(result_df)
                
                # Auto-generate visualization
                if len(result_df.columns) >= 2:
                    fig = px.bar(
                        result_df.head(15),
                        x=result_df.columns[0],
                        y=result_df.columns[1],
                        title=selected_query
                    )
                    st.plotly_chart(fig, use_container_width=True)

def show_visualizations_page():
    """Interactive visualizations"""
    st.header("📈 Interactive Visualizations")
    
    # Culture distribution
    df_culture = execute_query(query_by_culture)
    if df_culture is not None:
        fig1 = px.bar(df_culture, x='culture', y='artifact_count',
                      title='Top 10 Cultures by Artifact Count')
        st.plotly_chart(fig1, use_container_width=True)
    
    # Color distribution
    df_colors = execute_query(query_top_colors)
    if df_colors is not None:
        fig2 = px.pie(df_colors, values='frequency', names='color',
                      title='Color Distribution in Collection')
        st.plotly_chart(fig2, use_container_width=True)

if __name__ == "__main__":
    main()
```

### Run the Streamlit App

```bash
streamlit run app.py
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def get_last_artifact_id():
    """Get the maximum artifact ID already loaded"""
    query = "SELECT MAX(id) as max_id FROM artifactmetadata"
    df = execute_query(query)
    return df['max_id'].iloc[0] if df is not None else 0

def fetch_new_artifacts_only():
    """Fetch only artifacts not yet in database"""
    last_id = get_last_artifact_id()
    
    # Modify API call to filter by ID
    # This prevents duplicate data loading
    pass
```

### Pattern 2: Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def robust_etl_pipeline(num_records):
    """ETL with comprehensive error handling"""
    try:
        logger.info(f"Starting ETL for {num_records} records")
        
        artifacts = fetch_artifacts(num_records)
        logger.info(f"Extracted {len(artifacts)} artifacts")
        
        df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        logger.info("Transformation complete")
        
        load_to_database(df_metadata, df_media, df_colors)
        logger.info("Load complete")
        
        return True
        
    except Exception as e:
        logger.error(f"ETL pipeline failed: {str(e)}")
        return False
```

### Pattern 3: Data Quality Checks

```python
def validate_data_quality(df_metadata):
    """Check data quality metrics"""
    checks = {
        'total_records': len(df_metadata),
        'null_titles': df_metadata['title'].isnull().sum(),
        'null_cultures': df_metadata['culture'].isnull().sum(),
        'duplicate_ids': df_metadata['id'].duplicated().sum(),
        'accession_year_range': (
            df_metadata['accessionyear'].min(),
            df_metadata['accessionyear'].max()
        )
    }
    
    return checks
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_rate_limit(num_records, delay=1):
    """Add delay between API calls"""
    artifacts = []
    
    for page in range(1, (num_records // 100) + 2):
        response = requests.get(url, params={'page': page})
        artifacts.extend(response.json()['records'])
        time.sleep(delay)  # Respect rate limits
    
    return artifacts
```

### Database Connection Issues

```python
from mysql.connector import pooling

# Use connection pooling for better performance
db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_connection():
    return db_pool.get_connection()
```

### Missing Environment Variables

```python
import sys

required_vars = ['HARVARD_API_KEY', 'DB_HOST', 'DB_USER', 'DB_PASSWORD']

for var in required_vars:
    if not os.getenv(var):
        print(f"Error: {var} not set in environment")
        sys.exit(1)
```

### Large Dataset Memory Issues

```python
def batch_load(artifacts, batch_size=100):
    """Load data in batches to avoid memory issues"""
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        df_meta, df_media, df_colors = transform_artifacts(batch)
        load_to_database(df_meta, df_media, df_colors)
```

This skill provides comprehensive guidance for building ETL pipelines with the Harvard Art Museums API, implementing SQL analytics, and creating interactive Streamlit dashboards for museum collection data.
