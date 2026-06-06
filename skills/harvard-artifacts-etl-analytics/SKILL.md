---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build ETL pipeline for Harvard Art Museums data
  - create analytics dashboard from museum API
  - extract Harvard artifacts data to SQL database
  - visualize museum collection data with Streamlit
  - implement art museum data engineering pipeline
  - query and analyze Harvard Art Museums collection
  - set up artifact metadata ETL workflow
  - build interactive museum data analytics app
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Collects artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color data
- **SQL Database**: Stores structured data in relational tables (MySQL/TiDB Cloud)
- **Analytics Engine**: Executes 20+ predefined analytical SQL queries
- **Visualization Dashboard**: Interactive Streamlit app with Plotly charts

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
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
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

## Core Components

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

class HarvardAPICollector:
    def __init__(self):
        self.api_key = os.getenv('HARVARD_API_KEY')
        self.base_url = "https://api.harvardartmuseums.org/object"
    
    def fetch_artifacts(self, page=1, size=100):
        """Fetch artifacts with pagination"""
        params = {
            'apikey': self.api_key,
            'page': page,
            'size': size
        }
        response = requests.get(self.base_url, params=params)
        response.raise_for_status()
        return response.json()
    
    def collect_all_artifacts(self, max_pages=10):
        """Collect multiple pages of artifacts"""
        all_artifacts = []
        for page in range(1, max_pages + 1):
            data = self.fetch_artifacts(page=page)
            all_artifacts.extend(data.get('records', []))
            if not data.get('info', {}).get('next'):
                break
        return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

class ArtifactETL:
    def __init__(self):
        self.conn = self.create_connection()
    
    def create_connection(self):
        """Create database connection"""
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
    
    def extract_metadata(self, artifacts):
        """Extract artifact metadata"""
        metadata = []
        for artifact in artifacts:
            metadata.append({
                'objectid': artifact.get('objectid'),
                'title': artifact.get('title'),
                'culture': artifact.get('culture'),
                'century': artifact.get('century'),
                'classification': artifact.get('classification'),
                'medium': artifact.get('medium'),
                'department': artifact.get('department'),
                'dated': artifact.get('dated'),
                'period': artifact.get('period')
            })
        return pd.DataFrame(metadata)
    
    def extract_media(self, artifacts):
        """Extract media/image data"""
        media = []
        for artifact in artifacts:
            objectid = artifact.get('objectid')
            images = artifact.get('images', [])
            for img in images:
                media.append({
                    'objectid': objectid,
                    'imageid': img.get('imageid'),
                    'baseimageurl': img.get('baseimageurl'),
                    'width': img.get('width'),
                    'height': img.get('height')
                })
        return pd.DataFrame(media)
    
    def extract_colors(self, artifacts):
        """Extract color data"""
        colors = []
        for artifact in artifacts:
            objectid = artifact.get('objectid')
            color_list = artifact.get('colors', [])
            for color in color_list:
                colors.append({
                    'objectid': objectid,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'percent': color.get('percent')
                })
        return pd.DataFrame(colors)
    
    def load_to_database(self, df, table_name, batch_size=1000):
        """Batch insert data into SQL table"""
        if df.empty:
            print(f"No data to insert into {table_name}")
            return
        
        cursor = self.conn.cursor()
        
        # Create insert query
        cols = ', '.join(df.columns)
        placeholders = ', '.join(['%s'] * len(df.columns))
        query = f"INSERT IGNORE INTO {table_name} ({cols}) VALUES ({placeholders})"
        
        # Batch insert
        for i in range(0, len(df), batch_size):
            batch = df.iloc[i:i+batch_size]
            data = [tuple(row) for row in batch.values]
            cursor.executemany(query, data)
            self.conn.commit()
        
        cursor.close()
        print(f"Loaded {len(df)} records into {table_name}")
```

### 3. Database Schema Setup

```python
def create_tables(connection):
    """Create database tables"""
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            medium TEXT,
            department VARCHAR(255),
            dated VARCHAR(255),
            period VARCHAR(255)
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            imageid INT,
            baseimageurl TEXT,
            width INT,
            height INT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            percent FLOAT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    connection.commit()
    cursor.close()
    print("Database tables created successfully")
