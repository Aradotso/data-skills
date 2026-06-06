---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build ETL pipeline for Harvard Art Museums API
  - create analytics dashboard with Streamlit and SQL
  - extract and transform Harvard museum artifact data
  - visualize museum collection data with Plotly
  - set up data engineering pipeline for art museum API
  - query and analyze Harvard artifacts database
  - implement museum data ETL with Python and MySQL
  - create interactive art collection analytics app
---

# Harvard Artifacts ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipeline construction, SQL database design, analytical queries, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection application:
- Extracts artifact data from Harvard Art Museums API with pagination handling
- Transforms nested JSON into normalized relational tables
- Loads data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries on artifact metadata, media, and color data
- Visualizes insights through interactive Streamlit dashboards with Plotly charts

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests plotly mysql-connector-python python-dotenv
```

### Project Setup

```bash
# Clone repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install requirements
pip install -r requirements.txt

# Create .env file for credentials
touch .env
```

### Environment Configuration

Create a `.env` file with:

```plaintext
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

Get Harvard API key from: https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dOd0Z8MpY2rvQ/viewform

## Key Components

### 1. ETL Pipeline Structure

```python
import requests
import pandas as pd
import mysql.connector
from typing import List, Dict
import os
from dotenv import load_dotenv

load_dotenv()

class HarvardArtifactsETL:
    def __init__(self):
        self.api_key = os.getenv('HARVARD_API_KEY')
        self.base_url = "https://api.harvardartmuseums.org/object"
        self.db_config = {
            'host': os.getenv('DB_HOST'),
            'user': os.getenv('DB_USER'),
            'password': os.getenv('DB_PASSWORD'),
            'database': os.getenv('DB_NAME')
        }
    
    def extract_artifacts(self, page: int = 1, size: int = 100) -> Dict:
        """Extract artifacts from Harvard API with pagination"""
        params = {
            'apikey': self.api_key,
            'page': page,
            'size': size,
            'hasimage': 1  # Only artifacts with images
        }
        response = requests.get(self.base_url, params=params)
        response.raise_for_status()
        return response.json()
    
    def transform_metadata(self, records: List[Dict]) -> pd.DataFrame:
        """Transform artifact records into metadata DataFrame"""
        metadata = []
        for record in records:
            metadata.append({
                'objectid': record.get('id'),
                'title': record.get('title'),
                'culture': record.get('culture'),
                'century': record.get('century'),
                'classification': record.get('classification'),
                'department': record.get('department'),
                'dated': record.get('dated'),
                'period': record.get('period'),
                'technique': record.get('technique'),
                'medium': record.get('medium')
            })
        return pd.DataFrame(metadata)
    
    def transform_media(self, records: List[Dict]) -> pd.DataFrame:
        """Transform media/image data"""
        media_records = []
        for record in records:
            objectid = record.get('id')
            images = record.get('images', [])
            for img in images:
                media_records.append({
                    'objectid': objectid,
                    'baseimageurl': img.get('baseimageurl'),
                    'iiifbaseuri': img.get('iiifbaseuri'),
                    'imageid': img.get('imageid'),
                    'format': img.get('format'),
                    'width': img.get('width'),
                    'height': img.get('height')
                })
        return pd.DataFrame(media_records)
    
    def transform_colors(self, records: List[Dict]) -> pd.DataFrame:
        """Transform color data from artifacts"""
        color_records = []
        for record in records:
            objectid = record.get('id')
            colors = record.get('colors', [])
            for color in colors:
                color_records.append({
                    'objectid': objectid,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
        return pd.DataFrame(color_records)
    
    def load_to_database(self, df: pd.DataFrame, table_name: str):
        """Load DataFrame to MySQL table"""
        conn = mysql.connector.connect(**self.db_config)
        cursor = conn.cursor()
        
        # Create insert query
        columns = ', '.join(df.columns)
        placeholders = ', '.join(['%s'] * len(df.columns))
        insert_query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
        
        # Batch insert
        data = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data)
        
        conn.commit()
        cursor.close()
        conn.close()
        
        return len(data)
```

