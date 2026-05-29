---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifact data
  - create a data analytics app with the Harvard Art Museums API
  - set up a Streamlit dashboard for art collection data
  - design a SQL database schema for artifact metadata
  - implement batch data collection from Harvard museums
  - analyze art museum data with SQL queries
  - build an interactive art analytics visualization
  - create a data engineering project with museum API
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards.

## What This Project Does

The Harvard Art Museums ETL Analytics application:
- Extracts artifact data from the Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into relational database tables
- Loads structured data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries for insights
- Visualizes results through interactive Plotly charts in Streamlit

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

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

### Environment Variables

Create a `.env` file in the project root:

```env
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### API Key Setup

1. Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Store it in `.env` as `HARVARD_API_KEY`
3. Never commit the `.env` file to version control

## Database Schema

The application uses three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(300),
    medium VARCHAR(300),
    dated VARCHAR(200),
    url VARCHAR(500)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_url VARCHAR(500),
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return None

# Collect multiple pages with rate limiting
import time

def collect_artifacts_batch(num_pages=5):
    """
    Collect artifacts across multiple pages
    """
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page, size=100)
        
        if data and 'records' in data:
            all_artifacts.extend(data['records'])
            time.sleep(1)  # Rate limiting - 1 second between requests
        else:
            break
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def extract_artifact_metadata(artifacts):
    """
    Extract metadata from API response
    """
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'period': artifact.get('period', '')[:200],
            'technique': artifact.get('technique', '')[:300],
            'medium': artifact.get('medium', '')[:300],
            'dated': artifact.get('dated', '')[:200],
            'url': artifact.get('url', '')[:500]
        })
    
    return pd.DataFrame(metadata)

def extract_artifact_media(artifacts):
    """
    Extract media information
    """
    media = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media.append({
                'artifact_id': artifact_id,
                'media_url': image.get('baseimageurl', '')[:500],
                'media_type': image.get('format', 'image')[:100]
            })
    
    return pd.DataFrame(media)

def extract_artifact_colors(artifacts):
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
                'color': color.get('color', '')[:50],
                'percentage': color.get('percent', 0.0)
            })
    
    return pd.DataFrame(colors)
```

### 3. Database Loading

```python
def get_db_connection():
    """
    Create database connection
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

def load_metadata(df):
    """
    Load artifact metadata into database
    """
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, 
     period, technique, medium, dated, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    try:
        records = df.to_records(index=False)
        cursor.executemany(insert_query, records.tolist())
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
        return True
    except Error as e:
        print(f"Error loading metadata: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()

def load_media(df):
    """
    Load media data into database
    """
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, media_url, media_type)
    VALUES (%s, %s, %s)
    """
    
    try:
        records = df.to_records(index=False)
        cursor.executemany(insert_query, records.tolist())
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
        return True
    except Error as e:
        print(f"Error loading media: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

### 4. Streamlit Application

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museum Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    else:
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection")
    
    num_pages = st.number_input(
        "Number of pages to collect",
        min_value=1,
        max_value=50,
        value=5
    )
    
    if st.button("Start Collection"):
        with st.spinner("Collecting data..."):
            artifacts = collect_artifacts_batch(num_pages)
            
            # ETL Process
            metadata_df = extract_artifact_metadata(artifacts)
            media_df = extract_artifact_media(artifacts)
            colors_df = extract_artifact_colors(artifacts)
            
            # Load to database
            load_metadata(metadata_df)
            load_media(media_df)
            load_colors(colors_df)
            
            st.success(f"Collected {len(artifacts)} artifacts!")
            st.dataframe(metadata_df.head())

def show_sql_analytics():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Top 10 Cultures by Artifact Count": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
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
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            GROUP BY department
            ORDER BY count DESC
        """,
        "Top Colors Used": """
            SELECT color, COUNT(*) as count, AVG(percentage) as avg_percentage
            FROM artifactcolors
            GROUP BY color
            ORDER BY count DESC
            LIMIT 10
        """
    }
    
    selected_query = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        connection = get_db_connection()
        if connection:
            df = pd.read_sql(queries[selected_query], connection)
            connection.close()
            
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) == 2:
                fig = px.bar(
                    df,
                    x=df.columns[0],
                    y=df.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Workflow

```python
def run_etl_pipeline(num_pages=5):
    """
    Execute complete ETL pipeline
    """
    print("Starting ETL Pipeline...")
    
    # Extract
    print("1. Extracting data from API...")
    artifacts = collect_artifacts_batch(num_pages)
    
    # Transform
    print("2. Transforming data...")
    metadata_df = extract_artifact_metadata(artifacts)
    media_df = extract_artifact_media(artifacts)
    colors_df = extract_artifact_colors(artifacts)
    
    # Load
    print("3. Loading data to database...")
    success = True
    success &= load_metadata(metadata_df)
    success &= load_media(media_df)
    success &= load_colors(colors_df)
    
    if success:
        print("✅ ETL Pipeline completed successfully!")
    else:
        print("❌ ETL Pipeline encountered errors")
    
    return success
```

### Running SQL Analytics

```python
def run_analytics_query(query_name):
    """
    Execute and visualize analytics query
    """
    analytics_queries = {
        "classification_analysis": """
            SELECT classification, COUNT(*) as total_artifacts
            FROM artifactmetadata
            WHERE classification IS NOT NULL
            GROUP BY classification
            ORDER BY total_artifacts DESC
            LIMIT 15
        """
    }
    
    connection = get_db_connection()
    if not connection:
        return None
    
    try:
        df = pd.read_sql(analytics_queries[query_name], connection)
        return df
    finally:
        connection.close()
```

## Troubleshooting

### API Rate Limiting
```python
# Add delays between requests
import time

for page in range(1, num_pages + 1):
    data = fetch_artifacts(page=page)
    time.sleep(1)  # Wait 1 second between requests
```

### Database Connection Issues
```python
# Test connection
def test_db_connection():
    connection = get_db_connection()
    if connection and connection.is_connected():
        print("✅ Database connected successfully")
        connection.close()
        return True
    else:
        print("❌ Database connection failed")
        return False
```

### Memory Management for Large Datasets
```python
# Process in chunks
def process_in_chunks(artifacts, chunk_size=100):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        metadata_df = extract_artifact_metadata(chunk)
        load_metadata(metadata_df)
```

### Handling Missing Data
```python
# Safe extraction with defaults
def safe_get(artifact, key, default='', max_length=None):
    value = artifact.get(key, default)
    if max_length and value:
        return str(value)[:max_length]
    return value
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Run ETL pipeline standalone
python etl_pipeline.py

# Execute specific analytics
python analytics.py --query=culture_distribution
```

This skill provides everything needed to build, deploy, and extend the Harvard Art Museums ETL Analytics application with proper data engineering practices.
