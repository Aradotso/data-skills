---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and visualize Harvard Art Museums API data
  - set up data engineering pipeline with Streamlit
  - analyze museum collection data with SQL
  - build artifact data visualization app
  - create museum data warehouse application
  - process Harvard Art Museums API with Python
---

# Harvard Art Museums ETL Analytics Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering solution for the Harvard Art Museums API. It demonstrates:

- **API Integration**: Fetching paginated artifact data from Harvard Art Museums
- **ETL Pipeline**: Extracting, transforming, and loading nested JSON into relational tables
- **SQL Storage**: MySQL/TiDB Cloud database with normalized schema
- **Analytics**: 20+ predefined analytical queries
- **Visualization**: Interactive Streamlit dashboard with Plotly charts

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup

```bash
# Clone repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install requirements
pip install -r requirements.txt

# Set up environment variables
cat > .env << EOF
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_db_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
EOF
```

### Get Harvard Art Museums API Key

1. Visit: https://www.harvardartmuseums.org/collections/api
2. Request an API key (free)
3. Add to `.env` file as `HARVARD_API_KEY`

## Database Schema

The application uses three core tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    url TEXT,
    total_unique_pageviews INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Media/images table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(50),
    base_url TEXT,
    image_url TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Color analysis table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

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
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts, info = fetch_artifacts(api_key, page=1, size=50)
print(f"Fetched {len(artifacts)} artifacts")
print(f"Total available: {info['totalrecords']}")
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class HarvardETL:
    def __init__(self, db_config):
        self.db_config = db_config
        self.conn = None
        
    def connect_db(self):
        """Establish database connection"""
        self.conn = mysql.connector.connect(**self.db_config)
        return self.conn.cursor()
    
    def extract(self, api_key, num_pages=5):
        """Extract data from API"""
        all_artifacts = []
        
        for page in range(1, num_pages + 1):
            artifacts, _ = fetch_artifacts(api_key, page=page)
            all_artifacts.extend(artifacts)
            print(f"Extracted page {page}")
        
        return all_artifacts
    
    def transform(self, artifacts: List[Dict]) -> tuple:
        """Transform nested JSON to relational format"""
        metadata_records = []
        media_records = []
        color_records = []
        
        for artifact in artifacts:
            # Metadata
            metadata_records.append({
                'id': artifact.get('id'),
                'title': artifact.get('title', 'Unknown')[:500],
                'culture': artifact.get('culture', 'Unknown')[:255],
                'period': artifact.get('period', 'Unknown')[:255],
                'century': artifact.get('century', 'Unknown')[:100],
                'classification': artifact.get('classification', 'Unknown')[:255],
                'department': artifact.get('department', 'Unknown')[:255],
                'dated': artifact.get('dated', 'Unknown')[:255],
                'url': artifact.get('url', ''),
                'total_unique_pageviews': artifact.get('totalpageviews', 0)
            })
            
            # Media
            if 'images' in artifact and artifact['images']:
                for img in artifact['images']:
                    media_records.append({
                        'artifact_id': artifact.get('id'),
                        'media_type': 'image',
                        'base_url': img.get('baseimageurl', ''),
                        'image_url': img.get('iiifbaseuri', '')
                    })
            
            # Colors
            if 'colors' in artifact and artifact['colors']:
                for color in artifact['colors']:
                    color_records.append({
                        'artifact_id': artifact.get('id'),
                        'color_hex': color.get('hex', ''),
                        'color_name': color.get('color', 'Unknown'),
                        'percentage': color.get('percent', 0.0)
                    })
        
        return (
            pd.DataFrame(metadata_records),
            pd.DataFrame(media_records),
            pd.DataFrame(color_records)
        )
    
    def load(self, df_metadata, df_media, df_colors):
        """Load data into MySQL database"""
        cursor = self.connect_db()
        
        # Insert metadata
        for _, row in df_metadata.iterrows():
            sql = """
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, classification, department, dated, url, total_unique_pageviews)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
            """
            cursor.execute(sql, tuple(row))
        
        # Insert media
        for _, row in df_media.iterrows():
            sql = """
            INSERT INTO artifactmedia (artifact_id, media_type, base_url, image_url)
            VALUES (%s, %s, %s, %s)
            """
            cursor.execute(sql, tuple(row))
        
        # Insert colors
        for _, row in df_colors.iterrows():
            sql = """
            INSERT INTO artifactcolors (artifact_id, color_hex, color_name, percentage)
            VALUES (%s, %s, %s, %s)
            """
            cursor.execute(sql, tuple(row))
        
        self.conn.commit()
        cursor.close()
        print(f"Loaded {len(df_metadata)} artifacts to database")

