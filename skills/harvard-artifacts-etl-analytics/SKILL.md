---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create a data engineering project with Harvard API
  - set up artifact collection analytics with Streamlit
  - build SQL analytics dashboard for museum artifacts
  - extract and transform Harvard museum data
  - visualize artifact data from Harvard API
  - create data pipeline for art museum collections
  - analyze Harvard artifacts with SQL queries
---

# Harvard Artifacts ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for collecting, processing, and analyzing artifact data from the Harvard Art Museums API. It demonstrates production-grade ETL pipelines with SQL analytics and interactive Streamlit dashboards.

**Architecture Flow:** API → ETL → SQL Database → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Expected requirements:
# streamlit
# pandas
# requests
# mysql-connector-python
# plotly
# python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

The application uses three main tables with the following schema:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    division VARCHAR(255),
    department VARCHAR(255),
    classification VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dated_start INT,
    dated_end INT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(50),
    base_url VARCHAR(1000),
    caption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
artifacts = data.get('records', [])
total_records = data.get('info', {}).get('totalrecords', 0)
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform raw API data into structured metadata
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'division': artifact.get('division', '')[:255],
            'department': artifact.get('department', '')[:255],
            'classification': artifact.get('classification', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'period': artifact.get('period', '')[:255],
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'dated_start': artifact.get('datebegin'),
            'dated_end': artifact.get('dateend')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """
    Extract and transform media/image data
    """
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media = {
                'artifact_id': artifact_id,
                'media_type': 'image',
                'base_url': img.get('baseimageurl', '')[:1000],
                'caption': img.get('caption', '')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """
    Extract color palette data
    """
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', '')[:10],
                'color_name': color.get('color', '')[:100],
                'percentage': color.get('percent', 0.0)
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """
    Create database connection using environment variables
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def batch_insert_metadata(df, connection):
    """
    Batch insert artifact metadata into database
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, division, department, classification, 
     dated, period, technique, medium, dated_start, dated_end)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE 
    title=VALUES(title), culture=VALUES(culture)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    
    return cursor.rowcount

def load_all_data(metadata_df, media_df, colors_df):
    """
    Load all transformed data into database
    """
    conn = get_db_connection()
    if not conn:
        return False
    
    try:
        metadata_count = batch_insert_metadata(metadata_df, conn)
        
        # Similar batch inserts for media and colors
        cursor = conn.cursor()
        
        media_query = """
        INSERT INTO artifactmedia (artifact_id, media_type, base_url, caption)
        VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(media_query, media_df.to_records(index=False).tolist())
        
        colors_query = """
        INSERT INTO artifactcolors (artifact_id, color_hex, color_name, percentage)
        VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(colors_query, colors_df.to_records(index=False).tolist())
        
        conn.commit()
        return True
        
    except Error as e:
        print(f"Database error: {e}")
        conn.rollback()
        return False
    finally:
        conn.close()
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    else:
        show_visualizations_page()

def show_data_collection_page():
    st.header("📥 Data Collection from Harvard API")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_records = st.number_input(
            "Number of records to fetch",
            min_value=10,
            max_value=1000,
            value=100,
            step=10
        )
    
    with col2:
        start_page = st.number_input(
            "Starting page",
            min_value=1,
            value=1
        )
    
    if st.button("🚀 Start ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            api_key = os.getenv('HARVARD_API_KEY')
            
            # Fetch data
            data = fetch_artifacts(api_key, page=start_page, size=num_records)
            artifacts = data.get('records', [])
            
            st.success(f"✅ Fetched {len(artifacts)} artifacts")
            
            # Transform data
            metadata_df = transform_artifact_metadata(artifacts)
            media_df = transform_artifact_media(artifacts)
            colors_df = transform_artifact_colors(artifacts)
            
            st.info(f"Transformed: {len(metadata_df)} metadata, {len(media_df)} media, {len(colors_df)} colors")
            
            # Load to database
            if load_all_data(metadata_df, media_df, colors_df):
                st.success("✅ Data successfully loaded to database!")
            else:
                st.error("❌ Error loading data to database")

def show_analytics_page():
    st.header("📊 SQL Analytics")
    
    # Predefined queries
    queries = {
        "Top 10 Cultures by Artifact Count": """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY artifact_count DESC
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
            SELECT department, COUNT(*) as total
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY total DESC
        """,
        "Most Common Colors": """
            SELECT color_name, COUNT(*) as usage_count
            FROM artifactcolors
            GROUP BY color_name
            ORDER BY usage_count DESC
            LIMIT 15
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        conn = get_db_connection()
        if conn:
            df = pd.read_sql(queries[selected_query], conn)
            conn.close()
            
            st.dataframe(df, use_container_width=True)
            
            # Auto-generate visualization
            if len(df.columns) == 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                            title=selected_query)
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Analytics Queries

### Artifacts with Most Media

```sql
SELECT 
    m.id,
    m.title,
    COUNT(media.id) as media_count
FROM artifactmetadata m
JOIN artifactmedia media ON m.id = media.artifact_id
GROUP BY m.id, m.title
ORDER BY media_count DESC
LIMIT 20;
```

### Color Analysis by Culture

```sql
SELECT 
    m.culture,
    c.color_name,
    AVG(c.percentage) as avg_percentage
FROM artifactmetadata m
JOIN artifactcolors c ON m.id = c.artifact_id
WHERE m.culture IS NOT NULL
GROUP BY m.culture, c.color_name
HAVING AVG(c.percentage) > 10
ORDER BY m.culture, avg_percentage DESC;
```

### Temporal Distribution

```sql
SELECT 
    CASE 
        WHEN dated_start < 0 THEN 'BCE'
        ELSE 'CE'
    END as era,
    COUNT(*) as artifact_count
FROM artifactmetadata
WHERE dated_start IS NOT NULL
GROUP BY era;
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, size, max_retries=3):
    """
    Fetch with exponential backoff on rate limit
    """
    for attempt in range(max_retries):
        try:
            data = fetch_artifacts(api_key, page, size)
            return data
        except Exception as e:
            if "429" in str(e):  # Rate limit
                wait_time = (2 ** attempt) * 5
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

- Verify environment variables are loaded correctly
- Check firewall rules for database host
- Ensure database user has INSERT/SELECT privileges
- For TiDB Cloud, enable public access or configure IP whitelist

### Memory Management for Large Datasets

```python
def fetch_in_batches(api_key, total_records, batch_size=100):
    """
    Process large datasets in batches
    """
    total_pages = (total_records // batch_size) + 1
    
    for page in range(1, total_pages + 1):
        data = fetch_artifacts(api_key, page, batch_size)
        artifacts = data.get('records', [])
        
        # Transform and load immediately
        metadata_df = transform_artifact_metadata(artifacts)
        media_df = transform_artifact_media(artifacts)
        colors_df = transform_artifact_colors(artifacts)
        
        load_all_data(metadata_df, media_df, colors_df)
        
        # Clear memory
        del metadata_df, media_df, colors_df
```

## Best Practices

1. **Always use environment variables** for sensitive credentials
2. **Implement pagination** for large API responses
3. **Use batch inserts** for database performance
4. **Add error handling** around API calls and DB operations
5. **Cache API responses** when developing to avoid rate limits
6. **Index foreign keys** in database for query performance
7. **Validate data types** before database insertion
