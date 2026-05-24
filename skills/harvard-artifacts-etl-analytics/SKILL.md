---
name: harvard-artifacts-etl-analytics
description: ETL pipeline and analytics application for Harvard Art Museums API with Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering pipeline with museum artifacts
  - set up Harvard Arts API data collection and analytics
  - implement artifact metadata extraction and SQL storage
  - visualize museum collection data with Streamlit
  - process Harvard Art Museums API into relational database
  - analyze art museum data with SQL queries
  - create an end-to-end data pipeline for art collections
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for collecting, processing, and analyzing artifact data from the Harvard Art Museums API. It implements a complete ETL pipeline that extracts artifact metadata, transforms nested JSON into relational structures, loads data into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics through a Streamlit dashboard.

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

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

**Required packages:**
- streamlit
- pandas
- requests
- mysql-connector-python (or pymysql)
- plotly
- python-dotenv

## Configuration

### API Configuration

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

Store credentials in `.env` file:
```env
HARVARD_API_KEY=your_api_key
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Setup

Create the database schema with three core tables:

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

def create_database_schema():
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD')
    )
    cursor = conn.cursor()
    
    # Create database
    cursor.execute(f"CREATE DATABASE IF NOT EXISTS {os.getenv('DB_NAME')}")
    cursor.execute(f"USE {os.getenv('DB_NAME')}")
    
    # Create artifactmetadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            accession_year INT,
            object_number VARCHAR(100),
            technique VARCHAR(500),
            medium VARCHAR(500),
            dimensions VARCHAR(500),
            provenance TEXT,
            description TEXT,
            url VARCHAR(500)
        )
    """)
    
    # Create artifactmedia table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_url VARCHAR(1000),
            image_width INT,
            image_height INT,
            media_type VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Create artifactcolors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_hex VARCHAR(10),
            color_name VARCHAR(100),
            color_percentage DECIMAL(5,2),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    conn.commit()
    cursor.close()
    conn.close()

create_database_schema()
```

## ETL Pipeline Implementation

### Extract: Fetch Data from Harvard API

```python
import requests
import time

class HarvardAPIExtractor:
    def __init__(self, api_key):
        self.api_key = api_key
        self.base_url = "https://api.harvardartmuseums.org/object"
        
    def extract_artifacts(self, num_pages=10, page_size=100):
        """Extract artifacts with pagination"""
        artifacts = []
        
        for page in range(1, num_pages + 1):
            params = {
                'apikey': self.api_key,
                'size': page_size,
                'page': page,
                'hasimage': 1  # Only artifacts with images
            }
            
            try:
                response = requests.get(self.base_url, params=params)
                response.raise_for_status()
                data = response.json()
                
                artifacts.extend(data.get('records', []))
                print(f"Extracted page {page}: {len(data.get('records', []))} artifacts")
                
                # Rate limiting
                time.sleep(0.5)
                
            except requests.exceptions.RequestException as e:
                print(f"Error fetching page {page}: {e}")
                continue
                
        return artifacts

# Usage
extractor = HarvardAPIExtractor(os.getenv('HARVARD_API_KEY'))
raw_artifacts = extractor.extract_artifacts(num_pages=5)
```

### Transform: Convert JSON to Relational Format

```python
import pandas as pd

class ArtifactTransformer:
    def transform_metadata(self, artifacts):
        """Transform artifact metadata"""
        metadata = []
        
        for artifact in artifacts:
            metadata.append({
                'id': artifact.get('id'),
                'title': artifact.get('title'),
                'culture': artifact.get('culture'),
                'period': artifact.get('period'),
                'century': artifact.get('century'),
                'classification': artifact.get('classification'),
                'department': artifact.get('department'),
                'dated': artifact.get('dated'),
                'accession_year': artifact.get('accessionyear'),
                'object_number': artifact.get('objectnumber'),
                'technique': artifact.get('technique'),
                'medium': artifact.get('medium'),
                'dimensions': artifact.get('dimensions'),
                'provenance': artifact.get('provenance'),
                'description': artifact.get('description'),
                'url': artifact.get('url')
            })
        
        return pd.DataFrame(metadata)
    
    def transform_media(self, artifacts):
        """Transform media/image data"""
        media_records = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            images = artifact.get('images', [])
            
            for img in images:
                media_records.append({
                    'artifact_id': artifact_id,
                    'image_url': img.get('baseimageurl'),
                    'image_width': img.get('width'),
                    'image_height': img.get('height'),
                    'media_type': 'image'
                })
        
        return pd.DataFrame(media_records)
    
    def transform_colors(self, artifacts):
        """Transform color data"""
        color_records = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            colors = artifact.get('colors', [])
            
            for color in colors:
                color_records.append({
                    'artifact_id': artifact_id,
                    'color_hex': color.get('hex'),
                    'color_name': color.get('color'),
                    'color_percentage': color.get('percent')
                })
        
        return pd.DataFrame(color_records)

# Usage
transformer = ArtifactTransformer()
metadata_df = transformer.transform_metadata(raw_artifacts)
media_df = transformer.transform_media(raw_artifacts)
colors_df = transformer.transform_colors(raw_artifacts)
```

