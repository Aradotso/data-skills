---
name: harvard-art-museums-data-engineering-pipeline
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - extract and analyze Harvard museum artifacts
  - create a data engineering app with Streamlit and museum API
  - set up SQL database for art collections data
  - visualize Harvard Art Museums analytics
  - implement museum data pipeline with Python
  - query and analyze art museum metadata
  - build museum artifacts collection dashboard
---

# Harvard Art Museums Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL processes, SQL database design, analytics queries, and interactive Streamlit visualizations for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:
- Dynamic artifact data collection from Harvard Art Museums API
- ETL pipeline transforming nested JSON to relational database schema
- SQL database storage (MySQL/TiDB Cloud) with proper relationships
- 20+ predefined analytical queries for artifact insights
- Interactive Streamlit dashboard with Plotly visualizations
- Real-world data engineering patterns for analytics applications

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

Create a `.env` file or set environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_art_museums
```

### Getting Harvard API Key

1. Register at https://www.harvardartmuseums.org/collections/api
2. Obtain your API key
3. Store securely in environment variables

### Database Setup

```python
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

# Connect to database
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Create tables
def create_tables(conn):
    cursor = conn.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS artifactmetadata (
        id INT PRIMARY KEY,
        title VARCHAR(500),
        culture VARCHAR(255),
        period VARCHAR(255),
        century VARCHAR(255),
        classification VARCHAR(255),
        department VARCHAR(255),
        division VARCHAR(255),
        dated VARCHAR(255),
        url VARCHAR(500)
    )
    """)
    
    # Artifact Media Table
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS artifactmedia (
        media_id INT AUTO_INCREMENT PRIMARY KEY,
        artifact_id INT,
        image_url VARCHAR(1000),
        media_type VARCHAR(100),
        FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
    )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS artifactcolors (
        color_id INT AUTO_INCREMENT PRIMARY KEY,
        artifact_id INT,
        color_hex VARCHAR(10),
        color_percent FLOAT,
        FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
    )
    """)
    
    conn.commit()
    cursor.close()
```

## ETL Pipeline Implementation

### Extract: API Data Collection

```python
import requests
import time

def extract_artifacts(api_key, num_records=100, records_per_page=100):
    """
    Extract artifact data from Harvard Art Museums API
    """
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    
    params = {
        'apikey': api_key,
        'size': records_per_page,
        'page': 1
    }
    
    while len(artifacts) < num_records:
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            artifacts.extend(records)
            
            # Respect rate limits
            time.sleep(0.5)
            
            params['page'] += 1
            
            if len(artifacts) >= num_records:
                artifacts = artifacts[:num_records]
                break
                
        except requests.exceptions.RequestException as e:
            print(f"Error fetching data: {e}")
            break
    
    return artifacts
