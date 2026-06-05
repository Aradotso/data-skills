---
name: harvard-artifacts-etl-analytics
description: ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - extract data from Harvard Art Museums API
  - create analytics dashboard with Streamlit
  - design SQL schema for artifact collections
  - visualize museum artifact data with Plotly
  - implement paginated API data collection
  - query and analyze art museum datasets
  - build data engineering pipeline for cultural artifacts
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill covers building end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for cultural artifact data.

## What This Project Does

The Harvard Artifacts Collection app:
- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables
- **Loads** data into MySQL/TiDB Cloud databases
- **Analyzes** data using 20+ predefined SQL queries
- **Visualizes** results through interactive Plotly charts in a Streamlit interface

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

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

```python
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

# Create database connection
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Create tables
def create_tables(conn):
    cursor = conn.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            medium VARCHAR(500),
            dimensions VARCHAR(500),
            creditline TEXT,
            accession_number VARCHAR(100),
            INDEX idx_culture (culture),
            INDEX idx_century (century),
            INDEX idx_department (department)
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            media_type VARCHAR(50),
            base_url VARCHAR(500),
            format VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id),
            INDEX idx_artifact_id (artifact_id)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id),
            INDEX idx_artifact_id (artifact_id)
        )
    """)
    
    conn.commit()
    cursor.close()
```

## ETL Pipeline Implementation

### Extract: API Data Collection

```python
import requests
import time
import os

class HarvardAPIExtractor:
    def __init__(self, api_key=None):
        self.api_key = api_key or os.getenv('HARVARD_API_KEY')
        self.base_url = "https://api.harvardartmuseums.org/object"
        
    def extract_artifacts(self, num_pages=5, page_size=100):
        """Extract artifacts with pagination and rate limiting"""
        all_artifacts = []
        
        for page in range(1, num_pages + 1):
            params = {
                'apikey': self.api_key,
                'size': page_size,
                'page': page,
                'hasimage': 1  # Only artifacts with images
            }
            
            try:
                response = requests.get(self.base_url, params=params, timeout=30)
                response.raise_for_status()
                
                data = response.json()
                records = data.get('records', [])
                all_artifacts.extend(records)
                
                print(f"Extracted page {page}: {len(records)} artifacts")
                
                # Rate limiting - be respectful to API
                time.sleep(1)
                
            except requests.exceptions.RequestException as e:
                print(f"Error on page {page}: {e}")
                break
                
        return all_artifacts
```

### Transform: Data Normalization

```python
import pandas as pd

class ArtifactTransformer:
    @staticmethod
    def transform_metadata(artifacts):
        """Transform artifact data into metadata DataFrame"""
        metadata_records = []
        
        for artifact in artifacts:
            record = {
                'id': artifact.get('id'),
                'title': artifact.get('title', '')[:500],
                'culture': artifact.get('culture', '')[:200],
                'period': artifact.get('period', '')[:200],
                'century': artifact.get('century', '')[:100],
                'classification': artifact.get('classification', '')[:200],
                'department': artifact.get('department', '')[:200],
                'dated': artifact.get('dated', '')[:200],
                'medium': artifact.get('medium', '')[:500],
                'dimensions': artifact.get('dimensions', '')[:500],
                'creditline': artifact.get('creditline', ''),
                'accession_number': artifact.get('accessionNumber', '')[:100]
            }
            metadata_records.append(record)
            
        return pd.DataFrame(metadata_records)
    
    @staticmethod
    def transform_media(artifacts):
        """Transform media/image data"""
        media_records = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            images = artifact.get('images', [])
            
            for img in images:
                record = {
                    'artifact_id': artifact_id,
                    'media_type': 'image',
                    'base_url': img.get('baseimageurl', '')[:500],
                    'format': img.get('format', '')[:50]
                }
                media_records.append(record)
                
        return pd.DataFrame(media_records)
    
    @staticmethod
    def transform_colors(artifacts):
        """Transform color data"""
        color_records = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            colors = artifact.get('colors', [])
            
            for color in colors:
                record = {
                    'artifact_id': artifact_id,
                    'color': color.get('color', '')[:50],
                    'spectrum': color.get('spectrum', '')[:50],
                    'percent': color.get('percent', 0.0)
                }
                color_records.append(record)
                
        return pd.DataFrame(color_records)
```

### Load: Batch Insert to SQL

```python
def load_to_sql(df, table_name, conn, batch_size=100):
    """Batch insert DataFrame to SQL table"""
    cursor = conn.cursor()
    
    # Get column names
    columns = df.columns.tolist()
    placeholders = ', '.join(['%s'] * len(columns))
    column_str = ', '.join(columns)
    
    sql = f"INSERT IGNORE INTO {table_name} ({column_str}) VALUES ({placeholders})"
    
    # Convert DataFrame to list of tuples
    records = df.values.tolist()
    
    # Batch insert
    for i in range(0, len(records), batch_size):
        batch = records[i:i + batch_size]
        try:
            cursor.executemany(sql, batch)
            conn.commit()
            print(f"Inserted batch {i//batch_size + 1} into {table_name}")
        except Exception as e:
            print(f"Error inserting batch: {e}")
            conn.rollback()
    
    cursor.close()

# Complete ETL pipeline
def run_etl_pipeline(num_pages=5):
    """Run complete ETL pipeline"""
    # Extract
    extractor = HarvardAPIExtractor()
    artifacts = extractor.extract_artifacts(num_pages=num_pages)
    
    # Transform
    transformer = ArtifactTransformer()
    metadata_df = transformer.transform_metadata(artifacts)
    media_df = transformer.transform_media(artifacts)
    colors_df = transformer.transform_colors(artifacts)
    
    # Load
    conn = get_db_connection()
    create_tables(conn)
    
    load_to_sql(metadata_df, 'artifactmetadata', conn)
    load_to_sql(media_df, 'artifactmedia', conn)
    load_to_sql(colors_df, 'artifactcolors', conn)
    
    conn.close()
    print("ETL pipeline completed successfully!")
```

