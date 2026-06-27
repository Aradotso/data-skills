---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit, MySQL, and Plotly
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums API
  - create a data engineering project with Harvard artifacts
  - set up analytics dashboard for museum artifact data
  - integrate Harvard Art Museums API with SQL database
  - build Streamlit app for artifact data visualization
  - extract and analyze Harvard museum collection data
  - create SQL analytics queries for art museum data
  - develop end-to-end data pipeline for museum artifacts
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application using the Harvard Art Museums API. It demonstrates ETL pipelines, SQL analytics, and interactive visualization with Streamlit, simulating real-world data engineering workflows.

## What It Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON into relational SQL tables
- **SQL Analytics**: Executes predefined analytical queries on artifact metadata, media, and color data
- **Visualization**: Creates interactive dashboards using Streamlit and Plotly
- **Database Design**: Implements normalized schema with proper foreign key relationships

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
DB_HOST=your_db_host
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
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    creditline TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
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
        response.raise_for_status()
        return response.json()
    
    def fetch_all_artifacts(self, max_records=1000):
        """Fetch multiple pages of artifacts"""
        all_artifacts = []
        page = 1
        size = 100
        
        while len(all_artifacts) < max_records:
            data = self.fetch_artifacts(page=page, size=size)
            artifacts = data.get('records', [])
            
            if not artifacts:
                break
            
            all_artifacts.extend(artifacts)
            page += 1
        
        return all_artifacts[:max_records]
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class ArtifactETL:
    def __init__(self, db_config):
        self.db_config = db_config
        self.conn = None
    
    def connect_db(self):
        """Establish database connection"""
        self.conn = mysql.connector.connect(
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
                'title': artifact.get('title'),
                'culture': artifact.get('culture'),
                'century': artifact.get('century'),
                'classification': artifact.get('classification'),
                'department': artifact.get('department'),
                'division': artifact.get('division'),
                'dated': artifact.get('dated'),
                'period': artifact.get('period'),
                'creditline': artifact.get('creditline')
            })
        
        return pd.DataFrame(metadata)
    
    def extract_media(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract media information"""
        media_data = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            media_data.append({
                'artifact_id': artifact_id,
                'baseimageurl': artifact.get('baseimageurl'),
                'primaryimageurl': artifact.get('primaryimageurl'),
                'iiifbaseuri': artifact.get('iiifbaseuri')
            })
        
        return pd.DataFrame(media_data)
    
    def extract_colors(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract color information"""
        color_data = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            colors = artifact.get('colors', [])
            
            for color_info in colors:
                color_data.append({
                    'artifact_id': artifact_id,
                    'color': color_info.get('color'),
                    'spectrum': color_info.get('spectrum'),
                    'hue': color_info.get('hue'),
                    'percent': color_info.get('percent')
                })
        
        return pd.DataFrame(color_data)
    
    def load_to_db(self, df: pd.DataFrame, table_name: str):
        """Load dataframe to database"""
        cursor = self.conn.cursor()
        
        # Clear existing data
        cursor.execute(f"DELETE FROM {table_name}")
        
        # Prepare insert statement
        cols = ','.join(df.columns)
        placeholders = ','.join(['%s'] * len(df.columns))
        insert_sql = f"INSERT INTO {table_name} ({cols}) VALUES ({placeholders})"
        
        # Batch insert
        values = [tuple(row) for row in df.values]
        cursor.executemany(insert_sql, values)
        
        self.conn.commit()
        cursor.close()
    
    def run_pipeline(self, artifacts: List[Dict]):
        """Execute full ETL pipeline"""
        self.connect_db()
        
        # Extract
        metadata_df = self.extract_metadata(artifacts)
        media_df = self.extract_media(artifacts)
        colors_df = self.extract_colors(artifacts)
        
        # Load
        self.load_to_db(metadata_df, 'artifactmetadata')
        self.load_to_db(media_df, 'artifactmedia')
        self.load_to_db(colors_df, 'artifactcolors')
        
        self.conn.close()
```

### 3. SQL Analytics Queries

```python
class ArtifactAnalytics:
    
    QUERIES = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE culture IS NOT NULL 
            GROUP BY culture 
            ORDER BY count DESC 
            LIMIT 10
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE century IS NOT NULL 
            GROUP BY century 
            ORDER BY count DESC
        """,
        
        'artifacts_by_department': """
            SELECT department, COUNT(*) as count 
            FROM artifactmetadata 
            GROUP BY department 
            ORDER BY count DESC
        """,
        
        'media_availability': """
            SELECT 
                SUM(CASE WHEN primaryimageurl IS NOT NULL THEN 1 ELSE 0 END) as with_image,
                SUM(CASE WHEN primaryimageurl IS NULL THEN 1 ELSE 0 END) as without_image
            FROM artifactmedia
        """,
        
        'top_colors': """
            SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
            FROM artifactcolors
            WHERE color IS NOT NULL
            GROUP BY color
            ORDER BY count DESC
            LIMIT 10
        """,
        
        'artifacts_with_colors': """
            SELECT a.title, a.culture, c.color, c.percent
            FROM artifactmetadata a
            JOIN artifactcolors c ON a.id = c.artifact_id
            WHERE c.percent > 50
            ORDER BY c.percent DESC
            LIMIT 20
        """
    }
    
    def __init__(self, db_config):
        self.db_config = db_config
    
    def execute_query(self, query_name: str) -> pd.DataFrame:
        """Execute a predefined query"""
        conn = mysql.connector.connect(**self.db_config)
        query = self.QUERIES.get(query_name)
        
        if not query:
            raise ValueError(f"Query '{query_name}' not found")
        
        df = pd.read_sql(query, conn)
        conn.close()
        return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database config from env
    db_config = {
        'host': os.getenv('DB_HOST'),
        'port': int(os.getenv('DB_PORT', 3306)),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    # ETL Section
    st.header("📥 Data Collection & ETL")
    
    if st.button("Fetch & Load Artifacts"):
        with st.spinner("Fetching data from API..."):
            client = HarvardAPIClient()
            artifacts = client.fetch_all_artifacts(max_records=1000)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Running ETL pipeline..."):
            etl = ArtifactETL(db_config)
            etl.run_pipeline(artifacts)
            st.success("ETL pipeline completed!")
    
    # Analytics Section
    st.header("📊 Analytics")
    
    analytics = ArtifactAnalytics(db_config)
    
    query_options = list(analytics.QUERIES.keys())
    selected_query = st.selectbox("Select Analysis", query_options)
    
    if st.button("Run Analysis"):
        df = analytics.execute_query(selected_query)
        
        # Display table
        st.subheader("Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query.replace('_', ' ').title())
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

## Common Patterns

### Custom Query Execution

```python
def run_custom_query(query: str, db_config: dict) -> pd.DataFrame:
    """Execute custom SQL query"""
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Example usage
custom_sql = """
    SELECT a.culture, COUNT(*) as total, 
           AVG(c.percent) as avg_color_dominance
    FROM artifactmetadata a
    LEFT JOIN artifactcolors c ON a.id = c.artifact_id
    GROUP BY a.culture
    HAVING total > 10
    ORDER BY avg_color_dominance DESC
"""

results = run_custom_query(custom_sql, db_config)
```

### Incremental Data Loading

```python
def incremental_load(artifacts: List[Dict], db_config: dict):
    """Load only new artifacts"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    # Get existing IDs
    cursor.execute("SELECT id FROM artifactmetadata")
    existing_ids = set(row[0] for row in cursor.fetchall())
    
    # Filter new artifacts
    new_artifacts = [a for a in artifacts if a['id'] not in existing_ids]
    
    # Load only new ones
    if new_artifacts:
        etl = ArtifactETL(db_config)
        etl.run_pipeline(new_artifacts)
    
    conn.close()
    return len(new_artifacts)
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.HTTPError as e:
            if response.status_code == 429:  # Rate limited
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_db_connection(db_config):
    """Verify database connectivity"""
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        conn.close()
        return result[0] == 1
    except Exception as e:
        print(f"Connection failed: {e}")
        return False
```

### Handling Missing Data

```python
def safe_extract(artifact: dict, key: str, default=None):
    """Safely extract nested values"""
    value = artifact.get(key, default)
    return value if value not in [None, '', 'null'] else default

# Usage in extraction
metadata = {
    'id': artifact.get('id'),
    'title': safe_extract(artifact, 'title', 'Untitled'),
    'culture': safe_extract(artifact, 'culture', 'Unknown')
}
```
