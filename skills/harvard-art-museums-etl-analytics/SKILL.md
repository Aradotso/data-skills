---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering and analytics application using Harvard Art Museums API with ETL pipelines, SQL storage, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering app with museum artifacts
  - set up Harvard Art Museums API integration with SQL
  - implement artifact collection data analytics
  - build Streamlit dashboard for museum data
  - extract and analyze Harvard Art Museums collections
  - create SQL analytics for art museum artifacts
  - design ETL workflow for cultural heritage data
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL analytics, and interactive data visualization patterns commonly used in data engineering roles.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Dynamic artifact data collection from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load operations for artifact metadata, media, and color data
- **SQL Storage**: Relational database design with proper foreign key relationships
- **Analytics Engine**: 20+ predefined SQL queries for artifact analysis
- **Interactive Visualization**: Streamlit dashboard with Plotly charts for real-time analytics

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Full Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Database Setup

The project supports MySQL or TiDB Cloud. Set up your database credentials:

```bash
# Create .env file
touch .env
```

Add your configuration:

```env
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

## Configuration

### API Key Setup

1. Register at [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Generate an API key
3. Store in environment variable or `.env` file

### Database Schema

The application creates three main tables:

```python
# artifactmetadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(300),
    medium VARCHAR(300),
    dated VARCHAR(200),
    url VARCHAR(500)
);

# artifactmedia table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    publiccaption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

# artifactcolors table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

### Start Streamlit Dashboard

```bash
streamlit run app.py
```

Access the dashboard at `http://localhost:8501`

### Basic Usage Flow

1. **API Configuration**: Enter Harvard API key in the dashboard
2. **Data Collection**: Specify number of artifacts to collect
3. **ETL Execution**: Run the ETL pipeline to extract, transform, and load data
4. **Analytics**: Select and execute predefined SQL queries
5. **Visualization**: View results in interactive charts

## Key Code Patterns

### ETL Pipeline Implementation

```python
import requests
import pandas as pd
import mysql.connector
from typing import List, Dict

class HarvardArtETL:
    def __init__(self, api_key: str, db_config: Dict):
        self.api_key = api_key
        self.base_url = "https://api.harvardartmuseums.org/object"
        self.db_config = db_config
    
    def extract_artifacts(self, num_artifacts: int = 100) -> List[Dict]:
        """Extract artifact data from Harvard API with pagination"""
        artifacts = []
        page = 1
        per_page = 100
        
        while len(artifacts) < num_artifacts:
            params = {
                'apikey': self.api_key,
                'size': per_page,
                'page': page
            }
            
            response = requests.get(self.base_url, params=params)
            response.raise_for_status()
            
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            if not data.get('records'):
                break
            
            page += 1
        
        return artifacts[:num_artifacts]
    
    def transform_metadata(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform artifact data into metadata DataFrame"""
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
                'technique': artifact.get('technique', '')[:300],
                'medium': artifact.get('medium', '')[:300],
                'dated': artifact.get('dated', '')[:200],
                'url': artifact.get('url', '')[:500]
            })
        
        return pd.DataFrame(metadata)
    
    def transform_media(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform media data from nested JSON"""
        media_records = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            images = artifact.get('images', [])
            
            for image in images:
                media_records.append({
                    'artifact_id': artifact_id,
                    'iiifbaseuri': image.get('iiifbaseuri', '')[:500],
                    'baseimageurl': image.get('baseimageurl', '')[:500],
                    'publiccaption': image.get('publiccaption', '')
                })
        
        return pd.DataFrame(media_records)
    
    def transform_colors(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform color data from nested JSON"""
        color_records = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            colors = artifact.get('colors', [])
            
            for color in colors:
                color_records.append({
                    'artifact_id': artifact_id,
                    'color': color.get('color', '')[:50],
                    'spectrum': color.get('spectrum', '')[:50],
                    'hue': color.get('hue', '')[:50],
                    'percent': color.get('percent', 0.0)
                })
        
        return pd.DataFrame(color_records)
    
    def load_to_database(self, df: pd.DataFrame, table_name: str):
        """Load DataFrame to MySQL database with batch inserts"""
        conn = mysql.connector.connect(**self.db_config)
        cursor = conn.cursor()
        
        # Prepare column names and placeholders
        columns = ', '.join(df.columns)
        placeholders = ', '.join(['%s'] * len(df.columns))
        
        insert_query = f"""
            INSERT INTO {table_name} ({columns})
            VALUES ({placeholders})
            ON DUPLICATE KEY UPDATE id=id
        """
        
        # Batch insert
        data_tuples = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data_tuples)
        
        conn.commit()
        cursor.close()
        conn.close()
    
    def run_pipeline(self, num_artifacts: int = 100):
        """Execute full ETL pipeline"""
        print(f"Extracting {num_artifacts} artifacts...")
        artifacts = self.extract_artifacts(num_artifacts)
        
        print("Transforming metadata...")
        metadata_df = self.transform_metadata(artifacts)
        
        print("Transforming media...")
        media_df = self.transform_media(artifacts)
        
        print("Transforming colors...")
        colors_df = self.transform_colors(artifacts)
        
        print("Loading to database...")
        self.load_to_database(metadata_df, 'artifactmetadata')
        self.load_to_database(media_df, 'artifactmedia')
        self.load_to_database(colors_df, 'artifactcolors')
        
        print("ETL pipeline completed successfully!")
```

