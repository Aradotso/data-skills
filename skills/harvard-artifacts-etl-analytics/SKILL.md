---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit and SQL
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums API
  - create a data analytics dashboard with artifact data
  - extract and transform Harvard museum data into SQL
  - build a Streamlit app for art museum analytics
  - query Harvard Art Museums API and visualize results
  - set up ETL for cultural heritage data
  - analyze artifact collections with SQL and Python
  - create interactive museum data visualizations
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering pipelines using the Harvard Art Museums API. It covers API integration, ETL workflows, SQL database design, analytics queries, and interactive Streamlit dashboards with Plotly visualizations.

## What This Project Does

The Harvard Artifacts Collection application demonstrates a production-grade data pipeline:

1. **Extract**: Fetch artifact data from Harvard Art Museums API with pagination
2. **Transform**: Parse nested JSON into relational database schema
3. **Load**: Batch insert into MySQL/TiDB Cloud with proper relationships
4. **Analyze**: Execute analytical SQL queries on structured data
5. **Visualize**: Display results in interactive Streamlit dashboards

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Run the Streamlit app
streamlit run app.py
```

### Required Dependencies

```txt
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
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Register at: https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dOd0Z8MpY2rvQ/viewform
2. Receive key via email
3. Add to `.env` file

## Database Schema

The application uses three core tables:

```sql
-- Artifact metadata
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
    creditline TEXT
);

-- Artifact media
CREATE TABLE artifactmedia (
    artifact_id INT,
    baseimageurl VARCHAR(1000),
    primaryimageurl VARCHAR(1000),
    has_image BOOLEAN,
    total_images INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors
CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Integration

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

class HarvardAPIClient:
    def __init__(self):
        self.api_key = os.getenv('HARVARD_API_KEY')
        self.base_url = "https://api.harvardartmuseums.org"
    
    def fetch_artifacts(self, page=1, size=100):
        """Fetch artifacts with pagination support"""
        url = f"{self.base_url}/object"
        params = {
            'apikey': self.api_key,
            'page': page,
            'size': size
        }
        
        response = requests.get(url, params=params)
        response.raise_for_status()
        return response.json()
    
    def fetch_all_artifacts(self, max_records=1000):
        """Fetch multiple pages of artifacts"""
        all_artifacts = []
        page = 1
        size = 100
        
        while len(all_artifacts) < max_records:
            data = self.fetch_artifacts(page=page, size=size)
            records = data.get('records', [])
            
            if not records:
                break
                
            all_artifacts.extend(records)
            page += 1
            
        return all_artifacts[:max_records]
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class ArtifactETL:
    def __init__(self, db_config):
        self.db_config = db_config
        
    def extract(self, api_client: HarvardAPIClient, max_records=1000):
        """Extract data from API"""
        return api_client.fetch_all_artifacts(max_records)
    
    def transform(self, raw_data: List[Dict]) -> tuple:
        """Transform raw JSON into structured DataFrames"""
        metadata_records = []
        media_records = []
        color_records = []
        
        for artifact in raw_data:
            # Extract metadata
            metadata_records.append({
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
                'creditline': artifact.get('creditline')
            })
            
            # Extract media information
            media_records.append({
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl'),
                'primaryimageurl': artifact.get('primaryimageurl'),
                'has_image': 1 if artifact.get('primaryimageurl') else 0,
                'total_images': len(artifact.get('images', []))
            })
            
            # Extract color data
            colors = artifact.get('colors', [])
            for color in colors:
                color_records.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'color_hex': color.get('hex'),
                    'color_percent': color.get('percent')
                })
        
        return (
            pd.DataFrame(metadata_records),
            pd.DataFrame(media_records),
            pd.DataFrame(color_records)
        )
    
    def load(self, metadata_df, media_df, colors_df):
        """Load data into MySQL database"""
        conn = mysql.connector.connect(**self.db_config)
        cursor = conn.cursor()
        
        # Load metadata
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (id, title, culture, century, classification, department, 
                 division, dated, period, technique, medium, dimensions, creditline)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """, tuple(row))
        
        # Load media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (artifact_id, baseimageurl, primaryimageurl, has_image, total_images)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
        
        # Load colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (artifact_id, color, color_hex, color_percent)
                VALUES (%s, %s, %s, %s)
            """, tuple(row))
        
        conn.commit()
        cursor.close()
        conn.close()
```

