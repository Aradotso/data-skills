---
name: harvard-artifacts-etl-streamlit-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API data with Streamlit, MySQL, and Plotly visualizations
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums API
  - set up a Streamlit analytics dashboard with museum data
  - create a data engineering project with Harvard artifacts
  - extract and visualize Harvard Art Museums collection data
  - build SQL analytics for museum artifact metadata
  - integrate Harvard Art Museums API with MySQL database
  - develop an end-to-end data pipeline with Streamlit
  - analyze art museum data with Python and SQL
---

# Harvard Artifacts ETL & Analytics App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill helps you build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards with Plotly visualizations.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App provides a complete data pipeline:

1. **Data Collection**: Fetches artifact data from Harvard Art Museums API with pagination
2. **ETL Processing**: Transforms nested JSON into relational database tables
3. **SQL Storage**: Stores data in MySQL/TiDB with proper schema design
4. **Analytics**: Executes predefined SQL queries for insights
5. **Visualization**: Creates interactive dashboards using Streamlit and Plotly

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

### Prerequisites

- Python 3.8+
- MySQL or TiDB Cloud account
- Harvard Art Museums API key (get from https://www.harvardartmuseums.org/collections/api)

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

### Requirements

```txt
streamlit>=1.28.0
pandas>=2.0.0
requests>=2.31.0
mysql-connector-python>=8.1.0
plotly>=5.17.0
python-dotenv>=1.0.0
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Project Structure

```
harvard-artifacts-app/
├── app.py                 # Main Streamlit application
├── etl_pipeline.py        # ETL logic for data extraction and transformation
├── database.py            # Database connection and operations
├── queries.py             # Predefined SQL analytics queries
├── config.py              # Configuration management
├── requirements.txt       # Python dependencies
└── README.md
```

## Core Components

### 1. API Integration

```python
import requests
import os

class HarvardAPIClient:
    def __init__(self, api_key=None):
        self.api_key = api_key or os.getenv('HARVARD_API_KEY')
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
    
    def fetch_all_artifacts(self, max_pages=10):
        """Fetch multiple pages of artifacts"""
        all_artifacts = []
        for page in range(1, max_pages + 1):
            data = self.fetch_artifacts(page=page)
            all_artifacts.extend(data.get('records', []))
            if page >= data.get('info', {}).get('pages', 0):
                break
        return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
from typing import List, Dict

class ArtifactETL:
    def __init__(self, api_client):
        self.api_client = api_client
    
    def extract(self, max_pages=5):
        """Extract data from API"""
        return self.api_client.fetch_all_artifacts(max_pages=max_pages)
    
    def transform(self, raw_data: List[Dict]):
        """Transform nested JSON into relational tables"""
        # Artifact Metadata
        metadata_records = []
        media_records = []
        color_records = []
        
        for artifact in raw_data:
            artifact_id = artifact.get('id')
            
            # Metadata table
            metadata_records.append({
                'artifact_id': artifact_id,
                'title': artifact.get('title'),
                'culture': artifact.get('culture'),
                'period': artifact.get('period'),
                'century': artifact.get('century'),
                'classification': artifact.get('classification'),
                'department': artifact.get('department'),
                'dated': artifact.get('dated'),
                'medium': artifact.get('medium'),
                'dimensions': artifact.get('dimensions'),
                'creditline': artifact.get('creditline')
            })
            
            # Media table
            for media in artifact.get('images', []):
                media_records.append({
                    'artifact_id': artifact_id,
                    'image_id': media.get('imageid'),
                    'base_url': media.get('baseimageurl'),
                    'format': media.get('format'),
                    'width': media.get('width'),
                    'height': media.get('height')
                })
            
            # Color table
            for color in artifact.get('colors', []):
                color_records.append({
                    'artifact_id': artifact_id,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
        
        return {
            'metadata': pd.DataFrame(metadata_records),
            'media': pd.DataFrame(media_records),
            'colors': pd.DataFrame(color_records)
        }
    
    def load(self, dataframes: Dict[str, pd.DataFrame], db_connection):
        """Load data into SQL database"""
        cursor = db_connection.cursor()
        
        # Load metadata
        for _, row in dataframes['metadata'].iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (artifact_id, title, culture, period, century, classification, 
                 department, dated, medium, dimensions, creditline)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """, tuple(row))
        
        # Load media
        for _, row in dataframes['media'].iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (artifact_id, image_id, base_url, format, width, height)
                VALUES (%s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE base_url=VALUES(base_url)
            """, tuple(row))
        
        # Load colors
        for _, row in dataframes['colors'].iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
        
        db_connection.commit()
```

### 3. Database Schema

```python
import mysql.connector
import os

class DatabaseManager:
    def __init__(self):
        self.connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
    
    def create_schema(self):
        """Create database tables"""
        cursor = self.connection.cursor()
        
        # Metadata table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmetadata (
                artifact_id INT PRIMARY KEY,
                title VARCHAR(500),
                culture VARCHAR(255),
                period VARCHAR(255),
                century VARCHAR(100),
                classification VARCHAR(255),
                department VARCHAR(255),
                dated VARCHAR(255),
                medium TEXT,
                dimensions VARCHAR(500),
                creditline TEXT,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        
        # Media table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmedia (
                id INT AUTO_INCREMENT PRIMARY KEY,
                artifact_id INT,
                image_id INT,
                base_url VARCHAR(500),
                format VARCHAR(50),
                width INT,
                height INT,
                FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
            )
        """)
        
        # Colors table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactcolors (
                id INT AUTO_INCREMENT PRIMARY KEY,
                artifact_id INT,
                color VARCHAR(100),
                spectrum VARCHAR(100),
                hue VARCHAR(100),
                percent DECIMAL(5,2),
                FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
            )
        """)
        
        self.connection.commit()
