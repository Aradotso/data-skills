---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline and analytics for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with Harvard artifacts
  - set up data engineering workflow for museum collections
  - extract and transform Harvard Art Museums API data
  - build artifact collection analytics with Streamlit
  - create SQL analytics for museum artifact data
  - implement ETL for art museum metadata
  - visualize Harvard artifacts data with Plotly
---

# Harvard Artifacts Collection ETL & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides a complete data engineering solution for working with the Harvard Art Museums API. It demonstrates ETL pipeline design, SQL database modeling, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables (metadata, media, colors)
- **Loads** data into MySQL/TiDB Cloud using batch inserts
- **Analyzes** data with 20+ predefined SQL queries
- **Visualizes** results through interactive Plotly charts in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required Dependencies:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from: https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dAd8W9hbtnLpA/viewform

Store credentials in environment variables or `.env` file:

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Setup

Create the required tables:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accession_number VARCHAR(100),
    url TEXT,
    PRIMARY KEY (id)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    media_type VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(7),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv("HARVARD_API_KEY")
data = fetch_artifacts(api_key, page=1, size=100)
artifacts = data.get("records", [])
total_pages = data.get("info", {}).get("pages", 1)
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class HarvardArtifactsETL:
    def __init__(self, db_config: Dict, api_key: str):
        self.db_config = db_config
        self.api_key = api_key
        self.conn = None
    
    def connect_db(self):
        """Establish database connection"""
        self.conn = mysql.connector.connect(**self.db_config)
        return self.conn.cursor()
    
    def extract(self, num_pages: int = 5) -> List[Dict]:
        """Extract data from Harvard API"""
        all_artifacts = []
        
        for page in range(1, num_pages + 1):
            data = fetch_artifacts(self.api_key, page=page)
            artifacts = data.get("records", [])
            all_artifacts.extend(artifacts)
            print(f"Extracted page {page}/{num_pages}: {len(artifacts)} artifacts")
        
        return all_artifacts
    
    def transform(self, artifacts: List[Dict]) -> tuple:
        """Transform nested JSON into relational dataframes"""
        metadata_list = []
        media_list = []
        colors_list = []
        
        for artifact in artifacts:
            # Metadata
            metadata = {
                "id": artifact.get("id"),
                "title": artifact.get("title", "")[:500],
                "culture": artifact.get("culture", "")[:200],
                "century": artifact.get("century", "")[:100],
                "classification": artifact.get("classification", "")[:200],
                "department": artifact.get("department", "")[:200],
                "dated": artifact.get("dated", "")[:200],
                "accession_number": artifact.get("accessionNumber", "")[:100],
                "url": artifact.get("url", "")
            }
            metadata_list.append(metadata)
            
            # Media/Images
            images = artifact.get("images", [])
            for img in images:
                media_list.append({
                    "artifact_id": artifact.get("id"),
                    "image_url": img.get("baseimageurl", ""),
                    "media_type": "image"
                })
            
            # Colors
            colors = artifact.get("colors", [])
            for color in colors:
                colors_list.append({
                    "artifact_id": artifact.get("id"),
                    "color_hex": color.get("hex", ""),
                    "color_percent": color.get("percent", 0.0)
                })
        
        df_metadata = pd.DataFrame(metadata_list)
        df_media = pd.DataFrame(media_list)
        df_colors = pd.DataFrame(colors_list)
        
        return df_metadata, df_media, df_colors
    
    def load(self, df_metadata, df_media, df_colors):
        """Load data into SQL database with batch inserts"""
        cursor = self.connect_db()
        
        # Insert metadata
        for _, row in df_metadata.iterrows():
            cursor.execute("""
                INSERT IGNORE INTO artifactmetadata 
                (id, title, culture, century, classification, department, dated, accession_number, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
            """, tuple(row))
        
        # Insert media
        for _, row in df_media.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia (artifact_id, image_url, media_type)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        for _, row in df_colors.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        self.conn.commit()
        cursor.close()
        self.conn.close()
        
        print(f"Loaded {len(df_metadata)} artifacts, {len(df_media)} media, {len(df_colors)} colors")

