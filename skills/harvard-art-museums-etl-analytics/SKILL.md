---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build ETL pipeline for Harvard Art Museums data
  - create data engineering project with museum API
  - set up Harvard artifacts analytics dashboard
  - implement SQL analytics for art collection data
  - visualize Harvard museum data with Streamlit
  - extract and analyze art museum metadata
  - build end-to-end data pipeline for cultural artifacts
  - create interactive dashboard for museum collections
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build and work with end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL operations, SQL database design, analytical queries, and interactive Streamlit dashboards for cultural artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App:
- Extracts artifact data from the Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into normalized relational database tables
- Loads data into MySQL/TiDB Cloud with optimized batch inserts
- Provides 20+ predefined SQL analytical queries
- Visualizes results through interactive Plotly charts in Streamlit
- Simulates real-world data engineering workflows

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages (add to requirements.txt if missing)
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### API Key Setup

1. Get a free API key from [Harvard Art Museums API](https://harvardartmuseums.org/collections/api)
2. Create a `.env` file or configure in Streamlit:

```python
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
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
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    technique VARCHAR(255),
    provenance TEXT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    imagepermissionlevel INT,
    totalmedia INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
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

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Core Components

### 1. ETL Pipeline Implementation

```python
import requests
import pandas as pd
import mysql.connector
from typing import Dict, List
import time

class HarvardArtETL:
    """ETL pipeline for Harvard Art Museums API"""
    
    def __init__(self, api_key: str, db_config: Dict):
        self.api_key = api_key
        self.base_url = "https://api.harvardartmuseums.org/object"
        self.db_config = db_config
        
    def extract(self, num_records: int = 100, page_size: int = 100) -> List[Dict]:
        """Extract artifact data from API with pagination"""
        artifacts = []
        pages = (num_records + page_size - 1) // page_size
        
        for page in range(1, pages + 1):
            params = {
                'apikey': self.api_key,
                'size': page_size,
                'page': page
            }
            
            try:
                response = requests.get(self.base_url, params=params, timeout=10)
                response.raise_for_status()
                data = response.json()
                artifacts.extend(data.get('records', []))
                
                # Rate limiting
                time.sleep(0.5)
                
            except requests.exceptions.RequestException as e:
                print(f"Error fetching page {page}: {e}")
                break
                
        return artifacts[:num_records]
    
    def transform(self, artifacts: List[Dict]) -> tuple:
        """Transform raw API data into normalized tables"""
        metadata_records = []
        media_records = []
        color_records = []
        
        for artifact in artifacts:
            # Extract metadata
            metadata = {
                'id': artifact.get('id'),
                'title': artifact.get('title', '')[:500],
                'culture': artifact.get('culture', '')[:255],
                'period': artifact.get('period', '')[:255],
                'century': artifact.get('century', '')[:255],
                'classification': artifact.get('classification', '')[:255],
                'department': artifact.get('department', '')[:255],
                'dated': artifact.get('dated', '')[:255],
                'medium': artifact.get('medium', '')[:500],
                'dimensions': artifact.get('dimensions', '')[:500],
                'creditline': artifact.get('creditline', ''),
                'accessionyear': artifact.get('accessionyear'),
                'technique': artifact.get('technique', '')[:255],
                'provenance': artifact.get('provenance', '')
            }
            metadata_records.append(metadata)
            
            # Extract media info
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl', '')[:500],
                'primaryimageurl': artifact.get('primaryimageurl', '')[:500],
                'imagepermissionlevel': artifact.get('imagepermissionlevel'),
                'totalmedia': artifact.get('totalpageviews', 0)
            }
            media_records.append(media)
            
            # Extract color data
            colors = artifact.get('colors', [])
            for color in colors:
                color_record = {
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color', '')[:50],
                    'spectrum': color.get('spectrum', '')[:50],
                    'hue': color.get('hue', '')[:50],
                    'percent': color.get('percent', 0.0)
                }
                color_records.append(color_record)
        
        return (
            pd.DataFrame(metadata_records),
            pd.DataFrame(media_records),
            pd.DataFrame(color_records)
        )
    
    def load(self, metadata_df: pd.DataFrame, media_df: pd.DataFrame, 
             colors_df: pd.DataFrame) -> bool:
        """Load transformed data into MySQL database"""
        try:
            conn = mysql.connector.connect(**self.db_config)
            cursor = conn.cursor()
            
            # Insert metadata
            metadata_sql = """
                INSERT INTO artifactmetadata 
                (id, title, culture, period, century, classification, department, 
                 dated, medium, dimensions, creditline, accessionyear, technique, provenance)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """
            cursor.executemany(metadata_sql, metadata_df.values.tolist())
            
            # Insert media
            media_sql = """
                INSERT INTO artifactmedia 
                (artifact_id, baseimageurl, primaryimageurl, imagepermissionlevel, totalmedia)
                VALUES (%s, %s, %s, %s, %s)
            """
            cursor.executemany(media_sql, media_df.values.tolist())
            
            # Insert colors
            if not colors_df.empty:
                colors_sql = """
                    INSERT INTO artifactcolors 
                    (artifact_id, color, spectrum, hue, percent)
                    VALUES (%s, %s, %s, %s, %s)
                """
                cursor.executemany(colors_sql, colors_df.values.tolist())
            
            conn.commit()
            cursor.close()
            conn.close()
            return True
            
        except mysql.connector.Error as e:
            print(f"Database error: {e}")
            return False
