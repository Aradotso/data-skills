---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - create an ETL pipeline for museum data
  - build a data engineering app with Harvard Art Museums API
  - set up analytics dashboard for artifact collections
  - extract and analyze art museum data
  - build Streamlit app for museum artifacts
  - create SQL analytics for cultural heritage data
  - process Harvard Art Museums API data
  - design artifact metadata database schema
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates production-grade ETL pipelines, SQL database design, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Collect artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transform nested JSON into relational database structures
- **SQL Analytics**: Run 20+ predefined analytical queries on museum data
- **Interactive Dashboard**: Streamlit-based UI with Plotly visualizations
- **Database Design**: Proper schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables

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

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file:
```bash
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Setup

**MySQL/TiDB Cloud configuration:**

```python
import os
from dotenv import load_dotenv

load_dotenv()

DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

Environment variables:
```bash
DB_HOST=your_db_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

## Key Components

### 1. API Data Collection

```python
import requests
import pandas as pd
from typing import List, Dict

class HarvardAPIClient:
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.harvardartmuseums.org/object"
    
    def fetch_artifacts(self, page: int = 1, size: int = 100) -> Dict:
        """Fetch artifacts with pagination"""
        params = {
            'apikey': self.api_key,
            'page': page,
            'size': size
        }
        response = requests.get(self.base_url, params=params)
        response.raise_for_status()
        return response.json()
    
    def collect_batch(self, num_pages: int = 10) -> List[Dict]:
        """Collect multiple pages of artifacts"""
        all_artifacts = []
        for page in range(1, num_pages + 1):
            data = self.fetch_artifacts(page=page)
            all_artifacts.extend(data.get('records', []))
        return all_artifacts
```

### 2. ETL Pipeline

**Extract and Transform:**

```python
import pandas as pd
from typing import List, Dict, Tuple

def extract_metadata(artifacts: List[Dict]) -> pd.DataFrame:
    """Extract artifact metadata into DataFrame"""
    metadata = []
    for artifact in artifacts:
        metadata.append({
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'accession_year': artifact.get('accessionyear'),
            'object_number': artifact.get('objectnumber')
        })
    return pd.DataFrame(metadata)

def extract_media(artifacts: List[Dict]) -> pd.DataFrame:
    """Extract media/image data"""
    media = []
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        if images:
            for img in images:
                media.append({
                    'artifact_id': artifact_id,
                    'image_id': img.get('imageid'),
                    'base_url': img.get('baseimageurl'),
                    'width': img.get('width'),
                    'height': img.get('height'),
                    'format': img.get('format')
                })
        else:
            media.append({
                'artifact_id': artifact_id,
                'image_id': None,
                'base_url': None,
                'width': None,
                'height': None,
                'format': None
            })
    return pd.DataFrame(media)

def extract_colors(artifacts: List[Dict]) -> pd.DataFrame:
    """Extract color data"""
    colors = []
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        color_data = artifact.get('colors', [])
        
        for color in color_data:
            colors.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    return pd.DataFrame(colors) if colors else pd.DataFrame()
```

**Load to Database:**

```python
import mysql.connector
from mysql.connector import Error

class DatabaseLoader:
    def __init__(self, config: Dict):
        self.config = config
        self.connection = None
    
    def connect(self):
        """Establish database connection"""
        try:
            self.connection = mysql.connector.connect(**self.config)
            return self.connection
        except Error as e:
            raise Exception(f"Database connection failed: {e}")
    
    def create_tables(self):
        """Create database schema"""
        cursor = self.connection.cursor()
        
        # Metadata table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmetadata (
                artifact_id INT PRIMARY KEY,
                title VARCHAR(500),
                culture VARCHAR(200),
                century VARCHAR(100),
                classification VARCHAR(200),
                department VARCHAR(200),
                dated VARCHAR(200),
                accession_year INT,
                object_number VARCHAR(100)
            )
        """)
        
        # Media table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmedia (
                id INT AUTO_INCREMENT PRIMARY KEY,
                artifact_id INT,
                image_id INT,
                base_url VARCHAR(500),
                width INT,
                height INT,
                format VARCHAR(50),
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
                percent FLOAT,
                FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
            )
        """)
        
        self.connection.commit()
    
    def load_dataframe(self, df: pd.DataFrame, table_name: str):
        """Batch insert DataFrame into table"""
        cursor = self.connection.cursor()
        
        # Prepare insert statement
        cols = ','.join(df.columns)
        placeholders = ','.join(['%s'] * len(df.columns))
        query = f"INSERT IGNORE INTO {table_name} ({cols}) VALUES ({placeholders})"
        
        # Batch insert
        data = [tuple(row) for row in df.values]
        cursor.executemany(query, data)
        self.connection.commit()
        
        return cursor.rowcount
```

### 3. Complete ETL Workflow

```python
import os
from dotenv import load_dotenv

def run_etl_pipeline(num_pages: int = 10):
    """Execute complete ETL pipeline"""
    load_dotenv()
    
    # Extract
    print("Extracting data from Harvard API...")
    client = HarvardAPIClient(os.getenv('HARVARD_API_KEY'))
    artifacts = client.collect_batch(num_pages=num_pages)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    print("Transforming data...")
    df_metadata = extract_metadata(artifacts)
    df_media = extract_media(artifacts)
    df_colors = extract_colors(artifacts)
    
    # Load
    print("Loading data to database...")
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME'),
        'port': int(os.getenv('DB_PORT', 3306))
    }
    
    loader = DatabaseLoader(db_config)
    loader.connect()
    loader.create_tables()
    
    rows_metadata = loader.load_dataframe(df_metadata, 'artifactmetadata')
    rows_media = loader.load_dataframe(df_media, 'artifactmedia')
    rows_colors = loader.load_dataframe(df_colors, 'artifactcolors')
    
    print(f"Loaded {rows_metadata} metadata, {rows_media} media, {rows_colors} color records")
