---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create a data engineering workflow with Harvard API
  - set up SQL analytics for museum artifact data
  - build a Streamlit dashboard for Harvard artifacts
  - extract and transform Harvard Art Museums API data
  - query and visualize museum collection data
  - implement batch data ingestion from Harvard API
  - create relational tables from nested JSON artifact data
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON into relational SQL tables
- **Database Design**: Structured schema with artifact metadata, media, and color tables
- **SQL Analytics**: 20+ predefined analytical queries for artifact insights
- **Interactive Visualization**: Streamlit dashboards with Plotly charts

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install requirements
pip install -r requirements.txt

# Set up environment variables
cat > .env << EOF
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
EOF
```

### Get Harvard API Key

1. Visit https://docs.harvardartmuseums.org/
2. Register for a free API key
3. Add key to `.env` file

## Configuration

### Database Setup

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

# Database connection configuration
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

# Create connection
conn = mysql.connector.connect(**db_config)
cursor = conn.cursor()
```

### Create Database Schema

```python
def create_tables():
    """Create the relational database schema"""
    
    # Artifact Metadata table
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS artifactmetadata (
        id INT PRIMARY KEY,
        title VARCHAR(500),
        culture VARCHAR(200),
        period VARCHAR(200),
        century VARCHAR(100),
        classification VARCHAR(200),
        department VARCHAR(200),
        technique VARCHAR(500),
        medium VARCHAR(500),
        dated VARCHAR(200),
        division VARCHAR(200)
    )
    """)
    
    # Artifact Media table
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS artifactmedia (
        id INT AUTO_INCREMENT PRIMARY KEY,
        artifact_id INT,
        image_url VARCHAR(1000),
        FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
    )
    """)
    
    # Artifact Colors table
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
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

def collect_all_artifacts(api_key, max_records=1000):
    """Collect artifacts with pagination"""
    
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        print(f"Fetching page {page}...")
        
        data = fetch_artifacts(api_key, page=page, size=size)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_artifacts.extend(records)
        page += 1
        
        # Rate limiting
        time.sleep(0.5)
    
    return all_artifacts[:max_records]
```

### Transform: Process Nested JSON

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform nested JSON into flat DataFrames"""
    
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'period': artifact.get('period', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'technique': artifact.get('technique', ''),
            'medium': artifact.get('medium', ''),
            'dated': artifact.get('dated', ''),
            'division': artifact.get('division', '')
        }
        metadata_list.append(metadata)
        
        # Media/Images
        images = artifact.get('images', [])
        for image in images:
            media_list.append({
                'artifact_id': artifact.get('id'),
                'image_url': image.get('baseimageurl', '')
            })
        
        # Colors
        colors = artifact.get('colors', [])
        for color in colors:
            colors_list.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color', ''),
                'percentage': color.get('percent', 0.0)
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Batch Insert into SQL

```python
def load_to_database(metadata_df, media_df, colors_df):
    """Load transformed data into SQL database"""
    
    # Insert metadata
    metadata_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, classification, department, 
     technique, medium, dated, division)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE id=id
    """
    
    cursor.executemany(metadata_query, metadata_df.values.tolist())
    
    # Insert media
    media_query = """
    INSERT INTO artifactmedia (artifact_id, image_url)
    VALUES (%s, %s)
    """
    
    cursor.executemany(media_query, media_df.values.tolist())
    
    # Insert colors
    colors_query = """
    INSERT INTO artifactcolors (artifact_id, color, percentage)
    VALUES (%s, %s, %s)
    """
    
    cursor.executemany(colors_query, colors_df.values.tolist())
    
    conn.commit()
    print(f"Loaded {len(metadata_df)} artifacts successfully")
```

## Streamlit Dashboard Implementation

### Main Application

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
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        data_collection_page()
    elif page == "SQL Analytics":
        sql_analytics_page()
    elif page == "Visualizations":
        visualizations_page()

def data_collection_page():
    st.header("📥 Data Collection from Harvard API")
    
    api_key = st.text_input("Enter API Key", type="password", 
                             value=os.getenv('HARVARD_API_KEY', ''))
    
    num_records = st.slider("Number of Records", 100, 5000, 1000)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Fetching artifacts..."):
            artifacts = collect_all_artifacts(api_key, num_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
            st.success("Data loaded to database!")
```

## SQL Analytics Queries

### Common Analytical Queries

```python
# Sample analytical queries dictionary
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY artifact_count DESC
    """,
    
    "Most Common Colors": """
        SELECT color, 
               COUNT(*) as occurrences,
               AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY occurrences DESC
        LIMIT 15
    """,
    
    "Artifacts with Most Images": """
        SELECT am.id, am.title, COUNT(media.id) as image_count
        FROM artifactmetadata am
        JOIN artifactmedia media ON am.id = media.artifact_id
        GROUP BY am.id, am.title
        ORDER BY image_count DESC
        LIMIT 20
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
}

def sql_analytics_page():
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Query", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Run Query"):
        query = ANALYTICAL_QUERIES[query_name]
        
        cursor.execute(query)
        results = cursor.fetchall()
        columns = [desc[0] for desc in cursor.description]
        
        df = pd.DataFrame(results, columns=columns)
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Error Handling for API Calls

```python
def safe_api_call(api_key, page, max_retries=3):
    """API call with retry logic"""
    
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except Exception as e:
            if attempt == max_retries - 1:
                st.error(f"Failed after {max_retries} attempts: {str(e)}")
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

### Visualization Helper

```python
def create_chart(df, chart_type='bar'):
    """Create interactive charts from query results"""
    
    if len(df.columns) < 2:
        st.warning("Not enough columns for visualization")
        return
    
    x_col, y_col = df.columns[0], df.columns[1]
    
    if chart_type == 'bar':
        fig = px.bar(df, x=x_col, y=y_col)
    elif chart_type == 'pie':
        fig = px.pie(df, names=x_col, values=y_col)
    elif chart_type == 'line':
        fig = px.line(df, x=x_col, y=y_col)
    
    st.plotly_chart(fig, use_container_width=True)
```

## Troubleshooting

### API Rate Limiting

```python
# Add delays between requests
time.sleep(0.5)  # 500ms delay

# Check response headers for rate limit info
if 'X-RateLimit-Remaining' in response.headers:
    remaining = int(response.headers['X-RateLimit-Remaining'])
    if remaining < 10:
        time.sleep(5)  # Longer pause when near limit
```

### Database Connection Issues

```python
def get_db_connection():
    """Create database connection with error handling"""
    try:
        conn = mysql.connector.connect(**db_config)
        return conn
    except mysql.connector.Error as e:
        st.error(f"Database connection failed: {e}")
        st.info("Check DB_HOST, DB_USER, DB_PASSWORD in .env")
        return None
```

### Handling NULL Values

```python
# Use COALESCE in SQL queries
query = """
SELECT COALESCE(culture, 'Unknown') as culture, 
       COUNT(*) as count
FROM artifactmetadata
GROUP BY culture
"""

# Or filter in Python
df = df[df['culture'].notna()]
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

The dashboard will open with three main sections:
1. **Data Collection**: ETL pipeline interface
2. **SQL Analytics**: Run predefined queries
3. **Visualizations**: Interactive charts and graphs