```

### 2. Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px
import os
from dotenv import load_dotenv

load_dotenv()

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar configuration
    with st.sidebar:
        st.header("⚙️ Configuration")
        
        api_key = st.text_input(
            "Harvard API Key",
            value=os.getenv('HARVARD_API_KEY', ''),
            type="password"
        )
        
        st.subheader("Database Config")
        db_config = {
            'host': st.text_input("Host", os.getenv('DB_HOST', 'localhost')),
            'user': st.text_input("User", os.getenv('DB_USER', 'root')),
            'password': st.text_input("Password", os.getenv('DB_PASSWORD', ''), type="password"),
            'database': st.text_input("Database", os.getenv('DB_NAME', 'harvard_artifacts'))
        }
    
    # Main tabs
    tab1, tab2, tab3 = st.tabs(["📥 ETL Pipeline", "📊 Analytics", "📈 Visualizations"])
    
    with tab1:
        st.header("Extract, Transform, Load")
        
        num_records = st.slider("Number of records to fetch", 10, 500, 100)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Running ETL..."):
                etl = HarvardArtETL(api_key, db_config)
                
                # Extract
                st.info("Extracting data from API...")
                artifacts = etl.extract(num_records)
                st.success(f"Extracted {len(artifacts)} artifacts")
                
                # Transform
                st.info("Transforming data...")
                metadata_df, media_df, colors_df = etl.transform(artifacts)
                st.success("Transformation complete")
                
                # Load
                st.info("Loading into database...")
                success = etl.load(metadata_df, media_df, colors_df)
                
                if success:
                    st.success("✅ ETL Pipeline completed successfully!")
                    
                    col1, col2, col3 = st.columns(3)
                    col1.metric("Metadata Records", len(metadata_df))
                    col2.metric("Media Records", len(media_df))
                    col3.metric("Color Records", len(colors_df))
                else:
                    st.error("❌ ETL Pipeline failed")
    
    with tab2:
        st.header("SQL Analytics Queries")
        
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
                WHERE century IS NOT NULL AND century != ''
                GROUP BY century
                ORDER BY count DESC
            """,
            "Department Distribution": """
                SELECT department, COUNT(*) as count
                FROM artifactmetadata
                WHERE department IS NOT NULL
                GROUP BY department
                ORDER BY count DESC
            """,
            "Media Availability": """
                SELECT 
                    CASE 
                        WHEN primaryimageurl IS NOT NULL THEN 'Has Image'
                        ELSE 'No Image'
                    END as image_status,
                    COUNT(*) as count
                FROM artifactmedia
                GROUP BY image_status
            """,
            "Top 15 Colors Used": """
                SELECT color, SUM(percent) as total_percent, COUNT(*) as frequency
                FROM artifactcolors
                WHERE color IS NOT NULL
                GROUP BY color
                ORDER BY frequency DESC
                LIMIT 15
            """,
            "Artifacts by Classification": """
                SELECT classification, COUNT(*) as count
                FROM artifactmetadata
                WHERE classification IS NOT NULL
                GROUP BY classification
                ORDER BY count DESC
                LIMIT 10
            """,
            "Accession Year Trends": """
                SELECT accessionyear, COUNT(*) as count
                FROM artifactmetadata
                WHERE accessionyear IS NOT NULL AND accessionyear > 1800
                GROUP BY accessionyear
                ORDER BY accessionyear
            """
        }
        
        selected_query = st.selectbox("Select Analysis", list(queries.keys()))
        
        if st.button("Execute Query"):
            try:
                conn = mysql.connector.connect(**db_config)
                df = pd.read_sql(queries[selected_query], conn)
                conn.close()
                
                st.dataframe(df, use_container_width=True)
                
                # Auto-visualization
                if len(df.columns) == 2:
                    fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                                title=selected_query)
                    st.plotly_chart(fig, use_container_width=True)
                    
            except Exception as e:
                st.error(f"Query error: {e}")
    
    with tab3:
        st.header("Interactive Visualizations")
        st.info("Execute queries in the Analytics tab to generate visualizations")

if __name__ == "__main__":
    main()
```

