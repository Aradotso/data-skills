---
name: harvard-art-museums-data-engineering-pipeline
description: End-to-end ETL pipeline for Harvard Art Museums API with SQL analytics and Streamlit visualization
triggers:
  - build a data pipeline for museum artifacts
  - create ETL workflow with Harvard Art Museums API
  - analyze art collection data with SQL and visualization
  - set up artifact metadata extraction pipeline
  - build analytics dashboard for museum collections
  - implement data engineering pipeline with Streamlit
  - extract and analyze Harvard museum data
  - create art artifacts ETL application
---

# Harvard Art Museums Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides a complete data engineering solution for extracting, transforming, and analyzing artifact data from the Harvard Art Museums API. It demonstrates production-grade ETL pipelines, relational database design, SQL analytics, and interactive visualization using Streamlit.

## What This Project Does

- **API Integration**: Fetches artifact metadata, media files, and color information from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON responses into normalized relational tables
- **Database Storage**: Stores data in MySQL/TiDB Cloud with proper schema design and foreign key relationships
- **SQL Analytics**: Executes 20+ predefined analytical queries for insights
- **Visualization**: Renders interactive dashboards using Plotly and Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
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
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### API Key Setup

1. Register at [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Obtain your API key
3. Add to environment variables or Streamlit secrets

### Streamlit Secrets (Alternative)

Create `.streamlit/secrets.toml`:

```toml
[api]
harvard_key = "your_api_key_here"

[database]
host = "your_database_host"
port = 3306
user = "your_username"
password = "your_password"
database = "harvard_artifacts"
```

## Running the Application

```bash
# Start the Streamlit application
streamlit run app.py

# Access at http://localhost:8501
```

## Database Schema

The pipeline creates three normalized tables:

```sql
-- Artifact Metadata (Main Table)
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    technique VARCHAR(300),
    medium VARCHAR(300),
    dimensions VARCHAR(500),
    dated VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    contact VARCHAR(500),
    description TEXT,
    provenance TEXT,
    url VARCHAR(500),
    INDEX idx_culture (culture),
    INDEX idx_century (century),
    INDEX idx_department (department)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_url VARCHAR(500),
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE
);
```

## Core ETL Pipeline Code

### API Data Extraction

```python
import requests
import pandas as pd
from typing import List, Dict
import time

class HarvardAPIClient:
    """Client for Harvard Art Museums API"""
    
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.harvardartmuseums.org"
        
    def fetch_artifacts(self, page: int = 1, size: int = 100) -> Dict:
        """Fetch artifacts with pagination"""
        endpoint = f"{self.base_url}/object"
        params = {
            "apikey": self.api_key,
            "page": page,
            "size": size
        }
        
        response = requests.get(endpoint, params=params)
        response.raise_for_status()
        return response.json()
    
    def collect_artifacts(self, total_records: int = 1000) -> List[Dict]:
        """Collect multiple pages of artifacts"""
        artifacts = []
        page = 1
        size = 100
        
        while len(artifacts) < total_records:
            try:
                data = self.fetch_artifacts(page=page, size=size)
                records = data.get('records', [])
                
                if not records:
                    break
                
                artifacts.extend(records)
                page += 1
                
                # Rate limiting
                time.sleep(0.5)
                
            except Exception as e:
                print(f"Error on page {page}: {e}")
                break
        
        return artifacts[:total_records]
```

### Data Transformation

```python
def transform_artifacts(artifacts: List[Dict]) -> tuple:
    """Transform raw API data into normalized DataFrames"""
    
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'contact': artifact.get('contact'),
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance'),
            'url': artifact.get('url')
        }
        metadata_records.append(metadata)
        
        # Extract media/images
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        for img in images:
            media = {
                'artifact_id': artifact_id,
                'base_url': img.get('baseimageurl'),
                'format': img.get('format'),
                'height': img.get('height'),
                'width': img.get('width')
            }
            media_records.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color_obj in colors:
            color = {
                'artifact_id': artifact_id,
                'color': color_obj.get('color'),
                'spectrum': color_obj.get('spectrum'),
                'percent': color_obj.get('percent')
            }
            color_records.append(color)
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### Database Loading

```python
import mysql.connector
from mysql.connector import Error

class DatabaseLoader:
    """Handle database operations"""
    
    def __init__(self, host: str, user: str, password: str, database: str):
        self.config = {
            'host': host,
            'user': user,
            'password': password,
            'database': database
        }
    
    def get_connection(self):
        """Create database connection"""
        return mysql.connector.connect(**self.config)
    
    def load_metadata(self, df: pd.DataFrame):
        """Bulk insert artifact metadata"""
        conn = self.get_connection()
        cursor = conn.cursor()
        
        insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, technique, 
         medium, dimensions, dated, department, division, contact, 
         description, provenance, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
        """
        
        data = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data)
        conn.commit()
        
        cursor.close()
        conn.close()
    
    def load_media(self, df: pd.DataFrame):
        """Bulk insert media records"""
        if df.empty:
            return
        
        conn = self.get_connection()
        cursor = conn.cursor()
        
        insert_query = """
        INSERT INTO artifactmedia (artifact_id, base_url, format, height, width)
        VALUES (%s, %s, %s, %s, %s)
        """
        
        data = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data)
        conn.commit()
        
        cursor.close()
        conn.close()
    
    def load_colors(self, df: pd.DataFrame):
        """Bulk insert color records"""
        if df.empty:
            return
        
        conn = self.get_connection()
        cursor = conn.cursor()
        
        insert_query = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, percent)
        VALUES (%s, %s, %s, %s)
        """
        
        data = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data)
        conn.commit()
        
        cursor.close()
        conn.close()