### 2. Database Schema Setup

```python
def create_database_schema():
    """Create normalized database schema"""
    
    create_metadata_table = """
    CREATE TABLE IF NOT EXISTS artifactmetadata (
        objectid INT PRIMARY KEY,
        title VARCHAR(500),
        culture VARCHAR(200),
        century VARCHAR(100),
        classification VARCHAR(200),
        department VARCHAR(200),
        dated VARCHAR(200),
        period VARCHAR(200),
        technique VARCHAR(500),
        medium VARCHAR(500),
        INDEX idx_culture (culture),
        INDEX idx_century (century),
        INDEX idx_department (department)
    );
    """
    
    create_media_table = """
    CREATE TABLE IF NOT EXISTS artifactmedia (
        id INT AUTO_INCREMENT PRIMARY KEY,
        objectid INT,
        baseimageurl VARCHAR(500),
        iiifbaseuri VARCHAR(500),
        imageid INT,
        format VARCHAR(50),
        width INT,
        height INT,
        FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid),
        INDEX idx_objectid (objectid)
    );
    """
    
    create_colors_table = """
    CREATE TABLE IF NOT EXISTS artifactcolors (
        id INT AUTO_INCREMENT PRIMARY KEY,
        objectid INT,
        color VARCHAR(50),
        spectrum VARCHAR(50),
        hue VARCHAR(50),
        percent FLOAT,
        FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid),
        INDEX idx_objectid (objectid),
        INDEX idx_color (color)
    );
    """
    
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    cursor.execute(create_metadata_table)
    cursor.execute(create_media_table)
    cursor.execute(create_colors_table)
    
    conn.commit()
    cursor.close()
    conn.close()
```

### 3. Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px
import plotly.graph_objects as go

