---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering pipeline with the Harvard API
  - analyze Harvard Art Museums artifacts with SQL
  - set up a Streamlit dashboard for museum data
  - extract and visualize Harvard museum collection data
  - process Harvard Art Museums API data into SQL
  - create analytics for art museum artifacts
  - build a data pipeline for museum collections
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end ETL (Extract, Transform, Load) pipelines and analytics applications using the Harvard Art Museums API. The project demonstrates real-world data engineering patterns including API integration, batch processing, SQL database design, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App provides:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into relational database tables
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper schema design
- **Analytics Queries**: 20+ predefined SQL queries for artifact insights
- **Visualization Dashboard**: Interactive Streamlit app with Plotly charts

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

## Installation

### Prerequisites

```bash
# Python 3.8+ required
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
cp .env.example .env
# Edit .env with your credentials
```

### Required Environment Variables

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

## Database Schema

The project uses three main tables with foreign key relationships:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    description TEXT,
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        page: Page number for pagination
        size: Number of records per page (max 100)
    
    Returns:
        JSON response with artifact data
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

# Fetch multiple pages
def fetch_all_artifacts(num_pages=5):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page, size=100)
        all_artifacts.extend(data.get('records', []))
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifacts(raw_data):
    """
    Transform raw API data into structured DataFrames
    
    Args:
        raw_data: List of artifact dictionaries from API
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Untitled'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'description': artifact.get('description'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media/images
        if artifact.get('images'):
            for image in artifact['images']:
                media = {
                    'artifact_id': artifact.get('id'),
                    'image_url': image.get('baseimageurl'),
                    'media_type': 'image'
                }
                media_list.append(media)
        
        # Extract colors
        if artifact.get('colors'):
            for color_data in artifact['colors']:
                color = {
                    'artifact_id': artifact.get('id'),
                    'color': color_data.get('color'),
                    'percentage': color_data.get('percent')
                }
                colors_list.append(color)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

def get_db_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            port=int(os.getenv('DB_PORT', 3306))
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_to_database(metadata_df, media_df, colors_df):
    """
    Load transformed data into SQL database
    
    Args:
        metadata_df: DataFrame with artifact metadata
        media_df: DataFrame with media/images
        colors_df: DataFrame with color data
    """
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Insert metadata
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (id, title, culture, period, century, classification, 
                 department, division, dated, description, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE
                title=VALUES(title), culture=VALUES(culture)
            """, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia (artifact_id, image_url, media_type)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors (artifact_id, color, percentage)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        connection.commit()
        print(f"Successfully loaded {len(metadata_df)} artifacts")
        return True
        
    except Error as e:
        print(f"Database error: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

### 3. Analytics Queries

```python
def execute_query(query):
    """
    Execute SQL query and return results as DataFrame
    
    Args:
        query: SQL query string
    
    Returns:
        pandas DataFrame with query results
    """
    connection = get_db_connection()
    if not connection:
        return None
    
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return None
    finally:
        connection.close()

# Sample analytics queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Top Colors in Collection": """
        SELECT color, COUNT(*) as occurrences, 
               AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY occurrences DESC
        LIMIT 15
    """,
    
    "Artifacts with Most Images": """
        SELECT m.id, m.title, COUNT(med.id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia med ON m.id = med.artifact_id
        GROUP BY m.id, m.title
        ORDER BY image_count DESC
        LIMIT 10
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count,
               COUNT(DISTINCT classification) as classifications
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
}
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Data Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar for ETL operations
    with st.sidebar:
        st.header("ETL Operations")
        
        if st.button("🔄 Fetch New Data"):
            with st.spinner("Fetching artifacts from API..."):
                artifacts = fetch_all_artifacts(num_pages=3)
                st.success(f"Fetched {len(artifacts)} artifacts")
                
                # Transform
                metadata_df, media_df, colors_df = transform_artifacts(artifacts)
                
                # Load
                if load_to_database(metadata_df, media_df, colors_df):
                    st.success("Data loaded successfully!")
                else:
                    st.error("Failed to load data")
    
    # Main analytics section
    st.header("📊 Analytics Dashboard")
    
    # Query selector
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        
        with st.spinner("Executing query..."):
            results_df = execute_query(query)
            
            if results_df is not None and not results_df.empty:
                # Display results table
                st.subheader("Query Results")
                st.dataframe(results_df, use_container_width=True)
                
                # Auto-generate visualization
                if len(results_df.columns) >= 2:
                    st.subheader("Visualization")
                    
                    # Create bar chart
                    fig = px.bar(
                        results_df,
                        x=results_df.columns[0],
                        y=results_df.columns[1],
                        title=query_name,
                        labels={results_df.columns[0]: results_df.columns[0].title(),
                               results_df.columns[1]: results_df.columns[1].title()}
                    )
                    
                    st.plotly_chart(fig, use_container_width=True)
            else:
                st.warning("No results found")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Batch Processing with Rate Limiting

```python
import time

def batch_fetch_with_rate_limit(total_pages, batch_size=5, delay=2):
    """Fetch data in batches with rate limiting"""
    all_data = []
    
    for i in range(0, total_pages, batch_size):
        batch_end = min(i + batch_size, total_pages)
        
        for page in range(i + 1, batch_end + 1):
            data = fetch_artifacts(page=page)
            all_data.extend(data.get('records', []))
            
        # Rate limit between batches
        if batch_end < total_pages:
            time.sleep(delay)
    
    return all_data
```

### Error Handling and Retry Logic

```python
from requests.exceptions import RequestException
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
            time.sleep(wait_time)
```

### Incremental Updates

```python
def get_latest_artifact_id():
    """Get the latest artifact ID in database"""
    query = "SELECT MAX(id) as max_id FROM artifactmetadata"
    result = execute_query(query)
    return result['max_id'].iloc[0] if result is not None else 0

def fetch_new_artifacts_only():
    """Fetch only artifacts newer than what's in DB"""
    latest_id = get_latest_artifact_id()
    
    # Implement logic to fetch only new artifacts
    # This depends on API capabilities
    pass
```

## Troubleshooting

### API Key Issues

```python
# Verify API key is loaded
import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv('HARVARD_API_KEY')

if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment variables")

# Test API connection
def test_api_connection():
    try:
        response = fetch_artifacts(page=1, size=1)
        print("API connection successful!")
        return True
    except Exception as e:
        print(f"API connection failed: {e}")
        return False
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        connection = get_db_connection()
        if connection and connection.is_connected():
            print("Database connection successful!")
            cursor = connection.cursor()
            cursor.execute("SELECT VERSION()")
            version = cursor.fetchone()
            print(f"MySQL version: {version[0]}")
            cursor.close()
            connection.close()
            return True
    except Error as e:
        print(f"Database connection failed: {e}")
        return False
```

### Handling Null Values

```python
def clean_dataframe(df):
    """Clean DataFrame before database insertion"""
    # Replace NaN with None for SQL NULL
    df = df.where(pd.notnull(df), None)
    
    # Truncate long text fields
    if 'description' in df.columns:
        df['description'] = df['description'].str[:5000]
    
    if 'title' in df.columns:
        df['title'] = df['title'].str[:500]
    
    return df
```

### Memory Management for Large Datasets

```python
def process_in_chunks(artifacts, chunk_size=100):
    """Process large datasets in chunks"""
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        metadata_df, media_df, colors_df = transform_artifacts(chunk)
        
        # Clean before loading
        metadata_df = clean_dataframe(metadata_df)
        
        load_to_database(metadata_df, media_df, colors_df)
        print(f"Processed chunk {i // chunk_size + 1}")
```

This skill provides comprehensive guidance for building ETL pipelines and analytics applications using the Harvard Art Museums API with modern Python data engineering tools.
