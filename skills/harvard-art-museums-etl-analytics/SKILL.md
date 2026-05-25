---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for museum artifact data
  - create a data engineering project with Harvard Art Museums API
  - set up analytics dashboard for art collection data
  - extract and transform Harvard museum artifacts
  - build streamlit app for art collection analytics
  - create SQL analytics for museum artifact data
  - develop end-to-end data pipeline for art museums
  - visualize Harvard Art Museums collection data
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL analytics, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App is a complete data pipeline that:

- **Extracts** artifact data from the Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables (artifacts, media, colors)
- **Loads** structured data into MySQL/TiDB Cloud databases
- **Analyzes** data using 20+ predefined SQL queries
- **Visualizes** results through interactive Streamlit dashboards with Plotly charts

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
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
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

**Get your Harvard API key:** Register at https://www.harvardartmuseums.org/collections/api

### Database Setup

The project requires three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    division VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    technique VARCHAR(500),
    accessionyear INT,
    totalpageviews INT,
    totaluniquepageviews INT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(50),
    baseimageurl VARCHAR(500),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
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
    url = f"https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'fields': 'id,title,culture,century,classification,division,department,dated,technique,accessionyear,totalpageviews,totaluniquepageviews,images,colors'
    }
    
    response = requests.get(url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Fetch multiple pages with rate limiting
import time

def collect_artifacts(num_pages=5):
    """Collect artifacts from multiple pages"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        try:
            data = fetch_artifacts(page=page)
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            print(f"Collected page {page}: {len(artifacts)} artifacts")
            time.sleep(1)  # Rate limiting
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifacts(raw_artifacts):
    """Transform raw API data into normalized tables"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_artifacts:
        # Metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'division': artifact.get('division', '')[:255],
            'department': artifact.get('department', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'technique': artifact.get('technique', '')[:500],
            'accessionyear': artifact.get('accessionyear'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        })
        
        # Media/Images
        images = artifact.get('images', [])
        for image in images:
            media_records.append({
                'artifact_id': artifact.get('id'),
                'media_type': 'image',
                'baseimageurl': image.get('baseimageurl', '')[:500],
                'height': image.get('height'),
                'width': image.get('width')
            })
        
        # Colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color', '')[:50],
                'hue': color.get('hue', '')[:50],
                'percent': color.get('percent', 0.0)
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Load transformed data into MySQL database"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        cursor = connection.cursor()
        
        # Insert metadata
        for _, row in metadata_df.iterrows():
            insert_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, division, department, 
             dated, technique, accessionyear, totalpageviews, totaluniquepageviews)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
            """
            cursor.execute(insert_query, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            insert_query = """
            INSERT INTO artifactmedia 
            (artifact_id, media_type, baseimageurl, height, width)
            VALUES (%s, %s, %s, %s, %s)
            """
            cursor.execute(insert_query, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            insert_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, hue, percent)
            VALUES (%s, %s, %s, %s)
            """
            cursor.execute(insert_query, tuple(row))
        
        connection.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 3. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Collection Analytics")
    st.markdown("**End-to-End Data Engineering & Analytics Platform**")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigate",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    else:
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection from Harvard API")
    
    num_pages = st.number_input("Number of pages to collect", 
                                 min_value=1, max_value=50, value=5)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Collecting artifacts..."):
            artifacts = collect_artifacts(num_pages=num_pages)
            
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
            
        st.success(f"✅ Loaded {len(metadata_df)} artifacts successfully!")

def show_sql_analytics():
    st.header("📊 SQL Analytics Dashboard")
    
    queries = {
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
        "Top Departments": """
            SELECT department, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE department IS NOT NULL 
            GROUP BY department 
            ORDER BY count DESC
        """,
        "Color Distribution": """
            SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY count DESC 
            LIMIT 10
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        df = execute_query(queries[selected_query])
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=selected_query)
            st.plotly_chart(fig)

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df = pd.read_sql(query, connection)
    connection.close()
    return df

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Batch Processing with Error Handling

```python
def batch_load_artifacts(artifacts, batch_size=100):
    """Load artifacts in batches with error handling"""
    total = len(artifacts)
    
    for i in range(0, total, batch_size):
        batch = artifacts[i:i + batch_size]
        try:
            metadata_df, media_df, colors_df = transform_artifacts(batch)
            load_to_database(metadata_df, media_df, colors_df)
            print(f"Processed batch {i//batch_size + 1}: {len(batch)} artifacts")
        except Exception as e:
            print(f"Error in batch {i//batch_size + 1}: {e}")
            continue
```

### Advanced Analytics Queries

```python
# Artifacts with images
query_with_images = """
SELECT 
    a.title, 
    a.culture, 
    COUNT(m.id) as image_count
FROM artifactmetadata a
LEFT JOIN artifactmedia m ON a.id = m.artifact_id
GROUP BY a.id
HAVING image_count > 0
ORDER BY image_count DESC
LIMIT 20
"""

# Color analysis by department
color_by_dept = """
SELECT 
    a.department,
    c.color,
    COUNT(*) as count,
    AVG(c.percent) as avg_percent
FROM artifactmetadata a
JOIN artifactcolors c ON a.id = c.artifact_id
WHERE a.department IS NOT NULL
GROUP BY a.department, c.color
ORDER BY a.department, count DESC
"""
```

## Troubleshooting

### API Rate Limiting
```python
import time
from functools import wraps

def rate_limit(calls_per_second=1):
    """Decorator to rate limit API calls"""
    min_interval = 1.0 / calls_per_second
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            wait_time = min_interval - elapsed
            if wait_time > 0:
                time.sleep(wait_time)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_second=2)
def fetch_artifacts_limited(page=1):
    return fetch_artifacts(page=page)
```

### Database Connection Pooling
```python
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_connection():
    return db_pool.get_connection()
```

### Handling Missing Data
```python
def safe_transform(artifact):
    """Safely extract data with defaults"""
    return {
        'id': artifact.get('id'),
        'title': artifact.get('title', 'Untitled')[:500],
        'culture': artifact.get('culture', 'Unknown')[:255],
        'century': artifact.get('century', 'Unknown')[:100],
        'accessionyear': artifact.get('accessionyear') or None,
        'totalpageviews': artifact.get('totalpageviews', 0) or 0
    }
```

This skill provides AI agents with the knowledge to help developers build complete ETL pipelines and analytics applications using the Harvard Art Museums API, from data collection to interactive visualization.
