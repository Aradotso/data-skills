---
name: harvard-artifacts-data-pipeline
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums API
  - create a data engineering pipeline with Streamlit dashboard
  - fetch and analyze Harvard artifacts data
  - set up Harvard Art Museums data analytics app
  - implement SQL analytics on museum artifacts
  - visualize Harvard Art Museums collection data
  - build end-to-end data pipeline with API integration
  - create museum artifacts analytics dashboard
---

# Harvard Artifacts Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for collecting, processing, and analyzing artifacts from the Harvard Art Museums API. It implements a complete ETL pipeline that extracts artifact data, transforms it into relational structures, loads it into SQL databases (MySQL/TiDB), and presents analytics through an interactive Streamlit dashboard.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly
```

## Configuration

### Environment Variables

Set up your configuration using environment variables:

```bash
# Harvard Art Museums API Key
export HARVARD_API_KEY="your_api_key_here"

# Database Configuration
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
export DB_PORT="3306"
```

### API Key Setup

1. Register at [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Obtain your API key
3. Store it securely using environment variables

## Database Schema

The application uses three main tables with foreign key relationships:

```sql
-- Artifact Metadata (Main Table)
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    dated VARCHAR(200),
    url TEXT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
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

### 1. ETL Pipeline Implementation

**Extract Data from API:**

```python
import requests
import pandas as pd
import os

def extract_artifacts(api_key, num_pages=5, page_size=100):
    """Extract artifact data from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            if 'records' in data:
                all_artifacts.extend(data['records'])
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts
```

**Transform Data:**

```python
def transform_artifacts(raw_data):
    """Transform raw API data into structured dataframes"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'period': artifact.get('period', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'technique': artifact.get('technique', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'url': artifact.get('url', '')
        }
        metadata_records.append(metadata)
        
        # Extract media
        if 'images' in artifact and artifact['images']:
            for img in artifact['images']:
                media_records.append({
                    'artifact_id': artifact.get('id'),
                    'image_url': img.get('baseimageurl', ''),
                    'media_type': 'image'
                })
        
        # Extract colors
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
```

**Load into Database:**

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """Load transformed data into MySQL database"""
    try:
        connection = mysql.connector.connect(
            host=db_config['host'],
            user=db_config['user'],
            password=db_config['password'],
            database=db_config['database'],
            port=db_config.get('port', 3306)
        )
        
        cursor = connection.cursor()
        
        # Insert metadata
        for _, row in metadata_df.iterrows():
            sql = """
                INSERT INTO artifactmetadata 
                (id, title, culture, period, century, classification, 
                 department, technique, dated, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """
            cursor.execute(sql, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            sql = """
                INSERT INTO artifactmedia (artifact_id, image_url, media_type)
                VALUES (%s, %s, %s)
            """
            cursor.execute(sql, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            sql = """
                INSERT INTO artifactcolors 
                (artifact_id, color_hex, color_name, percentage)
                VALUES (%s, %s, %s, %s)
            """
            cursor.execute(sql, tuple(row))
        
        connection.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 2. Streamlit Dashboard

**Main Application Structure:**

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Data Analytics")
    st.markdown("### End-to-End Data Engineering Pipeline")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME'),
        'port': int(os.getenv('DB_PORT', 3306))
    }
    
    # ETL Section
    st.header("1️⃣ ETL Pipeline")
    if st.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            api_key = os.getenv('HARVARD_API_KEY')
            raw_data = extract_artifacts(api_key, num_pages=5)
            st.success(f"Extracted {len(raw_data)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df, db_config)
            st.success("Data loaded to database")
    
    # Analytics Section
    st.header("2️⃣ SQL Analytics")
    
    queries = {
        "Artifacts by Culture": """
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
        "Top Departments": """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            GROUP BY department
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        "Color Distribution": """
            SELECT color_name, AVG(percentage) as avg_percentage
            FROM artifactcolors
            GROUP BY color_name
            ORDER BY avg_percentage DESC
            LIMIT 10
        """
    }
    
    selected_query = st.selectbox("Select Analytics Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        result_df = execute_query(queries[selected_query], db_config)
        st.dataframe(result_df)
        
        # Visualization
        if len(result_df) > 0:
            fig = px.bar(result_df, x=result_df.columns[0], 
                        y=result_df.columns[1],
                        title=selected_query)
            st.plotly_chart(fig)

def execute_query(query, db_config):
    """Execute SQL query and return results as DataFrame"""
    try:
        connection = mysql.connector.connect(**db_config)
        df = pd.read_sql(query, connection)
        connection.close()
        return df
    except Error as e:
        st.error(f"Query error: {e}")
        return pd.DataFrame()

if __name__ == "__main__":
    main()
```

### 3. Advanced Analytics Queries

```python
ANALYTICS_QUERIES = {
    "Media Availability Analysis": """
        SELECT 
            m.classification,
            COUNT(DISTINCT m.id) as total_artifacts,
            COUNT(DISTINCT media.artifact_id) as artifacts_with_media,
            ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / 
                  COUNT(DISTINCT m.id), 2) as media_percentage
        FROM artifactmetadata m
        LEFT JOIN artifactmedia media ON m.id = media.artifact_id
        GROUP BY m.classification
        HAVING total_artifacts > 5
        ORDER BY media_percentage DESC
    """,
    
    "Cultural Period Analysis": """
        SELECT 
            culture,
            period,
            COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture != 'Unknown' AND period != 'Unknown'
        GROUP BY culture, period
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "Dominant Colors by Department": """
        SELECT 
            m.department,
            c.color_name,
            AVG(c.percentage) as avg_percentage
        FROM artifactmetadata m
        JOIN artifactcolors c ON m.id = c.artifact_id
        GROUP BY m.department, c.color_name
        HAVING COUNT(*) > 10
        ORDER BY m.department, avg_percentage DESC
    """
}
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Batch Processing for Large Datasets

```python
def batch_insert(df, table_name, db_config, batch_size=1000):
    """Insert data in batches for better performance"""
    connection = mysql.connector.connect(**db_config)
    cursor = connection.cursor()
    
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        # Perform batch insert
        # ... insert logic
        connection.commit()
    
    cursor.close()
    connection.close()
```

### Rate Limiting API Calls

```python
import time

def extract_with_rate_limit(api_key, num_pages, delay=1):
    """Extract data with rate limiting"""
    artifacts = []
    
    for page in range(1, num_pages + 1):
        data = fetch_page(api_key, page)
        artifacts.extend(data)
        time.sleep(delay)  # Respect API rate limits
    
    return artifacts
```

## Troubleshooting

**API Connection Issues:**
- Verify API key is set correctly: `echo $HARVARD_API_KEY`
- Check API rate limits (typically 2500 requests/day)
- Ensure network connectivity to api.harvardartmuseums.org

**Database Connection Errors:**
- Verify database credentials and host accessibility
- Check firewall rules for database port (default 3306)
- Ensure database and tables exist before loading data

**Memory Issues with Large Datasets:**
- Use batch processing for large data loads
- Limit API pagination (reduce `num_pages`)
- Process data in chunks rather than loading all at once

**Streamlit Performance:**
- Cache database connections with `@st.cache_resource`
- Use `@st.cache_data` for expensive computations
- Limit query result sizes for visualization
