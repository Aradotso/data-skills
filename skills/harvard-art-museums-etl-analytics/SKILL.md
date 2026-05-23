---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics application for Harvard Art Museums API with Streamlit dashboards
triggers:
  - build an ETL pipeline for Harvard Art Museums API
  - create analytics dashboard for museum artifact data
  - set up Harvard artifacts data engineering pipeline
  - query and visualize Harvard Art Museums collection
  - extract and transform museum API data
  - build Streamlit app for art collection analytics
  - connect to Harvard Art Museums API with Python
  - design SQL schema for museum artifacts
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build and work with end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL operations, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection application:
- Extracts artifact data from Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into normalized relational tables
- Loads data into MySQL/TiDB Cloud databases
- Provides 20+ predefined analytical SQL queries
- Visualizes results through interactive Plotly charts in Streamlit

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

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

### Dependencies
```python
# requirements.txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from typing import List, Dict

class HarvardAPIClient:
    """Client for Harvard Art Museums API"""
    
    def __init__(self, api_key: str = None):
        self.api_key = api_key or os.getenv('HARVARD_API_KEY')
        self.base_url = "https://api.harvardartmuseums.org"
        
    def fetch_artifacts(self, size: int = 100, page: int = 1) -> Dict:
        """Fetch artifacts with pagination"""
        endpoint = f"{self.base_url}/object"
        params = {
            'apikey': self.api_key,
            'size': size,
            'page': page
        }
        
        response = requests.get(endpoint, params=params)
        response.raise_for_status()
        return response.json()
    
    def fetch_all_artifacts(self, max_records: int = 1000) -> List[Dict]:
        """Fetch multiple pages of artifacts"""
        artifacts = []
        page = 1
        size = 100
        
        while len(artifacts) < max_records:
            data = self.fetch_artifacts(size=size, page=page)
            records = data.get('records', [])
            
            if not records:
                break
                
            artifacts.extend(records)
            page += 1
            
            # Rate limiting
            import time
            time.sleep(0.5)
        
        return artifacts[:max_records]

# Usage
client = HarvardAPIClient()
artifacts = client.fetch_all_artifacts(max_records=500)
```

### 2. ETL Pipeline

```python
import pandas as pd
from typing import Tuple

class ArtifactETL:
    """ETL processor for Harvard artifacts data"""
    
    def __init__(self, raw_data: List[Dict]):
        self.raw_data = raw_data
    
    def transform(self) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
        """Transform raw JSON into normalized dataframes"""
        
        # Metadata table
        metadata_records = []
        for artifact in self.raw_data:
            metadata_records.append({
                'objectid': artifact.get('objectid'),
                'title': artifact.get('title'),
                'culture': artifact.get('culture'),
                'period': artifact.get('period'),
                'century': artifact.get('century'),
                'classification': artifact.get('classification'),
                'medium': artifact.get('medium'),
                'department': artifact.get('department'),
                'division': artifact.get('division'),
                'dated': artifact.get('dated'),
                'url': artifact.get('url')
            })
        
        metadata_df = pd.DataFrame(metadata_records)
        
        # Media table
        media_records = []
        for artifact in self.raw_data:
            object_id = artifact.get('objectid')
            images = artifact.get('images', [])
            
            for img in images:
                media_records.append({
                    'objectid': object_id,
                    'imageid': img.get('imageid'),
                    'baseimageurl': img.get('baseimageurl'),
                    'iiifbaseuri': img.get('iiifbaseuri'),
                    'format': img.get('format'),
                    'width': img.get('width'),
                    'height': img.get('height')
                })
        
        media_df = pd.DataFrame(media_records) if media_records else pd.DataFrame()
        
        # Colors table
        color_records = []
        for artifact in self.raw_data:
            object_id = artifact.get('objectid')
            colors = artifact.get('colors', [])
            
            for color in colors:
                color_records.append({
                    'objectid': object_id,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
        
        colors_df = pd.DataFrame(color_records) if color_records else pd.DataFrame()
        
        return metadata_df, media_df, colors_df

# Usage
etl = ArtifactETL(artifacts)
metadata, media, colors = etl.transform()
```

### 3. Database Schema & Loading