# Usage
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

etl = HarvardETL(db_config)
artifacts = etl.extract(api_key, num_pages=3)
df_meta, df_media, df_colors = etl.transform(artifacts)
etl.load(df_meta, df_media, df_colors)
```

### 3. Streamlit Dashboard

```python
import streamlit as st
import pandas as pd
import plotly.express as px
from dotenv import load_dotenv

load_dotenv()

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Database connection
@st.cache_resource
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

conn = get_db_connection()

# Analytics queries
queries = {
    "Top 10 Cultures": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE culture != 'Unknown'
        GROUP BY culture 
        ORDER BY count DESC 
        LIMIT 10
    """,
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century IS NOT NULL
        GROUP BY century 
        ORDER BY count DESC
    """,
    "Department Distribution": """
        SELECT department, COUNT(*) as count 
        FROM artifactmetadata 
        GROUP BY department 
        ORDER BY count DESC
    """,
    "Most Popular Colors": """
        SELECT color_name, COUNT(*) as frequency, AVG(percentage) as avg_percentage
        FROM artifactcolors 
        GROUP BY color_name 
        ORDER BY frequency DESC 
        LIMIT 10
    """,
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
            COUNT(*) as total_media_files,
            AVG(media_count) as avg_media_per_artifact
        FROM artifactmedia m
        JOIN (
            SELECT artifact_id, COUNT(*) as media_count
            FROM artifactmedia
            GROUP BY artifact_id
        ) sub ON m.artifact_id = sub.artifact_id
    """
}

# Query selector
selected_query = st.selectbox("Select Analysis", list(queries.keys()))

if st.button("Run Analysis"):
    with st.spinner("Executing query..."):
        df_result = pd.read_sql(queries[selected_query], conn)
        
        st.subheader("Query Results")
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) == 2 and len(df_result) > 1:
            col1, col2 = df_result.columns
            fig = px.bar(df_result, x=col1, y=col2, title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

# Sidebar stats
st.sidebar.header("📊 Database Statistics")
total_artifacts = pd.read_sql("SELECT COUNT(*) as count FROM artifactmetadata", conn)
st.sidebar.metric("Total Artifacts", total_artifacts['count'][0])

total_media = pd.read_sql("SELECT COUNT(*) as count FROM artifactmedia", conn)
st.sidebar.metric("Total Media Files", total_media['count'][0])
```

## Common Patterns

### Batch Data Collection

```python
def batch_collect(api_key, total_records=1000, batch_size=100):
    """Collect data in batches to avoid rate limits"""
    import time
    
    num_pages = (total_records // batch_size) + 1
    all_data = []
    
    for page in range(1, num_pages + 1):
        try:
            artifacts, _ = fetch_artifacts(api_key, page, batch_size)
            all_data.extend(artifacts)
            print(f"Batch {page}/{num_pages} complete")
            time.sleep(1)  # Rate limiting
        except Exception as e:
            print(f"Error on page {page}: {e}")
            continue
    
    return all_data
```

### Error Handling in ETL

```python
def safe_transform(artifact, field, default='Unknown', max_length=None):
    """Safely extract and transform field with defaults"""
    value = artifact.get(field, default)
    if max_length and len(str(value)) > max_length:
        value = str(value)[:max_length]
    return value

# Usage in transform
metadata_records.append({
    'title': safe_transform(artifact, 'title', 'Untitled', 500),
    'culture': safe_transform(artifact, 'culture', max_length=255)
})
```

### Incremental Updates

```python
def get_last_artifact_id(conn):
    """Get last processed artifact ID"""
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(api_key, conn):
    """Only load new artifacts"""
    last_id = get_last_artifact_id(conn)
    
    # Fetch only artifacts with ID > last_id
    # (Note: Harvard API doesn't support ID filtering directly,
    # so filter after fetch)
    artifacts = fetch_artifacts(api_key)
    new_artifacts = [a for a in artifacts if a['id'] > last_id]
    
    return new_artifacts
```

## Analytical Queries

### Top Viewed Artifacts

```sql
SELECT title, culture, century, total_unique_pageviews
FROM artifactmetadata
ORDER BY total_unique_pageviews DESC
LIMIT 20;
```

### Color Distribution Analysis

```sql
SELECT 
    am.department,
    ac.color_name,
    COUNT(*) as usage_count,
    AVG(ac.percentage) as avg_percentage
FROM artifactcolors ac
JOIN artifactmetadata am ON ac.artifact_id = am.id
GROUP BY am.department, ac.color_name
ORDER BY usage_count DESC;
```

### Artifacts Without Media

```sql
SELECT am.id, am.title, am.culture, am.century
FROM artifactmetadata am
LEFT JOIN artifactmedia media ON am.id = media.artifact_id
WHERE media.artifact_id IS NULL;
```

## Configuration

### Environment Variables

Create `.env` file:

```bash
# API Configuration
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=gateway01.your-tidb-cloud.prod.aws.tidbcloud.com
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts

# Application Settings
RECORDS_PER_PAGE=100
MAX_PAGES=10
RATE_LIMIT_DELAY=1
```

### Streamlit Configuration

Create `.streamlit/config.toml`:

```toml
[theme]
primaryColor = "#A51C30"
backgroundColor = "#FFFFFF"
secondaryBackgroundColor = "#F0F2F6"
textColor = "#262730"

[server]
maxUploadSize = 200
enableCORS = false
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Run ETL pipeline independently
python etl_pipeline.py

# Access at http://localhost:8501
```

## Troubleshooting

### API Rate Limiting

```python
# Add delay between requests
import time

for page in range(1, num_pages):
    artifacts = fetch_artifacts(api_key, page)
    time.sleep(1)  # 1 second delay
```

### Database Connection Issues

```python
# Test connection
try:
    conn = mysql.connector.connect(**db_config)
    print("Database connected successfully")
except mysql.connector.Error as err:
    print(f"Error: {err}")
    # Check: firewall rules, credentials, host availability
```

### Memory Issues with Large Datasets

```python
# Use chunked processing
def process_in_chunks(artifacts, chunk_size=100):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        df_meta, df_media, df_colors = transform(chunk)
        load(df_meta, df_media, df_colors)
        print(f"Processed chunk {i//chunk_size + 1}")
```

### Missing API Key

```python
api_key = os.getenv('HARVARD_API_KEY')
if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment variables")
```

## Best Practices

1. **Always use environment variables** for sensitive data
2. **Implement retry logic** for API calls
3. **Use ON DUPLICATE KEY UPDATE** for idempotent loads
4. **Index frequently queried columns** (culture, century, department)
5. **Cache Streamlit database connections** with `@st.cache_resource`
6. **Validate data** before inserting into database
7. **Log ETL runs** for debugging and monitoring

## Advanced Usage

### Custom Analytics Query

```python
def run_custom_query(query_name, sql_query):
    """Execute custom analytical query"""
    df = pd.read_sql(sql_query, conn)
    
    # Auto-detect visualization type
    if 'count' in df.columns or 'total' in df.columns:
        fig = px.bar(df, x=df.columns[0], y=df.columns[1])
    else:
        fig = px.scatter(df, x=df.columns[0], y=df.columns[1])
    
    return df, fig
```

### Export Results

```python
# Export query results to CSV
df_result.to_csv('analytics_results.csv', index=False)

# Export to Excel with formatting
with pd.ExcelWriter('artifacts_analysis.xlsx') as writer:
    df_meta.to_excel(writer, sheet_name='Metadata', index=False)
    df_stats.to_excel(writer, sheet_name='Statistics', index=False)
```

This skill enables AI agents to build, configure, and extend Harvard Art Museums ETL pipelines with SQL analytics and interactive visualizations.
