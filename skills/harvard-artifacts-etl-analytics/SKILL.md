---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data with SQL storage and Streamlit visualization
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create a data engineering project with Harvard artifacts API
  - set up analytics dashboard for museum collection data
  - extract and analyze Harvard Art Museums artifacts
  - build streamlit app for art collection analytics
  - query and visualize museum artifact data with SQL
  - implement end-to-end data pipeline for art museums
  - process Harvard API data into relational database
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering & Analytics App is an end-to-end data pipeline that demonstrates real-world ETL practices using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational structures, loads data into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics through Streamlit dashboards with Plotly visualizations.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Core Dependencies:**
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

```env
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Schema

The application uses three main tables with relational structure:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    department VARCHAR(255),
    classification VARCHAR(255),
    objectnumber VARCHAR(100),
    division VARCHAR(255),
    url TEXT,
    primaryimageurl TEXT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    mediatype VARCHAR(100),
    mediaurl TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Core Components

### 1. API Data Collection

**Fetching artifacts with pagination:**

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_artifacts=100):
    """Extract artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(artifacts) < num_artifacts:
        params = {
            'apikey': api_key,
            'page': page,
            'size': size,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data['records'])
            
            if len(data['records']) < size:
                break  # No more data
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_artifacts]
```

### 2. ETL Pipeline

**Transform and load data:**

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifacts(raw_data):
    """Transform nested JSON into flat DataFrames"""
    
    # Metadata transformation
    metadata = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Untitled'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'classification': artifact.get('classification'),
            'objectnumber': artifact.get('objectnumber'),
            'division': artifact.get('division'),
            'url': artifact.get('url'),
            'primaryimageurl': artifact.get('primaryimageurl')
        })
        
        # Extract media (images)
        if 'images' in artifact and artifact['images']:
            for img in artifact['images']:
                media_list.append({
                    'artifact_id': artifact.get('id'),
                    'mediatype': 'image',
                    'mediaurl': img.get('baseimageurl')
                })
        
        # Extract color data
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                colors_list.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'percentage': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

def load_to_database(df_metadata, df_media, df_colors):
    """Load DataFrames into SQL database"""
    
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT')),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        if connection.is_connected():
            cursor = connection.cursor()
            
            # Insert metadata
            for _, row in df_metadata.iterrows():
                cursor.execute("""
                    INSERT INTO artifactmetadata 
                    (id, title, culture, century, dated, department, 
                     classification, objectnumber, division, url, primaryimageurl)
                    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                    ON DUPLICATE KEY UPDATE title=VALUES(title)
                """, tuple(row))
            
            # Insert media
            for _, row in df_media.iterrows():
                cursor.execute("""
                    INSERT INTO artifactmedia (artifact_id, mediatype, mediaurl)
                    VALUES (%s, %s, %s)
                """, tuple(row))
            
            # Insert colors
            for _, row in df_colors.iterrows():
                cursor.execute("""
                    INSERT INTO artifactcolors 
                    (artifact_id, color, spectrum, percentage)
                    VALUES (%s, %s, %s, %s)
                """, tuple(row))
            
            connection.commit()
            print("Data loaded successfully!")
            
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 3. SQL Analytics Queries

**Example analytical queries:**

```python
# Sample queries for the analytics dashboard

ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
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
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Color Usage Analysis": """
        SELECT color, COUNT(*) as usage_count, 
               AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
            COUNT(*) as total_media_items
        FROM artifactmedia m
    """,
    
    "Artifacts Without Images": """
        SELECT COUNT(*) as count
        FROM artifactmetadata
        WHERE primaryimageurl IS NULL
    """
}

def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT')),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```

### 4. Streamlit Dashboard

**Building the interactive interface:**

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    st.sidebar.header("Navigation")
    page = st.sidebar.radio(
        "Select Page",
        ["Data Collection", "Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "Analytics":
        show_analytics()
    else:
        show_visualizations()

def show_data_collection():
    """ETL Pipeline Interface"""
    st.header("📥 Data Collection & ETL")
    
    num_artifacts = st.number_input(
        "Number of artifacts to collect",
        min_value=10,
        max_value=1000,
        value=100,
        step=10
    )
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            raw_data = fetch_artifacts(num_artifacts)
            st.success(f"Fetched {len(raw_data)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_meta, df_media, df_colors = transform_artifacts(raw_data)
            st.success("Data transformed")
        
        with st.spinner("Loading to database..."):
            load_to_database(df_meta, df_media, df_colors)
            st.success("Data loaded successfully!")

def show_analytics():
    """SQL Analytics Interface"""
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        
        with st.spinner("Executing query..."):
            df_result = execute_query(query)
            
        st.subheader("Query Results")
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) == 2:
            fig = px.bar(
                df_result,
                x=df_result.columns[0],
                y=df_result.columns[1],
                title=query_name
            )
            st.plotly_chart(fig, use_container_width=True)

def show_visualizations():
    """Pre-built Visualizations"""
    st.header("📈 Data Visualizations")
    
    col1, col2 = st.columns(2)
    
    with col1:
        df_culture = execute_query(ANALYTICS_QUERIES["Artifacts by Culture"])
        fig1 = px.pie(
            df_culture,
            names='culture',
            values='count',
            title='Artifact Distribution by Culture'
        )
        st.plotly_chart(fig1)
    
    with col2:
        df_century = execute_query(ANALYTICS_QUERIES["Artifacts by Century"])
        fig2 = px.bar(
            df_century,
            x='century',
            y='count',
            title='Artifacts by Century'
        )
        st.plotly_chart(fig2)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Batch Processing Large Datasets

```python
def batch_insert(df, table_name, batch_size=1000):
    """Insert data in batches for better performance"""
    connection = mysql.connector.connect(...)
    cursor = connection.cursor()
    
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        # Insert batch logic
        connection.commit()
    
    connection.close()
```

### Error Handling in ETL

```python
def safe_extract(artifact, key, default=None):
    """Safely extract nested JSON values"""
    try:
        return artifact.get(key, default)
    except (KeyError, TypeError):
        return default
```

## Troubleshooting

**API Rate Limiting:**
- The Harvard API has rate limits; add delays between requests
- Use `time.sleep(0.5)` between paginated calls

**Database Connection Errors:**
- Verify credentials in `.env` file
- Check if database exists: `CREATE DATABASE IF NOT EXISTS harvard_artifacts;`
- Ensure tables are created before loading data

**Missing Data:**
- Not all artifacts have images; use `hasimage=1` parameter
- Handle NULL values in queries with `WHERE field IS NOT NULL`

**Memory Issues with Large Datasets:**
- Process data in chunks
- Use generators instead of loading all data at once
- Consider using database cursors for large result sets