```python
import mysql.connector
from contextlib import contextmanager

class DatabaseManager:
    """Manager for MySQL database operations"""
    
    def __init__(self):
        self.config = {
            'host': os.getenv('DB_HOST'),
            'user': os.getenv('DB_USER'),
            'password': os.getenv('DB_PASSWORD'),
            'database': os.getenv('DB_NAME')
        }
    
    @contextmanager
    def get_connection(self):
        """Context manager for database connections"""
        conn = mysql.connector.connect(**self.config)
        try:
            yield conn
        finally:
            conn.close()
    
    def create_schema(self):
        """Create database schema"""
        with self.get_connection() as conn:
            cursor = conn.cursor()
            
            # Metadata table
            cursor.execute("""
                CREATE TABLE IF NOT EXISTS artifactmetadata (
                    objectid INT PRIMARY KEY,
                    title TEXT,
                    culture VARCHAR(255),
                    period VARCHAR(255),
                    century VARCHAR(100),
                    classification VARCHAR(255),
                    medium TEXT,
                    department VARCHAR(255),
                    division VARCHAR(255),
                    dated VARCHAR(255),
                    url TEXT
                )
            """)
            
            # Media table
            cursor.execute("""
                CREATE TABLE IF NOT EXISTS artifactmedia (
                    id INT AUTO_INCREMENT PRIMARY KEY,
                    objectid INT,
                    imageid INT,
                    baseimageurl TEXT,
                    iiifbaseuri TEXT,
                    format VARCHAR(50),
                    width INT,
                    height INT,
                    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
                )
            """)
            
            # Colors table
            cursor.execute("""
                CREATE TABLE IF NOT EXISTS artifactcolors (
                    id INT AUTO_INCREMENT PRIMARY KEY,
                    objectid INT,
                    color VARCHAR(50),
                    spectrum VARCHAR(50),
                    hue VARCHAR(50),
                    percent FLOAT,
                    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
                )
            """)
            
            conn.commit()
    
    def load_data(self, metadata_df: pd.DataFrame, 
                  media_df: pd.DataFrame, 
                  colors_df: pd.DataFrame):
        """Batch load dataframes into database"""
        with self.get_connection() as conn:
            cursor = conn.cursor()
            
            # Load metadata
            for _, row in metadata_df.iterrows():
                cursor.execute("""
                    INSERT INTO artifactmetadata 
                    (objectid, title, culture, period, century, classification, 
                     medium, department, division, dated, url)
                    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                    ON DUPLICATE KEY UPDATE title=VALUES(title)
                """, tuple(row))
            
            # Load media
            if not media_df.empty:
                for _, row in media_df.iterrows():
                    cursor.execute("""
                        INSERT INTO artifactmedia 
                        (objectid, imageid, baseimageurl, iiifbaseuri, format, width, height)
                        VALUES (%s, %s, %s, %s, %s, %s, %s)
                    """, tuple(row))
            
            # Load colors
            if not colors_df.empty:
                for _, row in colors_df.iterrows():
                    cursor.execute("""
                        INSERT INTO artifactcolors 
                        (objectid, color, spectrum, hue, percent)
                        VALUES (%s, %s, %s, %s, %s)
                    """, tuple(row))
            
            conn.commit()

# Usage
db = DatabaseManager()
db.create_schema()
db.load_data(metadata, media, colors)
```

### 4. Analytics Queries