```

### Transform: Data Processing

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform raw API data into structured dataframes
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'period': artifact.get('period', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'division': artifact.get('division', ''),
            'dated': artifact.get('dated', ''),
            'url': artifact.get('url', '')
        }
        metadata_records.append(metadata)
        
        # Extract media information
        images = artifact.get('images', [])
        for image in images:
            media = {
                'artifact_id': artifact.get('id'),
                'image_url': image.get('baseimageurl', ''),
                'media_type': 'image'
            }
            media_records.append(media)
        
        # Extract color information
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex', ''),
                'color_percent': color.get('percent', 0.0)
            }
            color_records.append(color_data)
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### Load: Database Insertion

```python
def load_to_database(df_metadata, df_media, df_colors, conn):
    """
    Load transformed data into SQL database
    """
    cursor = conn.cursor()
    
    # Insert metadata
    for _, row in df_metadata.iterrows():
        cursor.execute("""
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, division, dated, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture), period=VALUES(period)
        """, tuple(row))
    
    # Insert media
    for _, row in df_media.iterrows():
        cursor.execute("""
        INSERT INTO artifactmedia (artifact_id, image_url, media_type)
        VALUES (%s, %s, %s)
        """, (row['artifact_id'], row['image_url'], row['media_type']))
    
    # Insert colors
    for _, row in df_colors.iterrows():
        cursor.execute("""
        INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
        VALUES (%s, %s, %s)
        """, (row['artifact_id'], row['color_hex'], row['color_percent']))
    
    conn.commit()
    cursor.close()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Data Engineering App")
    st.markdown("---")
    
    # Sidebar navigation
    menu = st.sidebar.selectbox(
        "Navigation",
        ["Home", "Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if menu == "Home":
        show_home()
    elif menu == "Data Collection":
        show_data_collection()
    elif menu == "SQL Analytics":
        show_sql_analytics()
    elif menu == "Visualizations":
        show_visualizations()

def show_home():
    st.header("Welcome to Harvard Art Museums Analytics")
    st.write("""
    This application demonstrates:
    - ETL pipeline for museum data
    - SQL database design and analytics
    - Interactive data visualizations
    """)
    
    # Display statistics
    conn = get_db_connection()
    cursor = conn.cursor()
    
    cursor.execute("SELECT COUNT(*) FROM artifactmetadata")
    total_artifacts = cursor.fetchone()[0]
    
    col1, col2, col3 = st.columns(3)
    col1.metric("Total Artifacts", total_artifacts)
    
    cursor.close()
    conn.close()

def show_data_collection():
    st.header("📥 Data Collection")
    
    num_records = st.number_input(
        "Number of records to collect",
        min_value=10,
        max_value=1000,
        value=100,
        step=10
    )
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            api_key = os.getenv('HARVARD_API_KEY')
            raw_data = extract_artifacts(api_key, num_records)
            st.success(f"Extracted {len(raw_data)} records")
        
        with st.spinner("Transforming data..."):
            df_meta, df_media, df_colors = transform_artifacts(raw_data)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            conn = get_db_connection()
            load_to_database(df_meta, df_media, df_colors, conn)
            conn.close()
            st.success("Data loaded to database")

def show_sql_analytics():
    st.header("📊 SQL Analytics Dashboard")
    
    queries = {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE culture != '' 
            GROUP BY culture 
            ORDER BY count DESC 
            LIMIT 10
        """,
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE century != '' 
            GROUP BY century 
            ORDER BY count DESC
        """,
        "Department Distribution": """
            SELECT department, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE department != '' 
            GROUP BY department
        """,
        "Artifacts with Images": """
            SELECT 
                CASE WHEN EXISTS (
                    SELECT 1 FROM artifactmedia 
                    WHERE artifactmedia.artifact_id = artifactmetadata.id
                ) THEN 'With Images' ELSE 'Without Images' END as has_images,
                COUNT(*) as count
            FROM artifactmetadata
            GROUP BY has_images
        """,
        "Top Colors in Collection": """
            SELECT color_hex, COUNT(*) as count, AVG(color_percent) as avg_percent
            FROM artifactcolors
            GROUP BY color_hex
            ORDER BY count DESC
            LIMIT 10
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        conn = get_db_connection()
        df_result = pd.read_sql(queries[selected_query], conn)
        conn.close()
        
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) == 2:
            fig = px.bar(
                df_result,
                x=df_result.columns[0],
                y=df_result.columns[1],
                title=selected_query
            )
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Analytical SQL Queries

### Common Analysis Patterns

```python
# Query 1: Classification distribution
query_classification = """
SELECT classification, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE classification IS NOT NULL AND classification != ''
GROUP BY classification
ORDER BY artifact_count DESC
LIMIT 15
"""

# Query 2: Artifacts by period and culture
query_period_culture = """
SELECT period, culture, COUNT(*) as count
FROM artifactmetadata
WHERE period != '' AND culture != ''
GROUP BY period, culture
ORDER BY count DESC
LIMIT 20
"""

# Query 3: Media availability analysis
query_media_stats = """
SELECT 
    COUNT(DISTINCT am.id) as total_artifacts,
    COUNT(DISTINCT media.artifact_id) as artifacts_with_media,
    COUNT(media.media_id) as total_media_items,
    ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as media_percentage
FROM artifactmetadata am
LEFT JOIN artifactmedia media ON am.id = media.artifact_id
"""

# Query 4: Color diversity per artifact
query_color_diversity = """
SELECT 
    artifact_id,
    COUNT(*) as color_count,
    GROUP_CONCAT(color_hex) as colors
FROM artifactcolors
GROUP BY artifact_id
HAVING color_count > 5
ORDER BY color_count DESC
LIMIT 10
"""

# Query 5: Department-wise image coverage
query_dept_images = """
SELECT 
    am.department,
    COUNT(DISTINCT am.id) as total_artifacts,
    COUNT(DISTINCT media.artifact_id) as artifacts_with_images,
    ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as image_coverage_pct
FROM artifactmetadata am
LEFT JOIN artifactmedia media ON am.id = media.artifact_id
WHERE am.department != ''
GROUP BY am.department
ORDER BY image_coverage_pct DESC
"""
```

## Visualization Patterns

```python
import plotly.graph_objects as go

def create_culture_distribution_chart(df):
    """Bar chart for culture distribution"""
    fig = px.bar(
        df,
        x='culture',
        y='count',
        title='Artifact Distribution by Culture',
        labels={'culture': 'Culture', 'count': 'Number of Artifacts'},
        color='count',
        color_continuous_scale='Viridis'
    )
    fig.update_layout(xaxis_tickangle=-45)
    return fig

def create_timeline_chart(df):
    """Timeline visualization for centuries"""
    fig = px.line(
        df,
        x='century',
        y='count',
        title='Artifacts Timeline by Century',
        markers=True
    )
    return fig

def create_color_pie_chart(df):
    """Pie chart for color distribution"""
    fig = px.pie(
        df,
        values='count',
        names='color_hex',
        title='Top Colors in Collection',
        color='color_hex',
        color_discrete_map={row['color_hex']: row['color_hex'] for _, row in df.iterrows()}
    )
    return fig

def create_department_treemap(df):
    """Treemap for department distribution"""
    fig = px.treemap(
        df,
        path=['department'],
        values='count',
        title='Department Distribution'
    )
    return fig
```

## Common Patterns

### Batch Processing for Large Datasets

```python
def batch_insert(dataframe, table_name, conn, batch_size=1000):
    """Insert data in batches for better performance"""
    cursor = conn.cursor()
    total_rows = len(dataframe)
    
    for start_idx in range(0, total_rows, batch_size):
        end_idx = min(start_idx + batch_size, total_rows)
        batch = dataframe.iloc[start_idx:end_idx]
        
        # Prepare batch insert
        placeholders = ', '.join(['%s'] * len(batch.columns))
        columns = ', '.join(batch.columns)
        
        query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
        
        cursor.executemany(query, batch.values.tolist())
        conn.commit()
        
        print(f"Inserted {end_idx}/{total_rows} rows")
    
    cursor.close()
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

def safe_api_call(url, params, max_retries=3):
    """API call with retry logic"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            logging.error(f"Attempt {attempt + 1} failed: {e}")
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

### API Rate Limiting

```python
# Add delay between requests
import time

def rate_limited_fetch(api_key, page, delay=0.5):
    time.sleep(delay)
    response = requests.get(
        "https://api.harvardartmuseums.org/object",
        params={'apikey': api_key, 'page': page}
    )
    return response.json()
```

### Database Connection Issues

```python
def get_robust_connection(max_retries=3):
    """Retry database connection"""
    for attempt in range(max_retries):
        try:
            conn = mysql.connector.connect(
                host=os.getenv('DB_HOST'),
                user=os.getenv('DB_USER'),
                password=os.getenv('DB_PASSWORD'),
                database=os.getenv('DB_NAME'),
                connection_timeout=10
            )
            return conn
        except mysql.connector.Error as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2)
```

### Handling Missing Data

```python
def clean_artifact_data(df):
    """Clean and standardize artifact data"""
    # Fill empty strings with None for proper SQL handling
    df = df.replace('', None)
    
    # Truncate long strings to fit database constraints
    if 'title' in df.columns:
        df['title'] = df['title'].str[:500]
    
    # Remove duplicates
    df = df.drop_duplicates(subset=['id'])
    
    return df
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Run with specific port
streamlit run app.py --server.port 8501

# Run in background
nohup streamlit run app.py &
```

This skill provides comprehensive guidance for building data engineering pipelines with museum APIs, SQL analytics, and interactive dashboards.
