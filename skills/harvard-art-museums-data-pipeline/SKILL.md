---
name: harvard-art-museums-data-pipeline
description: Build end-to-end data pipelines using Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline with harvard art museums api
  - create etl pipeline for museum artifacts
  - set up harvard art museums data analytics app
  - implement artifact collection data engineering
  - build streamlit dashboard for museum data
  - create sql analytics for harvard art api
  - design museum artifact data warehouse
  - fetch and analyze harvard art museums data
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering solution for extracting, transforming, and analyzing Harvard Art Museums collection data. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards.

## What It Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into relational database tables
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper schema design
- **Analytics**: Runs 20+ predefined SQL queries for insights
- **Visualization**: Creates interactive Plotly charts in Streamlit dashboards

## Architecture

```
Harvard API → Extract → Transform → Load (SQL) → Query → Visualize (Streamlit)
```

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

Create a `.env` file:

```env
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit: https://www.harvardartmuseums.org/collections/api
2. Request an API key
3. Add to `.env` file

## Database Schema

### Core Tables

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    medium VARCHAR(500),
    technique VARCHAR(500),
    period VARCHAR(200),
    provenance TEXT,
    creditline TEXT,
    url VARCHAR(500)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    imagepermissionlevel INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
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

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
print(f"Total records: {data['info']['totalrecords']}")
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class HarvardETL:
    def __init__(self, db_config: Dict):
        self.db_config = db_config
        self.conn = None
    
    def connect_db(self):
        """Connect to MySQL database"""
        self.conn = mysql.connector.connect(
            host=self.db_config['host'],
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
                'dated': artifact.get('dated'),
                'medium': artifact.get('medium'),
                'technique': artifact.get('technique'),
                'period': artifact.get('period'),
                'provenance': artifact.get('provenance'),
                'creditline': artifact.get('creditline'),
                'url': artifact.get('url')
            })
        return pd.DataFrame(metadata)
    
    def extract_media(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract media information"""
        media = []
        for artifact in artifacts:
            if artifact.get('images'):
                for img in artifact['images']:
                    media.append({
                        'artifact_id': artifact['id'],
                        'baseimageurl': img.get('baseimageurl'),
                        'primaryimageurl': artifact.get('primaryimageurl'),
                        'imagepermissionlevel': img.get('imagepermissionlevel')
                    })
        return pd.DataFrame(media)
    
    def extract_colors(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract color information"""
        colors = []
        for artifact in artifacts:
            if artifact.get('colors'):
                for color in artifact['colors']:
                    colors.append({
                        'artifact_id': artifact['id'],
                        'color': color.get('color'),
                        'spectrum': color.get('spectrum'),
                        'hue': color.get('hue'),
                        'percent': color.get('percent')
                    })
        return pd.DataFrame(colors)
    
    def load_to_db(self, df: pd.DataFrame, table_name: str):
        """Load DataFrame to database"""
        cursor = self.conn.cursor()
        
        # Create insert query
        cols = ','.join(df.columns)
        placeholders = ','.join(['%s'] * len(df.columns))
        query = f"INSERT IGNORE INTO {table_name} ({cols}) VALUES ({placeholders})"
        
        # Batch insert
        data = [tuple(row) for row in df.values]
        cursor.executemany(query, data)
        self.conn.commit()
        
        print(f"Loaded {cursor.rowcount} rows into {table_name}")

# Usage
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

etl = HarvardETL(db_config)
etl.connect_db()

# Extract and load
artifacts = fetch_artifacts(api_key)['records']
metadata_df = etl.extract_metadata(artifacts)
media_df = etl.extract_media(artifacts)
colors_df = etl.extract_colors(artifacts)

etl.load_to_db(metadata_df, 'artifactmetadata')
etl.load_to_db(media_df, 'artifactmedia')
etl.load_to_db(colors_df, 'artifactcolors')
```

### 3. SQL Analytics Queries

