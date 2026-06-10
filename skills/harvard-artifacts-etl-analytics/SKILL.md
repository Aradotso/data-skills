---
name: harvard-artifacts-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - how do I build an ETL pipeline for the Harvard Art Museums API
  - show me how to extract and transform Harvard artifacts data
  - help me set up the Harvard artifacts collection analytics app
  - create a data pipeline with Harvard Art Museums API
  - how do I use the Harvard artifacts data engineering project
  - build an analytics dashboard for museum artifacts data
  - implement ETL for Harvard Art Museums with SQL
  - set up Streamlit visualization for artifacts collection
---

# Harvard Artifacts Collection Data Engineering & Analytics App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming it into relational structures, loading into SQL databases (MySQL/TiDB Cloud), and visualizing insights through a Streamlit dashboard. It includes 20+ analytical SQL queries and interactive Plotly visualizations.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

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

### Environment Setup

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_db_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Schema

The project uses three main tables with foreign key relationships:

```sql
-- Artifact Metadata (main table)
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    objectnumber VARCHAR(100),
    dated VARCHAR(200),
    period VARCHAR(200),
    url TEXT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    primaryimageurl TEXT,
    has_images BOOLEAN,
    totalpageviews INT,
    totaluniquepageviews INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
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

## Core Components

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv("HARVARD_API_KEY")
data = fetch_artifacts(api_key, page=1, size=100)
total_records = data['info']['totalrecords']
artifacts = data['records']
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class ArtifactETL:
    def __init__(self, db_config):
        self.db_config = db_config
        self.connection = None
    
    def connect_db(self):
        """Establish database connection"""
        self.connection = mysql.connector.connect(**self.db_config)
        return self.connection
    
    def extract_metadata(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract and transform metadata"""
        metadata = []
        for artifact in artifacts:
            metadata.append({
                'id': artifact.get('id'),
                'title': artifact.get('title', '')[:500],
                'culture': artifact.get('culture', '')[:200],
                'century': artifact.get('century', '')[:100],
                'classification': artifact.get('classification', '')[:200],
                'department': artifact.get('department', '')[:200],
                'technique': artifact.get('technique', '')[:500],
                'objectnumber': artifact.get('objectnumber', '')[:100],
                'dated': artifact.get('dated', '')[:200],
                'period': artifact.get('period', '')[:200],
                'url': artifact.get('url', '')
            })
        return pd.DataFrame(metadata)
    
    def extract_media(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract media information"""
        media = []
        for artifact in artifacts:
            media.append({
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl', ''),
                'primaryimageurl': artifact.get('primaryimageurl', ''),
                'has_images': 1 if artifact.get('images') else 0,
                'totalpageviews': artifact.get('totalpageviews', 0),
                'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
            })
        return pd.DataFrame(media)
    
    def extract_colors(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract color data"""
        colors = []
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            color_list = artifact.get('colors', [])
            for color_data in color_list:
                colors.append({
                    'artifact_id': artifact_id,
                    'color': color_data.get('color', ''),
                    'spectrum': color_data.get('spectrum', ''),
                    'hue': color_data.get('hue', ''),
                    'percent': color_data.get('percent', 0.0)
                })
        return pd.DataFrame(colors) if colors else pd.DataFrame()
    
    def load_to_db(self, df: pd.DataFrame, table_name: str):
        """Batch insert data into database"""
        cursor = self.connection.cursor()
        
        # Create insert query
        cols = ', '.join(df.columns)
        placeholders = ', '.join(['%s'] * len(df.columns))
        insert_query = f"INSERT IGNORE INTO {table_name} ({cols}) VALUES ({placeholders})"
        
        # Batch insert
        data_tuples = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data_tuples)
        self.connection.commit()
        cursor.close()
        
        return cursor.rowcount

# Usage
db_config = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

etl = ArtifactETL(db_config)
etl.connect_db()

# Extract
metadata_df = etl.extract_metadata(artifacts)
media_df = etl.extract_media(artifacts)
colors_df = etl.extract_colors(artifacts)

# Load
etl.load_to_db(metadata_df, 'artifactmetadata')
etl.load_to_db(media_df, 'artifactmedia')
if not colors_df.empty:
    etl.load_to_db(colors_df, 'artifactcolors')
```

