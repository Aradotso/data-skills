---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with MySQL/TiDB storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifact data
  - set up Harvard Art Museums API data collection
  - implement museum data warehouse with SQL analytics
  - visualize Harvard Art Museums collection data
  - extract and analyze art museum API data
  - build Streamlit app for museum artifact analytics
  - create data engineering pipeline for art collections
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to build and work with the Harvard Art Museums ETL pipeline and analytics application. The project demonstrates production-grade data engineering patterns: API integration, ETL pipelines, relational database design, SQL analytics, and interactive visualization using Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is a complete data engineering solution that:

- **Extracts** artifact data from the Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables (metadata, media, colors)
- **Loads** data into MySQL/TiDB Cloud with batch operations
- **Analyzes** data using 20+ predefined SQL analytical queries
- **Visualizes** results through interactive Streamlit dashboards with Plotly charts

The application follows the architecture: `API → ETL → SQL → Analytics → Visualization`

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
DB_HOST=your_db_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    description TEXT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accession_number VARCHAR(100),
    provenance TEXT,
    copyright TEXT,
    url VARCHAR(500),
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    thumbnail_url VARCHAR(1000),
    has_image BOOLEAN,
    total_images INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id) ON DELETE CASCADE
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id) ON DELETE CASCADE
);
```

## Key Commands

### Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Run on specific port
streamlit run app.py --server.port 8501

# Run with custom configuration
streamlit run app.py --server.headless true
```

## API Integration Patterns

### Harvard Art Museums API Client

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
            'size': size,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(self.base_url, params=params)
        response.raise_for_status()
        return response.json()
    
    def fetch_artifacts_batch(self, total_pages=10):
        """Fetch multiple pages of artifacts"""
        all_artifacts = []
        
        for page in range(1, total_pages + 1):
            print(f"Fetching page {page}/{total_pages}")
            data = self.fetch_artifacts(page=page)
            all_artifacts.extend(data.get('records', []))
            
            # Respect rate limits
            import time
            time.sleep(0.5)
        
        return all_artifacts

# Usage
client = HarvardAPIClient()
artifacts = client.fetch_artifacts_batch(total_pages=5)
```

## ETL Pipeline Implementation

### Extract Transform Load Pattern

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class ArtifactETL:
    def __init__(self, db_config):
        self.db_config = db_config
        self.connection = None
        
    def connect(self):
        """Establish database connection"""
        self.connection = mysql.connector.connect(**self.db_config)
        return self.connection
    
    def extract_metadata(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract and transform artifact metadata"""
        metadata_list = []
        
        for artifact in artifacts:
            metadata = {
                'artifact_id': artifact.get('id'),
                'title': artifact.get('title'),
                'culture': artifact.get('culture'),
                'period': artifact.get('period'),
                'century': artifact.get('century'),
                'classification': artifact.get('classification'),
                'department': artifact.get('department'),
                'division': artifact.get('division'),
                'dated': artifact.get('dated'),
                'description': artifact.get('description'),
                'technique': artifact.get('technique'),
                'medium': artifact.get('medium'),
                'dimensions': artifact.get('dimensions'),
                'creditline': artifact.get('creditline'),
                'accession_number': artifact.get('accessionyear'),
                'provenance': artifact.get('provenance'),
                'copyright': artifact.get('copyright'),
                'url': artifact.get('url')
            }
            metadata_list.append(metadata)
        
        return pd.DataFrame(metadata_list)
    
    def extract_media(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract media information"""
        media_list = []
        
        for artifact in artifacts:
            primary_image = artifact.get('primaryimageurl')
            images = artifact.get('images', [])
            
            media = {
                'artifact_id': artifact.get('id'),
                'image_url': primary_image,
                'thumbnail_url': images[0].get('baseimageurl') if images else None,
                'has_image': 1 if primary_image else 0,
                'total_images': len(images)
            }
            media_list.append(media)
        
        return pd.DataFrame(media_list)
    
    def extract_colors(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract color data"""
        color_list = []
        
        for artifact in artifacts:
            colors = artifact.get('colors', [])
            
            for color in colors:
                color_data = {
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                }
                color_list.append(color_data)
        
        return pd.DataFrame(color_list)
    
    def load_dataframe(self, df: pd.DataFrame, table_name: str):
        """Load DataFrame into SQL table using batch insert"""
        if df.empty:
            return
        
        cursor = self.connection.cursor()
        
        # Prepare column names and placeholders
        columns = ', '.join(df.columns)
        placeholders = ', '.join(['%s'] * len(df.columns))
        
        insert_query = f"""
            INSERT INTO {table_name} ({columns}) 
            VALUES ({placeholders})
            ON DUPLICATE KEY UPDATE {', '.join([f'{col}=VALUES({col})' for col in df.columns if col != df.columns[0]])}
        """
        
        # Batch insert
        data = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data)
        self.connection.commit()
        
        print(f"Loaded {len(df)} records into {table_name}")
        cursor.close()
    
    def run_pipeline(self, artifacts: List[Dict]):
        """Execute complete ETL pipeline"""
        self.connect()
        
        # Extract and transform
        metadata_df = self.extract_metadata(artifacts)
        media_df = self.extract_media(artifacts)
        colors_df = self.extract_colors(artifacts)
        
        # Load
        self.load_dataframe(metadata_df, 'artifactmetadata')
        self.load_dataframe(media_df, 'artifactmedia')
        self.load_dataframe(colors_df, 'artifactcolors')
        
        self.connection.close()
        print("ETL pipeline completed successfully")

# Usage
db_config = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

etl = ArtifactETL(db_config)
etl.run_pipeline(artifacts)
```

## SQL Analytics Queries