```

## Complete ETL Pipeline Example

```python
import os
from dotenv import load_dotenv

def run_etl_pipeline(num_records: int = 500):
    """Execute complete ETL pipeline"""
    
    # Load configuration
    load_dotenv()
    api_key = os.getenv('HARVARD_API_KEY')
    
    # Initialize clients
    api_client = HarvardAPIClient(api_key)
    db_loader = DatabaseLoader(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    # Extract
    print(f"Extracting {num_records} artifacts...")
    artifacts = api_client.collect_artifacts(total_records=num_records)
    
    # Transform
    print("Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(artifacts)
    
    # Load
    print("Loading to database...")
    db_loader.load_metadata(df_metadata)
    db_loader.load_media(df_media)
    db_loader.load_colors(df_colors)
    
    print("ETL pipeline completed successfully!")
    return {
        'metadata_count': len(df_metadata),
        'media_count': len(df_media),
        'color_count': len(df_colors)
    }

# Run the pipeline
if __name__ == "__main__":
    results = run_etl_pipeline(num_records=1000)
    print(f"Loaded: {results}")
```

## SQL Analytics Queries

### Example Analytical Queries

```python
# Sample analytics queries for insights
ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "artifacts_with_images": """
        SELECT 
            CASE WHEN media_count > 0 THEN 'With Images' ELSE 'Without Images' END as status,
            COUNT(*) as artifact_count
        FROM (
            SELECT a.id, COUNT(m.media_id) as media_count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY a.id
        ) as counts
        GROUP BY status
    """,
    
    "top_colors": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "department_distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "classification_breakdown": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """
}
```

### Execute Queries and Visualize

```python
import plotly.express as px

def execute_and_visualize(query: str, query_name: str, db_config: dict):
    """Execute SQL query and create visualization"""
    
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    
    if df.empty:
        return None, None
    
    # Create appropriate chart based on data
    if len(df.columns) == 2:
        fig = px.bar(
            df, 
            x=df.columns[0], 
            y=df.columns[1],
            title=query_name.replace('_', ' ').title(),
            labels={df.columns[0]: df.columns[0].title(), 
                    df.columns[1]: df.columns[1].title()}
        )
        fig.update_layout(xaxis_tickangle=-45)
    else:
        fig = None
    
    return df, fig
```

## Streamlit Application Structure

```python
import streamlit as st

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    # Database configuration
    with st.sidebar.expander("Database Settings"):
        db_host = st.text_input("DB Host", value=os.getenv('DB_HOST', ''))
        db_user = st.text_input("DB User", value=os.getenv('DB_USER', ''))
        db_password = st.text_input("DB Password", type="password")
        db_name = st.text_input("DB Name", value=os.getenv('DB_NAME', ''))
    
    # ETL Section
    st.header("📥 Data Collection (ETL)")
    num_records = st.number_input("Number of Records", min_value=10, max_value=5000, value=500)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            try:
                results = run_etl_pipeline(num_records)
                st.success(f"✅ Loaded {results['metadata_count']} artifacts!")
                st.json(results)
            except Exception as e:
                st.error(f"Error: {e}")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    
    query_choice = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys()),
        format_func=lambda x: x.replace('_', ' ').title()
    )
    
    if st.button("Execute Query"):
        db_config = {
            'host': db_host,
            'user': db_user,
            'password': db_password,
            'database': db_name
        }
        
        with st.spinner("Executing query..."):
            df, fig = execute_and_visualize(
                ANALYTICS_QUERIES[query_choice], 
                query_choice, 
                db_config
            )
            
            if df is not None:
                st.dataframe(df)
                if fig:
                    st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Incremental Data Loading