### 3. Streamlit Application

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Artifacts Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                     value=os.getenv("HARVARD_API_KEY", ""))
    
    # Data collection
    if st.sidebar.button("Collect Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(api_key, page=1, size=100)
            etl = ArtifactETL(db_config)
            etl.connect_db()
            
            # ETL process
            metadata_df = etl.extract_metadata(artifacts['records'])
            media_df = etl.extract_media(artifacts['records'])
            colors_df = etl.extract_colors(artifacts['records'])
            
            etl.load_to_db(metadata_df, 'artifactmetadata')
            etl.load_to_db(media_df, 'artifactmedia')
            if not colors_df.empty:
                etl.load_to_db(colors_df, 'artifactcolors')
            
            st.success(f"Loaded {len(metadata_df)} artifacts!")
    
    # Analytics section
    st.header("📊 SQL Analytics")
    
    queries = {
        "Top 10 Cultures by Artifact Count": """
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
            WHERE century IS NOT NULL 
            GROUP BY century 
            ORDER BY count DESC
        """,
        "Top Colors Used": """
            SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY color
            ORDER BY frequency DESC
            LIMIT 10
        """,
        "Department Distribution": """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """,
        "Image Availability": """
            SELECT has_images, COUNT(*) as count
            FROM artifactmedia
            GROUP BY has_images
        """
    }
    
    query_name = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        connection = mysql.connector.connect(**db_config)
        df_result = pd.read_sql(queries[query_name], connection)
        connection.close()
        
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) == 2:
            fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Analytical Queries

### Cultural Distribution Analysis

```sql
SELECT culture, century, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL AND century IS NOT NULL
GROUP BY culture, century
ORDER BY artifact_count DESC
LIMIT 20;
```

### Media Engagement Metrics

```sql
SELECT 
    am.department,
    AVG(ame.totalpageviews) as avg_views,
    SUM(ame.has_images) as total_with_images
FROM artifactmetadata am
JOIN artifactmedia ame ON am.id = ame.artifact_id
GROUP BY am.department
ORDER BY avg_views DESC;
```

### Color Palette Analysis

```sql
SELECT 
    ac.spectrum,
    ac.color,
    COUNT(DISTINCT ac.artifact_id) as artifact_count,
    AVG(ac.percent) as avg_coverage
FROM artifactcolors ac
GROUP BY ac.spectrum, ac.color
HAVING artifact_count > 5
ORDER BY artifact_count DESC;
```

### Cross-Table Insights

```sql
SELECT 
    am.classification,
    COUNT(DISTINCT am.id) as total_artifacts,
    AVG(ame.totalpageviews) as avg_popularity,
    COUNT(DISTINCT ac.color) as unique_colors
FROM artifactmetadata am
LEFT JOIN artifactmedia ame ON am.id = ame.artifact_id
LEFT JOIN artifactcolors ac ON am.id = ac.artifact_id
GROUP BY am.classification
HAVING total_artifacts > 10
ORDER BY avg_popularity DESC;
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            if response.status_code == 429:  # Rate limit
                time.sleep(2 ** attempt)  # Exponential backoff
                continue
            return response.json()
        except Exception as e:
            if attempt == max_retries - 1:
                raise e
            time.sleep(1)
```

### Database Connection Issues

```python
from mysql.connector import Error

try:
    connection = mysql.connector.connect(**db_config)
    if connection.is_connected():
        print("Successfully connected to database")
except Error as e:
    print(f"Error: {e}")
    # Check: firewall rules, credentials, host reachability
```

### Memory Optimization for Large Datasets

```python
def batch_process_artifacts(api_key, total_pages, batch_size=10):
    """Process artifacts in batches to manage memory"""
    etl = ArtifactETL(db_config)
    etl.connect_db()
    
    for page in range(1, total_pages + 1, batch_size):
        artifacts = fetch_artifacts(api_key, page, size=100)
        
        # Process and load immediately
        metadata_df = etl.extract_metadata(artifacts['records'])
        etl.load_to_db(metadata_df, 'artifactmetadata')
        
        # Clear memory
        del metadata_df, artifacts
```

## Best Practices

1. **API Key Security**: Always use environment variables, never hardcode
2. **Batch Processing**: Insert data in batches for performance (100-1000 rows)
3. **Error Handling**: Implement retry logic for API calls
4. **Data Validation**: Check for NULL values and string length limits before insertion
5. **Index Optimization**: Create indexes on frequently queried columns (culture, century, department)

```sql
CREATE INDEX idx_culture ON artifactmetadata(culture);
CREATE INDEX idx_century ON artifactmetadata(century);
CREATE INDEX idx_department ON artifactmetadata(department);
CREATE INDEX idx_artifact_colors ON artifactcolors(artifact_id, color);
```