### SQL Analytics Queries

```python
class ArtifactAnalytics:
    def __init__(self, db_config: Dict):
        self.db_config = db_config
    
    def execute_query(self, query: str) -> pd.DataFrame:
        """Execute SQL query and return results as DataFrame"""
        conn = mysql.connector.connect(**self.db_config)
        df = pd.read_sql(query, conn)
        conn.close()
        return df
    
    # Sample analytical queries
    QUERIES = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 15
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'media_availability': """
            SELECT 
                CASE 
                    WHEN baseimageurl IS NOT NULL THEN 'Has Image'
                    ELSE 'No Image'
                END as image_status,
                COUNT(*) as count
            FROM artifactmedia
            GROUP BY image_status
        """,
        
        'top_colors': """
            SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
            FROM artifactcolors
            WHERE color IS NOT NULL
            GROUP BY color
            ORDER BY usage_count DESC
            LIMIT 20
        """,
        
        'artifacts_by_department': """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """,
        
        'classification_distribution': """
            SELECT classification, COUNT(*) as count
            FROM artifactmetadata
            WHERE classification IS NOT NULL AND classification != ''
            GROUP BY classification
            ORDER BY count DESC
            LIMIT 15
        """
    }
```

### Streamlit Dashboard Integration

```python
import streamlit as st
import plotly.express as px
import os
from dotenv import load_dotenv

load_dotenv()

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    api_key = st.sidebar.text_input(
        "Harvard API Key",
        value=os.getenv('HARVARD_API_KEY', ''),
        type='password'
    )
    
    db_config = {
        'host': os.getenv('DB_HOST'),
        'port': int(os.getenv('DB_PORT', 3306)),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    # ETL Section
    st.header("📥 Data Collection")
    
    num_artifacts = st.number_input(
        "Number of artifacts to collect",
        min_value=10,
        max_value=1000,
        value=100,
        step=10
    )
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            etl = HarvardArtETL(api_key, db_config)
            etl.run_pipeline(num_artifacts)
            st.success(f"Successfully loaded {num_artifacts} artifacts!")
    
    # Analytics Section
    st.header("📊 Analytics Dashboard")
    
    analytics = ArtifactAnalytics(db_config)
    
    query_name = st.selectbox(
        "Select Analysis",
        list(analytics.QUERIES.keys())
    )
    
    if st.button("Execute Query"):
        query = analytics.QUERIES[query_name]
        
        with st.spinner("Executing query..."):
            df = analytics.execute_query(query)
            
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df,
                    x=df.columns[0],
                    y=df.columns[1],
                    title=query_name.replace('_', ' ').title()
                )
                st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

### Rate Limiting and Error Handling

```python
import time
from typing import Optional

class RateLimitedAPIClient:
    def __init__(self, api_key: str, requests_per_second: float = 2):
        self.api_key = api_key
        self.min_interval = 1.0 / requests_per_second
        self.last_request_time = 0
    
    def make_request(self, url: str, params: Dict) -> Optional[Dict]:
        """Make API request with rate limiting and retry logic"""
        # Rate limiting
        elapsed = time.time() - self.last_request_time
        if elapsed < self.min_interval:
            time.sleep(self.min_interval - elapsed)
        
        max_retries = 3
        for attempt in range(max_retries):
            try:
                params['apikey'] = self.api_key
                response = requests.get(url, params=params, timeout=30)
                self.last_request_time = time.time()
                
                if response.status_code == 429:  # Rate limit exceeded
                    retry_after = int(response.headers.get('Retry-After', 60))
                    print(f"Rate limit hit, waiting {retry_after} seconds...")
                    time.sleep(retry_after)
                    continue
                
                response.raise_for_status()
                return response.json()
                
            except requests.exceptions.RequestException as e:
                print(f"Request failed (attempt {attempt + 1}/{max_retries}): {e}")
                if attempt < max_retries - 1:
                    time.sleep(2 ** attempt)  # Exponential backoff
                else:
                    raise
        
        return None
```

## Common Patterns

### Pattern 1: Incremental ETL Updates

```python
def incremental_etl(self, last_update_id: int):
    """Load only new artifacts since last ETL run"""
    query = f"""
        SELECT MAX(id) as max_id FROM artifactmetadata
    """
    
    conn = mysql.connector.connect(**self.db_config)
    cursor = conn.cursor()
    cursor.execute(query)
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    
    current_max_id = result[0] if result[0] else 0
    
    # Fetch only new artifacts
    params = {
        'apikey': self.api_key,
        'sort': 'id',
        'sortorder': 'asc',
        'size': 100
    }
    
    if current_max_id:
        params['q'] = f'id:>{current_max_id}'
    
    # Continue with extraction...
