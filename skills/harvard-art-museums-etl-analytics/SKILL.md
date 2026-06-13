---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with Harvard artifacts API
  - extract and transform museum collection data
  - set up data pipeline with Streamlit visualization
  - analyze Harvard Art Museums collection data
  - build museum data engineering application
  - create artifact analytics with SQL and Python
  - develop ETL workflow for museum API data
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON data into relational SQL tables
- **Database Schema**: Structured tables for artifact metadata, media, and color information
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboards**: Streamlit-based visualization with Plotly charts

**Architecture Flow**: API → ETL → SQL Database → Analytics Queries → Streamlit Visualization

## Installation

### Prerequisites

```bash
# Python 3.8+
# MySQL or TiDB Cloud database
# Harvard Art Museums API key (get from https://harvardartmuseums.org/collections/api)
```

### Setup

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

### Required Dependencies

```python
# requirements.txt
streamlit>=1.20.0
pandas>=1.5.0
requests>=2.28.0
mysql-connector-python>=8.0.0
plotly>=5.13.0
python-dotenv>=0.21.0
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Core Components

### 1. API Data Collection

```python
import requests
import os
from typing import Dict, List

class HarvardAPIClient:
    """Client for Harvard Art Museums API"""
    
    def __init__(self, api_key: str = None):
        self.api_key = api_key or os.getenv('HARVARD_API_KEY')
        self.base_url = "https://api.harvardartmuseums.org"
        
    def fetch_objects(self, page: int = 1, size: int = 100) -> Dict:
        """Fetch artifacts with pagination"""
        endpoint = f"{self.base_url}/object"
        params = {
            'apikey': self.api_key,
            'page': page,
            'size': size
        }
        
        response = requests.get(endpoint, params=params)
        response.raise_for_status()
        return response.json()
    
    def fetch_all_objects(self, max_records: int = 1000) -> List[Dict]:
        """Fetch multiple pages of artifacts"""
        all_objects = []
        page = 1
        size = 100
        
        while len(all_objects) < max_records:
            data = self.fetch_objects(page=page, size=size)
            records = data.get('records', [])
            
            if not records:
                break
                
            all_objects.extend(records)
            page += 1
            
            # Rate limiting
            import time
            time.sleep(0.5)
        
        return all_objects[:max_records]
```

### 2. ETL Pipeline

```python
import pandas as pd
from typing import Tuple

