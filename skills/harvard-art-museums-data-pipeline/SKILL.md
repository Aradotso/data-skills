---
name: harvard-art-museums-data-pipeline
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build a data pipeline for museum artifacts
  - create an ETL workflow with Harvard Art Museums API
  - set up artifact collection analytics with Streamlit
  - implement SQL analytics for art museum data
  - build a museum data engineering application
  - create interactive visualizations for artifact collections
  - design a relational database for museum API data
  - develop a Harvard Art Museums data app
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering solution for collecting, transforming, storing, and analyzing artifact data from the Harvard Art Museums API. It demonstrates production-grade ETL pipelines with SQL database design, batch processing, and interactive analytics dashboards using Streamlit and Plotly.

**Key Components:**
- API data extraction with pagination and rate limiting
- ETL transformations for nested JSON to relational tables
- MySQL/TiDB Cloud storage with proper schema design
- Pre-built SQL analytics queries
- Interactive Streamlit dashboard with Plotly visualizations

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
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

### Getting API Access

1. Register at [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Obtain your API key
3. Store in environment variables or Streamlit secrets

### Database Setup

Create the database schema:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    dated VARCHAR(100),
    culture VARCHAR(200),
    century VARCHAR(100),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(500),
    classification VARCHAR(200),
    url TEXT,
    lastupdate DATETIME
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    iiifbaseuri TEXT,
    primaryimageurl TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(100),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will open in your browser at http://localhost:8501
```

## Core Implementation Patterns

### 1. API Data Extraction

```python
import requests
import os
from typing import Dict, List, Optional

class HarvardAPIClient:
    """Client for Harvard Art Museums API with pagination support"""
    
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.harvardartmuseums.org"
        
    def fetch_artifacts(self, page: int = 1, size: int = 100) -> Dict:
        """Fetch artifacts with pagination"""
        endpoint = f"{self.base_url}/object"
        params = {
            'apikey': self.api_key,
            'page': page,
            'size': size,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(endpoint, params=params)
        response.raise_for_status()
        return response.json()
    
    def fetch_multiple_pages(self, num_pages: int = 5, size: int = 100) -> List[Dict]:
        """Fetch multiple pages of artifacts"""
        all_artifacts = []
        
        for page in range(1, num_pages + 1):
            print(f"Fetching page {page}...")
            data = self.fetch_artifacts(page=page, size=size)
            all_artifacts.extend(data.get('records', []))
            
        return all_artifacts

# Usage
api_key = os.getenv('HARVARD_API_KEY')
client = HarvardAPIClient(api_key)
artifacts = client.fetch_multiple_pages(num_pages=10)
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
from typing import Tuple

class ArtifactETL:
    """ETL transformations for Harvard artifact data"""
    
    @staticmethod
    def extract_metadata(artifacts: List[Dict]) -> pd.DataFrame:
        """Extract core metadata fields"""
        metadata_records = []
        
        for artifact in artifacts:
            record = {
                'id': artifact.get('id'),
                'title': artifact.get('title', ''),
                'dated': artifact.get('dated', ''),
                'culture': artifact.get('culture', ''),
                'century': artifact.get('century', ''),
                'department': artifact.get('department', ''),
                'division': artifact.get('division', ''),
                'technique': artifact.get('technique', ''),
                'classification': artifact.get('classification', ''),
                'url': artifact.get('url', ''),
                'lastupdate': artifact.get('lastupdate', '')
            }
            metadata_records.append(record)
        
        return pd.DataFrame(metadata_records)
    
    @staticmethod
    def extract_media(artifacts: List[Dict]) -> pd.DataFrame:
        """Extract media and image URLs"""
        media_records = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            media = {
                'artifact_id': artifact_id,
                'baseimageurl': artifact.get('baseimageurl', ''),
                'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri', '') if artifact.get('images') else '',
                'primaryimageurl': artifact.get('primaryimageurl', '')
            }
            media_records.append(media)
        
        return pd.DataFrame(media_records)
    
    @staticmethod
    def extract_colors(artifacts: List[Dict]) -> pd.DataFrame:
        """Extract color information"""
        color_records = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            colors = artifact.get('colors', [])
            
            for color in colors:
                record = {
                    'artifact_id': artifact_id,
                    'color': color.get('color', ''),
                    'spectrum': color.get('spectrum', ''),
                    'percent': color.get('percent', 0.0)
                }
                color_records.append(record)
        
        return pd.DataFrame(color_records)
    
    @classmethod
    def transform_all(cls, artifacts: List[Dict]) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
        """Transform all artifact data into three DataFrames"""
        metadata_df = cls.extract_metadata(artifacts)
        media_df = cls.extract_media(artifacts)
        colors_df = cls.extract_colors(artifacts)
        
        return metadata_df, media_df, colors_df
```

### 3. Database Loading

```python
import mysql.connector
from typing import List

class ArtifactDB:
    """Database operations for artifact data"""
    
    def __init__(self, host: str, port: int, user: str, password: str, database: str):
        self.config = {
            'host': host,
            'port': port,
            'user': user,
            'password': password,
            'database': database
        }
    
    def get_connection(self):
        """Create database connection"""
        return mysql.connector.connect(**self.config)
    
    def load_metadata(self, df: pd.DataFrame):
        """Batch insert metadata"""
        conn = self.get_connection()
        cursor = conn.cursor()
        
        insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, dated, culture, century, department, division, technique, classification, url, lastupdate)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), dated=VALUES(dated), culture=VALUES(culture)
        """
        
        data = df.values.tolist()
        cursor.executemany(insert_query, data)
        conn.commit()
        
        cursor.close()
        conn.close()
        
        print(f"Loaded {len(df)} metadata records")
    
    def load_media(self, df: pd.DataFrame):
        """Batch insert media data"""
        conn = self.get_connection()
        cursor = conn.cursor()
        
        insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, iiifbaseuri, primaryimageurl)
        VALUES (%s, %s, %s, %s)
        """
        
        data = df.values.tolist()
        cursor.executemany(insert_query, data)
        conn.commit()
        
        cursor.close()
        conn.close()
        
        print(f"Loaded {len(df)} media records")
    
    def execute_query(self, query: str) -> pd.DataFrame:
        """Execute SELECT query and return DataFrame"""
        conn = self.get_connection()
        df = pd.read_sql(query, conn)
        conn.close()
        return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_analytics_dashboard():
    """Main Streamlit dashboard"""
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    with st.sidebar:
        st.header("Configuration")
        api_key = st.text_input("Harvard API Key", type="password", value=os.getenv('HARVARD_API_KEY', ''))
        num_pages = st.slider("Pages to fetch", 1, 20, 5)
        
        if st.button("Collect Data"):
            with st.spinner("Fetching artifacts..."):
                client = HarvardAPIClient(api_key)
                artifacts = client.fetch_multiple_pages(num_pages=num_pages)
                st.success(f"Collected {len(artifacts)} artifacts")
    
    # Analytics queries
    st.header("📊 SQL Analytics")
    
    queries = {
        "Artifacts by Culture": """
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
            GROUP BY department 
            ORDER BY count DESC
        """,
        "Top Colors": """
            SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY frequency DESC 
            LIMIT 10
        """
    }
    
    query_name = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        db = ArtifactDB(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        df = db.execute_query(queries[query_name])
        
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Visualization
        if len(df) > 0:
            col1, col2 = df.columns[:2]
            fig = px.bar(df, x=col1, y=col2, title=query_name)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    create_analytics_dashboard()
```

## Common Analytics Queries

### Artifact Insights

```sql
-- Artifacts with complete metadata
SELECT COUNT(*) as total_complete
FROM artifactmetadata
WHERE culture IS NOT NULL 
  AND century IS NOT NULL 
  AND department IS NOT NULL;

-- Media availability analysis
SELECT 
    CASE 
        WHEN primaryimageurl IS NOT NULL AND primaryimageurl != '' THEN 'Has Image'
        ELSE 'No Image'
    END as image_status,
    COUNT(*) as count
FROM artifactmedia
GROUP BY image_status;

-- Color spectrum distribution
SELECT 
    spectrum,
    COUNT(DISTINCT artifact_id) as artifact_count,
    AVG(percent) as avg_coverage
FROM artifactcolors
GROUP BY spectrum
ORDER BY artifact_count DESC;
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(client, page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return client.fetch_artifacts(page=page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_db_connection():
    """Verify database connectivity"""
    try:
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD')
        )
        print("✓ Database connection successful")
        conn.close()
        return True
    except Exception as e:
        print(f"✗ Connection failed: {e}")
        return False
```

### Empty Data Handling

```python
def safe_extract(artifacts: List[Dict]) -> pd.DataFrame:
    """Extract with fallback for missing fields"""
    if not artifacts:
        return pd.DataFrame()
    
    df = ArtifactETL.extract_metadata(artifacts)
    
    # Fill missing values
    df['culture'] = df['culture'].fillna('Unknown')
    df['century'] = df['century'].fillna('Unknown')
    
    return df
```

## Best Practices

1. **Always use environment variables** for credentials
2. **Implement pagination** for large datasets
3. **Use batch inserts** for performance (100-1000 records per batch)
4. **Handle NULL values** in SQL queries with COALESCE
5. **Cache results** in Streamlit with `@st.cache_data`
6. **Log ETL operations** for debugging and monitoring
