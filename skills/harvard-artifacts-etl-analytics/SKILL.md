---
name: harvard-artifacts-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - analyze Harvard Art Museums collection
  - create a data engineering app with Streamlit
  - set up SQL analytics for art artifacts
  - extract and transform API data to database
  - visualize museum collection analytics
  - implement batch data ingestion pipeline
  - query and analyze artifact metadata
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates real-world ETL patterns, SQL database design, analytical queries, and interactive Streamlit dashboards for artifact collection analytics.

## What This Project Does

The Harvard Artifacts Collection app provides a complete data engineering workflow:

1. **Extract** - Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
2. **Transform** - Processes nested JSON into normalized relational tables
3. **Load** - Batch inserts into MySQL/TiDB Cloud with proper foreign key relationships
4. **Analyze** - Executes 20+ predefined analytical SQL queries
5. **Visualize** - Renders interactive Plotly charts in Streamlit dashboards

**Data Pipeline Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
cp .env.example .env
# Edit .env with your credentials
```

### Required Environment Variables

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Connection
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Dependencies

```python
# requirements.txt
streamlit>=1.28.0
pandas>=2.0.0
requests>=2.31.0
mysql-connector-python>=8.1.0
plotly>=5.17.0
python-dotenv>=1.0.0
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

class HarvardAPIExtractor:
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
    
    def extract_all_artifacts(self, max_pages=10):
        """Extract multiple pages of artifacts"""
        all_artifacts = []
        
        for page in range(1, max_pages + 1):
            data = self.fetch_artifacts(page=page)
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            # Check if more pages available
            if page >= data.get('info', {}).get('pages', 0):
                break
        
        return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
from typing import List, Dict, Tuple

