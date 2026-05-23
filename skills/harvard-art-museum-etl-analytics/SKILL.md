---
name: harvard-art-museum-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API data with Python, SQL, and Streamlit
triggers:
  - build an art museum data pipeline
  - create ETL pipeline for Harvard museum API
  - analyze Harvard Art Museums collection data
  - set up museum artifact analytics dashboard
  - extract and visualize art collection data
  - build museum data engineering project
  - create cultural artifact analytics app
  - process Harvard museum API with SQL
---

# Harvard Art Museum ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is an end-to-end data engineering solution that demonstrates production-grade ETL pipelines using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational structures, loads data into SQL databases, and provides interactive analytics dashboards via Streamlit.

**Architecture**: API → ETL → SQL → Analytics → Visualization

**Key Components**:
- API integration with Harvard Art Museums
- ETL pipeline with pagination and rate limiting
- Relational database design (MySQL/TiDB Cloud)
- SQL analytics queries
- Streamlit dashboard with Plotly visualizations

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Core Dependencies**:
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

### Get Harvard API Key

Visit [Harvard Art Museums API](https://harvardartmuseums.org/collections/api) to request a free API key.

### Database Setup

```sql
CREATE DATABASE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    period VARCHAR(255),
    dated VARCHAR(255),
    url TEXT,
    totalpageviews INT,
    totaluniquepageviews INT
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri TEXT,
    baseimageurl TEXT,
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
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
# Start the Streamlit app
streamlit run app.py
```

The app will open at `http://localhost:8501`

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
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
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

def fetch_all_artifacts(max_pages=10):
    """
    Fetch multiple pages with rate limiting
    """
    import time
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page, size=100)
        all_artifacts.extend(data['records'])
        
        # Rate limiting
        time.sleep(1)
        
        if not data.get('info', {}).get('next'):
            break
    
    return all_artifacts
```

### Transform: Process JSON to Relational Format

```python
import pandas as pd

def transform_metadata(artifacts):
    """
    Transform artifact metadata into structured DataFrame
    """
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'technique': artifact.get('technique', '')[:500],
            'period': artifact.get('period', ''),
            'dated': artifact.get('dated', ''),
            'url': artifact.get('url', ''),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        })
    
    return pd.DataFrame(metadata)

def transform_media(artifacts):
    """
    Extract media/image information
    """
    media = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media.append({
                'artifact_id': artifact_id,
                'iiifbaseuri': img.get('iiifbaseuri', ''),
                'baseimageurl': img.get('baseimageurl', ''),
                'format': img.get('format', '')
            })
    
    return pd.DataFrame(media)

def transform_colors(artifacts):
    """
    Extract color information
    """
    colors = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        color_data = artifact.get('colors', [])
        
        for color in color_data:
            colors.append({
                'artifact_id': artifact_id,
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            })
    
    return pd.DataFrame(colors)
```

### Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """
    Create database connection
    """
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df_metadata):
    """
    Batch insert metadata using executemany for performance
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, 
     technique, period, dated, url, totalpageviews, totaluniquepageviews)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data = df_metadata.values.tolist()
    cursor.executemany(insert_query, data)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(data)} metadata records")

def load_media(df_media):
    """
    Load media/image data
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, iiifbaseuri, baseimageurl, format)
    VALUES (%s, %s, %s, %s)
    """
    
    data = df_media.values.tolist()
    cursor.executemany(insert_query, data)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(data)} media records")

def load_colors(df_colors):
    """
    Load color data
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data = df_colors.values.tolist()
    cursor.executemany(insert_query, data)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(data)} color records")
```

## Complete ETL Script

```python
# etl_pipeline.py
import os
from dotenv import load_dotenv

load_dotenv()

def run_etl_pipeline(max_pages=5):
    """
    Execute complete ETL pipeline
    """
    print("Starting ETL Pipeline...")
    
    # Extract
    print("Step 1: Extracting data from API...")
    artifacts = fetch_all_artifacts(max_pages=max_pages)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    print("Step 2: Transforming data...")
    df_metadata = transform_metadata(artifacts)
    df_media = transform_media(artifacts)
    df_colors = transform_colors(artifacts)
    
    # Load
    print("Step 3: Loading data to database...")
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    
    print("ETL Pipeline completed successfully!")

if __name__ == "__main__":
    run_etl_pipeline(max_pages=10)
```

## Streamlit Dashboard Implementation