def run_analytics_dashboard():
    """Main Streamlit analytics dashboard"""
    
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    st.title("🎨 Harvard Art Museums Collection Analytics")
    
    # Sidebar for ETL operations
    with st.sidebar:
        st.header("ETL Pipeline")
        
        if st.button("Extract Data from API"):
            with st.spinner("Extracting artifacts..."):
                etl = HarvardArtifactsETL()
                data = etl.extract_artifacts(page=1, size=100)
                st.success(f"Extracted {len(data['records'])} records")
        
        if st.button("Transform & Load to Database"):
            with st.spinner("Processing..."):
                etl = HarvardArtifactsETL()
                data = etl.extract_artifacts(page=1, size=100)
                records = data['records']
                
                # Transform
                metadata_df = etl.transform_metadata(records)
                media_df = etl.transform_media(records)
                colors_df = etl.transform_colors(records)
                
                # Load
                etl.load_to_database(metadata_df, 'artifactmetadata')
                etl.load_to_database(media_df, 'artifactmedia')
                etl.load_to_database(colors_df, 'artifactcolors')
                
                st.success("Data loaded successfully!")
    
    # Analytics Section
    st.header("📊 Analytics Queries")
    
    query_options = {
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
        "Department-wise Collection": """
            SELECT department, COUNT(*) as total_artifacts 
            FROM artifactmetadata 
            GROUP BY department 
            ORDER BY total_artifacts DESC
        """,
        "Most Common Colors in Collection": """
            SELECT color, COUNT(*) as frequency, AVG(percent) as avg_coverage
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY frequency DESC 
            LIMIT 15
        """,
        "Image Format Distribution": """
            SELECT format, COUNT(*) as count 
            FROM artifactmedia 
            WHERE format IS NOT NULL 
            GROUP BY format
        """,
        "Artifacts with Multiple Images": """
            SELECT m.objectid, m.title, COUNT(am.id) as image_count
            FROM artifactmetadata m
            JOIN artifactmedia am ON m.objectid = am.objectid
            GROUP BY m.objectid, m.title
            HAVING image_count > 1
            ORDER BY image_count DESC
            LIMIT 20
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        conn = mysql.connector.connect(**db_config)
        df = pd.read_sql(query_options[selected_query], conn)
        conn.close()
        
        # Display results
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    run_analytics_dashboard()
```

### 4. Running Complete ETL Pipeline

```python
def full_etl_pipeline(num_pages: int = 5):
    """Execute complete ETL pipeline for multiple pages"""
    
    etl = HarvardArtifactsETL()
    
    # Create schema
    create_database_schema()
    
    all_metadata = []
    all_media = []
    all_colors = []
    
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}...")
        
        # Extract
        data = etl.extract_artifacts(page=page, size=100)
        records = data.get('records', [])
        
        if not records:
            break
        
        # Transform
        metadata_df = etl.transform_metadata(records)
        media_df = etl.transform_media(records)
        colors_df = etl.transform_colors(records)
        
        all_metadata.append(metadata_df)
        all_media.append(media_df)
        all_colors.append(colors_df)
    
    # Combine all pages
    final_metadata = pd.concat(all_metadata, ignore_index=True)
    final_media = pd.concat(all_media, ignore_index=True)
    final_colors = pd.concat(all_colors, ignore_index=True)
    
    # Load to database
    etl.load_to_database(final_metadata, 'artifactmetadata')
    etl.load_to_database(final_media, 'artifactmedia')
    etl.load_to_database(final_colors, 'artifactcolors')
    
    print(f"ETL Complete: {len(final_metadata)} artifacts loaded")
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def extract_with_rate_limit(etl, pages: int, delay: float = 1.0):
    """Extract data with rate limiting"""
    for page in range(1, pages + 1):
        data = etl.extract_artifacts(page=page)
        # Process data...
        time.sleep(delay)  # Respect API rate limits
```

### Error Handling in ETL

```python
def safe_etl_execution():
    """ETL with error handling"""
    try:
        etl = HarvardArtifactsETL()
        data = etl.extract_artifacts()
        
        if data.get('records'):
            metadata_df = etl.transform_metadata(data['records'])
            etl.load_to_database(metadata_df, 'artifactmetadata')
        else:
            print("No records found")
            
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
    except mysql.connector.Error as e:
        print(f"Database Error: {e}")
    except Exception as e:
        print(f"Unexpected Error: {e}")
```

### Custom Analytics Query

```python
def run_custom_analytics(query: str) -> pd.DataFrame:
    """Execute custom SQL analytics query"""
    conn = mysql.connector.connect(**db_config)
    try:
        df = pd.read_sql(query, conn)
        return df
    finally:
        conn.close()

# Example usage
query = """
SELECT 
    m.culture,
    COUNT(DISTINCT m.objectid) as artifacts,
    COUNT(DISTINCT c.color) as unique_colors,
    AVG(am.width) as avg_image_width
FROM artifactmetadata m
LEFT JOIN artifactcolors c ON m.objectid = c.objectid
LEFT JOIN artifactmedia am ON m.objectid = am.objectid
WHERE m.culture IS NOT NULL
GROUP BY m.culture
HAVING artifacts > 5
ORDER BY artifacts DESC
"""

results = run_custom_analytics(query)
```

## Troubleshooting

### API Key Issues

If you get authentication errors:
```python
# Verify API key is loaded
import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv('HARVARD_API_KEY')
print(f"API Key loaded: {api_key is not None}")
```

### Database Connection Errors

```python
# Test database connection
def test_db_connection():
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        print("✓ Database connection successful")
    except mysql.connector.Error as e:
        print(f"✗ Database error: {e}")

test_db_connection()
```

### Empty Results from API

```python
# Check API response structure
data = etl.extract_artifacts(page=1, size=10)
print(f"Total records: {data.get('info', {}).get('totalrecords')}")
print(f"Records returned: {len(data.get('records', []))}")
```

### Streamlit Not Starting

```bash
# Ensure Streamlit is installed
pip install --upgrade streamlit

# Run with verbose output
streamlit run app.py --logger.level=debug
```

This skill provides comprehensive knowledge for building production-ready ETL pipelines and analytics dashboards using the Harvard Art Museums API.