### Predefined Analytical Queries

```python
ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
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
            has_image,
            COUNT(*) as count,
            ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM artifactmedia), 2) as percentage
        FROM artifactmedia
        GROUP BY has_image
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
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "artifacts_by_classification": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "media_coverage": """
        SELECT 
            CASE 
                WHEN total_images = 0 THEN 'No Images'
                WHEN total_images = 1 THEN '1 Image'
                WHEN total_images BETWEEN 2 AND 5 THEN '2-5 Images'
                ELSE '6+ Images'
            END as image_range,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_range
        ORDER BY count DESC
    """,
    
    "artifacts_with_provenance": """
        SELECT 
            CASE WHEN provenance IS NOT NULL THEN 'Has Provenance' ELSE 'No Provenance' END as status,
            COUNT(*) as count
        FROM artifactmetadata
        GROUP BY status
    """
}
```

### Query Execution Pattern

```python
import pandas as pd

class SQLAnalytics:
    def __init__(self, db_config):
        self.db_config = db_config
    
    def execute_query(self, query: str) -> pd.DataFrame:
        """Execute SQL query and return DataFrame"""
        connection = mysql.connector.connect(**self.db_config)
        df = pd.read_sql(query, connection)
        connection.close()
        return df
    
    def get_analytics(self, query_name: str) -> pd.DataFrame:
        """Get predefined analytics query results"""
        if query_name not in ANALYTICS_QUERIES:
            raise ValueError(f"Unknown query: {query_name}")
        
        return self.execute_query(ANALYTICS_QUERIES[query_name])

# Usage
analytics = SQLAnalytics(db_config)
culture_data = analytics.get_analytics("artifacts_by_culture")
```

## Streamlit Dashboard Patterns

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["Data Collection", "Analytics Dashboard", "ETL Status"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "Analytics Dashboard":
        show_analytics_dashboard()
    elif page == "ETL Status":
        show_etl_status()

def show_data_collection():
    st.header("📥 Data Collection")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    with col2:
        records_per_page = st.number_input("Records per page", min_value=10, max_value=100, value=100)
    
    if st.button("Start Data Collection"):
        with st.spinner("Fetching data from Harvard API..."):
            client = HarvardAPIClient()
            artifacts = client.fetch_artifacts_batch(total_pages=num_pages)
            
            st.success(f"Fetched {len(artifacts)} artifacts")
            
            # Run ETL
            with st.spinner("Running ETL pipeline..."):
                etl = ArtifactETL(db_config)
                etl.run_pipeline(artifacts)
            
            st.success("ETL pipeline completed!")

def show_analytics_dashboard():
    st.header("📊 Analytics Dashboard")
    
    analytics = SQLAnalytics(db_config)
    
    # Query selector
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            df = analytics.get_analytics(query_name)
            
            # Display results
            st.subheader("Query Results")
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

def show_etl_status():
    st.header("🔧 ETL Pipeline Status")
    
    analytics = SQLAnalytics(db_config)
    
    # Database statistics
    col1, col2, col3 = st.columns(3)
    
    with col1:
        metadata_count = analytics.execute_query("SELECT COUNT(*) as count FROM artifactmetadata")
        st.metric("Total Artifacts", metadata_count.iloc[0]['count'])
    
    with col2:
        media_count = analytics.execute_query("SELECT COUNT(*) as count FROM artifactmedia")
        st.metric("Media Records", media_count.iloc[0]['count'])
    
    with col3:
        color_count = analytics.execute_query("SELECT COUNT(*) as count FROM artifactcolors")
        st.metric("Color Records", color_count.iloc[0]['count'])

if __name__ == "__main__":
    main()
```

## Common Patterns

### Error Handling and Retry Logic

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(url, params, max_retries=3):
    """Fetch data with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Request failed, retrying in {wait_time}s...")
            time.sleep(wait_time)
```

### Database Connection Pooling

```python
from mysql.connector import pooling

class DatabasePool:
    _pool = None
    
    @classmethod
    def get_pool(cls, db_config, pool_size=5):
        if cls._pool is None:
            cls._pool = pooling.MySQLConnectionPool(
                pool_name="harvard_pool",
                pool_size=pool_size,
                **db_config
            )
        return cls._pool
    
    @classmethod
    def get_connection(cls, db_config):
        pool = cls.get_pool(db_config)
        return pool.get_connection()
```

## Troubleshooting

### API Rate Limiting

```python
# Add delay between requests
import time

for page in range(1, total_pages + 1):
    artifacts = fetch_artifacts(page)
    time.sleep(0.5)  # 500ms delay to respect rate limits
```

### Database Connection Issues

```python
# Test connection before running ETL
try:
    connection = mysql.connector.connect(**db_config)
    print("Database connection successful")
    connection.close()
except mysql.connector.Error as err:
    print(f"Database connection failed: {err}")
    st.error(f"Cannot connect to database: {err}")
```

### Memory Management for Large Datasets

```python
# Process in chunks
def process_in_chunks(artifacts, chunk_size=1000):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        etl.run_pipeline(chunk)
        print(f"Processed {i + len(chunk)}/{len(artifacts)} artifacts")
```

### Missing API Key

```python
# Validate environment variables
def validate_config():
    required_vars = ['HARVARD_API_KEY', 'DB_HOST', 'DB_USER', 'DB_PASSWORD']
    missing = [var for var in required_vars if not os.getenv(var)]
    
    if missing:
        raise EnvironmentError(f"Missing required environment variables: {', '.join(missing)}")

validate_config()
```

This skill provides comprehensive coverage of building ETL pipelines with the Harvard Art Museums API, implementing SQL analytics, and creating interactive dashboards with Streamlit.
