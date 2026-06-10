---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with museum artifact data
  - extract and transform Harvard Art Museums API data
  - set up SQL database for artifact collection analysis
  - visualize museum data with Streamlit and Plotly
  - implement data engineering pipeline for art collections
  - analyze Harvard artifacts with SQL queries
  - build data warehouse for museum collections
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering Analytics App is an end-to-end data pipeline that demonstrates:

- **API Integration**: Fetching artifact data from Harvard Art Museums API
- **ETL Pipeline**: Extracting, transforming, and loading nested JSON into relational tables
- **SQL Analytics**: Running analytical queries on structured museum data
- **Interactive Visualization**: Building Streamlit dashboards with Plotly charts

This project simulates real-world data engineering workflows used in analytics roles, handling pagination, rate limiting, and complex nested data structures.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Register for a free API key at: https://www.harvardartmuseums.org/collections/api

```python
# Set environment variable
export HARVARD_API_KEY="your_api_key_here"

# Or create .env file
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Setup

The project supports MySQL or TiDB Cloud. Configure connection parameters:

```python
# .env file
DB_HOST=your_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### 3. Database Schema

```sql
-- Create database
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
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
    accession_year INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
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

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from typing import List, Dict

class HarvardAPIClient:
    def __init__(self):
        self.api_key = os.getenv('HARVARD_API_KEY')
        self.base_url = "https://api.harvardartmuseums.org/object"
    
    def fetch_artifacts(self, size: int = 100, page: int = 1) -> Dict:
        """Fetch artifacts with pagination support"""
        params = {
            'apikey': self.api_key,
            'size': size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(self.base_url, params=params)
        response.raise_for_status()
        return response.json()
    
    def collect_batch(self, total_records: int = 500) -> List[Dict]:
        """Collect multiple pages of artifacts"""
        all_artifacts = []
        page = 1
        size = 100
        
        while len(all_artifacts) < total_records:
            data = self.fetch_artifacts(size=size, page=page)
            artifacts = data.get('records', [])
            
            if not artifacts:
                break
                
            all_artifacts.extend(artifacts)
            page += 1
            
            # Respect rate limits
            import time
            time.sleep(0.5)
        
        return all_artifacts[:total_records]
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class ArtifactETL:
    def __init__(self, db_config: Dict):
        self.db_config = db_config
        self.conn = None
    
    def connect(self):
        """Establish database connection"""
        self.conn = mysql.connector.connect(**self.db_config)
        return self.conn.cursor()
    
    def transform_metadata(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform artifact metadata into DataFrame"""
        metadata = []
        
        for artifact in artifacts:
            metadata.append({
                'id': artifact.get('id'),
                'title': artifact.get('title'),
                'culture': artifact.get('culture'),
                'period': artifact.get('period'),
                'century': artifact.get('century'),
                'classification': artifact.get('classification'),
                'department': artifact.get('department'),
                'division': artifact.get('division'),
                'dated': artifact.get('dated'),
                'url': artifact.get('url'),
                'accession_year': artifact.get('accessionyear')
            })
        
        return pd.DataFrame(metadata)
    
    def transform_media(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract and flatten media information"""
        media_records = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            images = artifact.get('images', [])
            
            for image in images:
                media_records.append({
                    'artifact_id': artifact_id,
                    'baseimageurl': image.get('baseimageurl'),
                    'format': image.get('format'),
                    'height': image.get('height'),
                    'width': image.get('width')
                })
        
        return pd.DataFrame(media_records)
    
    def transform_colors(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract color analysis data"""
        color_records = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            colors = artifact.get('colors', [])
            
            for color in colors:
                color_records.append({
                    'artifact_id': artifact_id,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
        
        return pd.DataFrame(color_records)
    
    def load_to_sql(self, df: pd.DataFrame, table_name: str):
        """Batch insert DataFrame into SQL table"""
        cursor = self.connect()
        
        # Clear existing data
        cursor.execute(f"DELETE FROM {table_name}")
        
        # Prepare insert statement
        columns = ', '.join(df.columns)
        placeholders = ', '.join(['%s'] * len(df.columns))
        insert_sql = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
        
        # Batch insert
        data_tuples = [tuple(row) for row in df.values]
        cursor.executemany(insert_sql, data_tuples)
        
        self.conn.commit()
        cursor.close()
        
        return len(df)
```

### 3. SQL Analytics Queries