### 3. Analytics Queries

```python
class ArtifactAnalytics:
    def __init__(self, db_config):
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
            WHERE culture IS NOT NULL 
            GROUP BY culture 
            ORDER BY count DESC 
            LIMIT 10
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE century IS NOT NULL 
            GROUP BY century 
            ORDER BY count DESC
        """,
        
        'image_availability': """
            SELECT has_image, COUNT(*) as count 
            FROM artifactmedia 
            GROUP BY has_image
        """,
        
        'top_colors': """
            SELECT color, COUNT(*) as frequency 
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY frequency DESC 
            LIMIT 10
        """,
        
        'department_distribution': """
            SELECT department, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE department IS NOT NULL 
            GROUP BY department 
            ORDER BY count DESC
        """
    }
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Database configuration from environment
    db_config = {
        'host': os.getenv('DB_HOST'),
        'port': int(os.getenv('DB_PORT', 3306)),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    analytics = ArtifactAnalytics(db_config)
    
    # Sidebar for query selection
    st.sidebar.header("Select Analysis")
    query_name = st.sidebar.selectbox(
        "Choose Query",
        list(analytics.QUERIES.keys())
    )
    
    # Execute query
    if st.sidebar.button("Run Query"):
        with st.spinner("Executing query..."):
            df = analytics.execute_query(analytics.QUERIES[query_name])
            
            # Display results
            st.subheader("Query Results")
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
    
    # ETL Section
    st.sidebar.header("ETL Pipeline")
    if st.sidebar.button("Run ETL"):
        with st.spinner("Running ETL pipeline..."):
            api_client = HarvardAPIClient()
            etl = ArtifactETL(db_config)
            
            # Extract
            raw_data = etl.extract(api_client, max_records=500)
            st.success(f"Extracted {len(raw_data)} records")
            
            # Transform
            metadata_df, media_df, colors_df = etl.transform(raw_data)
            st.success("Data transformed successfully")
            
            # Load
            etl.load(metadata_df, media_df, colors_df)
            st.success("Data loaded into database")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(api_client, pages, delay=1):
    """Fetch data with rate limiting"""
    results = []
    for page in range(1, pages + 1):
        data = api_client.fetch_artifacts(page=page)
        results.extend(data.get('records', []))
        time.sleep(delay)  # Respect API rate limits
    return results
```

### Batch Insert for Performance

```python
def batch_insert(cursor, table, columns, data, batch_size=1000):
    """Insert data in batches for better performance"""
    placeholders = ', '.join(['%s'] * len(columns))
    query = f"INSERT INTO {table} ({', '.join(columns)}) VALUES ({placeholders})"
    
    for i in range(0, len(data), batch_size):
        batch = data[i:i + batch_size]
        cursor.executemany(query, batch)
```

### Error Handling for API Calls

```python
def safe_api_call(api_client, page, retries=3):
    """API call with retry logic"""
    for attempt in range(retries):
        try:
            return api_client.fetch_artifacts(page=page)
        except requests.exceptions.RequestException as e:
            if attempt == retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

### API Key Issues
- Verify key is set in `.env`: `echo $HARVARD_API_KEY`
- Check key validity by testing in browser: `https://api.harvardartmuseums.org/object?apikey=YOUR_KEY`

### Database Connection Errors
```python
# Test connection
try:
    conn = mysql.connector.connect(**db_config)
    print("✓ Database connected")
    conn.close()
except mysql.connector.Error as e:
    print(f"✗ Connection failed: {e}")
```

### Missing Data Handling
```python
# Handle None values in DataFrame
df = df.fillna({
    'title': 'Untitled',
    'culture': 'Unknown',
    'century': 'Unknown'
})
```

### Streamlit Caching
```python
@st.cache_data(ttl=3600)
def load_analytics_data(query):
    """Cache query results for 1 hour"""
    return analytics.execute_query(query)
```

## Key Commands

```bash
# Run the application
streamlit run app.py

# Run on specific port
streamlit run app.py --server.port 8501

# Run with auto-reload during development
streamlit run app.py --server.runOnSave true

# Clear Streamlit cache
streamlit cache clear
```

This skill provides the foundation for building production ETL pipelines with cultural heritage data, applicable to other museum APIs and data engineering projects.
