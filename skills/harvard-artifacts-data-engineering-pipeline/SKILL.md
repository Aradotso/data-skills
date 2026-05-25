---
name: harvard-artifacts-data-engineering-pipeline
description: Build end-to-end data pipelines with Harvard Art Museums API, ETL processing, SQL analytics, and Streamlit visualization
triggers:
  - how do I extract Harvard museum artifacts data
  - build an ETL pipeline for art museum data
  - create analytics dashboard for Harvard Art Museums API
  - how to use Harvard artifacts collection app
  - set up data engineering pipeline with museum data
  - analyze Harvard Art Museums data with SQL
  - visualize museum artifacts with Streamlit
  - configure Harvard museum API data pipeline
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL operations, SQL database design, analytics queries, and interactive Streamlit visualizations for museum artifact data.

## What It Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON data into relational SQL tables
- **Database Design**: Structured schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **SQL Analytics**: 20+ predefined analytical queries for artifact insights
- **Visualization**: Interactive Streamlit dashboards with Plotly charts

Architecture flow: `API → ETL → SQL → Analytics → Visualization`

## Installation

### Prerequisites

```bash
# Python 3.8+
# MySQL or TiDB Cloud account
# Harvard Art Museums API key (get from https://harvardartmuseums.org/collections/api)
```

### Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"

# Run the Streamlit app
streamlit run app.py
```

### Requirements File

```txt
streamlit>=1.28.0
pandas>=2.0.0
requests>=2.31.0
plotly>=5.17.0
mysql-connector-python>=8.1.0
python-dotenv>=1.0.0
```

## Configuration

### Database Connection

```python
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

def get_db_connection():
    """Create database connection using environment variables"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```

### API Configuration

```python
import requests
import os

API_BASE_URL = "https://api.harvardartmuseums.org/object"
API_KEY = os.getenv('HARVARD_API_KEY')

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts with pagination"""
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size
    }
    response = requests.get(API_BASE_URL, params=params)
    response.raise_for_status()
    return response.json()
```

## Database Schema

### Create Tables

```python
def create_tables(connection):
    """Create relational tables for artifact data"""
    cursor = connection.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            dated VARCHAR(200),
            culture VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            division VARCHAR(200),
            technique VARCHAR(500),
            period VARCHAR(200),
            provenance TEXT,
            description TEXT,
            url VARCHAR(500),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_url VARCHAR(1000),
            media_type VARCHAR(100),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_hex VARCHAR(10),
            color_name VARCHAR(100),
            percentage FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()
    cursor.close()
```

## ETL Pipeline Implementation

### Complete ETL Process

```python
import pandas as pd
import requests
import time

class HarvardArtifactETL:
    def __init__(self, api_key, db_connection):
        self.api_key = api_key
        self.db_connection = db_connection
        self.base_url = "https://api.harvardartmuseums.org/object"
    
    def extract(self, num_pages=5, page_size=100):
        """Extract artifacts from API with pagination"""
        all_artifacts = []
        
        for page in range(1, num_pages + 1):
            params = {
                'apikey': self.api_key,
                'page': page,
                'size': page_size
            }
            
            response = requests.get(self.base_url, params=params)
            if response.status_code == 200:
                data = response.json()
                all_artifacts.extend(data.get('records', []))
                time.sleep(0.5)  # Rate limiting
            else:
                print(f"Failed to fetch page {page}: {response.status_code}")
        
        return all_artifacts
    
    def transform(self, artifacts):
        """Transform nested JSON into relational format"""
        metadata_list = []
        media_list = []
        colors_list = []
        
        for artifact in artifacts:
            # Transform metadata
            metadata = {
                'id': artifact.get('id'),
                'title': artifact.get('title', '')[:500],
                'dated': artifact.get('dated', ''),
                'culture': artifact.get('culture', ''),
                'century': artifact.get('century', ''),
                'classification': artifact.get('classification', ''),
                'department': artifact.get('department', ''),
                'division': artifact.get('division', ''),
                'technique': artifact.get('technique', ''),
                'period': artifact.get('period', ''),
                'provenance': artifact.get('provenance', ''),
                'description': artifact.get('description', ''),
                'url': artifact.get('url', '')
            }
            metadata_list.append(metadata)
            
            # Transform media
            for image in artifact.get('images', []):
                media_list.append({
                    'artifact_id': artifact.get('id'),
                    'image_url': image.get('baseimageurl'),
                    'media_type': 'image'
                })
            
            # Transform colors
            for color in artifact.get('colors', []):
                colors_list.append({
                    'artifact_id': artifact.get('id'),
                    'color_hex': color.get('hex'),
                    'color_name': color.get('color'),
                    'percentage': color.get('percent')
                })
        
        return (
            pd.DataFrame(metadata_list),
            pd.DataFrame(media_list),
            pd.DataFrame(colors_list)
        )
    
    def load(self, metadata_df, media_df, colors_df):
        """Load transformed data into SQL database"""
        cursor = self.db_connection.cursor()
        
        # Load metadata
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (id, title, dated, culture, century, classification, department, 
                 division, technique, period, provenance, description, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE
                title=VALUES(title), dated=VALUES(dated)
            """, tuple(row))
        
        # Load media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia (artifact_id, image_url, media_type)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        # Load colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors (artifact_id, color_hex, color_name, percentage)
                VALUES (%s, %s, %s, %s)
            """, tuple(row))
        
        self.db_connection.commit()
        cursor.close()
    
    def run_pipeline(self, num_pages=5):
        """Execute complete ETL pipeline"""
        print("Starting ETL pipeline...")
        
        # Extract
        print(f"Extracting {num_pages} pages of artifacts...")
        artifacts = self.extract(num_pages=num_pages)
        print(f"Extracted {len(artifacts)} artifacts")
        
        # Transform
        print("Transforming data...")
        metadata_df, media_df, colors_df = self.transform(artifacts)
        print(f"Transformed: {len(metadata_df)} metadata, {len(media_df)} media, {len(colors_df)} colors")
        
        # Load
        print("Loading data into database...")
        self.load(metadata_df, media_df, colors_df)
        print("ETL pipeline completed successfully!")
