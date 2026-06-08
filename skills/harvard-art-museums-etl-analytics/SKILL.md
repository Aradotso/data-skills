---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API data with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifact data
  - connect to Harvard Art Museums API
  - create analytics dashboard with Streamlit
  - set up artifact data warehouse
  - extract and transform museum collection data
  - visualize art museum data with SQL queries
  - implement batch data ingestion for artifacts
  - analyze museum artifacts by culture and century
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering solution for the Harvard Art Museums API, featuring:

- **ETL Pipeline**: Extract artifact data from Harvard API, transform nested JSON into relational format, load into SQL database
- **SQL Analytics**: 20+ predefined analytical queries for artifact insights
- **Streamlit Dashboard**: Interactive visualization interface with Plotly charts
- **Batch Processing**: Efficient pagination and rate-limited data collection

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Install core packages if requirements.txt unavailable
pip install streamlit pandas requests mysql-connector-python plotly sqlalchemy
```

## Configuration

### API Key Setup

Obtain a free API key from [Harvard Art Museums](https://harvardartmuseums.org/collections/api).

Store credentials in environment variables:

```bash
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

Or create a `.env` file:

```
HARVARD_API_KEY=your_api_key_here
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Schema

The project uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    creditline TEXT,
    accession_number VARCHAR(100),
    url TEXT
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_image_url TEXT,
    format VARCHAR(50),
    description TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. ETL Pipeline

**Extract**: Fetch data from Harvard API with pagination

```python
import requests
import pandas as pd
import os

def extract_artifacts(api_key, pages=10, per_page=100):
    """Extract artifact data from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, pages + 1):
        params = {
            "apikey": api_key,
            "size": per_page,
            "page": page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}/{pages}")
        else:
            print(f"Error on page {page}: {response.status_code}")
            break
    
    return all_artifacts
```

**Transform**: Convert nested JSON to relational format

```python
def transform_artifacts(artifacts):
    """Transform raw artifact data into structured tables"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'creditline': artifact.get('creditline'),
            'accession_number': artifact.get('accessionyear'),
            'url': artifact.get('url')
        }
        metadata_records.append(metadata)
        
        # Extract media information
        if artifact.get('primaryimageurl'):
            media = {
                'artifact_id': artifact.get('id'),
                'base_image_url': artifact.get('primaryimageurl'),
                'format': 'image',
                'description': artifact.get('title')
            }
            media_records.append(media)
        
        # Extract color information
        for color in artifact.get('colors', []):
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percentage': color.get('percent')
            }
            color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

**Load**: Batch insert into SQL database

```python
import mysql.connector
from sqlalchemy import create_engine

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """Load transformed data into SQL database"""
    connection_string = f"mysql+mysqlconnector://{db_config['user']}:{db_config['password']}@{db_config['host']}/{db_config['database']}"
    engine = create_engine(connection_string)
    
    # Load with replace to handle duplicates
    metadata_df.to_sql('artifactmetadata', engine, if_exists='append', index=False)
    media_df.to_sql('artifactmedia', engine, if_exists='append', index=False)
    colors_df.to_sql('artifactcolors', engine, if_exists='append', index=False)
    
    print(f"Loaded {len(metadata_df)} artifacts, {len(media_df)} media, {len(colors_df)} colors")
```

### 2. Streamlit Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    menu = st.sidebar.selectbox(
        "Select Module",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if menu == "Data Collection":
        data_collection_module()
    elif menu == "SQL Analytics":
        sql_analytics_module()
    elif menu == "Visualizations":
        visualization_module()

def data_collection_module():
    st.header("📥 Data Collection from API")
    
    api_key = st.text_input("Harvard API Key", type="password", value=os.getenv("HARVARD_API_KEY", ""))
    pages = st.slider("Number of pages to fetch", 1, 50, 10)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Extracting data..."):
            artifacts = extract_artifacts(api_key, pages=pages)
            st.success(f"Extracted {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Data transformed")
        
        with st.spinner("Loading to database..."):
            db_config = {
                'host': os.getenv('DB_HOST'),
                'user': os.getenv('DB_USER'),
                'password': os.getenv('DB_PASSWORD'),
                'database': os.getenv('DB_NAME')
            }
            load_to_database(metadata_df, media_df, colors_df, db_config)
            st.success("Data loaded successfully!")
