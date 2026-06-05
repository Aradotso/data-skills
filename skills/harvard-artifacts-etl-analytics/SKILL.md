---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - set up data engineering workflow with Harvard API
  - create analytics dashboard for museum artifacts
  - extract and load Harvard museum data to SQL
  - build Streamlit app for artifact analytics
  - query and visualize Harvard Art Museums collection
  - implement museum data pipeline with Python
  - analyze Harvard artifacts with SQL queries
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application for the Harvard Art Museums API. It demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What It Does

- **Extracts** artifact data from Harvard Art Museums API with pagination handling
- **Transforms** nested JSON into relational database schema (metadata, media, colors)
- **Loads** data into MySQL/TiDB Cloud with batch inserts
- **Analyzes** data using 20+ predefined SQL queries
- **Visualizes** results through interactive Plotly charts in Streamlit

Architecture: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Expected dependencies include:
# streamlit
# pandas
# requests
# mysql-connector-python
# plotly
```

## Configuration

### Environment Variables

Set up your configuration using environment variables:

```bash
# Harvard Art Museums API Key
export HARVARD_API_KEY="your_api_key_here"

# Database Configuration
export DB_HOST="your_db_host"
export DB_PORT="4000"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

### API Key Setup

Get your free API key from Harvard Art Museums:
1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key
3. Store in environment variable or Streamlit secrets

### Streamlit Secrets (Alternative)

Create `.streamlit/secrets.toml`:

```toml
HARVARD_API_KEY = "your_api_key"

[database]
host = "your_db_host"
port = 4000
user = "your_db_user"
password = "your_db_password"
database = "harvard_artifacts"
```

## Database Schema

### Tables Structure

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    dated VARCHAR(200),
    accession_year INT,
    PRIMARY KEY (id)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    image_height INT,
    image_width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Usage

### Extract Data from API

```python
import requests
import pandas as pd
import os

def fetch_artifacts(api_key, size=100, page=1):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts, info = fetch_artifacts(api_key, size=50, page=1)
print(f"Total records: {info['totalrecords']}")
print(f"Pages: {info['pages']}")
```

### Transform Data

```python
def transform_artifacts(records):
    """Transform API records into structured dataframes"""
    
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in records:
        # Extract metadata
        metadata = {
            'id': record.get('id'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'technique': record.get('technique'),
            'dated': record.get('dated'),
            'accession_year': record.get('accessionyear')
        }
        metadata_list.append(metadata)
        
        # Extract media (images)
        if record.get('images'):
            for img in record['images']:
                media = {
                    'artifact_id': record['id'],
                    'image_url': img.get('baseimageurl'),
                    'image_height': img.get('height'),
                    'image_width': img.get('width')
                }
                media_list.append(media)
        
        # Extract colors
        if record.get('colors'):
            for color in record['colors']:
                color_data = {
                    'artifact_id': record['id'],
                    'color_hex': color.get('hex'),
                    'color_percent': color.get('percent')
                }
                colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

# Usage
df_metadata, df_media, df_colors = transform_artifacts(artifacts)
```

### Load Data to SQL

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df, connection):
    """Bulk insert metadata with duplicate handling"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, technique, dated, accession_year)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    print(f"Inserted {cursor.rowcount} metadata records")

def load_media(df, connection):
    """Bulk insert media data"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia (artifact_id, image_url, image_height, image_width)
        VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    print(f"Inserted {cursor.rowcount} media records")

def load_colors(df, connection):
    """Bulk insert color data"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
        VALUES (%s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    print(f"Inserted {cursor.rowcount} color records")

# Usage
conn = get_db_connection()
load_metadata(df_metadata, conn)
load_media(df_media, conn)
load_colors(df_colors, conn)
conn.close()
```

## Analytical SQL Queries

### Common Analytics Patterns

```python
# Query 1: Artifacts by Culture
query_by_culture = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query 2: Century Distribution
query_by_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Query 3: Department Statistics
query_by_department = """
    SELECT department, 
           COUNT(*) as total_artifacts,
           COUNT(DISTINCT classification) as unique_classifications
    FROM artifactmetadata
    GROUP BY department
    ORDER BY total_artifacts DESC
"""

# Query 4: Most Common Colors
query_top_colors = """
    SELECT c.color_hex, 
           COUNT(*) as usage_count,
           AVG(c.color_percent) as avg_percent
    FROM artifactcolors c
    GROUP BY c.color_hex
    ORDER BY usage_count DESC
    LIMIT 20
"""

# Query 5: Artifacts with Most Images
query_media_rich = """
    SELECT m.artifact_id, 
           a.title,
           COUNT(*) as image_count
    FROM artifactmedia m
    JOIN artifactmetadata a ON m.artifact_id = a.id
    GROUP BY m.artifact_id, a.title
    ORDER BY image_count DESC
    LIMIT 15
