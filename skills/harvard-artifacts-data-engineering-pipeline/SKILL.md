---
name: harvard-artifacts-data-engineering-pipeline
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API using Python, SQL, and Streamlit
triggers:
  - how do I use the Harvard Art Museums API for data engineering
  - build an ETL pipeline for museum artifact data
  - create analytics dashboard with Harvard museum data
  - setup data pipeline with Harvard Art Museums API
  - extract and analyze Harvard artifacts collection data
  - build Streamlit app for museum data visualization
  - query Harvard Art Museums API and store in SQL
  - create data engineering project with museum API
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytics queries, and interactive Streamlit dashboards.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App:
- Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- Performs ETL operations to transform nested JSON into relational tables
- Stores structured data in SQL databases (MySQL/TiDB Cloud)
- Executes analytical SQL queries for insights
- Visualizes results using Streamlit and Plotly

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    url TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact media table
CREATE TABLE IF NOT EXISTS artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    primaryimageurl TEXT,
    imagepermissionlevel INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE IF NOT EXISTS artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Integration

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

class HarvardAPIClient:
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
        
        if response.status_code == 200:
            return response.json()
        else:
            raise Exception(f"API Error: {response.status_code}")
    
    def fetch_all_artifacts(self, max_pages=10):
        """Fetch multiple pages of artifacts"""
        all_artifacts = []
        
        for page in range(1, max_pages + 1):
            data = self.fetch_artifacts(page=page)
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            # Check if more pages exist
            if page >= data.get('info', {}).get('pages', 0):
                break
        
        return all_artifacts
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
        self.connection = mysql.connector.connect(
            host=self.db_config['host'],
            port=self.db_config['port'],
            user=self.db_config['user'],
            password=self.db_config['password'],
            database=self.db_config['database']
        )
    
    def extract_metadata(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract artifact metadata"""
        metadata = []
        
        for artifact in artifacts:
            metadata.append({
                'id': artifact.get('id'),
                'title': artifact.get('title', '')[:500],
                'culture': artifact.get('culture', '')[:200],
                'period': artifact.get('period', '')[:200],
                'century': artifact.get('century', '')[:100],
                'classification': artifact.get('classification', '')[:200],
                'department': artifact.get('department', '')[:200],
                'division': artifact.get('division', '')[:200],
                'dated': artifact.get('dated', '')[:200],
                'url': artifact.get('url', '')
            })
        
        return pd.DataFrame(metadata)
    
    def extract_media(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract media information"""
        media = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            media.append({
                'artifact_id': artifact_id,
                'baseimageurl': artifact.get('baseimageurl', ''),
                'primaryimageurl': artifact.get('primaryimageurl', ''),
                'imagepermissionlevel': artifact.get('imagepermissionlevel', 0)
            })
        
        return pd.DataFrame(media)
    
    def extract_colors(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract color information"""
        colors = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            color_list = artifact.get('colors', [])
            
            for color in color_list:
                colors.append({
                    'artifact_id': artifact_id,
                    'color': color.get('color', ''),
                    'spectrum': color.get('spectrum', ''),
                    'hue': color.get('hue', ''),
                    'percent': color.get('percent', 0.0)
                })
        
        return pd.DataFrame(colors) if colors else pd.DataFrame()
    
    def load_data(self, df: pd.DataFrame, table_name: str):
        """Load data into SQL table"""
        cursor = self.connection.cursor()
        
        # Generate INSERT statement
        columns = ', '.join(df.columns)
        placeholders = ', '.join(['%s'] * len(df.columns))
        
        insert_query = f"""
            INSERT INTO {table_name} ({columns}) 
            VALUES ({placeholders})
            ON DUPLICATE KEY UPDATE {', '.join([f'{col}=VALUES({col})' for col in df.columns if col != 'id'])}
        """
        
        # Batch insert
        data_tuples = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data_tuples)
        self.connection.commit()
        cursor.close()
    
    def run_etl(self, artifacts: List[Dict]):
        """Execute full ETL pipeline"""
        self.connect_db()
        
        # Extract
        metadata_df = self.extract_metadata(artifacts)
        media_df = self.extract_media(artifacts)
        colors_df = self.extract_colors(artifacts)
        
        # Load
        if not metadata_df.empty:
            self.load_data(metadata_df, 'artifactmetadata')
        
        if not media_df.empty:
            self.load_data(media_df, 'artifactmedia')
        
        if not colors_df.empty:
            self.load_data(colors_df, 'artifactcolors')
        
        self.connection.close()
```

### 3. Analytics Queries

```python
class ArtifactAnalytics:
    def __init__(self, db_config):
        self.db_config = db_config
    
    def execute_query(self, query: str) -> pd.DataFrame:
        """Execute SQL query and return DataFrame"""
        connection = mysql.connector.connect(**self.db_config)
        df = pd.read_sql(query, connection)
        connection.close()
        return df
    
    # Sample analytical queries
    QUERIES = {
        "artifacts_by_culture": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 20
        """,
        
        "artifacts_by_century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century
            ORDER BY count DESC
        """,
        
        "artifacts_with_images": """
            SELECT 
                CASE 
                    WHEN primaryimageurl IS NOT NULL AND primaryimageurl != '' THEN 'With Image'
                    ELSE 'Without Image'
                END as image_status,
                COUNT(*) as count
            FROM artifactmedia
            GROUP BY image_status
        """,
        
        "top_departments": """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL AND department != ''
            GROUP BY department
            ORDER BY count DESC
            LIMIT 10
        """,
        
        "color_distribution": """
            SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
            FROM artifactcolors
            WHERE color IS NOT NULL
            GROUP BY color
            ORDER BY count DESC
            LIMIT 15
        """
    }
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px
from dotenv import load_dotenv

load_dotenv()

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    db_config = {
        'host': os.getenv('DB_HOST'),
        'port': int(os.getenv('DB_PORT', 3306)),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    # Initialize analytics
    analytics = ArtifactAnalytics(db_config)
    
    # Data Collection Tab
    tab1, tab2 = st.tabs(["📊 Analytics", "🔄 Data Collection"])
    
    with tab1:
        st.header("Analytics Dashboard")
        
        # Query selector
        query_name = st.selectbox(
            "Select Analysis",
            options=list(analytics.QUERIES.keys())
        )
        
        if st.button("Run Analysis"):
            with st.spinner("Executing query..."):
                df = analytics.execute_query(analytics.QUERIES[query_name])
                
                st.subheader("Results")
                st.dataframe(df)
                
                # Auto-visualization
                if len(df.columns) >= 2:
                    fig = px.bar(
                        df, 
                        x=df.columns[0], 
                        y=df.columns[1],
                        title=f"{query_name.replace('_', ' ').title()}"
                    )
                    st.plotly_chart(fig, use_container_width=True)
    
    with tab2:
        st.header("Data Collection")
        
        num_pages = st.slider("Number of pages to fetch", 1, 50, 5)
        
        if st.button("Fetch Artifacts"):
            with st.spinner("Fetching data from API..."):
                client = HarvardAPIClient()
                artifacts = client.fetch_all_artifacts(max_pages=num_pages)
                
                st.success(f"Fetched {len(artifacts)} artifacts")
                
                # Run ETL
                etl = ArtifactETL(db_config)
                etl.run_etl(artifacts)
                
                st.success("Data loaded successfully!")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(client, max_pages, delay=1):
    """Fetch artifacts with rate limiting"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        artifacts = client.fetch_artifacts(page=page)
        all_artifacts.extend(artifacts.get('records', []))
        
        # Rate limit: wait between requests
        time.sleep(delay)
    
    return all_artifacts
```

### Error Handling in ETL

```python
def safe_etl(artifacts, db_config):
    """ETL with error handling"""
    try:
        etl = ArtifactETL(db_config)
        etl.run_etl(artifacts)
        return True, "Success"
    except mysql.connector.Error as e:
        return False, f"Database error: {str(e)}"
    except Exception as e:
        return False, f"ETL error: {str(e)}"
```

## Troubleshooting

### API Key Issues
- Ensure `HARVARD_API_KEY` is set in `.env`
- Get API key from: https://www.harvardartmuseums.org/collections/api

### Database Connection Errors
- Verify database credentials in `.env`
- Check if database exists and tables are created
- Ensure MySQL server is running

### Streamlit Not Loading
```bash
# Clear Streamlit cache
streamlit cache clear

# Run with verbose logging
streamlit run app.py --logger.level=debug
```

### Empty Results
- Check if data exists in tables: `SELECT COUNT(*) FROM artifactmetadata;`
- Verify ETL completed successfully
- Re-run data collection if needed