## Analytical SQL Queries

```python
# Common analytical queries
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Color Distribution": """
        SELECT c.color, COUNT(*) as usage_count,
               ROUND(AVG(c.percent), 2) as avg_percent
        FROM artifactcolors c
        GROUP BY c.color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Media Coverage by Culture": """
        SELECT m.culture, COUNT(DISTINCT am.artifact_id) as artifacts_with_media
        FROM artifactmetadata m
        LEFT JOIN artifactmedia am ON m.id = am.artifact_id
        WHERE m.culture IS NOT NULL AND m.culture != ''
        GROUP BY m.culture
        ORDER BY artifacts_with_media DESC
        LIMIT 15
    """,
    
    "Classification Analysis": """
        SELECT classification, 
               COUNT(*) as total_artifacts,
               COUNT(DISTINCT century) as centuries_covered
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY total_artifacts DESC
        LIMIT 10
    """
}

def execute_query(query_name, conn):
    """Execute analytical query and return results"""
    query = ANALYTICAL_QUERIES.get(query_name)
    if not query:
        return None
    
    df = pd.read_sql(query, conn)
    return df
```

## Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar for ETL controls
    with st.sidebar:
        st.header("⚙️ ETL Pipeline")
        
        num_pages = st.number_input("Number of pages to extract", 
                                     min_value=1, max_value=50, value=5)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Running ETL pipeline..."):
                try:
                    run_etl_pipeline(num_pages=num_pages)
                    st.success("ETL completed successfully!")
                except Exception as e:
                    st.error(f"ETL failed: {e}")
    
    # Main analytics section
    st.header("📊 SQL Analytics")
    
    # Query selector
    query_name = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Execute Query"):
        try:
            conn = get_db_connection()
            df = execute_query(query_name, conn)
            conn.close()
            
            if df is not None and not df.empty:
                # Display table
                st.subheader("Query Results")
                st.dataframe(df)
                
                # Auto-generate visualization
                st.subheader("Visualization")
                
                # Determine best chart based on data
                if len(df.columns) == 2:
                    fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                                title=query_name)
                    st.plotly_chart(fig, use_container_width=True)
                
                # Download option
                csv = df.to_csv(index=False)
                st.download_button("Download Results", csv, 
                                  f"{query_name.replace(' ', '_')}.csv")
            else:
                st.warning("No results found")
                
        except Exception as e:
            st.error(f"Query execution failed: {e}")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def incremental_load(last_id=0):
    """Load only new artifacts since last ID"""
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'size': 100,
        'sort': 'id',
        'sortorder': 'asc',
        'after': last_id
    }
    
    response = requests.get("https://api.harvardartmuseums.org/object", 
                           params=params)
    return response.json().get('records', [])
```

### Pattern 2: Error Handling with Retry Logic

```python
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session_with_retries():
    """Create session with automatic retries"""
    session = requests.Session()
    retry = Retry(
        total=3,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504]
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('https://', adapter)
    return session
```

### Pattern 3: Data Quality Checks

```python
def validate_artifact_data(df):
    """Validate artifact data before loading"""
    checks = {
        'null_ids': df['id'].isnull().sum(),
        'duplicate_ids': df['id'].duplicated().sum(),
        'missing_titles': df['title'].isnull().sum(),
        'empty_cultures': (df['culture'] == '').sum()
    }
    
    print("Data Quality Report:")
    for check, count in checks.items():
        print(f"  {check}: {count}")
    
    return checks
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff
def fetch_with_backoff(url, params, max_retries=5):
    for i in range(max_retries):
        response = requests.get(url, params=params)
        if response.status_code == 429:
            wait_time = 2 ** i
            print(f"Rate limited. Waiting {wait_time}s...")
            time.sleep(wait_time)
        else:
            return response
    raise Exception("Max retries exceeded")
```

### Database Connection Issues
```python
def get_db_connection_with_retry(max_attempts=3):
    """Get database connection with retries"""
    for attempt in range(max_attempts):
        try:
            conn = mysql.connector.connect(
                host=os.getenv('DB_HOST'),
                port=int(os.getenv('DB_PORT', 3306)),
                user=os.getenv('DB_USER'),
                password=os.getenv('DB_PASSWORD'),
                database=os.getenv('DB_NAME'),
                connect_timeout=10
            )
            return conn
        except mysql.connector.Error as e:
            if attempt < max_attempts - 1:
                time.sleep(2 ** attempt)
            else:
                raise e
```

### Memory Management for Large Datasets
```python
def process_in_chunks(artifacts, chunk_size=1000):
    """Process large datasets in chunks"""
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        
        # Transform chunk
        metadata_df = ArtifactTransformer.transform_metadata(chunk)
        
        # Load chunk
        conn = get_db_connection()
        load_to_sql(metadata_df, 'artifactmetadata', conn)
        conn.close()
        
        print(f"Processed chunk {i//chunk_size + 1}")
```

### Handling Missing or Malformed Data
```python
def safe_get(data, key, default='', max_length=None):
    """Safely extract value with default and length limit"""
    value = data.get(key, default)
    if value is None:
        value = default
    value = str(value)
    if max_length:
        value = value[:max_length]
    return value
```