### Load: Batch Insert into SQL

```python
class SQLLoader:
    def __init__(self):
        self.conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
    def load_metadata(self, df):
        """Batch insert metadata"""
        cursor = self.conn.cursor()
        
        # Use INSERT IGNORE to handle duplicates
        insert_query = """
            INSERT IGNORE INTO artifactmetadata 
            (id, title, culture, period, century, classification, department, 
             dated, accession_year, object_number, technique, medium, 
             dimensions, provenance, description, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """
        
        data = df.values.tolist()
        cursor.executemany(insert_query, data)
        self.conn.commit()
        print(f"Loaded {cursor.rowcount} metadata records")
        
    def load_media(self, df):
        """Batch insert media records"""
        cursor = self.conn.cursor()
        
        insert_query = """
            INSERT INTO artifactmedia 
            (artifact_id, image_url, image_width, image_height, media_type)
            VALUES (%s, %s, %s, %s, %s)
        """
        
        data = df.values.tolist()
        cursor.executemany(insert_query, data)
        self.conn.commit()
        print(f"Loaded {cursor.rowcount} media records")
        
    def load_colors(self, df):
        """Batch insert color records"""
        cursor = self.conn.cursor()
        
        insert_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color_hex, color_name, color_percentage)
            VALUES (%s, %s, %s, %s)
        """
        
        data = df.values.tolist()
        cursor.executemany(insert_query, data)
        self.conn.commit()
        print(f"Loaded {cursor.rowcount} color records")

# Usage
loader = SQLLoader()
loader.load_metadata(metadata_df)
loader.load_media(media_df)
loader.load_colors(colors_df)
```

## Streamlit Analytics Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        data_collection_page()
    elif page == "SQL Analytics":
        sql_analytics_page()
    else:
        visualizations_page()

def data_collection_page():
    st.header("📥 Data Collection")
    
    api_key = st.text_input("Harvard API Key", type="password")
    num_pages = st.number_input("Number of pages to collect", 1, 50, 5)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Running ETL..."):
            # Run extraction
            extractor = HarvardAPIExtractor(api_key)
            artifacts = extractor.extract_artifacts(num_pages=num_pages)
            
            # Transform
            transformer = ArtifactTransformer()
            metadata = transformer.transform_metadata(artifacts)
            media = transformer.transform_media(artifacts)
            colors = transformer.transform_colors(artifacts)
            
            # Load
            loader = SQLLoader()
            loader.load_metadata(metadata)
            loader.load_media(media)
            loader.load_colors(colors)
            
            st.success(f"✅ Successfully loaded {len(artifacts)} artifacts!")

if __name__ == "__main__":
    main()
```

### SQL Analytics Queries

```python
def sql_analytics_page():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 15
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
        "Most Common Colors": """
            SELECT color_name, COUNT(*) as usage_count,
                   AVG(color_percentage) as avg_percentage
            FROM artifactcolors
            GROUP BY color_name
            ORDER BY usage_count DESC
            LIMIT 20
        """,
        "Artifacts with Most Images": """
            SELECT am.title, am.culture, COUNT(m.media_id) as image_count
            FROM artifactmetadata am
            JOIN artifactmedia m ON am.id = m.artifact_id
            GROUP BY am.id, am.title, am.culture
            ORDER BY image_count DESC
            LIMIT 10
        """,
        "Classification Analysis": """
            SELECT classification, COUNT(*) as count,
                   COUNT(DISTINCT culture) as cultures
            FROM artifactmetadata
            WHERE classification IS NOT NULL
            GROUP BY classification
            ORDER BY count DESC
        """
    }
    
    query_name = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        df = pd.read_sql(queries[query_name], conn)
        
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)
        
        conn.close()