```

### 4. SQL Analytics Queries

```python
ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "artifacts_by_department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "media_availability": """
        SELECT 
            CASE WHEN EXISTS (SELECT 1 FROM artifactmedia WHERE artifact_id = m.artifact_id) 
                 THEN 'Has Media' 
                 ELSE 'No Media' 
            END as media_status,
            COUNT(*) as count
        FROM artifactmetadata m
        GROUP BY media_status
    """,
    
    "top_colors": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "artifacts_with_dimensions": """
        SELECT 
            classification,
            COUNT(*) as total,
            SUM(CASE WHEN dimensions IS NOT NULL THEN 1 ELSE 0 END) as with_dimensions
        FROM artifactmetadata
        GROUP BY classification
        HAVING total > 10
        ORDER BY total DESC
    """
}

def execute_query(db_connection, query_name):
    """Execute a predefined analytics query"""
    cursor = db_connection.cursor(dictionary=True)
    cursor.execute(ANALYTICS_QUERIES[query_name])
    results = cursor.fetchall()
    return pd.DataFrame(results)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for configuration
    with st.sidebar:
        st.header("Configuration")
        api_key = st.text_input("Harvard API Key", type="password", value=os.getenv('HARVARD_API_KEY', ''))
        max_pages = st.slider("Pages to Fetch", 1, 20, 5)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Running ETL..."):
                api_client = HarvardAPIClient(api_key)
                etl = ArtifactETL(api_client)
                db_manager = DatabaseManager()
                
                # Extract
                raw_data = etl.extract(max_pages=max_pages)
                st.success(f"Extracted {len(raw_data)} artifacts")
                
                # Transform
                dataframes = etl.transform(raw_data)
                st.success("Data transformed")
                
                # Load
                etl.load(dataframes, db_manager.connection)
                st.success("Data loaded to database")
    
    # Analytics section
    st.header("📊 Analytics Dashboard")
    
    query_options = list(ANALYTICS_QUERIES.keys())
    selected_query = st.selectbox("Select Analysis", query_options)
    
    if st.button("Run Query"):
        db_manager = DatabaseManager()
        df = execute_query(db_manager.connection, selected_query)
        
        # Display results
        st.subheader("Query Results")
        st.dataframe(df, use_container_width=True)
        
        # Visualize
        if len(df) > 0 and len(df.columns) >= 2:
            st.subheader("Visualization")
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query.replace('_', ' ').title())
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Batch Processing with Error Handling

```python
def batch_insert(cursor, table, records, batch_size=100):
    """Insert records in batches with error handling"""
    for i in range(0, len(records), batch_size):
        batch = records[i:i + batch_size]
        try:
            cursor.executemany(f"INSERT INTO {table} VALUES (%s, %s, %s)", batch)
        except mysql.connector.Error as err:
            st.warning(f"Batch {i//batch_size} failed: {err}")
            continue
```

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(api_client, max_pages, delay=0.5):
    """Fetch data with rate limiting"""
    artifacts = []
    for page in range(1, max_pages + 1):
        artifacts.extend(api_client.fetch_artifacts(page=page))
        time.sleep(delay)  # Respect API rate limits
    return artifacts
```

## Troubleshooting

### API Key Issues
- Ensure `HARVARD_API_KEY` environment variable is set
- Verify key at https://www.harvardartmuseums.org/collections/api
- Check for rate limit errors (429 status code)

### Database Connection Errors
- Verify `DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_NAME` are correct
- Ensure MySQL/TiDB service is running
- Check firewall rules for database port (default 3306)

### Memory Issues with Large Datasets
- Reduce `max_pages` parameter
- Implement streaming writes instead of loading all data into memory
- Use batch processing with smaller batch sizes

### Streamlit Performance
- Cache database connections with `@st.cache_resource`
- Cache query results with `@st.cache_data`
- Limit visualization data points

```python
@st.cache_resource
def get_db_connection():
    return DatabaseManager()

@st.cache_data(ttl=600)
def fetch_analytics(_db_conn, query_name):
    return execute_query(_db_conn, query_name)
```