```python
# Example analytical queries

ANALYTICS_QUERIES = {
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
    
    "image_availability": """
        SELECT 
            CASE WHEN EXISTS (
                SELECT 1 FROM artifactmedia WHERE artifact_id = am.id
            ) THEN 'Has Images' ELSE 'No Images' END as status,
            COUNT(*) as count
        FROM artifactmetadata am
        GROUP BY status
    """,
    
    "color_distribution": """
        SELECT spectrum, COUNT(*) as count
        FROM artifactcolors
        GROUP BY spectrum
        ORDER BY count DESC
    """,
    
    "artifacts_per_department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "accession_trends": """
        SELECT accession_year, COUNT(*) as count
        FROM artifactmetadata
        WHERE accession_year IS NOT NULL
        GROUP BY accession_year
        ORDER BY accession_year
    """,
    
    "top_colors": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """
}

def run_query(cursor, query_name: str) -> pd.DataFrame:
    """Execute analytical query and return results"""
    cursor.execute(ANALYTICS_QUERIES[query_name])
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    return pd.DataFrame(results, columns=columns)
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # API Key input
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    # Database connection
    db_config = {
        'host': st.sidebar.text_input("DB Host", os.getenv('DB_HOST', 'localhost')),
        'user': st.sidebar.text_input("DB User", os.getenv('DB_USER', 'root')),
        'password': st.sidebar.text_input("DB Password", type="password"),
        'database': st.sidebar.text_input("Database", os.getenv('DB_NAME', 'harvard_artifacts'))
    }
    
    # Data collection
    if st.sidebar.button("Fetch New Data"):
        with st.spinner("Collecting artifacts..."):
            client = HarvardAPIClient()
            artifacts = client.collect_batch(total_records=500)
            
            etl = ArtifactETL(db_config)
            
            # Transform and load
            metadata_df = etl.transform_metadata(artifacts)
            media_df = etl.transform_media(artifacts)
            colors_df = etl.transform_colors(artifacts)
            
            etl.load_to_sql(metadata_df, 'artifactmetadata')
            etl.load_to_sql(media_df, 'artifactmedia')
            etl.load_to_sql(colors_df, 'artifactcolors')
            
            st.success(f"Loaded {len(metadata_df)} artifacts!")
    
    # Analytics section
    st.header("SQL Analytics")
    
    query_choice = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        
        df = run_query(cursor, query_choice)
        
        # Display results
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(
                df, 
                x=df.columns[0], 
                y=df.columns[1],
                title=query_choice.replace('_', ' ').title()
            )
            st.plotly_chart(fig)
        
        cursor.close()
        conn.close()

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

### Pattern 1: Incremental Data Loading

```python
def incremental_load(etl: ArtifactETL, cursor, new_artifacts: List[Dict]):
    """Load only new artifacts not in database"""
    cursor.execute("SELECT id FROM artifactmetadata")
    existing_ids = set(row[0] for row in cursor.fetchall())
    
    new_records = [a for a in new_artifacts if a['id'] not in existing_ids]
    
    if new_records:
        df = etl.transform_metadata(new_records)
        etl.load_to_sql(df, 'artifactmetadata')
    
    return len(new_records)
```

### Pattern 2: Error Handling for API Calls

```python
def safe_fetch(client: HarvardAPIClient, max_retries: int = 3):
    """Fetch with exponential backoff"""
    import time
    
    for attempt in range(max_retries):
        try:
            return client.fetch_artifacts()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
```

### Pattern 3: Data Quality Checks

```python
def validate_artifacts(df: pd.DataFrame) -> pd.DataFrame:
    """Remove invalid records"""
    # Remove duplicates
    df = df.drop_duplicates(subset=['id'])
    
    # Remove rows with missing critical fields
    df = df.dropna(subset=['id', 'title'])
    
    # Standardize text fields
    df['culture'] = df['culture'].str.strip().str.title()
    
    return df
```

## Troubleshooting

**Issue: API rate limit exceeded**
```python
# Add delay between requests
import time
time.sleep(1)  # Wait 1 second between calls
```

**Issue: Database connection timeout**
```python
# Increase connection timeout
db_config = {
    'host': 'localhost',
    'connect_timeout': 30
}
```

**Issue: Memory error with large datasets**
```python
# Use chunked processing
def process_in_chunks(artifacts: List[Dict], chunk_size: int = 100):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        yield chunk
```

**Issue: Missing environment variables**
```python
from dotenv import load_dotenv
load_dotenv()

# Validate required vars
required = ['HARVARD_API_KEY', 'DB_HOST', 'DB_PASSWORD']
for var in required:
    if not os.getenv(var):
        raise ValueError(f"Missing required environment variable: {var}")
```

## Advanced Usage

### Custom Analytics Query

```python
def add_custom_query(query_name: str, sql: str):
    """Add custom analytical query"""
    ANALYTICS_QUERIES[query_name] = sql

# Example: Find artifacts with specific color palette
add_custom_query(
    "warm_color_artifacts",
    """
    SELECT am.title, am.culture, COUNT(DISTINCT ac.color) as color_count
    FROM artifactmetadata am
    JOIN artifactcolors ac ON am.id = ac.artifact_id
    WHERE ac.spectrum IN ('red', 'orange', 'yellow')
    GROUP BY am.id, am.title, am.culture
    HAVING color_count >= 2
    ORDER BY color_count DESC
    """
)
```

### Export Results

```python
def export_query_results(df: pd.DataFrame, filename: str):
    """Export query results to CSV"""
    df.to_csv(filename, index=False)
    
    # Or Excel
    df.to_excel(filename.replace('.csv', '.xlsx'), index=False)
```

This skill enables AI agents to build complete data engineering pipelines for museum collections with production-ready ETL, SQL analytics, and interactive visualizations.
