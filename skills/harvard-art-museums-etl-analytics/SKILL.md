---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for museum artifact data
  - show me how to use the Harvard Art Museums API
  - help me create a data engineering project with Streamlit
  - how to visualize art museum data with SQL analytics
  - build an artifact collection analytics dashboard
  - extract and analyze Harvard museum data
  - create a museum data pipeline with Python
  - implement ETL for art artifacts API
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates end-to-end data engineering: extracting artifact data from the Harvard Art Museums API, transforming it into relational structures, loading into SQL databases, and building interactive analytics dashboards with Streamlit.

## What It Does

- **API Integration**: Fetches artifact metadata, media, and color data from Harvard Art Museums API
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper schema design
- **Analytics**: Runs 20+ predefined SQL queries for insights
- **Visualization**: Interactive Plotly dashboards in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly pymysql
```

## Configuration

### API Key Setup

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
# Set environment variable
export HARVARD_API_KEY="your_api_key_here"

# Or in Python
import os
api_key = os.getenv('HARVARD_API_KEY')
```

### Database Configuration

```python
# config.py or environment variables
DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

## Database Schema

The project uses three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    department VARCHAR(255),
    classification VARCHAR(255),
    technique VARCHAR(500),
    period VARCHAR(255),
    dated VARCHAR(255),
    url TEXT,
    description TEXT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_name VARCHAR(100),
    color_hex VARCHAR(10),
    color_percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);
```

## ETL Pipeline Components

### 1. Extract: Fetch Data from API

```python
import requests
import os

def fetch_artifacts(api_key, num_pages=5, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}: {len(data.get('records', []))} artifacts")
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return all_artifacts

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts = fetch_artifacts(api_key, num_pages=3)
print(f"Total artifacts fetched: {len(artifacts)}")
```