```

### Pattern 2: Data Quality Validation

```python
def validate_data_quality(df: pd.DataFrame) -> Dict:
    """Run data quality checks on extracted data"""
    return {
        'total_records': len(df),
        'null_counts': df.isnull().sum().to_dict(),
        'duplicate_ids': df.duplicated(subset=['id']).sum(),
        'completeness': (1 - df.isnull().sum() / len(df)).to_dict()
    }
```

### Pattern 3: Parallel Processing for Large Datasets

```python
from concurrent.futures import ThreadPoolExecutor
import math

def parallel_extract(self, num_artifacts: int, workers: int = 4):
    """Extract artifacts using parallel workers"""
    artifacts_per_worker = math.ceil(num_artifacts / workers)
    
    def fetch_page_range(start_page: int, num_pages: int):
        artifacts = []
        for page in range(start_page, start_page + num_pages):
            params = {
                'apikey': self.api_key,
                'size': 100,
                'page': page
            }
            response = requests.get(self.base_url, params=params)
            artifacts.extend(response.json().get('records', []))
        return artifacts
    
    with ThreadPoolExecutor(max_workers=workers) as executor:
        futures = []
        pages_per_worker = math.ceil(artifacts_per_worker / 100)
        
        for i in range(workers):
            start_page = i * pages_per_worker + 1
            future = executor.submit(fetch_page_range, start_page, pages_per_worker)
            futures.append(future)
        
        all_artifacts = []
        for future in futures:
            all_artifacts.extend(future.result())
    
    return all_artifacts[:num_artifacts]
```

## Troubleshooting

### API Connection Issues

```python
# Test API connectivity
def test_api_connection(api_key: str) -> bool:
    """Verify Harvard API key is valid"""
    test_url = "https://api.harvardartmuseums.org/object"
    params = {'apikey': api_key, 'size': 1}
    
    try:
        response = requests.get(test_url, params=params, timeout=10)
        return response.status_code == 200
    except requests.exceptions.RequestException as e:
        print(f"API connection failed: {e}")
        return False
```

### Database Connection Issues

```python
def test_database_connection(db_config: Dict) -> bool:
    """Verify database connectivity"""
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        return True
    except mysql.connector.Error as e:
        print(f"Database connection failed: {e}")
        return False
```

### Common Error: Empty Response

```python
# Handle empty API responses
artifacts = self.extract_artifacts(num_artifacts)
if not artifacts:
    raise ValueError("No artifacts returned from API. Check API key and connectivity.")
```

### Common Error: Database Insert Failures

```python
# Use transactions for atomic inserts
try:
    conn = mysql.connector.connect(**self.db_config)
    cursor = conn.cursor()
    cursor.execute("START TRANSACTION")
    
    # Perform inserts
    cursor.executemany(insert_query, data_tuples)
    
    conn.commit()
except mysql.connector.Error as e:
    conn.rollback()
    print(f"Database insert failed: {e}")
    raise
finally:
    cursor.close()
    conn.close()
```

### Memory Management for Large Datasets

```python
# Process in chunks to avoid memory issues
def chunked_load(df: pd.DataFrame, table_name: str, chunk_size: int = 1000):
    """Load large DataFrame in chunks"""
    total_chunks = math.ceil(len(df) / chunk_size)
    
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i + chunk_size]
        self.load_to_database(chunk, table_name)
        print(f"Loaded chunk {i // chunk_size + 1}/{total_chunks}")
```

## Advanced Usage

### Custom Query Builder

```python
class QueryBuilder:
    """Build dynamic SQL queries for artifact analysis"""
    
    @staticmethod
    def filter_by_period(period: str) -> str:
        return f"""
            SELECT * FROM artifactmetadata
            WHERE period LIKE '%{period}%'
        """
    
    @staticmethod
    def artifacts_with_colors(min_colors: int = 5) -> str:
        return f"""
            SELECT 
                am.id,
                am.title,
                COUNT(ac.color) as color_count
            FROM artifactmetadata am
            JOIN artifactcolors ac ON am.id = ac.artifact_id
            GROUP BY am.id, am.title
            HAVING color_count >= {min_colors}
            ORDER BY color_count DESC
        """
    
    @staticmethod
    def artifacts_with_images() -> str:
        return """
            SELECT 
                am.id,
                am.title,
                COUNT(DISTINCT ami.baseimageurl) as image_count
            FROM artifactmetadata am
            JOIN artifactmedia ami ON am.id = ami.artifact_id
            WHERE ami.baseimageurl IS NOT NULL
            GROUP BY am.id, am.title
            HAVING image_count > 0
        """
```

This skill provides comprehensive guidance for building ETL pipelines and analytics dashboards using the Harvard Art Museums API, following industry-standard data engineering practices.