```

### 4. SQL Analytics Queries

```python
class AnalyticsEngine:
    def __init__(self, connection):
        self.connection = connection
    
    def execute_query(self, query: str) -> pd.DataFrame:
        """Execute SQL query and return DataFrame"""
        return pd.read_sql(query, self.connection)
    
    # Sample analytical queries
    QUERIES = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 20
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'department_distribution': """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """,
        
        'media_availability': """
            SELECT 
                CASE WHEN base_url IS NOT NULL THEN 'Has Image' ELSE 'No Image' END as status,
                COUNT(*) as count
            FROM artifactmedia
            GROUP BY status
        """,
        
        'top_colors': """
            SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
            FROM artifactcolors
            WHERE color IS NOT NULL
            GROUP BY color
            ORDER BY count DESC
            LIMIT 10
        """,
        
        'accession_by_year': """
            SELECT accession_year, COUNT(*) as count
            FROM artifactmetadata
            WHERE accession_year IS NOT NULL
            GROUP BY accession_year
            ORDER BY accession_year DESC
            LIMIT 20
        """
    }
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """Build Streamlit analytics dashboard"""
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Database connection
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    conn = mysql.connector.connect(**db_config)
    analytics = AnalyticsEngine(conn)
    
    # Sidebar for query selection
    st.sidebar.header("Select Analysis")
    query_name = st.sidebar.selectbox(
        "Choose Query",
        list(analytics.QUERIES.keys())
    )
    
    # Execute query
    query = analytics.QUERIES[query_name]
    df = analytics.execute_query(query)
    
    # Display results
    st.subheader(f"Results: {query_name.replace('_', ' ').title()}")
    st.dataframe(df)
    
    # Auto-generate visualization
    if len(df.columns) >= 2:
        fig = px.bar(
            df, 
            x=df.columns[0], 
            y=df.columns[1],
            title=f"{query_name.replace('_', ' ').title()}"
        )
        st.plotly_chart(fig, use_container_width=True)
    
    conn.close()

if __name__ == "__main__":
    create_dashboard()
```

## Running the Application

```bash
# Run ETL pipeline
python etl_pipeline.py

# Launch Streamlit dashboard
streamlit run app.py
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def get_latest_artifact_id(connection) -> int:
    """Get the latest artifact ID from database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(artifact_id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(api_key: str, db_config: Dict):
    """Load only new artifacts"""
    client = HarvardAPIClient(api_key)
    loader = DatabaseLoader(db_config)
    loader.connect()
    
    latest_id = get_latest_artifact_id(loader.connection)
    
    # Fetch new artifacts
    artifacts = client.fetch_artifacts()
    new_artifacts = [a for a in artifacts if a.get('id', 0) > latest_id]
    
    if new_artifacts:
        df_metadata = extract_metadata(new_artifacts)
        loader.load_dataframe(df_metadata, 'artifactmetadata')
```

### Pattern 2: Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_api_call(client: HarvardAPIClient, page: int, max_retries: int = 3):
    """API call with retry logic"""
    for attempt in range(max_retries):
        try:
            return client.fetch_artifacts(page=page)
        except requests.exceptions.RequestException as e:
            logger.warning(f"Attempt {attempt + 1} failed: {e}")
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

### Pattern 3: Data Quality Checks

```python
def validate_artifact_data(df: pd.DataFrame) -> bool:
    """Validate extracted data quality"""
    checks = {
        'no_duplicate_ids': df['artifact_id'].is_unique,
        'no_null_ids': df['artifact_id'].notna().all(),
        'valid_years': df['accession_year'].between(1800, 2030, inclusive='both').all()
    }
    
    for check_name, passed in checks.items():
        if not passed:
            logger.error(f"Data quality check failed: {check_name}")
            return False
    
    return True
```

## Troubleshooting

### API Rate Limiting

```python
import time

def rate_limited_fetch(client: HarvardAPIClient, pages: int, delay: float = 1.0):
    """Fetch with rate limiting"""
    artifacts = []
    for page in range(1, pages + 1):
        data = client.fetch_artifacts(page=page)
        artifacts.extend(data.get('records', []))
        time.sleep(delay)  # Respect rate limits
    return artifacts
```

### Database Connection Issues

```python
def test_database_connection(config: Dict) -> bool:
    """Test database connectivity"""
    try:
        conn = mysql.connector.connect(**config)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        conn.close()
        return result[0] == 1
    except Error as e:
        logger.error(f"Database connection test failed: {e}")
        return False
```

### Memory Optimization for Large Datasets

```python
def chunked_load(artifacts: List[Dict], chunk_size: int = 1000):
    """Load data in chunks to manage memory"""
    loader = DatabaseLoader(db_config)
    loader.connect()
    
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        df_metadata = extract_metadata(chunk)
        loader.load_dataframe(df_metadata, 'artifactmetadata')
        logger.info(f"Loaded chunk {i // chunk_size + 1}")
```

This skill provides complete guidance for building production-ready ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit.
