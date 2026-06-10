---
name: harvard-artifacts-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - create a data engineering workflow for museum artifacts
  - set up Harvard artifacts collection analytics with Streamlit
  - implement SQL analytics for Harvard Art Museums data
  - build a data pipeline from API to database to visualization
  - extract and transform Harvard museum data into relational tables
  - create interactive dashboards for artifact collection analytics
  - design a data warehouse for museum artifact metadata
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming nested JSON into relational structures, loading into SQL databases (MySQL/TiDB Cloud), and visualizing analytics through Streamlit dashboards. It showcases real-world ETL patterns, API pagination handling, and SQL-based analytics.

## Architecture

```
Harvard Art Museums API → ETL Pipeline → SQL Database → Analytics Queries → Streamlit Dashboard
```

**Core Components:**
- **Extraction**: API requests with pagination and rate limiting
- **Transformation**: JSON to relational table mapping (artifacts, media, colors)
- **Loading**: Batch SQL inserts with foreign key relationships
- **Analytics**: 20+ predefined SQL queries
- **Visualization**: Plotly charts integrated with Streamlit

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

### Environment Variables

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

### Database Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    division VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    medium VARCHAR(500),
    technique VARCHAR(500),
    period VARCHAR(200),
    provenance TEXT,
    description TEXT,
    accessionyear INT,
    verificationlevel INT,
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Media/images table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    primaryimageurl VARCHAR(500),
    imagecount INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## API Integration Patterns

### Basic API Request with Pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

class HarvardAPIClient:
    def __init__(self):
        self.api_key = os.getenv('HARVARD_API_KEY')
        self.base_url = 'https://api.harvardartmuseums.org'
    
    def fetch_objects(self, page=1, size=100):
        """Fetch objects with pagination"""
        endpoint = f"{self.base_url}/object"
        params = {
            'apikey': self.api_key,
            'page': page,
            'size': size,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(endpoint, params=params)
        response.raise_for_status()
        return response.json()
    
    def fetch_all_objects(self, max_records=1000):
        """Fetch multiple pages of objects"""
        all_records = []
        page = 1
        size = 100
        
        while len(all_records) < max_records:
            data = self.fetch_objects(page=page, size=size)
            records = data.get('records', [])
            
            if not records:
                break
            
            all_records.extend(records)
            page += 1
            
            # Respect rate limits
            import time
            time.sleep(0.5)
        
        return all_records[:max_records]
```

## ETL Pipeline Implementation

### Extract and Transform

```python
import pandas as pd

class ArtifactETL:
    def __init__(self, api_client):
        self.api_client = api_client
    
    def extract_metadata(self, raw_data):
        """Transform API response to metadata DataFrame"""
        metadata_records = []
        
        for record in raw_data:
            metadata = {
                'id': record.get('id'),
                'title': record.get('title'),
                'culture': record.get('culture'),
                'century': record.get('century'),
                'classification': record.get('classification'),
                'division': record.get('division'),
                'department': record.get('department'),
                'dated': record.get('dated'),
                'medium': record.get('medium'),
                'technique': record.get('technique'),
                'period': record.get('period'),
                'provenance': record.get('provenance'),
                'description': record.get('description'),
                'accessionyear': record.get('accessionyear'),
                'verificationlevel': record.get('verificationlevel'),
                'totalpageviews': record.get('totalpageviews'),
                'totaluniquepageviews': record.get('totaluniquepageviews')
            }
            metadata_records.append(metadata)
        
        return pd.DataFrame(metadata_records)
    
    def extract_media(self, raw_data):
        """Extract media/image information"""
        media_records = []
        
        for record in raw_data:
            artifact_id = record.get('id')
            images = record.get('images', [])
            
            media = {
                'artifact_id': artifact_id,
                'baseimageurl': record.get('baseimageurl'),
                'iiifbaseuri': record.get('iiifbaseuri'),
                'primaryimageurl': record.get('primaryimageurl'),
                'imagecount': len(images)
            }
            media_records.append(media)
        
        return pd.DataFrame(media_records)
    
    def extract_colors(self, raw_data):
        """Extract color information"""
        color_records = []
        
        for record in raw_data:
            artifact_id = record.get('id')
            colors = record.get('colors', [])
            
            for color in colors:
                color_data = {
                    'artifact_id': artifact_id,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                }
                color_records.append(color_data)
        
        return pd.DataFrame(color_records)
```

### Load to Database

```python
import mysql.connector
from mysql.connector import Error

class DatabaseLoader:
    def __init__(self):
        self.connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
    
    def load_metadata(self, df):
        """Batch insert metadata"""
        cursor = self.connection.cursor()
        
        insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, division, department, 
         dated, medium, technique, period, provenance, description, 
         accessionyear, verificationlevel, totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        
        records = df.to_records(index=False)
        cursor.executemany(insert_query, records.tolist())
        self.connection.commit()
        
        return cursor.rowcount
    
    def load_media(self, df):
        """Batch insert media data"""
        cursor = self.connection.cursor()
        
        insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, iiifbaseuri, primaryimageurl, imagecount)
        VALUES (%s, %s, %s, %s, %s)
        """
        
        records = df.to_records(index=False)
        cursor.executemany(insert_query, records.tolist())
        self.connection.commit()
        
        return cursor.rowcount
    
    def load_colors(self, df):
        """Batch insert color data"""
        cursor = self.connection.cursor()
        
        insert_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        
        records = df.to_records(index=False)
        cursor.executemany(insert_query, records.tolist())
        self.connection.commit()
        
        return cursor.rowcount
```

## Analytics SQL Queries

### Example Queries

```python
ANALYTICS_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century Distribution": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Most Popular Artifacts by Page Views": """
        SELECT title, culture, totalpageviews
        FROM artifactmetadata
        WHERE totalpageviews IS NOT NULL
        ORDER BY totalpageviews DESC
        LIMIT 20
    """,
    
    "Color Distribution Across Artifacts": """
        SELECT color, COUNT(DISTINCT artifact_id) as artifact_count,
               AVG(percent) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY artifact_count DESC
    """,
    
    "Artifacts with Images vs Without": """
        SELECT 
            CASE WHEN imagecount > 0 THEN 'With Images' ELSE 'No Images' END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "Department Classification Analysis": """
        SELECT department, classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND classification IS NOT NULL
        GROUP BY department, classification
        ORDER BY department, count DESC
    """
}
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Collection Analytics")
    st.markdown("---")
    
    # Sidebar configuration
    with st.sidebar:
        st.header("⚙️ Configuration")
        
        # API section
        st.subheader("📡 API Data Collection")
        num_records = st.number_input("Number of records to fetch", 
                                      min_value=100, max_value=5000, 
                                      value=500, step=100)
        
        if st.button("🔄 Fetch & Load Data"):
            with st.spinner("Fetching data from API..."):
                run_etl_pipeline(num_records)
    
    # Main content tabs
    tab1, tab2, tab3 = st.tabs(["📊 Analytics", "🗄️ Data Explorer", "📈 Visualizations"])
    
    with tab1:
        display_analytics_dashboard()
    
    with tab2:
        display_data_explorer()
    
    with tab3:
        display_visualizations()

