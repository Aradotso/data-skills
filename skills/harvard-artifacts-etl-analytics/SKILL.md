---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - create an ETL pipeline for Harvard Art Museums data
  - build a data engineering app with the Harvard API
  - set up artifact collection analytics with SQL
  - visualize museum data with Streamlit and Plotly
  - extract and transform Harvard Art Museums API data
  - build a SQL analytics dashboard for artifact data
  - create a museum data pipeline with Python
  - analyze Harvard Art Museums collection with SQL queries
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application for the Harvard Art Museums API. It demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What It Does

The Harvard Artifacts Collection application:
- Fetches artifact data from the Harvard Art Museums API with pagination
- Extracts, transforms, and loads data into relational SQL databases
- Stores metadata, media details, and color information in normalized tables
- Executes analytical SQL queries for insights
- Visualizes results using interactive Plotly charts in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
- `streamlit`
- `pandas`
- `requests`
- `mysql-connector-python` or `pymysql`
- `plotly`
- `python-dotenv`

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from [Harvard Art Museums API](https://harvardartmuseums.org/collections/api).

Create a `.env` file or configure in your application:

```bash
HARVARD_API_KEY=your_api_key_here
```

### Database Configuration

Set up MySQL or TiDB Cloud connection:

```python
import os
from dotenv import load_dotenv
import mysql.connector

load_dotenv()

db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

conn = mysql.connector.connect(**db_config)
```

## Database Schema

The project uses three main tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    provenance TEXT,
    description TEXT,
    url VARCHAR(500)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    caption TEXT,
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
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

## API Integration

### Fetching Artifacts with Pagination

```python
import requests
import os

class HarvardAPIClient:
    BASE_URL = "https://api.harvardartmuseums.org/object"
    
    def __init__(self, api_key=None):
        self.api_key = api_key or os.getenv('HARVARD_API_KEY')
        
    def fetch_artifacts(self, num_pages=5, page_size=100):
        """Fetch artifacts with pagination"""
        all_artifacts = []
        
        for page in range(1, num_pages + 1):
            params = {
                'apikey': self.api_key,
                'size': page_size,
                'page': page
            }
            
            response = requests.get(self.BASE_URL, params=params)
            
            if response.status_code == 200:
                data = response.json()
                artifacts = data.get('records', [])
                all_artifacts.extend(artifacts)
                print(f"Fetched page {page}: {len(artifacts)} artifacts")
            else:
                print(f"Error on page {page}: {response.status_code}")
                break
                
        return all_artifacts
```

## ETL Pipeline

### Complete ETL Implementation

```python
import pandas as pd
import mysql.connector

class ArtifactETL:
    def __init__(self, db_config):
        self.db_config = db_config
        self.conn = None
        
    def connect(self):
        """Establish database connection"""
        self.conn = mysql.connector.connect(**self.db_config)
        return self.conn.cursor()
    
    def extract(self, artifacts):
        """Extract and transform artifact data"""
        metadata_list = []
        media_list = []
        colors_list = []
        
        for artifact in artifacts:
            # Extract metadata
            metadata = {
                'id': artifact.get('id'),
                'title': artifact.get('title'),
                'culture': artifact.get('culture'),
                'century': artifact.get('century'),
                'classification': artifact.get('classification'),
                'department': artifact.get('department'),
                'division': artifact.get('division'),
                'dated': artifact.get('dated'),
                'period': artifact.get('period'),
                'technique': artifact.get('technique'),
                'medium': artifact.get('medium'),
                'dimensions': artifact.get('dimensions'),
                'creditline': artifact.get('creditline'),
                'provenance': artifact.get('provenance'),
                'description': artifact.get('description'),
                'url': artifact.get('url')
            }
            metadata_list.append(metadata)
            
            # Extract media
            images = artifact.get('images', [])
            for img in images:
                media = {
                    'artifact_id': artifact.get('id'),
                    'image_url': img.get('baseimageurl'),
                    'caption': img.get('caption'),
                    'height': img.get('height'),
                    'width': img.get('width')
                }
                media_list.append(media)
            
            # Extract colors
            colors = artifact.get('colors', [])
            for color in colors:
                color_data = {
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                }
                colors_list.append(color_data)
        
        return (
            pd.DataFrame(metadata_list),
            pd.DataFrame(media_list),
            pd.DataFrame(colors_list)
        )
    
    def load(self, metadata_df, media_df, colors_df):
        """Load data into SQL database"""
        cursor = self.connect()
        
        # Load metadata
        for _, row in metadata_df.iterrows():
            insert_query = """
                INSERT INTO artifactmetadata 
                (id, title, culture, century, classification, department, 
                 division, dated, period, technique, medium, dimensions, 
                 creditline, provenance, description, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """
            cursor.execute(insert_query, tuple(row))
        
        # Load media
        for _, row in media_df.iterrows():
            insert_query = """
                INSERT INTO artifactmedia 
                (artifact_id, image_url, caption, height, width)
                VALUES (%s, %s, %s, %s, %s)
            """
            cursor.execute(insert_query, tuple(row))
        
        # Load colors
        for _, row in colors_df.iterrows():
            insert_query = """
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """
            cursor.execute(insert_query, tuple(row))
        
        self.conn.commit()
        cursor.close()
        print("Data loaded successfully!")
```

## Analytical SQL Queries

### Common Analytics Patterns

```python
# Top 10 artifacts by color diversity
query_color_diversity = """
SELECT 
    am.title,
    am.culture,
    COUNT(DISTINCT ac.color) as color_count
FROM artifactmetadata am
JOIN artifactcolors ac ON am.id = ac.artifact_id
GROUP BY am.id, am.title, am.culture
ORDER BY color_count DESC
LIMIT 10
"""

# Artifacts by century
query_century_distribution = """
SELECT 
    century,
    COUNT(*) as artifact_count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY artifact_count DESC
"""

# Media availability analysis
query_media_stats = """
SELECT 
    d.department,
    COUNT(DISTINCT m.artifact_id) as artifacts_with_images,
    COUNT(DISTINCT d.id) as total_artifacts,
    ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / COUNT(DISTINCT d.id), 2) as image_percentage
FROM artifactmetadata d
LEFT JOIN artifactmedia m ON d.id = m.artifact_id
GROUP BY d.department
ORDER BY image_percentage DESC
"""

# Most common colors across collection
query_color_frequency = """
SELECT 
    color,
    COUNT(*) as occurrence_count,
    AVG(percent) as avg_percent
FROM artifactcolors
GROUP BY color
ORDER BY occurrence_count DESC
LIMIT 20
"""

# Culture and classification breakdown
query_culture_classification = """
SELECT 
    culture,
    classification,
    COUNT(*) as count
FROM artifactmetadata
WHERE culture IS NOT NULL AND classification IS NOT NULL
GROUP BY culture, classification
ORDER BY count DESC
LIMIT 15
"""
```

## Streamlit Dashboard

### Building the Analytics App

```python
import streamlit as st
import pandas as pd
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Collection Analytics")

# Database connection
@st.cache_resource
def get_connection():
    return mysql.connector.connect(**db_config)

# Query executor
def run_query(query):
    conn = get_connection()
    df = pd.read_sql(query, conn)
    return df

# Sidebar with query selection
st.sidebar.header("Select Analysis")
analysis_type = st.sidebar.selectbox(
    "Choose an analytical query",
    [
        "Century Distribution",
        "Color Diversity",
        "Department Overview",
        "Media Availability",
        "Culture Classification"
    ]
)

# Query mapping
queries = {
    "Century Distribution": query_century_distribution,
    "Color Diversity": query_color_diversity,
    "Media Availability": query_media_stats,
    # ... add other queries
}

# Execute and display
if analysis_type in queries:
    st.subheader(f"Analysis: {analysis_type}")
    
    with st.spinner("Running query..."):
        result_df = run_query(queries[analysis_type])
    
    # Display table
    st.dataframe(result_df, use_container_width=True)
    
    # Auto-generate visualization
    if len(result_df.columns) >= 2:
        fig = px.bar(
            result_df.head(20),
            x=result_df.columns[0],
            y=result_df.columns[1],
            title=f"{analysis_type} - Top 20"
        )
        st.plotly_chart(fig, use_container_width=True)
```

## Running the Complete Pipeline

```python
# main.py - Complete workflow
import os
from dotenv import load_dotenv

load_dotenv()

def main():
    # Step 1: Fetch data from API
    client = HarvardAPIClient()
    artifacts = client.fetch_artifacts(num_pages=10, page_size=100)
    print(f"Total artifacts fetched: {len(artifacts)}")
    
    # Step 2: ETL Process
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    etl = ArtifactETL(db_config)
    metadata_df, media_df, colors_df = etl.extract(artifacts)
    
    print(f"Metadata records: {len(metadata_df)}")
    print(f"Media records: {len(media_df)}")
    print(f"Color records: {len(colors_df)}")
    
    etl.load(metadata_df, media_df, colors_df)
    
    # Step 3: Launch Streamlit dashboard
    # Run separately: streamlit run app.py

if __name__ == "__main__":
    main()
```

## Troubleshooting

**API Rate Limiting:**
```python
import time

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        response = requests.get(url, params=params)
        if response.status_code == 429:  # Rate limited
            wait_time = 2 ** attempt
            print(f"Rate limited. Waiting {wait_time}s...")
            time.sleep(wait_time)
            continue
        return response
    raise Exception("Max retries exceeded")
```

**Database Connection Errors:**
```python
# Use connection pooling
from mysql.connector import pooling

connection_pool = pooling.MySQLConnectionPool(
    pool_name="artifact_pool",
    pool_size=5,
    **db_config
)

conn = connection_pool.get_connection()
```

**Memory Issues with Large Datasets:**
```python
# Batch processing
def load_in_batches(df, table_name, batch_size=1000):
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        batch.to_sql(table_name, conn, if_exists='append', index=False)
        print(f"Loaded batch {i//batch_size + 1}")
```

**Missing Data Handling:**
```python
# Clean and validate before insertion
def clean_metadata(row):
    return {k: (v if v else None) for k, v in row.items()}

metadata_list = [clean_metadata(m) for m in metadata_list]
```

This skill enables AI agents to help developers build complete ETL pipelines, SQL analytics, and interactive dashboards using the Harvard Art Museums API.
