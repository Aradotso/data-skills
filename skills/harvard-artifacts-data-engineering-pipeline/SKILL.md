---
name: harvard-artifacts-data-engineering-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - show me how to create an ETL workflow for museum artifacts
  - help me set up analytics on Harvard museum data
  - how to extract and visualize art museum data
  - build a streamlit dashboard for Harvard artifacts
  - create SQL analytics for museum collection data
  - set up a data engineering project with museum APIs
  - how do I process Harvard Art Museums API data
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates an end-to-end data engineering pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational structures, loads it into SQL databases (MySQL/TiDB), and provides interactive analytics dashboards using Streamlit and Plotly.

**Architecture**: API → ETL → SQL → Analytics → Visualization

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

### 1. Harvard Art Museums API Key

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

```bash
# Create .env file
echo "HARVARD_API_KEY=your_api_key_here" > .env
echo "DB_HOST=your_database_host" >> .env
echo "DB_USER=your_database_user" >> .env
echo "DB_PASSWORD=your_database_password" >> .env
echo "DB_NAME=harvard_artifacts" >> .env
```

### 2. Database Setup

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Create artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    people VARCHAR(1000),
    url VARCHAR(500),
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Create artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    renditionnumber VARCHAR(50),
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Create artifact colors table
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

## Running the Application

```bash
streamlit run app.py
```

