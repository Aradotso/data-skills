---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline using Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifact data
  - create a data engineering project with Harvard Art Museums API
  - set up analytics dashboard for art collections
  - implement SQL analytics on artifact metadata
  - visualize museum data with Streamlit and Plotly
  - design relational database for art museum collections
  - build real-world data pipeline with Python and SQL
  - create interactive analytics app for cultural heritage data
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill helps you build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytics queries, and interactive visualization with Streamlit.

## What It Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON into relational SQL tables
- **Database Design**: Structured storage with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Visualization**: Streamlit dashboard with Plotly charts

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

### Prerequisites

```bash
# Install Python dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Environment Setup

Create a `.env` file:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit [Harvard Art Museums API](https://harvardartmuseums.org/collections/api)
2. Request an API key (free)
3. Add to `.env` file

### Database Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    dated VARCHAR(200),
    century VARCHAR(100),
    period VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accession_number VARCHAR(100),
    provenance TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(50),
    baseimageurl VARCHAR(1000),
    iiifbaseuri VARCHAR(1000),
    width INT,
    height INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Launch Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Core Components

### 1. API Data Collection

```python
import requests
import pandas as pd
from typing import Dict, List
import time

class HarvardAPIClient:
    """Client for Harvard Art Museums API"""
    
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.harvardartmuseums.org/object"
    
    def fetch_artifacts(self, page: int = 1, size: int = 100) -> Dict:
        """Fetch artifacts with pagination"""
        params = {
            'apikey': self.api_key,
            'page': page,
            'size': size,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(self.base_url, params=params)
        response.raise_for_status()
        return response.json()
    
    def collect_batch(self, num_pages: int = 5) -> List[Dict]:
        """Collect multiple pages of artifacts"""
        artifacts = []
        
        for page in range(1, num_pages + 1):
            print(f"Fetching page {page}...")
            data = self.fetch_artifacts(page=page)
            artifacts.extend(data.get('records', []))
            
            # Rate limiting
            time.sleep(0.5)
        
        return artifacts

# Usage
from dotenv import load_dotenv
import os

load_dotenv()
client = HarvardAPIClient(os.getenv('HARVARD_API_KEY'))
artifacts = client.collect_batch(num_pages=3)
```

### 2. ETL Pipeline

```python
import pandas as pd
from typing import List, Dict, Tuple

class ArtifactETL:
    """Extract, Transform, Load pipeline for artifact data"""
    
    @staticmethod
    def extract_metadata(artifacts: List[Dict]) -> pd.DataFrame:
        """Extract metadata from artifacts"""
        metadata = []
        
        for artifact in artifacts:
            metadata.append({
                'id': artifact.get('id'),
                'title': artifact.get('title'),
                'culture': artifact.get('culture'),
                'dated': artifact.get('dated'),
                'century': artifact.get('century'),
                'period': artifact.get('period'),
                'classification': artifact.get('classification'),
                'department': artifact.get('department'),
                'technique': artifact.get('technique'),
                'medium': artifact.get('medium'),
                'dimensions': artifact.get('dimensions'),
                'creditline': artifact.get('creditline'),
                'accession_number': artifact.get('accessionmethod'),
                'provenance': artifact.get('provenance')
            })
        
        return pd.DataFrame(metadata)
    
    @staticmethod
    def extract_media(artifacts: List[Dict]) -> pd.DataFrame:
        """Extract media/image data from artifacts"""
        media_records = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            images = artifact.get('images', [])
            
            for image in images:
                media_records.append({
                    'artifact_id': artifact_id,
                    'media_type': 'image',
                    'baseimageurl': image.get('baseimageurl'),
                    'iiifbaseuri': image.get('iiifbaseuri'),
                    'width': image.get('width'),
                    'height': image.get('height')
                })
        
        return pd.DataFrame(media_records)
    
    @staticmethod
    def extract_colors(artifacts: List[Dict]) -> pd.DataFrame:
        """Extract color data from artifacts"""
        color_records = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            colors = artifact.get('colors', [])
            
            for color in colors:
                color_records.append({
                    'artifact_id': artifact_id,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'percentage': color.get('percent')
                })
        
        return pd.DataFrame(color_records)
    
    @staticmethod
    def transform_all(artifacts: List[Dict]) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
        """Transform all artifact data into dataframes"""
        metadata_df = ArtifactETL.extract_metadata(artifacts)
        media_df = ArtifactETL.extract_media(artifacts)
        colors_df = ArtifactETL.extract_colors(artifacts)
        
        # Clean null values
        metadata_df = metadata_df.fillna('')
        media_df = media_df.fillna('')
        colors_df = colors_df.fillna(0)
        
        return metadata_df, media_df, colors_df

# Usage
etl = ArtifactETL()
metadata_df, media_df, colors_df = etl.transform_all(artifacts)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error
import pandas as pd

class DatabaseLoader:
    """Load transformed data into SQL database"""
    
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
        """Load metadata into artifactmetadata table"""
        conn = self.get_connection()
        cursor = conn.cursor()
        
        insert_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, dated, century, period, classification, 
             department, technique, medium, dimensions, creditline, 
             accession_number, provenance)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
            title=VALUES(title), culture=VALUES(culture)
        """
        
        data = df.values.tolist()
        cursor.executemany(insert_query, data)
        conn.commit()
        
        print(f"Loaded {cursor.rowcount} metadata records")
        cursor.close()
        conn.close()
    
    def load_media(self, df: pd.DataFrame):
        """Load media into artifactmedia table"""
        conn = self.get_connection()
        cursor = conn.cursor()
        
        insert_query = """
            INSERT INTO artifactmedia 
            (artifact_id, media_type, baseimageurl, iiifbaseuri, width, height)
            VALUES (%s, %s, %s, %s, %s, %s)
        """
        
        data = df.values.tolist()
        cursor.executemany(insert_query, data)
        conn.commit()
        
        print(f"Loaded {cursor.rowcount} media records")
        cursor.close()
        conn.close()
    
    def load_colors(self, df: pd.DataFrame):
        """Load colors into artifactcolors table"""
        conn = self.get_connection()
        cursor = conn.cursor()
        
        insert_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, percentage)
            VALUES (%s, %s, %s, %s)
        """
        
        data = df.values.tolist()
        cursor.executemany(insert_query, data)
        conn.commit()
        
        print(f"Loaded {cursor.rowcount} color records")
        cursor.close()
        conn.close()

