---
name: harvard-art-museums-data-engineering-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Streamlit, SQL, and Plotly
triggers:
  - build a data pipeline for museum artifacts
  - create an ETL workflow with Harvard Art Museums API
  - set up analytics dashboard for art collection data
  - integrate Harvard Art Museums API with SQL database
  - visualize museum artifact data with Streamlit
  - design data engineering pipeline for cultural heritage data
  - query and analyze art museum collections
  - build interactive analytics for artifact metadata
---

# Harvard Art Museums Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates an end-to-end data engineering and analytics application built using the Harvard Art Museums API. It showcases real-world ETL pipelines, SQL analytics, and interactive data visualization using Streamlit.

## What This Project Does

The application provides a complete data pipeline that:
- Collects artifact data from Harvard Art Museums API with pagination and rate limiting
- Performs ETL operations on nested JSON artifacts data
- Stores structured data in relational SQL databases (MySQL/TiDB Cloud)
- Executes analytical SQL queries on the data
- Visualizes results through interactive Streamlit dashboards with Plotly charts

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Core Dependencies:**
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

Get your free API key from [Harvard Art Museums API](https://harvardartmuseums.org/collections/api).

Create a `.env` file:
```bash
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

Set up MySQL or TiDB Cloud database credentials in your environment:

```python
# Database connection parameters
DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

### 3. Database Schema

Create the required tables:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    description TEXT,
    technique TEXT,
    medium TEXT,
    dimensions TEXT,
    provenance TEXT,
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    format VARCHAR(50),
    height INT,
    width INT,
    renditionnumber VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components & Usage

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(BASE_URL, params=params)
    response.raise_for_status()
    return response.json()

# Fetch multiple pages
all_artifacts = []
for page in range(1, 6):  # Fetch 5 pages
    data = fetch_artifacts(page=page)
    all_artifacts.extend(data.get('records', []))
    print(f"Fetched page {page}, Total artifacts: {len(all_artifacts)}")
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class ArtifactETL:
    def __init__(self, db_config):
        self.db_config = db_config
        self.connection = None
    
    def connect(self):
        """Establish database connection"""
        self.connection = mysql.connector.connect(**self.db_config)
        return self.connection
    
    def extract_metadata(self, artifact: Dict) -> Dict:
        """Extract metadata from artifact JSON"""
        return {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'period': artifact.get('period', '')[:255],
            'century': artifact.get('century', '')[:255],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'description': artifact.get('description', ''),
            'technique': artifact.get('technique', ''),
            'medium': artifact.get('medium', ''),
            'dimensions': artifact.get('dimensions', ''),
            'provenance': artifact.get('provenance', ''),
            'url': artifact.get('url', '')[:500]
        }
    
    def extract_media(self, artifact: Dict) -> List[Dict]:
        """Extract media information"""
        media_list = []
        images = artifact.get('images', [])
        
        for img in images:
            media_list.append({
                'artifact_id': artifact.get('id'),
                'baseimageurl': img.get('baseimageurl', '')[:500],
                'format': img.get('format', '')[:50],
                'height': img.get('height'),
                'width': img.get('width'),
                'renditionnumber': img.get('renditionnumber', '')[:50]
            })
        
        return media_list
    
    def extract_colors(self, artifact: Dict) -> List[Dict]:
        """Extract color information"""
        color_list = []
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_list.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'percentage': color.get('percent', 0.0)
            })
        
        return color_list
    
    def load_metadata(self, metadata_list: List[Dict]):
        """Batch insert metadata into database"""
        cursor = self.connection.cursor()
        
        query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, 
         dated, description, technique, medium, dimensions, provenance, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
        """
        
        values = [tuple(m.values()) for m in metadata_list]
        cursor.executemany(query, values)
        self.connection.commit()
        cursor.close()
    
    def load_media(self, media_list: List[Dict]):
        """Batch insert media into database"""
        if not media_list:
            return
        
        cursor = self.connection.cursor()
        
        query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, format, height, width, renditionnumber)
        VALUES (%s, %s, %s, %s, %s, %s)
        """
        
        values = [tuple(m.values()) for m in media_list]
        cursor.executemany(query, values)
        self.connection.commit()
        cursor.close()
    
    def load_colors(self, color_list: List[Dict]):
        """Batch insert colors into database"""
        if not color_list:
            return
        
        cursor = self.connection.cursor()
        
        query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, percentage)
        VALUES (%s, %s, %s, %s)
        """
        
        values = [tuple(c.values()) for c in color_list]
        cursor.executemany(query, values)
        self.connection.commit()
        cursor.close()
    
    def process_artifacts(self, artifacts: List[Dict]):
        """Complete ETL process for artifacts"""
        metadata_list = []
        media_list = []
        color_list = []
        
        # Extract
        for artifact in artifacts:
            metadata_list.append(self.extract_metadata(artifact))
            media_list.extend(self.extract_media(artifact))
            color_list.extend(self.extract_colors(artifact))
        
        # Load
        self.load_metadata(metadata_list)
        self.load_media(media_list)
        self.load_colors(color_list)
        
        print(f"Loaded {len(metadata_list)} artifacts, "
              f"{len(media_list)} media items, "
              f"{len(color_list)} color records")
```

### 3. Streamlit Application Structure

```python
import streamlit as st
import plotly.express as px

# Main app structure
def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Collection Analytics")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    elif page == "Visualizations":
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection")
    
    num_pages = st.number_input("Number of pages to fetch", 
                                 min_value=1, max_value=10, value=3)
    
    if st.button("Fetch Data"):
        with st.spinner("Fetching artifacts..."):
            etl = ArtifactETL(DB_CONFIG)
            etl.connect()
            
            all_artifacts = []
            for page in range(1, num_pages + 1):
                data = fetch_artifacts(page=page)
                all_artifacts.extend(data.get('records', []))
            
            etl.process_artifacts(all_artifacts)
            st.success(f"Successfully loaded {len(all_artifacts)} artifacts!")

def show_sql_analytics():
    st.header("📊 SQL Analytics")
    
    # Predefined analytical queries
    queries = {
        "Top 10 Cultures": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 10
        """,
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century
            ORDER BY count DESC
        """,
        "Department Distribution": """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL AND department != ''
            GROUP BY department
            ORDER BY count DESC
        """,
        "Top Color Spectrums": """
            SELECT spectrum, COUNT(*) as count, AVG(percentage) as avg_percentage
            FROM artifactcolors
            GROUP BY spectrum
            ORDER BY count DESC
            LIMIT 10
        """,
        "Media Format Distribution": """
            SELECT format, COUNT(*) as count
            FROM artifactmedia
            WHERE format IS NOT NULL
            GROUP BY format
            ORDER BY count DESC
        """
    }
    
    selected_query = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        connection = mysql.connector.connect(**DB_CONFIG)
        df = pd.read_sql(queries[selected_query], connection)
        connection.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

def show_visualizations():
    st.header("📈 Interactive Visualizations")
    
    connection = mysql.connector.connect(**DB_CONFIG)
    
    # Color analysis
    color_query = """
        SELECT color, SUM(percentage) as total_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY total_percentage DESC
        LIMIT 15
    """
    df_colors = pd.read_sql(color_query, connection)
    
    fig = px.pie(df_colors, values='total_percentage', names='color',
                 title='Color Distribution Across Artifacts')
    st.plotly_chart(fig, use_container_width=True)
    
    connection.close()

if __name__ == "__main__":
    main()
```

### 4. Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Rate-Limited API Requests

```python
import time

def fetch_with_rate_limit(pages, delay=1):
    """Fetch data with rate limiting"""
    artifacts = []
    for page in range(1, pages + 1):
        data = fetch_artifacts(page=page)
        artifacts.extend(data.get('records', []))
        time.sleep(delay)  # Respect API rate limits
    return artifacts
```

### Error Handling in ETL

```python
def safe_etl_process(artifacts):
    """ETL with error handling"""
    etl = ArtifactETL(DB_CONFIG)
    
    try:
        etl.connect()
        etl.process_artifacts(artifacts)
    except mysql.connector.Error as e:
        print(f"Database error: {e}")
        if etl.connection:
            etl.connection.rollback()
    except Exception as e:
        print(f"Error during ETL: {e}")
    finally:
        if etl.connection:
            etl.connection.close()
```

### Data Validation

```python
def validate_artifact(artifact: Dict) -> bool:
    """Validate artifact data before processing"""
    required_fields = ['id', 'title']
    return all(field in artifact for field in required_fields)

# Use in ETL
valid_artifacts = [a for a in artifacts if validate_artifact(a)]
```

## Troubleshooting

### API Key Issues
```python
# Verify API key is loaded
if not os.getenv('HARVARD_API_KEY'):
    raise ValueError("HARVARD_API_KEY not found in environment variables")
```

### Database Connection Errors
```python
# Test database connection
try:
    connection = mysql.connector.connect(**DB_CONFIG)
    print("Database connection successful")
    connection.close()
except mysql.connector.Error as e:
    print(f"Database connection failed: {e}")
```

### Duplicate Key Errors
Use `ON DUPLICATE KEY UPDATE` in INSERT statements to handle re-runs:
```sql
INSERT INTO artifactmetadata (...) VALUES (...)
ON DUPLICATE KEY UPDATE title=VALUES(title)
```

### Memory Issues with Large Datasets
```python
# Process in smaller batches
BATCH_SIZE = 100
for i in range(0, len(artifacts), BATCH_SIZE):
    batch = artifacts[i:i+BATCH_SIZE]
    etl.process_artifacts(batch)
```

### Streamlit Caching for Performance
```python
@st.cache_data
def load_analytics_data():
    """Cache SQL query results"""
    connection = mysql.connector.connect(**DB_CONFIG)
    df = pd.read_sql("SELECT * FROM artifactmetadata", connection)
    connection.close()
    return df
```