### 2. Transform: Process JSON to DataFrames

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform raw artifact JSON into normalized DataFrames
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        
        # Extract metadata
        metadata_list.append({
            'artifact_id': artifact_id,
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'technique': artifact.get('technique', 'Unknown'),
            'period': artifact.get('period', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'url': artifact.get('url', ''),
            'description': artifact.get('description', '')
        })
        
        # Extract media/images
        images = artifact.get('images', [])
        for img in images:
            media_list.append({
                'artifact_id': artifact_id,
                'image_url': img.get('baseimageurl', ''),
                'media_type': 'image'
            })
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            colors_list.append({
                'artifact_id': artifact_id,
                'color_name': color.get('color', 'Unknown'),
                'color_hex': color.get('hex', ''),
                'color_percentage': color.get('percent', 0.0)
            })
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors

# Usage
df_metadata, df_media, df_colors = transform_artifacts(artifacts)
print(f"Metadata records: {len(df_metadata)}")
print(f"Media records: {len(df_media)}")
print(f"Color records: {len(df_colors)}")
```

### 3. Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def create_connection(db_config):
    """Create database connection"""
    try:
        connection = mysql.connector.connect(**db_config)
        print("Successfully connected to database")
        return connection
    except Error as e:
        print(f"Error: {e}")
        return None

def load_data_to_sql(df_metadata, df_media, df_colors, db_config):
    """
    Load transformed DataFrames into SQL database with batch inserts
    """
    connection = create_connection(db_config)
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Load metadata (batch insert)
        metadata_query = """
        INSERT INTO artifactmetadata 
        (artifact_id, title, culture, century, department, classification, 
         technique, period, dated, url, description)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        metadata_values = df_metadata.values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Load media
        media_query = """
        INSERT INTO artifactmedia (artifact_id, image_url, media_type)
        VALUES (%s, %s, %s)
        """
        media_values = df_media.values.tolist()
        cursor.executemany(media_query, media_values)
        
        # Load colors
        colors_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color_name, color_hex, color_percentage)
        VALUES (%s, %s, %s, %s)
        """
        colors_values = df_colors.values.tolist()
        cursor.executemany(colors_query, colors_values)
        
        connection.commit()
        print(f"Successfully loaded {len(df_metadata)} artifacts to database")
        return True
        
    except Error as e:
        print(f"Error loading data: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()

# Usage
DB_CONFIG = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}
load_data_to_sql(df_metadata, df_media, df_colors, DB_CONFIG)
```

## Analytics Queries

### Common SQL Patterns

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "artifacts_by_century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century != 'Unknown'
        GROUP BY century
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture != 'Unknown'
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "top_colors": """
        SELECT color_name, 
               COUNT(*) as usage_count,
               AVG(color_percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "artifacts_with_images": """
        SELECT 
            m.department,
            COUNT(DISTINCT m.artifact_id) as total_artifacts,
            COUNT(DISTINCT med.artifact_id) as artifacts_with_images,
            ROUND(COUNT(DISTINCT med.artifact_id) * 100.0 / 
                  COUNT(DISTINCT m.artifact_id), 2) as image_percentage
        FROM artifactmetadata m
        LEFT JOIN artifactmedia med ON m.artifact_id = med.artifact_id
        GROUP BY m.department
        ORDER BY image_percentage DESC
    """
}

def execute_query(query, db_config):
    """Execute SQL query and return DataFrame"""
    connection = create_connection(db_config)
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

# Usage
df_result = execute_query(ANALYTICS_QUERIES['artifacts_by_century'], DB_CONFIG)
print(df_result)
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import plotly.express as px
import pandas as pd

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar configuration
st.sidebar.header("Database Configuration")
db_host = st.sidebar.text_input("Host", value=os.getenv('DB_HOST', 'localhost'))
db_user = st.sidebar.text_input("User", value=os.getenv('DB_USER', 'root'))
db_password = st.sidebar.text_input("Password", type="password")
db_name = st.sidebar.text_input("Database", value="harvard_artifacts")

# Query selection
query_options = list(ANALYTICS_QUERIES.keys())
selected_query = st.selectbox("Select Analysis", query_options)

if st.button("Run Query"):
    db_config = {
        'host': db_host,
        'user': db_user,
        'password': db_password,
        'database': db_name
    }
    
    with st.spinner("Executing query..."):
        df = execute_query(ANALYTICS_QUERIES[selected_query], db_config)
        
        if df is not None and not df.empty:
            st.success(f"Query returned {len(df)} rows")
            
            # Display table
            st.dataframe(df, use_container_width=True)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df, 
                    x=df.columns[0], 
                    y=df.columns[1],
                    title=selected_query.replace('_', ' ').title()
                )
                st.plotly_chart(fig, use_container_width=True)
        else:
            st.error("No data returned or query failed")
```

### ETL Pipeline in Streamlit

```python
# Add ETL section to dashboard
st.header("📥 ETL Pipeline")

api_key = st.text_input("Harvard API Key", type="password", 
                        value=os.getenv('HARVARD_API_KEY', ''))
num_pages = st.number_input("Number of pages to fetch", min_value=1, 
                            max_value=20, value=3)

if st.button("Run ETL Pipeline"):
    if not api_key:
        st.error("Please provide API key")
    else:
        progress_bar = st.progress(0)
        status_text = st.empty()
        
        # Extract
        status_text.text("Extracting data from API...")
        progress_bar.progress(25)
        artifacts = fetch_artifacts(api_key, num_pages)
        
        # Transform
        status_text.text("Transforming data...")
        progress_bar.progress(50)
        df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        
        # Load
        status_text.text("Loading to database...")
        progress_bar.progress(75)
        success = load_data_to_sql(df_metadata, df_media, df_colors, db_config)
        
        progress_bar.progress(100)
        
        if success:
            status_text.text("✅ ETL Pipeline completed successfully!")
            st.balloons()
        else:
            status_text.text("❌ ETL Pipeline failed")
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY="your_api_key"
export DB_HOST="your_db_host"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"

# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_rate_limit(api_key, num_pages, delay=1):
    """Add delay between requests to respect rate limits"""
    artifacts = []
    for page in range(1, num_pages + 1):
        response = requests.get(url, params=params)
        artifacts.extend(response.json().get('records', []))
        time.sleep(delay)  # Wait between requests
    return artifacts
```

### Database Connection Issues

```python
def test_connection(db_config):
    """Test database connectivity"""
    try:
        connection = mysql.connector.connect(**db_config)
        if connection.is_connected():
            print("✅ Database connection successful")
            db_info = connection.get_server_info()
            print(f"MySQL Server version: {db_info}")
            return True
    except Error as e:
        print(f"❌ Connection failed: {e}")
        return False
    finally:
        if connection and connection.is_connected():
            connection.close()
```

### Handling Missing Data

```python
def clean_artifact_data(artifact):
    """Handle missing or None values in artifact data"""
    return {
        'artifact_id': artifact.get('id', 0),
        'title': artifact.get('title') or 'Untitled',
        'culture': artifact.get('culture') or 'Unknown',
        'century': artifact.get('century') or 'Unknown',
        # Use None for SQL NULL, not empty strings
        'description': artifact.get('description') or None
    }
```

## Best Practices

1. **Use batch inserts** for better performance with `executemany()`
2. **Handle API pagination** properly to get all available data
3. **Implement error handling** for API failures and database errors
4. **Use environment variables** for sensitive credentials
5. **Add indexes** on frequently queried columns (artifact_id, culture, century)
6. **Cache Streamlit queries** using `@st.cache_data` decorator
7. **Normalize data** into proper relational tables with foreign keys

This skill enables AI agents to help developers build complete ETL pipelines with API integration, SQL analytics, and interactive dashboards using real museum data.
