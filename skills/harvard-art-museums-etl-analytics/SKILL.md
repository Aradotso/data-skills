---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics application for Harvard Art Museums API data with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and analyze Harvard Art Museums API data
  - set up artifact data engineering pipeline
  - query and visualize museum collection data
  - implement museum data ETL with SQL and Streamlit
  - analyze art museum collections with Python
  - create interactive art analytics dashboard
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering solution for working with the Harvard Art Museums API. It demonstrates production-grade ETL pipelines that extract artifact metadata, transform nested JSON into relational structures, load data into SQL databases (MySQL/TiDB Cloud), and visualize insights through an interactive Streamlit dashboard.

The application is structured around three core tables (`artifactmetadata`, `artifactmedia`, `artifactcolors`) with proper relational integrity, and includes 20+ pre-built analytical queries for exploring patterns in museum collections.

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

Create a `.env` file or set environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    technique VARCHAR(500),
    period VARCHAR(255),
    dated VARCHAR(255),
    medium VARCHAR(500),
    url VARCHAR(500)
);

-- Artifact media table
CREATE TABLE IF NOT EXISTS artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(500),
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE IF NOT EXISTS artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from typing import Dict, List, Optional

def fetch_artifacts(api_key: str, page: int = 1, size: int = 100) -> Optional[Dict]:
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        api_key: Harvard API key
        page: Page number (default: 1)
        size: Records per page (default: 100, max: 100)
    
    Returns:
        JSON response with artifact records
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    try:
        response = requests.get(base_url, params=params, timeout=30)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"API request failed: {e}")
        return None

def collect_artifacts_batch(api_key: str, num_pages: int = 5) -> List[Dict]:
    """
    Collect multiple pages of artifact data
    
    Args:
        api_key: Harvard API key
        num_pages: Number of pages to fetch
    
    Returns:
        List of all artifact records
    """
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        
        if data and 'records' in data:
            all_artifacts.extend(data['records'])
            print(f"Collected {len(data['records'])} artifacts from page {page}")
        else:
            print(f"No data returned for page {page}")
            break
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

def transform_artifacts(raw_data: List[Dict]) -> tuple:
    """
    Transform raw API data into normalized dataframes
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'division': artifact.get('division', '')[:255],
            'technique': artifact.get('technique', '')[:500],
            'period': artifact.get('period', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'medium': artifact.get('medium', '')[:500],
            'url': artifact.get('url', '')[:500]
        })
        
        # Extract media/images
        if artifact.get('images'):
            for img in artifact['images']:
                media_records.append({
                    'artifact_id': artifact.get('id'),
                    'image_url': img.get('baseimageurl', '')[:500],
                    'media_type': 'image'
                })
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_records.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color', '')[:50],
                    'percentage': color.get('percent', 0.0)
                })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

def load_to_database(metadata_df: pd.DataFrame, 
                     media_df: pd.DataFrame, 
                     colors_df: pd.DataFrame,
                     db_config: Dict):
    """
    Load transformed data into MySQL database
    
    Args:
        metadata_df: Artifact metadata DataFrame
        media_df: Media/images DataFrame
        colors_df: Colors DataFrame
        db_config: Database connection configuration
    """
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    try:
        # Insert metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, division, 
         technique, period, dated, medium, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture), century=VALUES(century)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        if not media_df.empty:
            media_query = """
            INSERT INTO artifactmedia (artifact_id, image_url, media_type)
            VALUES (%s, %s, %s)
            """
            cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        if not colors_df.empty:
            colors_query = """
            INSERT INTO artifactcolors (artifact_id, color, percentage)
            VALUES (%s, %s, %s)
            """
            cursor.executemany(colors_query, colors_df.values.tolist())
        
        conn.commit()
        print(f"Successfully loaded {len(metadata_df)} artifacts")
        
    except Exception as e:
        conn.rollback()
        print(f"Database load error: {e}")
    finally:
        cursor.close()
        conn.close()
```

### 3. Analytics Queries

```python
# Sample analytical queries for the Streamlit dashboard

ANALYTICS_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century Distribution": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department-wise Artifact Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Media Availability Analysis": """
        SELECT 
            CASE 
                WHEN image_count > 0 THEN 'With Images'
                ELSE 'Without Images'
            END as media_status,
            COUNT(*) as artifact_count
        FROM (
            SELECT a.id, COUNT(m.media_id) as image_count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY a.id
        ) as media_stats
        GROUP BY media_status
    """,
    
    "Top 10 Most Common Colors": """
        SELECT color, COUNT(*) as color_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY color_count DESC
        LIMIT 10
    """,
    
    "Classification Distribution": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL AND classification != ''
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """
}