# Usage
loader = DatabaseLoader(
    host=os.getenv('DB_HOST'),
    port=int(os.getenv('DB_PORT')),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

loader.load_metadata(metadata_df)
loader.load_media(media_df)
loader.load_colors(colors_df)
```

### 4. Analytics Queries

```python
class ArtifactAnalytics:
    """SQL analytics queries for artifact data"""
    
    QUERIES = {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture != ''
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 15
        """,
        
        "Artifacts by Century": """
            SELECT century, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE century != ''
            GROUP BY century
            ORDER BY artifact_count DESC
        """,
        
        "Top Departments": """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE department != ''
            GROUP BY department
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        
        "Media Availability": """
            SELECT 
                COUNT(DISTINCT artifact_id) as artifacts_with_media,
                COUNT(*) as total_media_items
            FROM artifactmedia
        """,
        
        "Color Distribution": """
            SELECT color, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
            FROM artifactcolors
            WHERE color != ''
            GROUP BY color
            ORDER BY usage_count DESC
            LIMIT 10
        """,
        
        "Artifacts by Classification": """
            SELECT classification, COUNT(*) as count
            FROM artifactmetadata
            WHERE classification != ''
            GROUP BY classification
            ORDER BY count DESC
            LIMIT 12
        """,
        
        "Average Image Dimensions": """
            SELECT 
                AVG(width) as avg_width,
                AVG(height) as avg_height,
                MAX(width) as max_width,
                MAX(height) as max_height
            FROM artifactmedia
            WHERE width > 0 AND height > 0
        """
    }
    
    @staticmethod
    def execute_query(query: str, db_config: dict) -> pd.DataFrame:
        """Execute SQL query and return results as DataFrame"""
        conn = mysql.connector.connect(**db_config)
        df = pd.read_sql(query, conn)
        conn.close()
        return df

# Usage
query = ArtifactAnalytics.QUERIES["Artifacts by Culture"]
results_df = ArtifactAnalytics.execute_query(query, loader.config)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px
import plotly.graph_objects as go

def create_dashboard():
    """Create Streamlit analytics dashboard"""
    
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("Real-time analytics on artifact collections")
    
    # Sidebar for configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    db_config = {
        'host': os.getenv('DB_HOST'),
        'port': int(os.getenv('DB_PORT')),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    # Query selection
    query_name = st.sidebar.selectbox(
        "Select Analytics Query",
        list(ArtifactAnalytics.QUERIES.keys())
    )
    
    if st.sidebar.button("Run Query"):
        with st.spinner("Executing query..."):
            query = ArtifactAnalytics.QUERIES[query_name]
            df = ArtifactAnalytics.execute_query(query, db_config)
            
            # Display results
            st.subheader(f"Results: {query_name}")
            st.dataframe(df, use_container_width=True)
            
            # Auto-generate visualization
            if len(df.columns) == 2 and len(df) > 0:
                fig = px.bar(
                    df,
                    x=df.columns[0],
                    y=df.columns[1],
                    title=query_name,
                    labels={df.columns[0]: df.columns[0].replace('_', ' ').title()}
                )
                fig.update_layout(height=500)
                st.plotly_chart(fig, use_container_width=True)
    
    # ETL Section
    st.sidebar.header("Data Collection")
    
    if st.sidebar.button("Fetch New Artifacts"):
        api_key = os.getenv('HARVARD_API_KEY')
        num_pages = st.sidebar.number_input("Pages to fetch", 1, 10, 3)
        
        with st.spinner(f"Fetching {num_pages} pages..."):
            client = HarvardAPIClient(api_key)
            artifacts = client.collect_batch(num_pages=num_pages)
            
            st.success(f"Collected {len(artifacts)} artifacts")
            
            # ETL
            etl = ArtifactETL()
            metadata_df, media_df, colors_df = etl.transform_all(artifacts)
            
            # Load
            loader = DatabaseLoader(**db_config)
            loader.load_metadata(metadata_df)
            loader.load_media(media_df)
            loader.load_colors(colors_df)
            
            st.success("Data loaded successfully!")

if __name__ == "__main__":
    create_dashboard()
```

## Common Patterns

### Complete ETL Workflow

```python
from dotenv import load_dotenv
import os

# Load environment variables
load_dotenv()

# 1. Extract: Fetch from API
client = HarvardAPIClient(os.getenv('HARVARD_API_KEY'))
artifacts = client.collect_batch(num_pages=5)

# 2. Transform: Convert to DataFrames
etl = ArtifactETL()
metadata_df, media_df, colors_df = etl.transform_all(artifacts)

# 3. Load: Insert into database
loader = DatabaseLoader(
    host=os.getenv('DB_HOST'),
    port=int(os.getenv('DB_PORT')),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

loader.load_metadata(metadata_df)
loader.load_media(media_df)
loader.load_colors(colors_df)

# 4. Analyze: Run queries
query = ArtifactAnalytics.QUERIES["Artifacts by Culture"]
results = ArtifactAnalytics.execute_query(query, loader.config)
print(results)
```

### Incremental Data Loading

```python
def incremental_load(last_artifact_id: int):
    """Load only new artifacts since last run"""
    
    # Fetch with filter
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'id': f'>{last_artifact_id}',
        'size': 100
    }
    
    response = requests.get(
        "https://api.harvardartmuseums.org/object",
        params=params
    )
    
    artifacts = response.json().get('records', [])
    
    if artifacts:
        metadata_df, media_df, colors_df = ArtifactETL.transform_all(artifacts)
        loader.load_metadata(metadata_df)
        loader.load_media(media_df)
        loader.load_colors(colors_df)
        
        return max(a['id'] for a in artifacts)
    
    return last_artifact_id
```

### Custom Analytics Query

```python
def run_custom_analytics(db_config: dict):
    """Run custom analytical queries"""
    
    custom_query = """
        SELECT 
            m.culture,
            m.century,
            COUNT(DISTINCT m.id) as artifact_count,
            COUNT(DISTINCT med.id) as media_count,
            AVG(c.percentage) as avg_color_dominance
        FROM artifactmetadata m
        LEFT JOIN artifactmedia med ON m.id = med.artifact_id
        LEFT JOIN artifactcolors c ON m.id = c.artifact_id
        WHERE m.culture != '' AND m.century != ''
        GROUP BY m.culture, m.century
        HAVING artifact_count > 5
        ORDER BY artifact_count DESC
        LIMIT 20
    """
    
    df = ArtifactAnalytics.execute_query(custom_query, db_config)
    return df
```

## Troubleshooting

### API Rate Limiting

```python
import time
from functools import wraps

def rate_limit(calls_per_second=2):
    """Decorator to limit API calls"""
    min_interval = 1.0 / calls_per_second
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            left_to_wait = min_interval - elapsed
            if left_to_wait > 0:
                time.sleep(left_to_wait)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

class HarvardAPIClient:
    @rate_limit(calls_per_second=1)
    def fetch_artifacts(self, page: int = 1, size: int = 100):
        # API call implementation
        pass
```

### Database Connection Issues

```python
from mysql.connector import Error
import time

def get_connection_with_retry(config, max_retries=3):
    """Create database connection with retry logic"""
    
    for attempt in range(max_retries):
        try:
            conn = mysql.connector.connect(**config)
            return conn
        except Error as e:
            print(f"Attempt {attempt + 1} failed: {e}")
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise
```

### Handling Missing Data

```python
def safe_extract_metadata(artifact: dict) -> dict:
    """Safely extract metadata with default values"""
    
    return {
        'id': artifact.get('id', 0),
        'title': artifact.get('title', 'Untitled'),
        'culture': artifact.get('culture', 'Unknown'),
        'dated': artifact.get('dated', ''),
        'century': artifact.get('century', ''),
        'period': artifact.get('period', ''),
        'classification': artifact.get('classification', 'Unclassified'),
        'department': artifact.get('department', ''),
        'technique': artifact.get('technique', ''),
        'medium': artifact.get('medium', ''),
        'dimensions': artifact.get('dimensions', ''),
        'creditline': artifact.get('creditline', ''),
        'accession_number': artifact.get('accessionmethod', ''),
        'provenance': artifact.get('provenance', '')
    }
```

### Memory Management for Large Datasets

```python
def chunked_load(artifacts: List[Dict], chunk_size: int = 1000):
    """Load data in chunks to manage memory"""
    
    loader = DatabaseLoader(**db_config)
    
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        
        metadata_df, media_df, colors_df = ArtifactETL.transform_all(chunk)
        
        loader.load_metadata(metadata_df)
        loader.load_media(media_df)
        loader.load_colors(colors_df)
        
        print(f"Loaded chunk {i // chunk_size + 1}")
```

## Key Concepts

- **ETL Pipeline**: Extract data from API, Transform into structured format, Load into database
- **Relational Design**: Normalized tables with foreign key relationships
- **Batch Processing**: Handle large datasets efficiently with pagination and chunking
- **Analytics Queries**: Pre-built SQL queries for common insights
- **Interactive Dashboards**: Real-time visualization with Streamlit and Plotly

This skill enables building production-grade data engineering applications with museum collections data.