```

### 4. Analytics Queries

```python
class AnalyticsEngine:
    def __init__(self, connection):
        self.conn = connection
    
    def execute_query(self, query):
        """Execute SQL query and return DataFrame"""
        return pd.read_sql(query, self.conn)
    
    # Sample analytical queries
    QUERIES = {
        "artifacts_by_culture": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 20
        """,
        
        "artifacts_by_century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        
        "media_availability": """
            SELECT 
                CASE WHEN m.objectid IS NOT NULL THEN 'With Media' ELSE 'No Media' END as media_status,
                COUNT(*) as count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.objectid = m.objectid
            GROUP BY media_status
        """,
        
        "top_colors": """
            SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY color
            ORDER BY usage_count DESC
            LIMIT 15
        """,
        
        "department_distribution": """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY artifact_count DESC
        """
    }
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Analytics Dashboard")
    
    # Initialize connections
    etl = ArtifactETL()
    analytics = AnalyticsEngine(etl.conn)
    
    # Sidebar for query selection
    st.sidebar.header("Select Analysis")
    query_name = st.sidebar.selectbox(
        "Choose a query",
        list(analytics.QUERIES.keys())
    )
    
    # Execute query
    if st.sidebar.button("Run Analysis"):
        with st.spinner("Executing query..."):
            query = analytics.QUERIES[query_name]
            df = analytics.execute_query(query)
            
            # Display results
            st.subheader(f"Results: {query_name.replace('_', ' ').title()}")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df, 
                    x=df.columns[0], 
                    y=df.columns[1],
                    title=f"{query_name.replace('_', ' ').title()}"
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

### Start the Streamlit App

```bash
streamlit run app.py
```

The app will open at `http://localhost:8501`

### Run ETL Pipeline

```python
# In Python script or Jupyter notebook
from etl_pipeline import HarvardAPICollector, ArtifactETL

# Collect data
collector = HarvardAPICollector()
artifacts = collector.collect_all_artifacts(max_pages=5)

# Run ETL
etl = ArtifactETL()
metadata_df = etl.extract_metadata(artifacts)
media_df = etl.extract_media(artifacts)
colors_df = etl.extract_colors(artifacts)

# Load to database
etl.load_to_database(metadata_df, 'artifactmetadata')
etl.load_to_database(media_df, 'artifactmedia')
etl.load_to_database(colors_df, 'artifactcolors')
```

## Common Patterns

### Rate-Limited API Calls

```python
import time

def fetch_with_rate_limit(collector, max_pages, delay=1):
    """Fetch data with rate limiting"""
    all_data = []
    for page in range(1, max_pages + 1):
        data = collector.fetch_artifacts(page=page)
        all_data.extend(data.get('records', []))
        time.sleep(delay)  # Respect API rate limits
    return all_data
```

### Error Handling in ETL

```python
def safe_etl_run(artifacts):
    """ETL with error handling"""
    try:
        etl = ArtifactETL()
        
        # Extract
        metadata = etl.extract_metadata(artifacts)
        media = etl.extract_media(artifacts)
        colors = etl.extract_colors(artifacts)
        
        # Load
        etl.load_to_database(metadata, 'artifactmetadata')
        etl.load_to_database(media, 'artifactmedia')
        etl.load_to_database(colors, 'artifactcolors')
        
        return True
    except Exception as e:
        print(f"ETL Error: {e}")
        return False
    finally:
        if etl.conn:
            etl.conn.close()
```

## Troubleshooting

### API Connection Issues

- **Problem**: API key not recognized
- **Solution**: Verify `HARVARD_API_KEY` is set in `.env` and valid. Get key from https://docs.harvardartmuseums.org/

### Database Connection Errors

- **Problem**: Can't connect to MySQL/TiDB
- **Solution**: Check `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD` in `.env`. Ensure database exists and is accessible.

### Empty Query Results

- **Problem**: Queries return no data
- **Solution**: Run ETL pipeline first to populate tables. Check table names match schema.

### Memory Issues with Large Datasets

- **Problem**: Out of memory when processing many artifacts
- **Solution**: Use batch processing with smaller page sizes and commit data more frequently:

```python
def batch_process(collector, pages_per_batch=2, total_pages=10):
    """Process data in batches"""
    etl = ArtifactETL()
    for start_page in range(1, total_pages, pages_per_batch):
        artifacts = []
        for page in range(start_page, min(start_page + pages_per_batch, total_pages + 1)):
            data = collector.fetch_artifacts(page=page)
            artifacts.extend(data.get('records', []))
        
        # Process batch
        metadata = etl.extract_metadata(artifacts)
        etl.load_to_database(metadata, 'artifactmetadata')
```

### Streamlit Caching

```python
@st.cache_data
def load_query_results(query):
    """Cache query results for performance"""
    etl = ArtifactETL()
    return pd.read_sql(query, etl.conn)
```

This skill provides everything needed to build, extend, and troubleshoot the Harvard Artifacts ETL and analytics pipeline.