The app will be available at `http://localhost:8501`

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Example: Fetch multiple pages
def collect_artifacts(num_pages=5):
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(page=page)
        all_artifacts.extend(data.get('records', []))
        print(f"Fetched page {page}, total artifacts: {len(all_artifacts)}")
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class ArtifactETL:
    def __init__(self):
        self.conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        self.cursor = self.conn.cursor()
    
    def transform_metadata(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform raw API data into metadata DataFrame"""
        metadata = []
        
        for artifact in artifacts:
            # Extract people as comma-separated string
            people_list = artifact.get('people', [])
            people_str = ', '.join([p.get('name', '') for p in people_list if p.get('name')])
            
            metadata.append({
                'id': artifact.get('id'),
                'title': artifact.get('title', '')[:500],
                'culture': artifact.get('culture', ''),
                'century': artifact.get('century', ''),
                'classification': artifact.get('classification', ''),
                'department': artifact.get('department', ''),
                'division': artifact.get('division', ''),
                'dated': artifact.get('dated', ''),
                'period': artifact.get('period', ''),
                'technique': artifact.get('technique', '')[:500],
                'medium': artifact.get('medium', '')[:500],
                'people': people_str[:1000],
                'url': artifact.get('url', ''),
                'totalpageviews': artifact.get('totalpageviews', 0),
                'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
            })
        
        return pd.DataFrame(metadata)
    
    def transform_media(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform media/image data"""
        media = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            images = artifact.get('images', [])
            
            for img in images:
                media.append({
                    'artifact_id': artifact_id,
                    'iiifbaseuri': img.get('iiifbaseuri', ''),
                    'baseimageurl': img.get('baseimageurl', ''),
                    'renditionnumber': img.get('renditionnumber', ''),
                    'format': img.get('format', ''),
                    'height': img.get('height', 0),
                    'width': img.get('width', 0)
                })
        
        return pd.DataFrame(media)
    
    def transform_colors(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform color data"""
        colors = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            color_list = artifact.get('colors', [])
            
            for color in color_list:
                colors.append({
                    'artifact_id': artifact_id,
                    'color': color.get('color', ''),
                    'spectrum': color.get('spectrum', ''),
                    'hue': color.get('hue', ''),
                    'percent': color.get('percent', 0.0)
                })
        
        return pd.DataFrame(colors)
    
    def load_to_sql(self, df: pd.DataFrame, table_name: str):
        """Batch insert DataFrame into SQL table"""
        if df.empty:
            print(f"No data to load for {table_name}")
            return
        
        # Use DataFrame's to_sql for efficient batch insert
        from sqlalchemy import create_engine
        
        db_url = f"mysql+mysqlconnector://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}/{os.getenv('DB_NAME')}"
        engine = create_engine(db_url)
        
        df.to_sql(table_name, engine, if_exists='append', index=False)
        print(f"Loaded {len(df)} records to {table_name}")
    
    def run_pipeline(self, num_pages=5):
        """Execute full ETL pipeline"""
        print("Starting ETL pipeline...")
        
        # Extract
        artifacts = collect_artifacts(num_pages)
        print(f"Extracted {len(artifacts)} artifacts")
        
        # Transform
        metadata_df = self.transform_metadata(artifacts)
        media_df = self.transform_media(artifacts)
        colors_df = self.transform_colors(artifacts)
        
        # Load
        self.load_to_sql(metadata_df, 'artifactmetadata')
        self.load_to_sql(media_df, 'artifactmedia')
        self.load_to_sql(colors_df, 'artifactcolors')
        
        print("ETL pipeline completed!")

# Run the pipeline
if __name__ == "__main__":
    etl = ArtifactETL()
    etl.run_pipeline(num_pages=10)
```

### 3. SQL Analytics Queries

```python
# Common analytical queries

ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN m.artifact_id IS NOT NULL THEN 'Has Media' ELSE 'No Media' END as media_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY media_status
    """,
    
    "Most Popular Artifacts": """
        SELECT title, culture, totalpageviews, totaluniquepageviews
        FROM artifactmetadata
        WHERE totalpageviews > 0
        ORDER BY totalpageviews DESC
        LIMIT 20
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Artifacts with Most Colors": """
        SELECT a.title, a.culture, COUNT(c.color) as color_count
        FROM artifactmetadata a
        JOIN artifactcolors c ON a.id = c.artifact_id
        GROUP BY a.id, a.title, a.culture
        ORDER BY color_count DESC
        LIMIT 20
    """
}

def execute_query(query: str):
    """Execute analytical query and return results"""
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for query selection
st.sidebar.header("Analytics Queries")
query_name = st.sidebar.selectbox(
    "Select Analysis",
    list(ANALYTICS_QUERIES.keys())
)

# Execute selected query
if st.button("Run Analysis"):
    with st.spinner("Executing query..."):
        query = ANALYTICS_QUERIES[query_name]
        results = execute_query(query)
        
        st.subheader(f"Results: {query_name}")
        st.dataframe(results)
        
        # Auto-generate visualization
        if len(results.columns) >= 2:
            fig = px.bar(
                results.head(20),
                x=results.columns[0],
                y=results.columns[1],
                title=query_name
            )
            st.plotly_chart(fig, use_container_width=True)

# ETL Pipeline Section
st.sidebar.header("Data Collection")
num_pages = st.sidebar.slider("Pages to fetch", 1, 20, 5)

if st.sidebar.button("Run ETL Pipeline"):
    with st.spinner("Running ETL pipeline..."):
        etl = ArtifactETL()
        etl.run_pipeline(num_pages=num_pages)
        st.success("ETL pipeline completed!")
```

## Common Patterns

### Handling API Rate Limits

```python
import time

def fetch_with_rate_limit(pages, delay=1):
    """Fetch artifacts with rate limiting"""
    artifacts = []
    
    for page in range(1, pages + 1):
        try:
            data = fetch_artifacts(page=page)
            artifacts.extend(data.get('records', []))
            time.sleep(delay)  # Rate limit delay
        except Exception as e:
            print(f"Error on page {page}: {e}")
            continue
    
    return artifacts
```

### Incremental Data Loading

```python
def get_max_artifact_id():
    """Get the latest artifact ID in database"""
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    conn.close()
    return result or 0

def incremental_load():
    """Load only new artifacts"""
    max_id = get_max_artifact_id()
    # Fetch artifacts with ID > max_id
    # Implementation depends on API capabilities
```

## Troubleshooting

### Database Connection Issues

```python
# Test database connection
def test_db_connection():
    try:
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        print("✓ Database connection successful")
        conn.close()
        return True
    except Exception as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### API Key Validation

```python
def validate_api_key():
    """Test if API key is valid"""
    try:
        data = fetch_artifacts(page=1, size=1)
        print("✓ API key is valid")
        return True
    except Exception as e:
        print(f"✗ API key validation failed: {e}")
        return False
```

### Handle Missing Data

```python
def safe_get(dictionary, key, default=''):
    """Safely get value from dictionary"""
    value = dictionary.get(key, default)
    return value if value is not None else default
```

## Best Practices

1. **Environment Variables**: Always use `.env` for credentials
2. **Batch Processing**: Use batch inserts for performance
3. **Error Handling**: Wrap API calls in try-except blocks
4. **Data Validation**: Check for null/empty values before inserting
5. **Indexing**: Add indexes on frequently queried columns (culture, century, department)
6. **Pagination**: Handle large datasets with pagination
7. **Logging**: Implement logging for production deployments

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

logger.info("Starting ETL pipeline")
logger.error(f"Failed to fetch page {page}: {error}")
```