# Usage
db_config = {
    "host": os.getenv("DB_HOST"),
    "user": os.getenv("DB_USER"),
    "password": os.getenv("DB_PASSWORD"),
    "database": os.getenv("DB_NAME")
}

etl = HarvardArtifactsETL(db_config, os.getenv("HARVARD_API_KEY"))
artifacts = etl.extract(num_pages=10)
df_metadata, df_media, df_colors = etl.transform(artifacts)
etl.load(df_metadata, df_media, df_colors)
```

### 3. SQL Analytics Queries

```python
def run_analytics_query(query: str, db_config: Dict) -> pd.DataFrame:
    """Execute SQL query and return results as DataFrame"""
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Sample analytical queries
ANALYTICS_QUERIES = {
    "artifacts_by_century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "artifacts_by_department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "top_colors": """
        SELECT color_hex, COUNT(*) as usage_count, 
               AVG(color_percent) as avg_percent
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY usage_count DESC
        LIMIT 20
    """,
    
    "artifacts_with_images": """
        SELECT 
            CASE WHEN media_count > 0 THEN 'With Images' ELSE 'No Images' END as image_status,
            COUNT(*) as artifact_count
        FROM (
            SELECT a.id, COUNT(m.media_id) as media_count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY a.id
        ) as img_stats
        GROUP BY image_status
    """,
    
    "classification_distribution": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 10
    """
}

# Execute query
df_results = run_analytics_query(ANALYTICS_QUERIES["artifacts_by_century"], db_config)
print(df_results)
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar configuration
with st.sidebar:
    st.header("Configuration")
    api_key = st.text_input("Harvard API Key", type="password", value=os.getenv("HARVARD_API_KEY", ""))
    
    st.header("ETL Pipeline")
    num_pages = st.slider("Pages to Extract", 1, 20, 5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            etl = HarvardArtifactsETL(db_config, api_key)
            artifacts = etl.extract(num_pages)
            df_metadata, df_media, df_colors = etl.transform(artifacts)
            etl.load(df_metadata, df_media, df_colors)
            st.success(f"Loaded {len(df_metadata)} artifacts!")

# Analytics section
st.header("📊 SQL Analytics")

query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))

if st.button("Run Query"):
    df = run_analytics_query(ANALYTICS_QUERIES[query_name], db_config)
    
    st.subheader("Query Results")
    st.dataframe(df)
    
    # Auto-generate visualization
    if len(df.columns) == 2:
        fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                     title=f"{query_name.replace('_', ' ').title()}")
        st.plotly_chart(fig, use_container_width=True)
    
    # Download results
    csv = df.to_csv(index=False)
    st.download_button("Download CSV", csv, f"{query_name}.csv", "text/csv")
```

## Common Patterns

### Incremental Data Loading

```python
def get_latest_artifact_id(db_config):
    """Get the highest artifact ID already in database"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    max_id = cursor.fetchone()[0] or 0
    cursor.close()
    conn.close()
    return max_id

def incremental_extract(api_key, last_id):
    """Fetch only new artifacts since last load"""
    # Harvard API doesn't support ID filtering directly
    # Implement by fetching latest pages and filtering by ID
    pass
```

### Rate Limiting

```python
import time

def fetch_with_rate_limit(api_key, page, delay=0.5):
    """Fetch with rate limiting to avoid API throttling"""
    data = fetch_artifacts(api_key, page)
    time.sleep(delay)  # Wait between requests
    return data
```

## Troubleshooting

**API Key Issues:**
- Ensure API key is valid from Harvard's registration form
- Check environment variable is properly loaded: `echo $HARVARD_API_KEY`

**Database Connection Errors:**
- Verify MySQL/TiDB credentials in `.env`
- Check firewall allows connections to database port (3306)
- Ensure database and tables exist before running ETL

**Memory Issues with Large Datasets:**
- Process data in smaller batches (reduce `num_pages`)
- Use chunked DataFrame operations: `df.to_sql(chunksize=1000)`

**Empty Results:**
- Check API response has `records` key
- Verify `hasimage=1` filter isn't too restrictive
- Inspect raw API response: `print(response.json())`

**Foreign Key Constraint Violations:**
- Ensure metadata is loaded before media/colors
- Use `INSERT IGNORE` to skip duplicates
- Check artifact IDs exist in parent table