class ArtifactETL:
    """ETL pipeline for artifact data"""
    
    def extract(self, api_client: HarvardAPIClient, max_records: int = 1000) -> List[Dict]:
        """Extract data from API"""
        print(f"Extracting {max_records} records from Harvard API...")
        return api_client.fetch_all_objects(max_records)
    
    def transform(self, raw_data: List[Dict]) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
        """Transform nested JSON into relational tables"""
        
        # Artifact Metadata
        metadata_records = []
        for obj in raw_data:
            metadata_records.append({
                'objectid': obj.get('objectid'),
                'title': obj.get('title'),
                'culture': obj.get('culture'),
                'period': obj.get('period'),
                'century': obj.get('century'),
                'classification': obj.get('classification'),
                'department': obj.get('department'),
                'technique': obj.get('technique'),
                'medium': obj.get('medium'),
                'dated': obj.get('dated'),
                'url': obj.get('url')
            })
        
        # Artifact Media
        media_records = []
        for obj in raw_data:
            objectid = obj.get('objectid')
            images = obj.get('images', [])
            
            if images:
                for img in images:
                    media_records.append({
                        'objectid': objectid,
                        'imageurl': img.get('baseimageurl'),
                        'iiifbaseuri': img.get('iiifbaseuri'),
                        'format': img.get('format'),
                        'width': img.get('width'),
                        'height': img.get('height')
                    })
        
        # Artifact Colors
        color_records = []
        for obj in raw_data:
            objectid = obj.get('objectid')
            colors = obj.get('colors', [])
            
            for color in colors:
                color_records.append({
                    'objectid': objectid,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
        
        return (
            pd.DataFrame(metadata_records),
            pd.DataFrame(media_records),
            pd.DataFrame(color_records)
        )
    
    def load(self, metadata_df: pd.DataFrame, media_df: pd.DataFrame, 
             colors_df: pd.DataFrame, db_connection):
        """Load data into SQL database"""
        cursor = db_connection.cursor()
        
        # Create tables
        self._create_tables(cursor)
        
        # Insert metadata
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (objectid, title, culture, period, century, classification, 
                 department, technique, medium, dated, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (objectid, imageurl, iiifbaseuri, format, width, height)
                VALUES (%s, %s, %s, %s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (objectid, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
        
        db_connection.commit()
        print("Data loaded successfully!")
    
    def _create_tables(self, cursor):
        """Create database schema"""
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmetadata (
                objectid INT PRIMARY KEY,
                title TEXT,
                culture VARCHAR(255),
                period VARCHAR(255),
                century VARCHAR(100),
                classification VARCHAR(255),
                department VARCHAR(255),
                technique TEXT,
                medium TEXT,
                dated VARCHAR(255),
                url TEXT
            )
        """)
        
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmedia (
                mediaid INT AUTO_INCREMENT PRIMARY KEY,
                objectid INT,
                imageurl TEXT,
                iiifbaseuri TEXT,
                format VARCHAR(50),
                width INT,
                height INT,
                FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
            )
        """)
        
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactcolors (
                colorid INT AUTO_INCREMENT PRIMARY KEY,
                objectid INT,
                color VARCHAR(50),
                spectrum VARCHAR(50),
                hue VARCHAR(50),
                percent FLOAT,
                FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
            )
        """)
```

### 3. Database Connection

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Create MySQL/TiDB connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            port=int(os.getenv('DB_PORT', 3306))
        )
        
        if connection.is_connected():
            print("Successfully connected to database")
            return connection
            
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None
```

### 4. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
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
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as occurrences, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY occurrences DESC
        LIMIT 15
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT m.objectid) as artifacts_with_images,
            COUNT(*) as total_images
        FROM artifactmedia m
    """,
    
    "Top Classifications": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 10
    """
}

def execute_query(connection, query_name: str) -> pd.DataFrame:
    """Execute analytical query and return DataFrame"""
    query = ANALYTICS_QUERIES.get(query_name)
    if not query:
        raise ValueError(f"Query '{query_name}' not found")
    
    return pd.read_sql(query, connection)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("End-to-end ETL Pipeline and Analytics Application")
    
    # Sidebar
    st.sidebar.header("Configuration")
    
    # Database connection
    if 'db_connection' not in st.session_state:
        st.session_state.db_connection = create_database_connection()
    
    # ETL Section
    st.header("📊 ETL Pipeline")
    
    col1, col2 = st.columns(2)
    
    with col1:
        max_records = st.number_input("Records to fetch", 100, 5000, 1000)
        
    with col2:
        if st.button("Run ETL Pipeline"):
            with st.spinner("Running ETL..."):
                api_client = HarvardAPIClient()
                etl = ArtifactETL()
                
                # Extract
                raw_data = etl.extract(api_client, max_records)
                st.success(f"Extracted {len(raw_data)} records")
                
                # Transform
                metadata, media, colors = etl.transform(raw_data)
                st.success("Data transformed")
                
                # Load
                etl.load(metadata, media, colors, st.session_state.db_connection)
                st.success("Data loaded to database")
    
    # Analytics Section
    st.header("📈 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        df = execute_query(st.session_state.db_connection, query_name)
        
        # Display table
        st.dataframe(df, use_container_width=True)
        
        # Auto visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Batch Processing with Rate Limiting

```python
def fetch_with_retry(api_client, page, max_retries=3):
    """Fetch with exponential backoff"""
    import time
    
    for attempt in range(max_retries):
        try:
            return api_client.fetch_objects(page=page)
        except requests.exceptions.RequestException as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
```

### Incremental ETL

```python
def incremental_load(connection, last_sync_date):
    """Load only new/updated records"""
    cursor = connection.cursor()
    cursor.execute("""
        SELECT MAX(objectid) FROM artifactmetadata
    """)
    max_id = cursor.fetchone()[0] or 0
    
    # Fetch only newer records from API
    # Filter by objectid > max_id
```

## Troubleshooting

### API Rate Limiting
- Add delays between requests (0.5-1 second)
- Implement exponential backoff for retries
- Cache responses locally for development

### Database Connection Issues
- Verify credentials in environment variables
- Check firewall rules for cloud databases (TiDB Cloud)
- Ensure database exists before running ETL

### Memory Issues with Large Datasets
- Process data in chunks/batches
- Use `chunksize` parameter in pandas
- Clear dataframes after loading to database

### Missing Data in Visualizations
- Check for NULL values in SQL queries
- Add `WHERE column IS NOT NULL` filters
- Validate data transformation logic

## Environment Variables

```bash
# .env file
HARVARD_API_KEY=your_api_key_from_harvardartmuseums
DB_HOST=your_db_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

Load with:

```python
from dotenv import load_dotenv
load_dotenv()
```