def execute_analytics_query(query: str, db_config: Dict) -> pd.DataFrame:
    """
    Execute analytical query and return results as DataFrame
    
    Args:
        query: SQL query string
        db_config: Database connection configuration
    
    Returns:
        Query results as pandas DataFrame
    """
    conn = mysql.connector.connect(**db_config)
    
    try:
        df = pd.read_sql(query, conn)
        return df
    except Exception as e:
        print(f"Query execution error: {e}")
        return pd.DataFrame()
    finally:
        conn.close()
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px
import os
from dotenv import load_dotenv

def main():
    st.set_page_config(page_title="Harvard Art Museum Analytics", layout="wide")
    
    load_dotenv()
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Database configuration
    db_config = {
        'host': os.getenv('DB_HOST'),
        'port': int(os.getenv('DB_PORT', 3306)),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    # Sidebar for data collection
    with st.sidebar:
        st.header("📥 Data Collection")
        api_key = os.getenv('HARVARD_API_KEY')
        num_pages = st.number_input("Pages to fetch", min_value=1, max_value=50, value=5)
        
        if st.button("Fetch & Load Data"):
            with st.spinner("Collecting artifacts..."):
                artifacts = collect_artifacts_batch(api_key, num_pages)
                metadata_df, media_df, colors_df = transform_artifacts(artifacts)
                load_to_database(metadata_df, media_df, colors_df, db_config)
                st.success(f"Loaded {len(metadata_df)} artifacts!")
    
    # Main dashboard
    st.header("📊 Analytics Queries")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        df = execute_analytics_query(query, db_config)
        
        if not df.empty:
            st.subheader("Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                            title=query_name)
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Running the Complete Pipeline

```python
from dotenv import load_dotenv
import os

# Load configuration
load_dotenv()

# Database config
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

# Execute ETL
api_key = os.getenv('HARVARD_API_KEY')
raw_data = collect_artifacts_batch(api_key, num_pages=10)
metadata_df, media_df, colors_df = transform_artifacts(raw_data)
load_to_database(metadata_df, media_df, colors_df, db_config)

# Run analytics
results = execute_analytics_query(
    ANALYTICS_QUERIES["Top 10 Cultures by Artifact Count"],
    db_config
)
print(results)
```

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(api_key: str, num_pages: int, delay: float = 1.0):
    """Fetch data with rate limiting to respect API limits"""
    artifacts = []
    
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(api_key, page=page)
        if data and 'records' in data:
            artifacts.extend(data['records'])
        time.sleep(delay)  # Wait between requests
    
    return artifacts
```

## Troubleshooting

### API Connection Issues

```python
# Verify API key
response = requests.get(
    "https://api.harvardartmuseums.org/object",
    params={'apikey': os.getenv('HARVARD_API_KEY'), 'size': 1}
)
print(f"Status: {response.status_code}")
print(response.json().get('info', 'No info available'))
```

### Database Connection Errors

```python
# Test database connection
try:
    conn = mysql.connector.connect(**db_config)
    print("Database connection successful")
    conn.close()
except mysql.connector.Error as e:
    print(f"Connection failed: {e}")
```

### Empty Query Results

- Ensure data has been loaded: `SELECT COUNT(*) FROM artifactmetadata;`
- Check for NULL/empty filters in WHERE clauses
- Verify foreign key relationships are intact

### Streamlit Not Starting

```bash
# Run with explicit port
streamlit run app.py --server.port 8501

# Check for port conflicts
lsof -i :8501
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

The dashboard will provide:
- Data collection interface in the sidebar
- 20+ pre-built analytical queries
- Interactive visualizations with Plotly
- Real-time SQL query execution and results