```

### Custom Visualization Dashboard

```python
def visualizations_page():
    st.header("📈 Data Visualizations")
    
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    # Culture distribution
    culture_query = """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """
    culture_df = pd.read_sql(culture_query, conn)
    
    fig1 = px.pie(culture_df, values='count', names='culture',
                  title='Top 10 Cultures in Collection')
    st.plotly_chart(fig1, use_container_width=True)
    
    # Color usage heatmap
    color_query = """
        SELECT color_name, AVG(color_percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color_name
        ORDER BY avg_percentage DESC
        LIMIT 15
    """
    color_df = pd.read_sql(color_query, conn)
    
    fig2 = px.bar(color_df, x='color_name', y='avg_percentage',
                  title='Average Color Usage Percentage',
                  color='avg_percentage', color_continuous_scale='viridis')
    st.plotly_chart(fig2, use_container_width=True)
    
    conn.close()
```

## Common Patterns

### Complete ETL Pipeline Function

```python
def run_complete_etl(api_key, num_pages=5):
    """Run the complete ETL pipeline"""
    # Extract
    print("Starting extraction...")
    extractor = HarvardAPIExtractor(api_key)
    artifacts = extractor.extract_artifacts(num_pages=num_pages)
    
    # Transform
    print("Transforming data...")
    transformer = ArtifactTransformer()
    metadata_df = transformer.transform_metadata(artifacts)
    media_df = transformer.transform_media(artifacts)
    colors_df = transformer.transform_colors(artifacts)
    
    # Load
    print("Loading to database...")
    loader = SQLLoader()
    loader.load_metadata(metadata_df)
    loader.load_media(media_df)
    loader.load_colors(colors_df)
    
    print(f"ETL Complete: {len(artifacts)} artifacts processed")
    return len(artifacts)
```

### Incremental Data Updates

```python
def get_latest_artifact_id():
    """Get the highest artifact ID in database"""
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

def incremental_update():
    """Fetch only new artifacts"""
    latest_id = get_latest_artifact_id()
    
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'size': 100,
        'sort': 'id',
        'sortorder': 'desc'
    }
    
    response = requests.get(
        "https://api.harvardartmuseums.org/object",
        params=params
    )
    
    new_artifacts = [
        a for a in response.json().get('records', [])
        if a.get('id') > latest_id
    ]
    
    if new_artifacts:
        transformer = ArtifactTransformer()
        loader = SQLLoader()
        
        loader.load_metadata(transformer.transform_metadata(new_artifacts))
        loader.load_media(transformer.transform_media(new_artifacts))
        loader.load_colors(transformer.transform_colors(new_artifacts))
        
    return len(new_artifacts)
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def get_session_with_retry():
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

### Database Connection Issues
```python
# Use connection pooling
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

### Memory Management for Large Datasets
```python
# Process in chunks
def load_in_chunks(df, chunk_size=1000):
    loader = SQLLoader()
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i+chunk_size]
        loader.load_metadata(chunk)
        print(f"Processed {i+len(chunk)}/{len(df)} records")
```

### Handling Missing Data
```python
# Clean data before loading
def clean_dataframe(df):
    # Replace None with empty strings for text columns
    text_columns = df.select_dtypes(include=['object']).columns
    df[text_columns] = df[text_columns].fillna('')
    
    # Replace None with 0 for numeric columns
    numeric_columns = df.select_dtypes(include=['int64', 'float64']).columns
    df[numeric_columns] = df[numeric_columns].fillna(0)
    
    return df
```

## Running the Application

```bash
# Launch Streamlit dashboard
streamlit run app.py

# Run ETL from command line
python -c "from etl import run_complete_etl; import os; run_complete_etl(os.getenv('HARVARD_API_KEY'), 10)"
```