```python
# app.py
import streamlit as st
import pandas as pd
import plotly.express as px
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

st.set_page_config(
    page_title="Harvard Artifacts Analytics",
    page_icon="🏛️",
    layout="wide"
)

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

def execute_query(query):
    """
    Execute SQL query and return results as DataFrame
    """
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Sidebar with query selection
st.sidebar.header("Analytics Queries")

queries = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    "Top Departments": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    "Most Viewed Artifacts": """
        SELECT title, culture, totalpageviews
        FROM artifactmetadata
        WHERE totalpageviews > 0
        ORDER BY totalpageviews DESC
        LIMIT 15
    """,
    "Color Distribution": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    "Media Format Analysis": """
        SELECT format, COUNT(*) as count
        FROM artifactmedia
        WHERE format IS NOT NULL
        GROUP BY format
        ORDER BY count DESC
    """
}

selected_query = st.sidebar.selectbox("Select Analysis", list(queries.keys()))

# Execute and display results
if st.sidebar.button("Run Analysis"):
    with st.spinner("Running query..."):
        df = execute_query(queries[selected_query])
        
        col1, col2 = st.columns([1, 2])
        
        with col1:
            st.subheader("Query Results")
            st.dataframe(df, use_container_width=True)
        
        with col2:
            st.subheader("Visualization")
            if len(df) > 0:
                x_col = df.columns[0]
                y_col = df.columns[1]
                
                fig = px.bar(
                    df,
                    x=x_col,
                    y=y_col,
                    title=selected_query,
                    labels={x_col: x_col.title(), y_col: y_col.title()}
                )
                st.plotly_chart(fig, use_container_width=True)

# Summary statistics
st.header("📊 Collection Summary")

col1, col2, col3 = st.columns(3)

with col1:
    total_artifacts = execute_query("SELECT COUNT(*) as count FROM artifactmetadata")
    st.metric("Total Artifacts", total_artifacts['count'].iloc[0])

with col2:
    total_images = execute_query("SELECT COUNT(*) as count FROM artifactmedia")
    st.metric("Total Images", total_images['count'].iloc[0])

with col3:
    unique_cultures = execute_query("SELECT COUNT(DISTINCT culture) as count FROM artifactmetadata WHERE culture IS NOT NULL")
    st.metric("Unique Cultures", unique_cultures['count'].iloc[0])
```

## Common Analytical Queries

```sql
-- Artifacts without images
SELECT COUNT(*) 
FROM artifactmetadata m
LEFT JOIN artifactmedia media ON m.id = media.artifact_id
WHERE media.artifact_id IS NULL;

-- Most common color per culture
SELECT m.culture, c.color, COUNT(*) as frequency
FROM artifactmetadata m
JOIN artifactcolors c ON m.id = c.artifact_id
WHERE m.culture IS NOT NULL
GROUP BY m.culture, c.color
ORDER BY m.culture, frequency DESC;

-- Classification distribution by department
SELECT department, classification, COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL AND classification IS NOT NULL
GROUP BY department, classification
ORDER BY department, count DESC;

-- Average pageviews by century
SELECT century, AVG(totalpageviews) as avg_views
FROM artifactmetadata
WHERE century IS NOT NULL AND totalpageviews > 0
GROUP BY century
ORDER BY avg_views DESC;
```

## Troubleshooting

### API Rate Limiting

```python
import time
from functools import wraps

def rate_limit(calls_per_second=1):
    """
    Decorator to rate limit API calls
    """
    min_interval = 1.0 / calls_per_second
    
    def decorator(func):
        last_called = [0.0]
        
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

@rate_limit(calls_per_second=1)
def fetch_artifacts_with_limit(page=1):
    return fetch_artifacts(page=page)
```

### Database Connection Pooling

```python
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    port=int(os.getenv('DB_PORT', 3306)),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_pooled_connection():
    return db_pool.get_connection()
```

### Handling Large Datasets

```python
def batch_load_data(df, table_name, batch_size=1000):
    """
    Load data in batches to avoid memory issues
    """
    conn = get_db_connection()
    
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        batch.to_sql(
            table_name,
            conn,
            if_exists='append',
            index=False,
            method='multi'
        )
        print(f"Loaded batch {i//batch_size + 1}")
    
    conn.close()
```

## Best Practices

1. **Always use environment variables** for sensitive credentials
2. **Implement retry logic** for API calls to handle transient failures
3. **Use batch inserts** (`executemany`) for better database performance
4. **Add error logging** to track ETL pipeline failures
5. **Validate data** before loading into database
6. **Create indexes** on frequently queried columns (culture, century, department)
7. **Schedule ETL jobs** using cron or Apache Airflow for production environments
