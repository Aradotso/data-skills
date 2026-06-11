---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I extract data from Harvard Art Museums API
  - build an ETL pipeline for art museum data
  - create analytics dashboard with Streamlit and museum data
  - analyze Harvard artifacts collection with SQL
  - set up data engineering pipeline for museum API
  - visualize art collection data with Plotly
  - transform nested JSON art data into relational tables
  - query museum artifact metadata with Python
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering solution for collecting, transforming, and analyzing Harvard Art Museums data. It demonstrates real-world ETL pipelines, SQL analytics, and interactive visualization using Streamlit.

## What It Does

- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables (metadata, media, colors)
- **Loads** data into MySQL/TiDB Cloud with optimized batch inserts
- **Analyzes** using 20+ predefined SQL queries for insights
- **Visualizes** results through interactive Plotly charts in Streamlit

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

Create a `.env` file:
```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Schema

The project uses three main tables with relationships:

```sql
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
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
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
        raise Exception(f"API request failed: {response.status_code}")

# Example: Fetch multiple pages
def collect_artifacts(total_pages=5):
    all_artifacts = []
    
    for page in range(1, total_pages + 1):
        records, info = fetch_artifacts(page=page, size=100)
        all_artifacts.extend(records)
        print(f"Fetched page {page}/{total_pages} - Total: {len(all_artifacts)}")
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifacts(raw_data):
    """Transform nested JSON into normalized dataframes"""
    
    # Metadata table
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', ''),
            'period': artifact.get('period', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'dated': artifact.get('dated', ''),
            'url': artifact.get('url', '')
        })
        
        # Extract media information
        if artifact.get('images'):
            for image in artifact['images']:
                media_records.append({
                    'artifact_id': artifact.get('id'),
                    'image_url': image.get('baseimageurl', ''),
                    'media_type': 'image'
                })
        
        # Extract color information
        if artifact.get('colors'):
            for color_obj in artifact['colors']:
                color_records.append({
                    'artifact_id': artifact.get('id'),
                    'color': color_obj.get('color', ''),
                    'percentage': color_obj.get('percent', 0)
                })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Load transformed data into MySQL database"""
    
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    cursor = connection.cursor()
    
    # Insert metadata (batch)
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, dated, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    cursor.executemany(metadata_query, metadata_df.values.tolist())
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia (artifact_id, image_url, media_type)
        VALUES (%s, %s, %s)
    """
    cursor.executemany(media_query, media_df.values.tolist())
    
    # Insert colors
    colors_query = """
        INSERT INTO artifactcolors (artifact_id, color, percentage)
        VALUES (%s, %s, %s)
    """
    cursor.executemany(colors_query, colors_df.values.tolist())
    
    connection.commit()
    cursor.close()
    connection.close()
    
    print(f"Loaded {len(metadata_df)} artifacts successfully")
```

### 3. Analytics Queries

```python
# Example analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY artifact_count DESC
    """,
    
    "Top Colors Across Collection": """
        SELECT color, 
               COUNT(DISTINCT artifact_id) as artifact_count,
               AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts with Most Images": """
        SELECT m.title, m.culture, COUNT(a.media_id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia a ON m.id = a.artifact_id
        GROUP BY m.id, m.title, m.culture
        ORDER BY image_count DESC
        LIMIT 20
    """,
    
    "Classification Distribution": """
        SELECT classification, COUNT(*) as count,
               ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 12
    """
}

def run_analytics_query(query_name):
    """Execute analytics query and return results"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df = pd.read_sql(ANALYTICS_QUERIES[query_name], connection)
    connection.close()
    
    return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Collection Analytics")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Module",
        ["Data Collection", "Analytics Dashboard", "Visualizations"]
    )
    
    if page == "Data Collection":
        st.header("📥 Collect Data from API")
        
        num_pages = st.number_input("Number of pages to fetch", 1, 50, 5)
        
        if st.button("Start ETL Pipeline"):
            with st.spinner("Fetching data from API..."):
                raw_data = collect_artifacts(total_pages=num_pages)
                st.success(f"Fetched {len(raw_data)} artifacts")
            
            with st.spinner("Transforming data..."):
                metadata_df, media_df, colors_df = transform_artifacts(raw_data)
                st.success("Data transformed successfully")
            
            with st.spinner("Loading to database..."):
                load_to_database(metadata_df, media_df, colors_df)
                st.success("Data loaded to database")
    
    elif page == "Analytics Dashboard":
        st.header("📊 SQL Analytics")
        
        query_name = st.selectbox("Select Query", list(ANALYTICS_QUERIES.keys()))
        
        if st.button("Run Query"):
            df = run_analytics_query(query_name)
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                           title=query_name)
                st.plotly_chart(fig, use_container_width=True)
    
    elif page == "Visualizations":
        st.header("📈 Interactive Visualizations")
        
        # Culture distribution
        df_culture = run_analytics_query("Artifacts by Culture")
        fig = px.pie(df_culture, names='culture', values='count',
                    title='Artifact Distribution by Culture')
        st.plotly_chart(fig)
        
        # Century timeline
        df_century = run_analytics_query("Artifacts by Century")
        fig = px.line(df_century, x='century', y='artifact_count',
                     title='Artifacts Across Centuries')
        st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_artifact_id():
    """Get the last artifact ID to support incremental loads"""
    connection = mysql.connector.connect(...)
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0
```

### Error Handling for API Requests

```python
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Retry {attempt + 1} after {wait_time}s")
            time.sleep(wait_time)
```

## Troubleshooting

**API Rate Limiting**: Add delays between requests
```python
import time
time.sleep(1)  # Wait 1 second between requests
```

**Database Connection Errors**: Check credentials and network access
```python
# Test connection
try:
    connection = mysql.connector.connect(...)
    print("Database connected successfully")
except Error as e:
    print(f"Connection failed: {e}")
```

**Missing Data Fields**: Use `.get()` with defaults
```python
culture = artifact.get('culture', 'Unknown')
```

**Memory Issues with Large Datasets**: Process in batches
```python
BATCH_SIZE = 1000
for i in range(0, len(df), BATCH_SIZE):
    batch = df[i:i+BATCH_SIZE]
    load_batch_to_db(batch)
```