```python
class AnalyticsQueries:
    """Predefined analytical queries"""
    
    QUERIES = {
        "artifacts_by_culture": """
            SELECT culture, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE culture IS NOT NULL
            GROUP BY culture 
            ORDER BY count DESC 
            LIMIT 10
        """,
        
        "artifacts_by_century": """
            SELECT century, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE century IS NOT NULL
            GROUP BY century 
            ORDER BY count DESC
        """,
        
        "artifacts_with_images": """
            SELECT 
                COUNT(DISTINCT m.objectid) as artifacts_with_images,
                COUNT(DISTINCT a.objectid) as total_artifacts,
                ROUND(COUNT(DISTINCT m.objectid) * 100.0 / COUNT(DISTINCT a.objectid), 2) as percentage
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.objectid = m.objectid
        """,
        
        "top_departments": """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        
        "color_distribution": """
            SELECT spectrum, COUNT(*) as count
            FROM artifactcolors
            WHERE spectrum IS NOT NULL
            GROUP BY spectrum
            ORDER BY count DESC
        """,
        
        "artifacts_by_classification": """
            SELECT classification, COUNT(*) as count
            FROM artifactmetadata
            WHERE classification IS NOT NULL
            GROUP BY classification
            ORDER BY count DESC
            LIMIT 15
        """
    }
    
    @staticmethod
    def execute_query(db_manager: DatabaseManager, query_name: str) -> pd.DataFrame:
        """Execute a named query and return results"""
        query = AnalyticsQueries.QUERIES.get(query_name)
        
        if not query:
            raise ValueError(f"Query '{query_name}' not found")
        
        with db_manager.get_connection() as conn:
            return pd.read_sql(query, conn)

# Usage
results = AnalyticsQueries.execute_query(db, "artifacts_by_culture")
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Initialize components
    db = DatabaseManager()
    
    # Sidebar
    st.sidebar.header("Analytics Options")
    query_choice = st.sidebar.selectbox(
        "Select Analysis",
        list(AnalyticsQueries.QUERIES.keys())
    )
    
    # Execute query
    if st.sidebar.button("Run Analysis"):
        with st.spinner("Executing query..."):
            results = AnalyticsQueries.execute_query(db, query_choice)
            
            # Display results
            st.subheader(f"Results: {query_choice.replace('_', ' ').title()}")
            st.dataframe(results, use_container_width=True)
            
            # Visualization
            if len(results.columns) >= 2:
                fig = px.bar(
                    results,
                    x=results.columns[0],
                    y=results.columns[1],
                    title=f"{query_choice.replace('_', ' ').title()}",
                    labels={results.columns[0]: results.columns[0].title(),
                            results.columns[1]: results.columns[1].title()}
                )
                st.plotly_chart(fig, use_container_width=True)
    
    # ETL Section
    st.sidebar.markdown("---")
    st.sidebar.header("Data Collection")
    
    num_records = st.sidebar.number_input("Number of records", 100, 5000, 500)
    
    if st.sidebar.button("Collect & Load Data"):
        with st.spinner("Fetching data from API..."):
            client = HarvardAPIClient()
            artifacts = client.fetch_all_artifacts(max_records=num_records)
            
            etl = ArtifactETL(artifacts)
            metadata, media, colors = etl.transform()
            
            db.load_data(metadata, media, colors)
            
            st.success(f"✅ Loaded {len(metadata)} artifacts successfully!")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete Pipeline Execution

```python
# Full ETL workflow
def run_complete_pipeline(num_artifacts: int = 1000):
    # Step 1: Extract
    client = HarvardAPIClient()
    raw_data = client.fetch_all_artifacts(max_records=num_artifacts)
    
    # Step 2: Transform
    etl = ArtifactETL(raw_data)
    metadata, media, colors = etl.transform()
    
    # Step 3: Load
    db = DatabaseManager()
    db.create_schema()
    db.load_data(metadata, media, colors)
    
    # Step 4: Analyze
    results = AnalyticsQueries.execute_query(db, "artifacts_by_culture")
    
    return results
```

### Custom Query Execution

```python
def run_custom_query(sql: str) -> pd.DataFrame:
    """Execute custom SQL query"""
    db = DatabaseManager()
    
    with db.get_connection() as conn:
        return pd.read_sql(sql, conn)

# Example
custom_results = run_custom_query("""
    SELECT a.century, COUNT(*) as count, AVG(c.percent) as avg_color_percent
    FROM artifactmetadata a
    JOIN artifactcolors c ON a.objectid = c.objectid
    WHERE a.century IS NOT NULL
    GROUP BY a.century
    ORDER BY count DESC
""")
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff
import time
from requests.exceptions import HTTPError

def fetch_with_retry(client, max_retries=3):
    for attempt in range(max_retries):
        try:
            return client.fetch_artifacts()
        except HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
```

### Database Connection Issues
```python
# Test connection
def test_database_connection():
    try:
        db = DatabaseManager()
        with db.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute("SELECT 1")
            print("✅ Database connection successful")
    except Exception as e:
        print(f"❌ Connection failed: {e}")
```

### Empty API Responses
```python
# Validate data before processing
def validate_artifacts(artifacts: List[Dict]) -> bool:
    if not artifacts:
        raise ValueError("No artifacts returned from API")
    
    required_fields = ['objectid', 'title']
    for artifact in artifacts:
        if not all(field in artifact for field in required_fields):
            raise ValueError("Missing required fields in artifact data")
    
    return True
```

## Configuration

Set environment variables in `.env`:
```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

Load in Python:
```python
from dotenv import load_dotenv
load_dotenv()
```