```

### 3. SQL Analytics Queries

```python
def sql_analytics_module():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Artifacts by Culture": """
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
            SELECT department, COUNT(*) as artifact_count 
            FROM artifactmetadata 
            GROUP BY department 
            ORDER BY artifact_count DESC
        """,
        "Color Distribution": """
            SELECT color, COUNT(*) as usage_count, AVG(percentage) as avg_percentage 
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY usage_count DESC 
            LIMIT 10
        """,
        "Media Availability": """
            SELECT 
                COUNT(DISTINCT am.artifact_id) as artifacts_with_media,
                COUNT(DISTINCT amd.id) as total_artifacts,
                ROUND(COUNT(DISTINCT am.artifact_id) * 100.0 / COUNT(DISTINCT amd.id), 2) as media_percentage
            FROM artifactmetadata amd
            LEFT JOIN artifactmedia am ON amd.id = am.artifact_id
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        result_df = execute_query(queries[selected_query])
        st.dataframe(result_df)
        
        # Auto-generate visualization
        if len(result_df.columns) == 2:
            fig = px.bar(result_df, x=result_df.columns[0], y=result_df.columns[1])
            st.plotly_chart(fig, use_container_width=True)

def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    engine = create_engine(f"mysql+mysqlconnector://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}/{os.getenv('DB_NAME')}")
    return pd.read_sql(query, engine)
```

### 4. Advanced Analytics Examples

```python
# Complex join query: Artifacts with most diverse colors
complex_query = """
    SELECT 
        am.title,
        am.culture,
        COUNT(DISTINCT ac.color) as color_diversity,
        GROUP_CONCAT(DISTINCT ac.color ORDER BY ac.percentage DESC) as colors
    FROM artifactmetadata am
    JOIN artifactcolors ac ON am.id = ac.artifact_id
    GROUP BY am.id, am.title, am.culture
    HAVING color_diversity > 3
    ORDER BY color_diversity DESC
    LIMIT 20
"""

# Time-based analysis
century_analysis = """
    SELECT 
        century,
        COUNT(*) as total_artifacts,
        COUNT(DISTINCT classification) as unique_classifications,
        COUNT(DISTINCT culture) as unique_cultures
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY century
"""

# Color spectrum analysis
color_spectrum = """
    SELECT 
        spectrum,
        COUNT(*) as usage_count,
        ROUND(AVG(percentage), 2) as avg_percentage,
        COUNT(DISTINCT artifact_id) as unique_artifacts
    FROM artifactcolors
    GROUP BY spectrum
    ORDER BY usage_count DESC
"""
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_artifact_id(engine):
    """Get the highest artifact ID already in database"""
    query = "SELECT MAX(id) as max_id FROM artifactmetadata"
    result = pd.read_sql(query, engine)
    return result['max_id'][0] or 0

def incremental_load(api_key, last_id):
    """Load only new artifacts since last ETL run"""
    url = f"https://api.harvardartmuseums.org/object?apikey={api_key}&id_after={last_id}"
    # Continue with extraction logic
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_pipeline(api_key, pages):
    """ETL pipeline with error handling"""
    try:
        artifacts = extract_artifacts(api_key, pages)
        logger.info(f"Extracted {len(artifacts)} artifacts")
        
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        logger.info("Transformation complete")
        
        load_to_database(metadata_df, media_df, colors_df, db_config)
        logger.info("Load complete")
        
        return True
    except requests.exceptions.RequestException as e:
        logger.error(f"API error: {e}")
        return False
    except Exception as e:
        logger.error(f"Pipeline error: {e}")
        return False
```

## Troubleshooting

**API Rate Limiting**: Add delays between requests
```python
import time
time.sleep(0.5)  # 500ms delay between API calls
```

**Database Connection Issues**: Verify credentials and network access
```python
try:
    engine = create_engine(connection_string)
    with engine.connect() as conn:
        conn.execute("SELECT 1")
    print("Database connection successful")
except Exception as e:
    print(f"Connection failed: {e}")
```

**Memory Issues with Large Datasets**: Process in smaller batches
```python
def batch_load(artifacts, batch_size=1000):
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        # Process batch
```

**Streamlit Caching**: Use for expensive operations
```python
@st.cache_data(ttl=3600)
def load_data_cached():
    return pd.read_sql("SELECT * FROM artifactmetadata", engine)
```