```python
def incremental_load(last_sync_id: int, batch_size: int = 100):
    """Load only new artifacts since last sync"""
    
    api_client = HarvardAPIClient(os.getenv('HARVARD_API_KEY'))
    
    # Fetch artifacts with ID filter
    endpoint = f"{api_client.base_url}/object"
    params = {
        "apikey": api_client.api_key,
        "q": f"id:>{last_sync_id}",
        "size": batch_size
    }
    
    response = requests.get(endpoint, params=params)
    new_artifacts = response.json().get('records', [])
    
    # Transform and load
    df_metadata, df_media, df_colors = transform_artifacts(new_artifacts)
    
    # Save to database
    db_loader = DatabaseLoader(...)
    db_loader.load_metadata(df_metadata)
    db_loader.load_media(df_media)
    db_loader.load_colors(df_colors)
    
    return len(new_artifacts)
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_run(num_records: int):
    """ETL with comprehensive error handling"""
    
    try:
        # Validate configuration
        required_vars = ['HARVARD_API_KEY', 'DB_HOST', 'DB_USER', 'DB_PASSWORD']
        for var in required_vars:
            if not os.getenv(var):
                raise ValueError(f"Missing environment variable: {var}")
        
        # Run pipeline
        logger.info(f"Starting ETL for {num_records} records")
        results = run_etl_pipeline(num_records)
        logger.info(f"ETL completed: {results}")
        
        return True, results
        
    except requests.RequestException as e:
        logger.error(f"API request failed: {e}")
        return False, {"error": "API connection failed"}
        
    except mysql.connector.Error as e:
        logger.error(f"Database error: {e}")
        return False, {"error": "Database operation failed"}
        
    except Exception as e:
        logger.error(f"Unexpected error: {e}")
        return False, {"error": str(e)}
```

## Troubleshooting

### API Rate Limiting

```python
from functools import wraps
import time

def rate_limited(max_per_second: float):
    """Decorator for rate limiting API calls"""
    min_interval = 1.0 / max_per_second
    
    def decorator(func):
        last_called = [0.0]
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            wait_time = min_interval - elapsed
            
            if wait_time > 0:
                time.sleep(wait_time)
            
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        
        return wrapper
    return decorator

# Apply to API calls
@rate_limited(2)  # Max 2 requests per second
def fetch_with_rate_limit(url, params):
    return requests.get(url, params=params)
```

### Database Connection Pooling

```python
from mysql.connector import pooling

class DatabasePool:
    """Connection pool for better performance"""
    
    def __init__(self, pool_name: str, pool_size: int = 5):
        self.pool = pooling.MySQLConnectionPool(
            pool_name=pool_name,
            pool_size=pool_size,
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
    
    def execute_query(self, query: str):
        conn = self.pool.get_connection()
        cursor = conn.cursor()
        cursor.execute(query)
        results = cursor.fetchall()
        cursor.close()
        conn.close()
        return results
```

### Handling Large Datasets

```python
def chunked_load(df: pd.DataFrame, chunk_size: int = 1000):
    """Load large DataFrames in chunks"""
    
    total_rows = len(df)
    db_loader = DatabaseLoader(...)
    
    for i in range(0, total_rows, chunk_size):
        chunk = df.iloc[i:i+chunk_size]
        db_loader.load_metadata(chunk)
        logger.info(f"Loaded chunk {i//chunk_size + 1}: {len(chunk)} records")
```

This skill provides everything needed to build, deploy, and extend the Harvard Art Museums data engineering pipeline with ETL workflows, SQL analytics, and interactive visualizations.