class ArtifactETL:
    def __init__(self, raw_artifacts: List[Dict]):
        self.raw_artifacts = raw_artifacts
    
    def transform(self) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
        """Transform nested JSON into normalized tables"""
        metadata_records = []
        media_records = []
        color_records = []
        
        for artifact in self.raw_artifacts:
            artifact_id = artifact.get('id')
            
            # Transform metadata
            metadata_records.append({
                'artifact_id': artifact_id,
                'title': artifact.get('title', 'Unknown'),
                'culture': artifact.get('culture', 'Unknown'),
                'period': artifact.get('period'),
                'century': artifact.get('century'),
                'classification': artifact.get('classification'),
                'department': artifact.get('department'),
                'technique': artifact.get('technique'),
                'dated': artifact.get('dated'),
                'description': artifact.get('description')
            })
            
            # Transform media (images)
            for media in artifact.get('images', []):
                media_records.append({
                    'artifact_id': artifact_id,
                    'media_id': media.get('imageid'),
                    'base_url': media.get('baseimageurl'),
                    'format': media.get('format'),
                    'width': media.get('width'),
                    'height': media.get('height')
                })
            
            # Transform colors
            for color in artifact.get('colors', []):
                color_records.append({
                    'artifact_id': artifact_id,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
        
        return (
            pd.DataFrame(metadata_records),
            pd.DataFrame(media_records),
            pd.DataFrame(color_records)
        )
```

### 3. Database Schema and Loading

```python
import mysql.connector
from mysql.connector import Error

class DatabaseLoader:
    def __init__(self):
        self.connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT', 3306),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        self.cursor = self.connection.cursor()
    
    def create_tables(self):
        """Create normalized tables with proper relationships"""
        
        # Metadata table
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmetadata (
                artifact_id INT PRIMARY KEY,
                title VARCHAR(500),
                culture VARCHAR(200),
                period VARCHAR(200),
                century VARCHAR(100),
                classification VARCHAR(200),
                department VARCHAR(200),
                technique VARCHAR(300),
                dated VARCHAR(200),
                description TEXT,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        
        # Media table
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmedia (
                id INT AUTO_INCREMENT PRIMARY KEY,
                artifact_id INT,
                media_id INT,
                base_url VARCHAR(500),
                format VARCHAR(50),
                width INT,
                height INT,
                FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
                    ON DELETE CASCADE
            )
        """)
        
        # Colors table
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactcolors (
                id INT AUTO_INCREMENT PRIMARY KEY,
                artifact_id INT,
                color VARCHAR(50),
                spectrum VARCHAR(50),
                hue VARCHAR(50),
                percent FLOAT,
                FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
                    ON DELETE CASCADE
            )
        """)
        
        self.connection.commit()
    
    def batch_insert(self, df: pd.DataFrame, table_name: str):
        """Batch insert with duplicate handling"""
        if df.empty:
            return
        
        columns = ', '.join(df.columns)
        placeholders = ', '.join(['%s'] * len(df.columns))
        
        # Use INSERT IGNORE to skip duplicates
        query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
        
        # Convert DataFrame to list of tuples
        values = [tuple(row) for row in df.values]
        
        self.cursor.executemany(query, values)
        self.connection.commit()
        
        print(f"Inserted {self.cursor.rowcount} rows into {table_name}")
    
    def close(self):
        self.cursor.close()
        self.connection.close()
```

### 4. Analytical Queries

```python
class AnalyticsEngine:
    def __init__(self, db_connection):
        self.connection = db_connection
        self.cursor = db_connection.cursor(dictionary=True)
    
    QUERIES = {
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
            LIMIT 20
        """,
        
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture != 'Unknown'
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 15
        """,
        
        'artifacts_by_department': """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """,
        
        'color_distribution': """
            SELECT color, AVG(percent) as avg_percent, COUNT(*) as count
            FROM artifactcolors
            GROUP BY color
            ORDER BY count DESC
            LIMIT 10
        """,
        
        'media_format_usage': """
            SELECT format, COUNT(*) as count
            FROM artifactmedia
            GROUP BY format
            ORDER BY count DESC
        """,
        
        'artifacts_with_images': """
            SELECT 
                m.artifact_id,
                m.title,
                m.culture,
                COUNT(med.id) as image_count
            FROM artifactmetadata m
            LEFT JOIN artifactmedia med ON m.artifact_id = med.artifact_id
            GROUP BY m.artifact_id, m.title, m.culture
            HAVING image_count > 0
            ORDER BY image_count DESC
            LIMIT 20
        """,
        
        'color_palette_by_culture': """
            SELECT 
                m.culture,
                c.color,
                AVG(c.percent) as avg_percent
            FROM artifactmetadata m
            JOIN artifactcolors c ON m.artifact_id = c.artifact_id
            WHERE m.culture != 'Unknown'
            GROUP BY m.culture, c.color
            HAVING COUNT(*) > 5
            ORDER BY m.culture, avg_percent DESC
        """
    }
    
    def execute_query(self, query_key: str) -> pd.DataFrame:
        """Execute predefined query and return DataFrame"""
        query = self.QUERIES.get(query_key)
        if not query:
            raise ValueError(f"Query '{query_key}' not found")
        
        self.cursor.execute(query)
        results = self.cursor.fetchall()
        return pd.DataFrame(results)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for configuration
    with st.sidebar:
        st.header("Configuration")
        
        # ETL Options
        st.subheader("ETL Pipeline")
        max_pages = st.number_input("Pages to Extract", 1, 50, 5)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Extracting data from API..."):
                extractor = HarvardAPIExtractor()
                artifacts = extractor.extract_all_artifacts(max_pages=max_pages)
                st.success(f"Extracted {len(artifacts)} artifacts")
            
            with st.spinner("Transforming data..."):
                etl = ArtifactETL(artifacts)
                metadata_df, media_df, colors_df = etl.transform()
                st.success("Data transformed successfully")
            
            with st.spinner("Loading to database..."):
                loader = DatabaseLoader()
                loader.create_tables()
                loader.batch_insert(metadata_df, 'artifactmetadata')
                loader.batch_insert(media_df, 'artifactmedia')
                loader.batch_insert(colors_df, 'artifactcolors')
                loader.close()
                st.success("Data loaded to database")
    
    # Main content area
    tab1, tab2, tab3 = st.tabs(["📊 Analytics", "🔍 Query Explorer", "📈 Visualizations"])
    
    with tab1:
        st.header("Key Metrics")
        
        # Connect to database
        db = DatabaseLoader()
        analytics = AnalyticsEngine(db.connection)
        
        # Display metrics
        col1, col2, col3 = st.columns(3)
        
        # Example: Artifacts by Department
        dept_df = analytics.execute_query('artifacts_by_department')
        col1.metric("Total Departments", len(dept_df))
        
        # Display chart
        st.subheader("Artifacts by Department")
        fig = px.bar(dept_df, x='department', y='count', 
                     title="Distribution of Artifacts by Department")
        st.plotly_chart(fig, use_container_width=True)
    
    with tab2:
        st.header("Query Explorer")
        
        query_options = list(AnalyticsEngine.QUERIES.keys())
        selected_query = st.selectbox("Select Analytics Query", query_options)
        
        if st.button("Execute Query"):
            result_df = analytics.execute_query(selected_query)
            st.dataframe(result_df, use_container_width=True)
            
            # Auto-generate visualization
            if len(result_df.columns) >= 2:
                x_col = result_df.columns[0]
                y_col = result_df.columns[1]
                fig = px.bar(result_df, x=x_col, y=y_col)
                st.plotly_chart(fig, use_container_width=True)
    
    with tab3:
        st.header("Custom Visualizations")
        
        # Color distribution
        color_df = analytics.execute_query('color_distribution')
        fig = px.pie(color_df, values='count', names='color',
                     title="Color Distribution Across Artifacts")
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Pattern 1: Incremental ETL

```python
def incremental_load(last_update_date):
    """Load only new/updated artifacts since last run"""
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'updatedafter': last_update_date.isoformat()
    }
    response = requests.get(base_url, params=params)
    return response.json()
```

### Pattern 2: Error Handling with Retries

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(url, params, max_retries=3):
    """Fetch data with exponential backoff"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=30)
            response.raise_for_status()
            return response.json()
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            time.sleep(wait_time)
```

### Pattern 3: Data Quality Checks

```python
def validate_artifact_data(df: pd.DataFrame) -> bool:
    """Validate DataFrame before loading"""
    checks = [
        ('artifact_id', df['artifact_id'].notna().all()),
        ('title', df['title'].notna().all()),
        ('no_duplicates', not df['artifact_id'].duplicated().any())
    ]
    
    for check_name, passed in checks:
        if not passed:
            st.error(f"Data quality check failed: {check_name}")
            return False
    return True
```

## Troubleshooting

### API Rate Limiting

```python
import time

def rate_limited_fetch(extractor, pages):
    """Add delay between API calls"""
    artifacts = []
    for page in range(1, pages + 1):
        data = extractor.fetch_artifacts(page=page)
        artifacts.extend(data.get('records', []))
        time.sleep(1)  # 1 second delay between requests
    return artifacts
```

### Database Connection Issues

```python
from mysql.connector import pooling

# Use connection pooling for better performance
connection_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    database=os.getenv('DB_NAME'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD')
)

def get_connection():
    return connection_pool.get_connection()
```

### Memory Management for Large Datasets

```python
def chunked_batch_insert(df: pd.DataFrame, table_name: str, chunk_size=1000):
    """Insert large DataFrames in chunks"""
    total_chunks = len(df) // chunk_size + 1
    
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i + chunk_size]
        batch_insert(chunk, table_name)
        print(f"Processed chunk {i//chunk_size + 1}/{total_chunks}")
```

### Handling Missing Data

```python
def clean_artifact_data(artifact: Dict) -> Dict:
    """Clean and normalize artifact data"""
    return {
        'artifact_id': artifact.get('id'),
        'title': artifact.get('title') or 'Untitled',
        'culture': artifact.get('culture') or 'Unknown',
        'period': artifact.get('period'),
        'century': artifact.get('century') or 'Unknown',
        'classification': artifact.get('classification') or 'Unclassified',
        'department': artifact.get('department') or 'General',
        'technique': artifact.get('technique'),
        'dated': artifact.get('dated'),
        'description': (artifact.get('description') or '')[:5000]  # Truncate long text
    }
```

## Advanced Usage

### Custom Analytics Query

```python
def add_custom_query(analytics_engine, query_name: str, sql: str):
    """Add custom SQL query to analytics engine"""
    analytics_engine.QUERIES[query_name] = sql
    result = analytics_engine.execute_query(query_name)
    return result

# Example usage
custom_sql = """
    SELECT 
        m.classification,
        COUNT(DISTINCT m.artifact_id) as artifact_count,
        COUNT(med.id) as total_images
    FROM artifactmetadata m
    LEFT JOIN artifactmedia med ON m.artifact_id = med.artifact_id
    GROUP BY m.classification
    HAVING artifact_count > 10
    ORDER BY artifact_count DESC
"""

result = add_custom_query(analytics, 'classification_analysis', custom_sql)
```

### Export Analytics Results

```python
def export_results(df: pd.DataFrame, filename: str, format: str = 'csv'):
    """Export query results to file"""
    if format == 'csv':
        df.to_csv(filename, index=False)
    elif format == 'excel':
        df.to_excel(filename, index=False, engine='openpyxl')
    elif format == 'json':
        df.to_json(filename, orient='records', indent=2)
    
    st.download_button(
        label=f"Download {format.upper()}",
        data=open(filename, 'rb').read(),
        file_name=filename,
        mime=f'application/{format}'
    )
```

This skill provides complete coverage for building, deploying, and extending the Harvard Artifacts ETL and analytics pipeline.