"""

# Execute query
def execute_query(query, connection):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)

# Usage
conn = get_db_connection()
df_result = execute_query(query_by_culture, conn)
conn.close()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar configuration
    with st.sidebar:
        st.header("Configuration")
        api_key = st.text_input("Harvard API Key", type="password", 
                                value=st.secrets.get("HARVARD_API_KEY", ""))
        
        st.header("ETL Pipeline")
        if st.button("Run ETL Pipeline"):
            run_etl_pipeline(api_key)
    
    # Main content tabs
    tab1, tab2, tab3 = st.tabs(["📊 Analytics", "🔍 Query Builder", "📈 Visualizations"])
    
    with tab1:
        show_analytics_dashboard()
    
    with tab2:
        show_query_builder()
    
    with tab3:
        show_visualizations()

def show_analytics_dashboard():
    """Display predefined analytics queries"""
    st.header("Predefined Analytics")
    
    queries = {
        "Artifacts by Culture": query_by_culture,
        "Century Distribution": query_by_century,
        "Department Statistics": query_by_department,
        "Top Colors": query_top_colors,
        "Media-Rich Artifacts": query_media_rich
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            conn = get_db_connection()
            df = execute_query(queries[selected_query], conn)
            conn.close()
            
            st.dataframe(df, use_container_width=True)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                           title=selected_query)
                st.plotly_chart(fig, use_container_width=True)

def run_etl_pipeline(api_key):
    """Execute full ETL pipeline"""
    progress = st.progress(0)
    status = st.empty()
    
    try:
        # Extract
        status.text("Extracting data from API...")
        artifacts, info = fetch_artifacts(api_key, size=100)
        progress.progress(33)
        
        # Transform
        status.text("Transforming data...")
        df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        progress.progress(66)
        
        # Load
        status.text("Loading to database...")
        conn = get_db_connection()
        load_metadata(df_metadata, conn)
        load_media(df_media, conn)
        load_colors(df_colors, conn)
        conn.close()
        progress.progress(100)
        
        status.text("✅ ETL Pipeline completed successfully!")
        st.success(f"Loaded {len(df_metadata)} artifacts")
        
    except Exception as e:
        st.error(f"Error: {str(e)}")

if __name__ == "__main__":
    main()
```

### Visualization Helpers

```python
def create_bar_chart(df, x_col, y_col, title):
    """Create interactive bar chart"""
    fig = px.bar(df, x=x_col, y=y_col, title=title,
                 labels={x_col: x_col.replace('_', ' ').title(),
                        y_col: y_col.replace('_', ' ').title()})
    fig.update_layout(xaxis_tickangle=-45)
    return fig

def create_pie_chart(df, values_col, names_col, title):
    """Create pie chart for distribution"""
    fig = px.pie(df, values=values_col, names=names_col, title=title)
    return fig

def create_timeline(df, date_col, value_col, title):
    """Create timeline visualization"""
    fig = px.line(df, x=date_col, y=value_col, title=title)
    return fig
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Pagination for Large Datasets

```python
def fetch_all_artifacts(api_key, max_pages=10):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        artifacts, info = fetch_artifacts(api_key, size=100, page=page)
        all_artifacts.extend(artifacts)
        
        if page >= info['pages']:
            break
    
    return all_artifacts
```

### Incremental ETL Updates

```python
def get_max_artifact_id(connection):
    """Get highest artifact ID in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(api_key):
    """Load only new artifacts"""
    conn = get_db_connection()
    max_id = get_max_artifact_id(conn)
    
    # Fetch only artifacts with ID > max_id
    # Process and load new data
    conn.close()
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, retries=3, delay=2):
    """Fetch with exponential backoff"""
    for attempt in range(retries):
        try:
            return fetch_artifacts(api_key)
        except Exception as e:
            if attempt < retries - 1:
                time.sleep(delay * (2 ** attempt))
            else:
                raise e
```

### Database Connection Issues

```python
def get_db_connection_with_retry(max_retries=3):
    """Connect with retry logic"""
    for attempt in range(max_retries):
        try:
            conn = mysql.connector.connect(
                host=os.getenv('DB_HOST'),
                port=int(os.getenv('DB_PORT', 3306)),
                user=os.getenv('DB_USER'),
                password=os.getenv('DB_PASSWORD'),
                database=os.getenv('DB_NAME'),
                connect_timeout=10
            )
            return conn
        except Error as e:
            if attempt < max_retries - 1:
                time.sleep(2)
            else:
                raise e
```

### Handling Missing Data

```python
def safe_transform(record, field, default=None):
    """Safely extract field with default"""
    return record.get(field, default) if record.get(field) != '' else default

# Usage in transform
metadata = {
    'id': record.get('id'),
    'title': safe_transform(record, 'title', 'Untitled'),
    'culture': safe_transform(record, 'culture', 'Unknown'),
    'century': safe_transform(record, 'century', 'Unknown')
}
```

This skill enables AI agents to help developers build complete ETL pipelines for museum data with SQL analytics and interactive dashboards.