def display_analytics_dashboard():
    st.header("SQL Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        db = DatabaseLoader()
        cursor = db.connection.cursor(dictionary=True)
        cursor.execute(ANALYTICS_QUERIES[query_name])
        results = cursor.fetchall()
        
        df = pd.DataFrame(results)
        st.dataframe(df, use_container_width=True)
        
        # Auto-generate chart
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)

def run_etl_pipeline(num_records):
    """Execute complete ETL pipeline"""
    api_client = HarvardAPIClient()
    etl = ArtifactETL(api_client)
    db_loader = DatabaseLoader()
    
    # Extract
    raw_data = api_client.fetch_all_objects(max_records=num_records)
    st.success(f"✅ Extracted {len(raw_data)} records")
    
    # Transform
    metadata_df = etl.extract_metadata(raw_data)
    media_df = etl.extract_media(raw_data)
    colors_df = etl.extract_colors(raw_data)
    st.success("✅ Transformed data into relational tables")
    
    # Load
    db_loader.load_metadata(metadata_df)
    db_loader.load_media(media_df)
    db_loader.load_colors(colors_df)
    st.success("✅ Loaded data to database")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Error Handling for API Requests

```python
def safe_api_request(url, params, max_retries=3):
    """API request with retry logic"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

### Null Handling in Transformation

```python
def safe_get(record, key, default=None):
    """Safely extract nested values"""
    value = record.get(key, default)
    return value if value not in [None, '', 'null'] else default

# Usage
metadata['culture'] = safe_get(record, 'culture', 'Unknown')
```

## Troubleshooting

**API Rate Limiting:**
```python
# Add delays between requests
import time
time.sleep(0.5)  # 500ms between requests
```

**Database Connection Issues:**
```python
# Test connection
try:
    connection.ping(reconnect=True, attempts=3, delay=5)
except Error as e:
    st.error(f"Database connection failed: {e}")
```

**Memory Issues with Large Datasets:**
```python
# Process in chunks
chunk_size = 100
for i in range(0, len(data), chunk_size):
    chunk = data[i:i+chunk_size]
    process_chunk(chunk)
```

**Missing API Key:**
```python
if not os.getenv('HARVARD_API_KEY'):
    st.error("⚠️ HARVARD_API_KEY not found in environment variables")
    st.stop()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```