### 3. Common Analytical Queries

```python
# Query patterns for Harvard Art Museum data analysis

# Time-based analysis
time_analysis = """
SELECT 
    FLOOR(accessionyear / 10) * 10 as decade,
    COUNT(*) as artifacts,
    COUNT(DISTINCT culture) as unique_cultures
FROM artifactmetadata
WHERE accessionyear IS NOT NULL
GROUP BY decade
ORDER BY decade DESC
"""

# Color spectrum analysis
color_spectrum = """
SELECT 
    spectrum,
    AVG(percent) as avg_coverage,
    COUNT(DISTINCT artifact_id) as artifact_count
FROM artifactcolors
GROUP BY spectrum
ORDER BY artifact_count DESC
"""

# Media richness score
media_score = """
SELECT 
    m.classification,
    AVG(CASE WHEN med.primaryimageurl IS NOT NULL THEN 1 ELSE 0 END) as image_rate,
    AVG(med.totalmedia) as avg_media_count,
    COUNT(*) as total_artifacts
FROM artifactmetadata m
JOIN artifactmedia med ON m.id = med.artifact_id
GROUP BY m.classification
HAVING total_artifacts > 10
ORDER BY image_rate DESC
"""

# Cultural period distribution
cultural_periods = """
SELECT 
    culture,
    period,
    century,
    COUNT(*) as count
FROM artifactmetadata
WHERE culture IS NOT NULL 
  AND period IS NOT NULL 
  AND century IS NOT NULL
GROUP BY culture, period, century
ORDER BY count DESC
LIMIT 50
```

## Troubleshooting

### API Rate Limiting

```python
# Implement exponential backoff
import time
from requests.exceptions import HTTPError

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    return None
```

### Database Connection Issues

```python
# Add connection pooling
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    **db_config
)

def get_connection():
    return db_pool.get_connection()
```

### Memory Management for Large Datasets

```python
# Process in chunks
def load_in_chunks(df, table_name, chunk_size=1000):
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i+chunk_size]
        # Insert chunk
        conn.commit()
    
    cursor.close()
    conn.close()
```

### Handling Missing or Null Data

```python
def clean_artifact_data(artifact):
    """Clean and validate artifact data"""
    return {
        'id': artifact.get('id'),
        'title': artifact.get('title', 'Untitled')[:500],
        'culture': artifact.get('culture') or 'Unknown',
        'century': artifact.get('century') or 'Unknown',
        'accessionyear': artifact.get('accessionyear') or None,
        # Add validation for all fields
    }
```

## Best Practices

1. **Always use environment variables for credentials**
2. **Implement proper error handling in ETL pipeline**
3. **Add logging for production deployments**
4. **Use database transactions for data integrity**
5. **Cache API responses to reduce calls**
6. **Validate data before database insertion**
7. **Create database indexes for frequently queried columns**

```sql
-- Performance indexes
CREATE INDEX idx_culture ON artifactmetadata(culture);
CREATE INDEX idx_century ON artifactmetadata(century);
CREATE INDEX idx_department ON artifactmetadata(department);
CREATE INDEX idx_artifact_id ON artifactmedia(artifact_id);
CREATE INDEX idx_color_artifact ON artifactcolors(artifact_id);
```
