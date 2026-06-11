---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines with Harvard Art Museums API data, SQL analytics, and Streamlit visualizations
triggers:
  - how do I extract data from Harvard Art Museums API
  - build an ETL pipeline with museum artifact data
  - create analytics dashboard with Streamlit and museum data
  - set up Harvard Art Museums data engineering project
  - visualize museum collection data with SQL and Plotly
  - implement artifact metadata ETL workflow
  - query Harvard Art Museums API with pagination
  - create museum data analytics application
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build complete data engineering and analytics pipelines using the Harvard Art Museums API. The project demonstrates real-world ETL patterns, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:
- **API Integration**: Paginated data collection from Harvard Art Museums API with rate limiting
- **ETL Pipeline**: Extract artifact metadata, transform nested JSON to relational format, load into SQL databases
- **Database Design**: Three normalized tables (artifactmetadata, artifactmedia, artifactcolors) with relationships
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboards**: Streamlit UI with Plotly visualizations

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### API Key Setup

1. Get a free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Create `.env` file in project root:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=your_database_name
```

### Database Setup

```python
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

# Database connection
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Create tables
def create_tables():
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            period VARCHAR(200),
            technique VARCHAR(500),
            medium VARCHAR(500),
            division VARCHAR(200)
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_url VARCHAR(1000),
            media_type VARCHAR(100),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            percentage FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    conn.commit()
    cursor.close()
    conn.close()
```

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
        raise Exception(f"API request failed: {response.status_code}")

def fetch_all_artifacts(max_pages=10):
    """
    Fetch multiple pages of artifacts
    """
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page)
        all_artifacts.extend(data.get('records', []))
        
        # Rate limiting
        import time
        time.sleep(1)
    
    return all_artifacts
```

### Transform: Process JSON to DataFrames

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform nested JSON into relational dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Metadata extraction
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'division': artifact.get('division')
        }
        metadata_list.append(metadata)
        
        # Media extraction
        if artifact.get('images'):
            for image in artifact['images']:
                media_list.append({
                    'artifact_id': artifact.get('id'),
                    'image_url': image.get('baseimageurl'),
                    'media_type': 'image'
                })
        
        # Color extraction
        if artifact.get('colors'):
            for color in artifact['colors']:
                colors_list.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'percentage': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Insert into Database

```python
def load_to_database(metadata_df, media_df, colors_df):
    """
    Batch insert dataframes into SQL database
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Load metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT IGNORE INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             dated, period, technique, medium, division)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Load media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia (artifact_id, image_url, media_type)
            VALUES (%s, %s, %s)
        """, tuple(row))
    
    # Load colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors (artifact_id, color, percentage)
            VALUES (%s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
    print(f"Loaded {len(metadata_df)} artifacts to database")
```

### Complete ETL Workflow

```python
def run_etl_pipeline(max_pages=5):
    """
    Execute complete ETL pipeline
    """
    print("Starting ETL pipeline...")
    
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_all_artifacts(max_pages=max_pages)
    
    # Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(artifacts)
    
    # Load
    print("Loading to database...")
    load_to_database(metadata_df, media_df, colors_df)
    
    print("ETL pipeline completed successfully!")
```

## Analytics Queries

### Sample Analytical Queries

```python
# Query 1: Top 10 cultures by artifact count
query_cultures = """
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
    ORDER BY count DESC
"""

# Query 3: Most common colors
query_colors = """
    SELECT color, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
    FROM artifactcolors
    GROUP BY color
    ORDER BY usage_count DESC
    LIMIT 15
"""

# Query 4: Departments with most artifacts
query_departments = """
    SELECT department, COUNT(*) as total_artifacts
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY total_artifacts DESC
"""

# Query 5: Media availability
query_media = """
    SELECT COUNT(DISTINCT artifact_id) as artifacts_with_media,
           (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts
    FROM artifactmedia
"""

def execute_query(query):
    """Execute SQL query and return results"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard

### Main Application

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Art Analytics", layout="wide")

st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar navigation
page = st.sidebar.selectbox(
    "Navigation",
    ["ETL Pipeline", "Analytics Queries", "Visualizations"]
)

if page == "ETL Pipeline":
    st.header("Run ETL Pipeline")
    
    max_pages = st.number_input("Number of pages to fetch", 
                                 min_value=1, max_value=50, value=5)
    
    if st.button("Run ETL"):
        with st.spinner("Running ETL pipeline..."):
            try:
                run_etl_pipeline(max_pages=max_pages)
                st.success("ETL pipeline completed!")
            except Exception as e:
                st.error(f"Error: {str(e)}")

elif page == "Analytics Queries":
    st.header("SQL Analytics")
    
    queries = {
        "Top Cultures": query_cultures,
        "Century Distribution": query_century,
        "Popular Colors": query_colors,
        "Department Stats": query_departments,
        "Media Coverage": query_media
    }
    
    selected_query = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        df = execute_query(queries[selected_query])
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=selected_query)
            st.plotly_chart(fig)

elif page == "Visualizations":
    st.header("Interactive Visualizations")
    
    # Culture distribution
    df_culture = execute_query(query_cultures)
    fig1 = px.bar(df_culture, x='culture', y='artifact_count',
                  title='Top 10 Cultures by Artifact Count')
    st.plotly_chart(fig1, use_container_width=True)
    
    # Color analysis
    df_colors = execute_query(query_colors)
    fig2 = px.pie(df_colors, values='usage_count', names='color',
                  title='Color Distribution Across Artifacts')
    st.plotly_chart(fig2, use_container_width=True)
```

## Common Patterns

### Error Handling for API Requests

```python
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch data with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1} after {wait_time}s")
                time.sleep(wait_time)
            else:
                raise e
```

### Incremental Data Loading

```python
def get_max_artifact_id():
    """Get the highest artifact ID already in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    return result or 0

def incremental_etl():
    """Only load new artifacts"""
    max_id = get_max_artifact_id()
    artifacts = fetch_all_artifacts()
    new_artifacts = [a for a in artifacts if a['id'] > max_id]
    
    if new_artifacts:
        metadata_df, media_df, colors_df = transform_artifacts(new_artifacts)
        load_to_database(metadata_df, media_df, colors_df)
```

## Troubleshooting

**API Rate Limiting**: Add delays between requests
```python
time.sleep(1)  # 1 second between requests
```

**Database Connection Issues**: Verify credentials and network access
```python
try:
    conn = get_db_connection()
    conn.ping(reconnect=True)
except Exception as e:
    print(f"Database connection failed: {e}")
```

**Missing Data Fields**: Handle None values gracefully
```python
artifact.get('culture', 'Unknown')
```

**Large Dataset Memory**: Use chunked processing
```python
def process_in_chunks(artifacts, chunk_size=100):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        metadata_df, media_df, colors_df = transform_artifacts(chunk)
        load_to_database(metadata_df, media_df, colors_df)
```
