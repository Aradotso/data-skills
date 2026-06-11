---
name: harvard-artifacts-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline for museum artifacts
  - create ETL pipeline with Harvard Art Museums API
  - set up analytics dashboard for art collection data
  - analyze Harvard museum artifacts with SQL
  - visualize museum collection data with Streamlit
  - build data engineering app for art museums
  - extract and transform Harvard API artifact data
  - create museum data warehouse with Python
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for collecting, transforming, and analyzing artifact data from the Harvard Art Museums API. It demonstrates production-grade ETL pipelines, relational database design, SQL analytics, and interactive visualization using Streamlit.

**Architecture Flow:** API → ETL → SQL Database → Analytics → Visualization

## Installation

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
```

## Core Components

### 1. API Integration

The project uses the Harvard Art Museums API to extract artifact data. Key endpoints:

```python
import requests
import os

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org"

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts with pagination"""
    url = f"{BASE_URL}/object"
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

# Fetch multiple pages
def collect_artifacts(num_pages=10):
    all_artifacts = []
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(page=page)
        all_artifacts.extend(data.get('records', []))
    return all_artifacts
```

### 2. ETL Pipeline

Extract, transform, and load artifact data into relational tables:

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def create_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def transform_artifacts(raw_data):
    """Transform nested JSON into relational format"""
    metadata = []
    media = []
    colors = []
    
    for artifact in raw_data:
        # Artifact metadata
        metadata.append({
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions')
        })
        
        # Media information
        if artifact.get('images'):
            for img in artifact['images']:
                media.append({
                    'artifact_id': artifact.get('id'),
                    'image_id': img.get('imageid'),
                    'base_url': img.get('baseimageurl'),
                    'width': img.get('width'),
                    'height': img.get('height'),
                    'format': img.get('format')
                })
        
        # Color data
        if artifact.get('colors'):
            for color in artifact['colors']:
                colors.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata),
        pd.DataFrame(media),
        pd.DataFrame(colors)
    )

def load_to_database(df_metadata, df_media, df_colors):
    """Load transformed data to SQL database"""
    conn = create_connection()
    cursor = conn.cursor()
    
    # Create tables
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            dated VARCHAR(100),
            medium TEXT,
            dimensions TEXT
        )
    """)
    
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_id INT,
            base_url TEXT,
            width INT,
            height INT,
            format VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            hue VARCHAR(100),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    # Batch insert
    for _, row in df_metadata.iterrows():
        cursor.execute("""
            INSERT IGNORE INTO artifactmetadata VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    for _, row in df_media.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia (artifact_id, image_id, base_url, width, height, format)
            VALUES (%s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    for _, row in df_colors.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
```

### 3. SQL Analytics Queries

Common analytical queries for insights:

```python
# Query 1: Artifact distribution by culture
query_culture = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query 2: Artifacts by century
query_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY century
"""

# Query 3: Media availability analysis
query_media = """
    SELECT 
        am.classification,
        COUNT(DISTINCT am.artifact_id) as total_artifacts,
        COUNT(DISTINCT media.artifact_id) as artifacts_with_images,
        ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT am.artifact_id), 2) as image_percentage
    FROM artifactmetadata am
    LEFT JOIN artifactmedia media ON am.artifact_id = media.artifact_id
    GROUP BY am.classification
    ORDER BY total_artifacts DESC
"""

# Query 4: Color usage patterns
query_colors = """
    SELECT 
        color,
        COUNT(*) as usage_count,
        AVG(percent) as avg_percentage
    FROM artifactcolors
    GROUP BY color
    ORDER BY usage_count DESC
    LIMIT 15
"""

# Query 5: Department-wise distribution
query_department = """
    SELECT 
        department,
        COUNT(*) as artifact_count,
        COUNT(DISTINCT classification) as unique_classifications
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY artifact_count DESC
"""

def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    conn = create_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### 4. Streamlit Application

Build interactive dashboard:

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Analysis",
        ["Overview", "Culture Analysis", "Media Analysis", "Color Patterns", "Department Stats"]
    )
    
    if page == "Culture Analysis":
        st.header("Artifact Distribution by Culture")
        df = execute_query(query_culture)
        
        # Display table
        st.dataframe(df)
        
        # Visualization
        fig = px.bar(
            df,
            x='culture',
            y='artifact_count',
            title='Top 10 Cultures by Artifact Count',
            labels={'artifact_count': 'Number of Artifacts', 'culture': 'Culture'}
        )
        st.plotly_chart(fig, use_container_width=True)
    
    elif page == "Color Patterns":
        st.header("Color Usage Analysis")
        df = execute_query(query_colors)
        
        col1, col2 = st.columns(2)
        
        with col1:
            st.subheader("Color Distribution")
            fig = px.pie(
                df,
                values='usage_count',
                names='color',
                title='Color Usage Distribution'
            )
            st.plotly_chart(fig)
        
        with col2:
            st.subheader("Average Color Percentage")
            fig = px.bar(
                df,
                x='color',
                y='avg_percentage',
                title='Average Percentage by Color'
            )
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Full ETL Pipeline Execution

```python
def run_etl_pipeline(num_pages=10):
    """Complete ETL pipeline"""
    # Extract
    st.info(f"Extracting data from {num_pages} pages...")
    raw_artifacts = collect_artifacts(num_pages=num_pages)
    
    # Transform
    st.info("Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(raw_artifacts)
    
    # Load
    st.info("Loading to database...")
    load_to_database(df_metadata, df_media, df_colors)
    
    st.success(f"Successfully loaded {len(df_metadata)} artifacts!")
    
    return {
        'artifacts': len(df_metadata),
        'images': len(df_media),
        'color_records': len(df_colors)
    }
```

### Rate Limiting for API Calls

```python
import time

def fetch_with_rate_limit(page, delay=0.5):
    """Fetch data with rate limiting"""
    data = fetch_artifacts(page=page)
    time.sleep(delay)  # Respect API rate limits
    return data
```

## Troubleshooting

### Database Connection Issues
- Verify environment variables are set correctly
- Check database credentials and network connectivity
- Ensure database exists and user has proper privileges

### API Rate Limiting
- Harvard API has rate limits (adjust `delay` parameter)
- Use smaller batch sizes if timeout errors occur
- Implement exponential backoff for retries

### Missing Data Handling
```python
# Handle None values during transformation
metadata.append({
    'culture': artifact.get('culture', 'Unknown'),
    'century': artifact.get('century', 'Undated')
})
```

### Large Dataset Performance
- Use batch inserts instead of row-by-row
- Add database indexes on frequently queried columns
- Consider pagination for Streamlit tables with large results

## Key Configuration

Set these environment variables before running:

```bash
HARVARD_API_KEY=      # Get from https://docs.harvardartmuseums.org/
DB_HOST=              # MySQL/TiDB host
DB_USER=              # Database username
DB_PASSWORD=          # Database password
DB_NAME=              # Database name (e.g., harvard_artifacts)
```