```python
# Sample analytical queries
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
        ORDER BY century
    """,
    
    "media_availability": """
        SELECT 
            CASE WHEN primaryimageurl IS NOT NULL THEN 'Has Image' ELSE 'No Image' END as status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY status
    """,
    
    "top_colors": """
        SELECT color, spectrum, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color, spectrum
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "department_distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "artifacts_with_provenance": """
        SELECT 
            CASE WHEN provenance IS NOT NULL THEN 'Has Provenance' ELSE 'No Provenance' END as status,
            COUNT(*) as count
        FROM artifactmetadata
        GROUP BY status
    """
}

def run_analytics(query_name: str, db_config: Dict) -> pd.DataFrame:
    """Execute analytical query and return results"""
    conn = mysql.connector.connect(**db_config)
    query = ANALYTICS_QUERIES[query_name]
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Data Analytics")
    st.markdown("---")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    db_config = {
        'host': st.sidebar.text_input("DB Host", os.getenv('DB_HOST')),
        'user': st.sidebar.text_input("DB User", os.getenv('DB_USER')),
        'password': st.sidebar.text_input("DB Password", type="password"),
        'database': st.sidebar.text_input("Database", os.getenv('DB_NAME'))
    }
    
    # Query selection
    st.sidebar.header("Analytics")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.sidebar.button("Run Analysis"):
        with st.spinner("Executing query..."):
            df = run_analytics(query_name, db_config)
            
            # Display results
            st.subheader(f"Results: {query_name.replace('_', ' ').title()}")
            st.dataframe(df)
            
            # Visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df, 
                    x=df.columns[0], 
                    y=df.columns[1],
                    title=query_name.replace('_', ' ').title()
                )
                st.plotly_chart(fig, use_container_width=True)
            
            # Download option
            csv = df.to_csv(index=False)
            st.download_button(
                "Download CSV",
                csv,
                f"{query_name}.csv",
                "text/csv"
            )

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Pagination for Large Datasets

```python
def fetch_all_artifacts(api_key, max_pages=10):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page, size=100)
        all_artifacts.extend(data['records'])
        
        # Rate limiting
        import time
        time.sleep(0.5)  # 2 requests per second max
        
        if page >= data['info']['pages']:
            break
    
    return all_artifacts
```

### Incremental Updates

```python
def get_latest_artifact_id(db_config):
    """Get the latest artifact ID in database"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    conn.close()
    return result or 0

def incremental_load(api_key, db_config):
    """Load only new artifacts"""
    latest_id = get_latest_artifact_id(db_config)
    
    # Fetch artifacts with ID > latest_id
    # Implementation depends on API capabilities
    pass
```

## Troubleshooting

### API Rate Limiting

```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except HTTPError as e:
            if e.response.status_code == 429:  # Rate limited
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_db_connection(db_config):
    """Test database connectivity"""
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        conn.close()
        print("✓ Database connection successful")
        return True
    except Exception as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Missing Data Handling

```python
def safe_extract(artifact, key, default=None):
    """Safely extract nested data"""
    try:
        value = artifact.get(key, default)
        return value if value else default
    except Exception:
        return default

# Usage in extraction
metadata.append({
    'id': safe_extract(artifact, 'id'),
    'title': safe_extract(artifact, 'title', 'Untitled'),
    'culture': safe_extract(artifact, 'culture', 'Unknown')
})
```

## Advanced Usage

### Custom Query Builder

```python
class QueryBuilder:
    def __init__(self, db_config):
        self.db_config = db_config
    
    def artifacts_by_filter(self, culture=None, century=None, department=None):
        """Build dynamic query with filters"""
        query = "SELECT * FROM artifactmetadata WHERE 1=1"
        params = []
        
        if culture:
            query += " AND culture = %s"
            params.append(culture)
        if century:
            query += " AND century = %s"
            params.append(century)
        if department:
            query += " AND department = %s"
            params.append(department)
        
        conn = mysql.connector.connect(**self.db_config)
        df = pd.read_sql(query, conn, params=params)
        conn.close()
        return df
```

This skill enables AI agents to help developers build complete data engineering pipelines using the Harvard Art Museums API, from extraction to visualization.