```

## SQL Analytics Queries

### Common Analytics Patterns

```python
def execute_analytics_query(connection, query_name):
    """Execute predefined analytics queries"""
    
    queries = {
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
        
        'top_colors': """
            SELECT color_name, COUNT(*) as frequency, AVG(percentage) as avg_percentage
            FROM artifactcolors
            WHERE color_name IS NOT NULL
            GROUP BY color_name
            ORDER BY frequency DESC
            LIMIT 10
        """,
        
        'media_availability': """
            SELECT 
                COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
                (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
                ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
            FROM artifactmedia m
        """,
        
        'department_distribution': """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """,
        
        'artifacts_with_provenance': """
            SELECT 
                COUNT(CASE WHEN provenance IS NOT NULL AND provenance != '' THEN 1 END) as with_provenance,
                COUNT(*) as total,
                ROUND(COUNT(CASE WHEN provenance IS NOT NULL AND provenance != '' THEN 1 END) * 100.0 / COUNT(*), 2) as percentage
            FROM artifactmetadata
        """
    }
    
    cursor = connection.cursor()
    cursor.execute(queries[query_name])
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    cursor.close()
    
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard

### Interactive Analytics Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """Create Streamlit analytics dashboard"""
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    query_option = st.sidebar.selectbox(
        "Select Analysis",
        [
            "Artifacts by Culture",
            "Artifacts by Century",
            "Top Colors",
            "Media Availability",
            "Department Distribution",
            "Provenance Analysis"
        ]
    )
    
    # Map selections to query names
    query_map = {
        "Artifacts by Culture": "artifacts_by_culture",
        "Artifacts by Century": "artifacts_by_century",
        "Top Colors": "top_colors",
        "Media Availability": "media_availability",
        "Department Distribution": "department_distribution",
        "Provenance Analysis": "artifacts_with_provenance"
    }
    
    # Execute query
    connection = get_db_connection()
    df = execute_analytics_query(connection, query_map[query_option])
    connection.close()
    
    # Display results
    st.subheader(f"Analysis: {query_option}")
    st.dataframe(df)
    
    # Create visualization
    if len(df) > 0 and len(df.columns) >= 2:
        fig = px.bar(
            df,
            x=df.columns[0],
            y=df.columns[1],
            title=query_option,
            labels={df.columns[0]: df.columns[0].replace('_', ' ').title(),
                    df.columns[1]: df.columns[1].replace('_', ' ').title()}
        )
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    create_dashboard()
```

## Complete Usage Example

```python
import os
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

# Initialize database connection
connection = get_db_connection()

# Create tables
create_tables(connection)

# Initialize ETL pipeline
etl = HarvardArtifactETL(
    api_key=os.getenv('HARVARD_API_KEY'),
    db_connection=connection
)

# Run ETL pipeline (5 pages = ~500 artifacts)
etl.run_pipeline(num_pages=5)

# Execute analytics
df_cultures = execute_analytics_query(connection, 'artifacts_by_culture')
print("Top 10 Cultures:")
print(df_cultures.head(10))

# Close connection
connection.close()
```

## Troubleshooting

### API Rate Limiting

```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(url, params, max_retries=3):
    """Fetch data with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            response.raise_for_status()
            return response.json()
        except HTTPError as e:
            if response.status_code == 429:  # Rate limit
                wait_time = (2 ** attempt) * 1
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise e
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        print("✓ Database connection successful")
        return True
    except Exception as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Handling Missing Data

```python
def safe_extract(artifact, key, default=''):
    """Safely extract nested values from artifact JSON"""
    value = artifact.get(key, default)
    return value if value is not None else default

# Use in transform
metadata = {
    'id': artifact.get('id'),
    'title': safe_extract(artifact, 'title', 'Untitled')[:500],
    'culture': safe_extract(artifact, 'culture', 'Unknown'),
    # ... other fields
}
```

## Best Practices

- **Batch Processing**: Load data in batches to optimize database performance
- **Environment Variables**: Always use environment variables for credentials
- **Error Handling**: Implement try-catch blocks for API and database operations
- **Data Validation**: Validate data types before insertion
- **Logging**: Add logging for ETL pipeline monitoring
- **Incremental Loads**: Track processed artifacts to avoid duplicates
