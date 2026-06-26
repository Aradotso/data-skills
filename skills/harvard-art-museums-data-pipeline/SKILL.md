---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - create an ETL process for museum artifact data
  - set up analytics dashboard for Harvard artifacts collection
  - query and visualize Harvard Art Museums data
  - build a Streamlit app with museum API data
  - extract and transform Harvard museum artifacts into SQL
  - analyze art collection data with Python and SQL
  - create interactive museum data visualizations
---

# Harvard Art Museums Data Pipeline & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

This project demonstrates building end-to-end data engineering pipelines using the Harvard Art Museums API. It covers API integration, ETL workflows, SQL database design, analytics queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color data into relational databases
- **SQL Analytics**: Stores data in MySQL/TiDB Cloud with proper schema design and foreign key relationships
- **Visualization**: Creates interactive Streamlit dashboards with Plotly charts for data exploration
- **Real-world Simulation**: Demonstrates production-grade data engineering patterns

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
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

Get your free API key from [Harvard Art Museums](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file or set environment variables:

```bash
HARVARD_API_KEY=your_api_key_here
```

### Database Configuration

Set up MySQL or TiDB Cloud connection:

```python
import os
import mysql.connector

# Database connection
conn = mysql.connector.connect(
    host=os.getenv('DB_HOST', 'localhost'),
    user=os.getenv('DB_USER', 'root'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME', 'harvard_artifacts')
)
```

### Database Schema

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    verificationlevel INT,
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(1000),
    primaryimageurl VARCHAR(1000),
    iiifbaseuri VARCHAR(1000),
    imagecount INT,
    videocount INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
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

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py
```

The app will open at `http://localhost:8501`

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os

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
    
    data = response.json()
    return data['records'], data['info']

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts, info = fetch_artifacts(api_key, page=1, size=50)
print(f"Fetched {len(artifacts)} artifacts")
print(f"Total available: {info['totalrecords']}")
```

### Transform: Process JSON to DataFrames

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform artifact data into structured metadata"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear'),
            'verificationlevel': artifact.get('verificationlevel', 0),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """Extract media information"""
    media_list = []
    
    for artifact in artifacts:
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'iiifbaseuri': artifact.get('iiifbaseuri'),
            'imagecount': artifact.get('totalpageviews', 0),
            'videocount': artifact.get('videocount', 0)
        }
        media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """Extract color data"""
    colors_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_data)
    
    return pd.DataFrame(colors_list)
```

### Load: Insert into Database

```python
def load_to_database(conn, df_metadata, df_media, df_colors):
    """Load transformed data into SQL database"""
    cursor = conn.cursor()
    
    # Insert metadata (use INSERT IGNORE to avoid duplicates)
    for _, row in df_metadata.iterrows():
        query = """
        INSERT IGNORE INTO artifactmetadata 
        (id, title, culture, period, century, dated, classification, 
         department, division, technique, medium, dimensions, creditline, 
         accessionyear, verificationlevel, totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """
        cursor.execute(query, tuple(row))
    
    # Insert media
    for _, row in df_media.iterrows():
        query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, primaryimageurl, iiifbaseuri, imagecount, videocount)
        VALUES (%s, %s, %s, %s, %s, %s)
        """
        cursor.execute(query, tuple(row))
    
    # Insert colors
    for _, row in df_colors.iterrows():
        query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.execute(query, tuple(row))
    
    conn.commit()
    cursor.close()
```

## Complete ETL Workflow

```python
import mysql.connector
import os
from dotenv import load_dotenv

def run_etl_pipeline(num_pages=5, page_size=100):
    """Run complete ETL pipeline"""
    load_dotenv()
    
    # Connect to database
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    api_key = os.getenv('HARVARD_API_KEY')
    
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}...")
        
        # Extract
        artifacts, info = fetch_artifacts(api_key, page=page, size=page_size)
        
        # Transform
        df_metadata = transform_artifact_metadata(artifacts)
        df_media = transform_artifact_media(artifacts)
        df_colors = transform_artifact_colors(artifacts)
        
        # Load
        load_to_database(conn, df_metadata, df_media, df_colors)
        
        print(f"Page {page} completed: {len(artifacts)} artifacts loaded")
    
    conn.close()
    print("ETL pipeline completed successfully!")

# Run the pipeline
if __name__ == "__main__":
    run_etl_pipeline(num_pages=10, page_size=50)
```

## SQL Analytics Queries

### Top Cultures by Artifact Count

```python
def get_top_cultures(conn, limit=10):
    """Get top cultures by artifact count"""
    query = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT %s
    """
    return pd.read_sql(query, conn, params=(limit,))
```

### Artifacts by Century

```python
def get_artifacts_by_century(conn):
    """Get artifact distribution by century"""
    query = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY century
    """
    return pd.read_sql(query, conn)
```

### Most Common Colors

```python
def get_top_colors(conn, limit=15):
    """Get most common colors in collection"""
    query = """
    SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
    FROM artifactcolors
    WHERE color IS NOT NULL
    GROUP BY color
    ORDER BY frequency DESC
    LIMIT %s
    """
    return pd.read_sql(query, conn, params=(limit,))
```

### Department Statistics

```python
def get_department_stats(conn):
    """Get artifact statistics by department"""
    query = """
    SELECT 
        department,
        COUNT(*) as total_artifacts,
        AVG(totalpageviews) as avg_pageviews,
        COUNT(DISTINCT culture) as unique_cultures
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY total_artifacts DESC
    """
    return pd.read_sql(query, conn)
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("Explore artifact data through interactive visualizations")
    
    # Database connection
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    query_option = st.sidebar.selectbox(
        "Select Analysis",
        [
            "Top Cultures",
            "Artifacts by Century",
            "Most Common Colors",
            "Department Statistics",
            "Media Availability"
        ]
    )
    
    # Execute selected query
    if query_option == "Top Cultures":
        df = get_top_cultures(conn, limit=15)
        st.subheader("Top 15 Cultures by Artifact Count")
        
        # Display table
        st.dataframe(df)
        
        # Create bar chart
        fig = px.bar(
            df,
            x='culture',
            y='artifact_count',
            title='Artifact Distribution by Culture',
            labels={'culture': 'Culture', 'artifact_count': 'Number of Artifacts'}
        )
        st.plotly_chart(fig, use_container_width=True)
    
    elif query_option == "Most Common Colors":
        df = get_top_colors(conn, limit=20)
        st.subheader("Most Common Colors in Collection")
        
        st.dataframe(df)
        
        fig = px.bar(
            df,
            x='color',
            y='frequency',
            title='Color Frequency Distribution',
            color='avg_percent',
            labels={'color': 'Color', 'frequency': 'Frequency'}
        )
        st.plotly_chart(fig, use_container_width=True)
    
    conn.close()

if __name__ == "__main__":
    main()
```

## Advanced Analytics Patterns

### Time-based Analysis

```python
def get_accession_trends(conn):
    """Analyze artifact acquisition trends over time"""
    query = """
    SELECT 
        accessionyear,
        COUNT(*) as artifacts_acquired,
        COUNT(DISTINCT culture) as cultures_added
    FROM artifactmetadata
    WHERE accessionyear IS NOT NULL 
        AND accessionyear > 1900
    GROUP BY accessionyear
    ORDER BY accessionyear
    """
    df = pd.read_sql(query, conn)
    
    # Create line chart
    fig = px.line(
        df,
        x='accessionyear',
        y='artifacts_acquired',
        title='Artifact Acquisitions Over Time'
    )
    return df, fig
```

### Cross-table Analysis

```python
def get_media_richness_by_culture(conn):
    """Analyze media availability across cultures"""
    query = """
    SELECT 
        m.culture,
        COUNT(DISTINCT m.id) as total_artifacts,
        SUM(CASE WHEN med.imagecount > 0 THEN 1 ELSE 0 END) as with_images,
        AVG(med.imagecount) as avg_images_per_artifact
    FROM artifactmetadata m
    LEFT JOIN artifactmedia med ON m.id = med.artifact_id
    WHERE m.culture IS NOT NULL
    GROUP BY m.culture
    HAVING total_artifacts > 10
    ORDER BY avg_images_per_artifact DESC
    LIMIT 20
    """
    return pd.read_sql(query, conn)
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_rate_limit(api_key, page, size=100, delay=0.5):
    """Fetch data with rate limiting"""
    try:
        artifacts, info = fetch_artifacts(api_key, page, size)
        time.sleep(delay)  # Respect API rate limits
        return artifacts, info
    except requests.exceptions.HTTPError as e:
        if e.response.status_code == 429:
            print("Rate limit hit, waiting 60 seconds...")
            time.sleep(60)
            return fetch_artifacts(api_key, page, size)
        raise
```

### Database Connection Issues

```python
def get_db_connection(retry=3):
    """Get database connection with retry logic"""
    for attempt in range(retry):
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
            if attempt < retry - 1:
                print(f"Connection attempt {attempt + 1} failed, retrying...")
                time.sleep(5)
            else:
                raise e
```

### Handling Missing Data

```python
def safe_transform(artifacts):
    """Transform data with null handling"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Untitled'),
            'culture': artifact.get('culture') or 'Unknown',
            'century': artifact.get('century') or 'Unknown',
            # Use .get() with defaults for all fields
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)
```

### Large Dataset Processing

```python
def batch_insert(conn, df, table_name, batch_size=1000):
    """Insert data in batches for better performance"""
    cursor = conn.cursor()
    total_rows = len(df)
    
    for i in range(0, total_rows, batch_size):
        batch = df.iloc[i:i+batch_size]
        # Insert batch logic here
        conn.commit()
        print(f"Processed {min(i+batch_size, total_rows)}/{total_rows} rows")
    
    cursor.close()
```

This skill provides comprehensive coverage of building data pipelines with the Harvard Art Museums API, from ETL implementation to interactive analytics dashboards.
