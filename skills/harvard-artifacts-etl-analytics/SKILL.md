---
name: harvard-artifacts-etl-analytics
description: ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with Harvard artifacts collection
  - extract and load Harvard museum API data to SQL
  - visualize Harvard art museum collection data
  - set up data engineering pipeline for museum artifacts
  - analyze Harvard art museums collection with SQL
  - build Streamlit app for art museum analytics
  - process Harvard API artifacts with ETL workflow
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It implements ETL pipelines to extract artifact metadata, transform nested JSON into relational tables, load data into SQL databases (MySQL/TiDB), and visualize analytics through an interactive Streamlit dashboard.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
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

Create a `.env` file or configure environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_db_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

```python
import mysql.connector
import os

def setup_database():
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD')
    )
    cursor = conn.cursor()
    
    # Create database
    cursor.execute("CREATE DATABASE IF NOT EXISTS harvard_artifacts")
    cursor.execute("USE harvard_artifacts")
    
    # Create tables
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            period VARCHAR(255),
            century VARCHAR(255),
            classification VARCHAR(255),
            department VARCHAR(255),
            dated VARCHAR(255),
            description TEXT,
            technique VARCHAR(500),
            medium VARCHAR(500),
            dimensions VARCHAR(500),
            url VARCHAR(500),
            creditline TEXT
        )
    """)
    
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            baseimageurl VARCHAR(500),
            iiifbaseuri VARCHAR(500),
            primaryimage BOOLEAN,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(20),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    conn.commit()
    cursor.close()
    conn.close()
```

## ETL Pipeline Implementation

### Extract: Fetch Data from Harvard API

```python
import requests
import os

def extract_artifacts(api_key, page=1, size=100):
    """Extract artifacts from Harvard Art Museums API"""
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
artifacts, info = extract_artifacts(api_key, page=1, size=50)
print(f"Total records: {info['totalrecords']}")
```

### Transform: Process Nested JSON

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """Transform raw API data into structured dataframes"""
    
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'description': artifact.get('description'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'url': artifact.get('url'),
            'creditline': artifact.get('creditline')
        }
        metadata_list.append(metadata)
        
        # Extract media (images)
        if artifact.get('images'):
            for image in artifact['images']:
                media = {
                    'artifact_id': artifact['id'],
                    'baseimageurl': image.get('baseimageurl'),
                    'iiifbaseuri': image.get('iiifbaseuri'),
                    'primaryimage': 1 if image.get('imageid') == artifact.get('primaryimageurl') else 0
                }
                media_list.append(media)
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_data = {
                    'artifact_id': artifact['id'],
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
```

### Load: Insert into SQL Database

```python
def load_to_database(metadata_df, media_df, colors_df):
    """Load transformed data into SQL database"""
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    cursor = conn.cursor()
    
    # Insert metadata (batch insert)
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, dated, 
         description, technique, medium, dimensions, url, creditline)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    cursor.executemany(metadata_query, metadata_df.values.tolist())
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, iiifbaseuri, primaryimage)
        VALUES (%s, %s, %s, %s)
    """
    cursor.executemany(media_query, media_df.values.tolist())
    
    # Insert colors
    colors_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    cursor.executemany(colors_query, colors_df.values.tolist())
    
    conn.commit()
    cursor.close()
    conn.close()
    
    return len(metadata_df), len(media_df), len(colors_df)
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Collection Analytics")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    elif page == "Visualizations":
        show_visualizations_page()

def show_etl_page():
    st.header("ETL Pipeline")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Number of pages to fetch", 1, 10, 1)
        page_size = st.number_input("Records per page", 10, 100, 50)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            api_key = os.getenv('HARVARD_API_KEY')
            all_artifacts = []
            
            for page in range(1, num_pages + 1):
                artifacts, info = extract_artifacts(api_key, page, page_size)
                all_artifacts.extend(artifacts)
                st.write(f"Fetched page {page}/{num_pages}")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(all_artifacts)
            st.success(f"Transformed {len(metadata_df)} artifacts")
        
        with st.spinner("Loading to database..."):
            counts = load_to_database(metadata_df, media_df, colors_df)
            st.success(f"Loaded: {counts[0]} metadata, {counts[1]} media, {counts[2]} colors")

def show_analytics_page():
    st.header("SQL Analytics Dashboard")
    
    # Predefined queries
    queries = {
        "Top 10 Cultures by Artifact Count": """
            SELECT culture, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE culture IS NOT NULL 
            GROUP BY culture 
            ORDER BY count DESC 
            LIMIT 10
        """,
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE century IS NOT NULL 
            GROUP BY century 
            ORDER BY century
        """,
        "Department Distribution": """
            SELECT department, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE department IS NOT NULL 
            GROUP BY department 
            ORDER BY count DESC
        """,
        "Most Common Colors": """
            SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY count DESC 
            LIMIT 10
        """,
        "Artifacts with Media": """
            SELECT 
                COUNT(DISTINCT a.id) as total_artifacts,
                COUNT(DISTINCT m.artifact_id) as with_media,
                ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / COUNT(DISTINCT a.id), 2) as percentage
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        df = pd.read_sql(queries[selected_query], conn)
        conn.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2 and df.shape[0] > 1:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Analytical Queries

### Classification Analysis

```python
query = """
SELECT 
    classification,
    COUNT(*) as artifact_count,
    COUNT(DISTINCT culture) as culture_variety,
    COUNT(DISTINCT century) as time_span
FROM artifactmetadata
WHERE classification IS NOT NULL
GROUP BY classification
ORDER BY artifact_count DESC
LIMIT 20
"""
```

### Color Spectrum Analysis

```python
query = """
SELECT 
    spectrum,
    COUNT(DISTINCT artifact_id) as artifact_count,
    AVG(percent) as avg_coverage,
    COUNT(*) as color_instances
FROM artifactcolors
GROUP BY spectrum
ORDER BY artifact_count DESC
"""
```

### Media Availability

```python
query = """
SELECT 
    a.department,
    COUNT(DISTINCT a.id) as total_artifacts,
    COUNT(DISTINCT m.artifact_id) as with_images,
    ROUND(AVG(m.primaryimage), 2) as primary_image_ratio
FROM artifactmetadata a
LEFT JOIN artifactmedia m ON a.id = m.artifact_id
GROUP BY a.department
ORDER BY total_artifacts DESC
"""
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Troubleshooting

### API Rate Limiting

```python
import time

def extract_with_rate_limit(api_key, pages, delay=1):
    """Extract data with rate limiting"""
    all_data = []
    for page in range(1, pages + 1):
        data, info = extract_artifacts(api_key, page)
        all_data.extend(data)
        time.sleep(delay)  # Respect API rate limits
    return all_data
```

### Database Connection Issues

```python
def get_db_connection(retries=3):
    """Get database connection with retry logic"""
    for attempt in range(retries):
        try:
            conn = mysql.connector.connect(
                host=os.getenv('DB_HOST'),
                user=os.getenv('DB_USER'),
                password=os.getenv('DB_PASSWORD'),
                database=os.getenv('DB_NAME'),
                connect_timeout=10
            )
            return conn
        except mysql.connector.Error as e:
            if attempt < retries - 1:
                time.sleep(2 ** attempt)
            else:
                raise e
```

### Handling Null Values

```python
def safe_transform(value, default=''):
    """Safely transform values handling None"""
    return value if value is not None else default

# Apply in transform
metadata = {
    'title': safe_transform(artifact.get('title')),
    'culture': safe_transform(artifact.get('culture')),
    # ...
}
```
