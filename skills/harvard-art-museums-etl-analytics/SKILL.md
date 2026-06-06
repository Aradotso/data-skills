---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for museum artifact data
  - create a data engineering app with Harvard Art Museums API
  - set up SQL analytics for art museum collections
  - visualize Harvard museum data with Streamlit
  - implement artifact data collection and transformation
  - query and analyze art museum metadata with SQL
  - design a relational database for museum artifacts
  - create interactive dashboards for cultural heritage data
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for cultural heritage data.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Collect artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON into relational database tables
- **SQL Database**: Structured storage with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **Analytics**: 20+ predefined SQL queries for insights on culture, century, media, and color patterns
- **Visualization**: Interactive Plotly dashboards within Streamlit

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

### Prerequisites

- Python 3.8+
- MySQL or TiDB Cloud database
- Harvard Art Museums API key (obtain from [https://www.harvardartmuseums.org/collections/api](https://www.harvardartmuseums.org/collections/api))

### Setup

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

### Required Dependencies

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Running the Application

```bash
# Launch the Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Configuration

### Database Connection

```python
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

def get_db_connection():
    """Establish MySQL database connection"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    return connection
```

### API Configuration

```python
import requests
import os

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    response = requests.get(BASE_URL, params=params)
    response.raise_for_status()
    return response.json()
```

## Database Schema

### Create Tables

```python
def create_tables(connection):
    """Create database schema for artifacts"""
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            division VARCHAR(255),
            dated VARCHAR(255),
            medium VARCHAR(500),
            dimensions VARCHAR(500),
            creditline TEXT,
            accessionyear INT,
            url VARCHAR(500),
            primaryimageurl TEXT
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            iiifbaseuri VARCHAR(500),
            baseimageurl TEXT,
            format VARCHAR(50),
            height INT,
            width INT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()
    cursor.close()
```

## ETL Pipeline

### Extract and Transform

```python
import pandas as pd

def extract_transform_artifacts(num_pages=5):
    """Extract and transform artifact data from API"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page, size=100)
        
        for record in data.get('records', []):
            artifact = {
                'id': record.get('id'),
                'title': record.get('title'),
                'culture': record.get('culture'),
                'century': record.get('century'),
                'classification': record.get('classification'),
                'department': record.get('department'),
                'division': record.get('division'),
                'dated': record.get('dated'),
                'medium': record.get('medium'),
                'dimensions': record.get('dimensions'),
                'creditline': record.get('creditline'),
                'accessionyear': record.get('accessionyear'),
                'url': record.get('url'),
                'primaryimageurl': record.get('primaryimageurl')
            }
            all_artifacts.append(artifact)
    
    return pd.DataFrame(all_artifacts)
```

### Load Data

```python
def load_artifact_metadata(df, connection):
    """Load artifact metadata into database"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, 
         division, dated, medium, dimensions, creditline, 
         accessionyear, url, primaryimageurl)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    for _, row in df.iterrows():
        cursor.execute(insert_query, tuple(row))
    
    connection.commit()
    cursor.close()

def load_artifact_media(artifact_id, images, connection):
    """Load media data for an artifact"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, iiifbaseuri, baseimageurl, format, height, width)
        VALUES (%s, %s, %s, %s, %s, %s)
    """
    
    for image in images:
        data = (
            artifact_id,
            image.get('iiifbaseuri'),
            image.get('baseimageurl'),
            image.get('format'),
            image.get('height'),
            image.get('width')
        )
        cursor.execute(insert_query, data)
    
    connection.commit()
    cursor.close()

def load_artifact_colors(artifact_id, colors, connection):
    """Load color data for an artifact"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    for color in colors:
        data = (
            artifact_id,
            color.get('color'),
            color.get('spectrum'),
            color.get('hue'),
            color.get('percent')
        )
        cursor.execute(insert_query, data)
    
    connection.commit()
    cursor.close()
```

## Analytics Queries

### Sample SQL Analytics

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN primaryimageurl IS NOT NULL THEN 'Has Image' ELSE 'No Image' END as image_status,
            COUNT(*) as count
        FROM artifactmetadata
        GROUP BY image_status
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Artifacts with Most Colors": """
        SELECT m.title, m.culture, COUNT(c.color) as color_count
        FROM artifactmetadata m
        JOIN artifactcolors c ON m.id = c.artifact_id
        GROUP BY m.id, m.title, m.culture
        ORDER BY color_count DESC
        LIMIT 10
    """
}

def execute_query(query, connection):
    """Execute SQL query and return results as DataFrame"""
    cursor = connection.cursor()
    cursor.execute(query)
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    cursor.close()
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    connection = get_db_connection()
    
    if page == "ETL Pipeline":
        show_etl_page(connection)
    elif page == "SQL Analytics":
        show_analytics_page(connection)
    elif page == "Visualizations":
        show_visualizations_page(connection)
    
    connection.close()

def show_etl_page(connection):
    """Display ETL pipeline controls"""
    st.header("📥 ETL Pipeline")
    
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=100, value=5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            df = extract_transform_artifacts(num_pages)
            st.success(f"Extracted {len(df)} artifacts")
        
        with st.spinner("Loading data into database..."):
            load_artifact_metadata(df, connection)
            st.success("Data loaded successfully!")
        
        st.dataframe(df.head(10))

def show_analytics_page(connection):
    """Display analytics queries"""
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        st.code(query, language='sql')
        
        df = execute_query(query, connection)
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], title=query_name)
            st.plotly_chart(fig, use_container_width=True)

def show_visualizations_page(connection):
    """Display custom visualizations"""
    st.header("📈 Data Visualizations")
    
    # Culture distribution
    df = execute_query(ANALYTICS_QUERIES["Artifacts by Culture"], connection)
    fig = px.pie(df, values='artifact_count', names='culture', title='Artifacts by Culture')
    st.plotly_chart(fig)
    
    # Century timeline
    df = execute_query(ANALYTICS_QUERIES["Artifacts by Century"], connection)
    fig = px.line(df, x='century', y='count', title='Artifacts Timeline by Century')
    st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(page, delay=0.5):
    """Fetch data with rate limiting"""
    time.sleep(delay)  # Prevent API throttling
    return fetch_artifacts(page=page)
```

### Batch Processing

```python
def batch_insert(records, batch_size=100):
    """Insert records in batches for performance"""
    for i in range(0, len(records), batch_size):
        batch = records[i:i+batch_size]
        # Insert batch
        connection.commit()
```

### Error Handling

```python
def safe_fetch(page):
    """Fetch with error handling"""
    try:
        return fetch_artifacts(page=page)
    except requests.exceptions.RequestException as e:
        print(f"Error fetching page {page}: {e}")
        return None
```

## Troubleshooting

### API Connection Issues

- **Problem**: `401 Unauthorized`
- **Solution**: Verify `HARVARD_API_KEY` environment variable is set correctly

### Database Connection Errors

- **Problem**: `Can't connect to MySQL server`
- **Solution**: Check database credentials and ensure MySQL service is running

```python
# Test database connection
try:
    conn = get_db_connection()
    print("Database connected successfully!")
    conn.close()
except Exception as e:
    print(f"Connection failed: {e}")
```

### Missing Data in Queries

- **Problem**: Queries return empty results
- **Solution**: Ensure ETL pipeline has run and data is loaded

```python
# Check table row counts
def check_data(connection):
    cursor = connection.cursor()
    cursor.execute("SELECT COUNT(*) FROM artifactmetadata")
    count = cursor.fetchone()[0]
    print(f"Artifacts in database: {count}")
    cursor.close()
```

### Streamlit Performance

- **Problem**: Slow dashboard loading
- **Solution**: Add caching to database queries

```python
@st.cache_data(ttl=3600)
def cached_query(query):
    connection = get_db_connection()
    result = execute_query(query, connection)
    connection.close()
    return result
```
