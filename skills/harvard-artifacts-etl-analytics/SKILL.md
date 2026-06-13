---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for the Harvard Art Museums API
  - create a data engineering pipeline for museum artifacts
  - set up analytics dashboard for Harvard art collection data
  - fetch and analyze Harvard museum artifact data
  - build a Streamlit app with Harvard Art Museums API
  - create SQL analytics for art museum collections
  - implement ETL workflow for cultural heritage data
  - visualize Harvard artifacts data with Plotly
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into SQL databases
- **Database Design**: Relational schema with proper foreign key relationships
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboards**: Streamlit-based visualization with Plotly charts

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required Dependencies:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from: https://harvardartmuseums.org/collections/api

Create a `.env` file or configure in Streamlit:
```bash
HARVARD_API_KEY=your_api_key_here
```

### Database Configuration

The project supports MySQL and TiDB Cloud. Configure connection parameters:

```python
import os
from dotenv import load_dotenv

load_dotenv()

DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

## Database Schema

### Core Tables

**artifactmetadata**
- `id` (PRIMARY KEY)
- `title`
- `culture`
- `period`
- `century`
- `classification`
- `department`
- `dated`
- `accession_number`

**artifactmedia**
- `media_id` (PRIMARY KEY)
- `artifact_id` (FOREIGN KEY)
- `base_url`
- `format`
- `description`
- `technique`

**artifactcolors**
- `color_id` (PRIMARY KEY)
- `artifact_id` (FOREIGN KEY)
- `color_hex`
- `color_name`
- `percentage`
- `css3`

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_pages=5, page_size=100):
    """
    Fetch artifact data from Harvard Art Museums API with pagination
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
            print(f"Fetched page {page}/{num_pages}")
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return all_artifacts
```

### Transform: Structure Data for SQL

```python
def transform_artifacts(raw_artifacts):
    """
    Transform raw JSON artifacts into relational dataframes
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'period': artifact.get('period', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'accession_number': artifact.get('accessionNumber', 'Unknown')
        })
        
        # Extract media/images
        for idx, image in enumerate(artifact.get('images', [])):
            media_records.append({
                'media_id': f"{artifact.get('id')}_{idx}",
                'artifact_id': artifact.get('id'),
                'base_url': image.get('baseimageurl', ''),
                'format': image.get('format', 'Unknown'),
                'description': image.get('description', ''),
                'technique': image.get('technique', 'Unknown')
            })
        
        # Extract colors
        for idx, color in enumerate(artifact.get('colors', [])):
            color_records.append({
                'color_id': f"{artifact.get('id')}_{idx}",
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex', '#000000'),
                'color_name': color.get('color', 'Unknown'),
                'percentage': color.get('percent', 0.0),
                'css3': color.get('css3', '#000000')
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def create_connection(config):
    """Create database connection"""
    try:
        connection = mysql.connector.connect(**config)
        print("Database connection successful")
        return connection
    except Error as e:
        print(f"Error: {e}")
        return None

def create_tables(connection):
    """Create database schema"""
    cursor = connection.cursor()
    
    # Create metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            accession_number VARCHAR(100)
        )
    """)
    
    # Create media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id VARCHAR(50) PRIMARY KEY,
            artifact_id INT,
            base_url VARCHAR(500),
            format VARCHAR(50),
            description TEXT,
            technique VARCHAR(200),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Create colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id VARCHAR(50) PRIMARY KEY,
            artifact_id INT,
            color_hex VARCHAR(7),
            color_name VARCHAR(100),
            percentage FLOAT,
            css3 VARCHAR(7),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()
    print("Tables created successfully")

def batch_insert_dataframe(connection, df, table_name):
    """Batch insert DataFrame into SQL table"""
    cursor = connection.cursor()
    
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    insert_query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    data_tuples = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} rows into {table_name}")
    except Error as e:
        print(f"Error inserting into {table_name}: {e}")
        connection.rollback()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import os

# Page configuration
st.set_page_config(
    page_title="Harvard Artifacts Analytics",
    page_icon="🏛️",
    layout="wide"
)

st.title("🏛️ Harvard Art Museums - ETL & Analytics Dashboard")

# Sidebar for API key and database config
with st.sidebar:
    st.header("⚙️ Configuration")
    
    api_key = st.text_input(
        "Harvard API Key",
        type="password",
        value=os.getenv('HARVARD_API_KEY', '')
    )
    
    st.header("📊 Database Settings")
    db_host = st.text_input("Host", value=os.getenv('DB_HOST', 'localhost'))
    db_user = st.text_input("User", value=os.getenv('DB_USER', 'root'))
    db_password = st.text_input("Password", type="password")
    db_name = st.text_input("Database", value="harvard_artifacts")

# Main tabs
tab1, tab2, tab3 = st.tabs(["📥 ETL Pipeline", "📊 SQL Analytics", "📈 Visualizations"])

with tab1:
    st.header("Extract, Transform, Load")
    
    num_pages = st.slider("Number of pages to fetch", 1, 10, 3)
    
    if st.button("▶️ Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            raw_data = fetch_artifacts(api_key, num_pages=num_pages)
            st.success(f"Fetched {len(raw_data)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata, df_media, df_colors = transform_artifacts(raw_data)
            st.success("Data transformation complete")
        
        with st.spinner("Loading into database..."):
            connection = create_connection({
                'host': db_host,
                'user': db_user,
                'password': db_password,
                'database': db_name
            })
            
            if connection:
                create_tables(connection)
                batch_insert_dataframe(connection, df_metadata, 'artifactmetadata')
                batch_insert_dataframe(connection, df_media, 'artifactmedia')
                batch_insert_dataframe(connection, df_colors, 'artifactcolors')
                connection.close()
                st.success("✅ ETL Pipeline Complete!")
```

## Analytical SQL Queries

### Pre-built Analytics

```python
ANALYTICAL_QUERIES = {
    "Artifacts by Department": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != 'Unknown'
        GROUP BY century
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture != 'Unknown'
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Media Format Distribution": """
        SELECT format, COUNT(*) as count
        FROM artifactmedia
        GROUP BY format
        ORDER BY count DESC
    """,
    
    "Top 10 Colors Used": """
        SELECT color_name, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts with Most Images": """
        SELECT m.title, m.culture, COUNT(media.media_id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia media ON m.id = media.artifact_id
        GROUP BY m.id, m.title, m.culture
        ORDER BY image_count DESC
        LIMIT 10
    """,
    
    "Color Diversity by Artifact": """
        SELECT m.title, m.classification, COUNT(DISTINCT c.color_name) as color_count
        FROM artifactmetadata m
        JOIN artifactcolors c ON m.id = c.artifact_id
        GROUP BY m.id, m.title, m.classification
        HAVING color_count >= 5
        ORDER BY color_count DESC
        LIMIT 10
    """
}

def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    try:
        df = pd.read_sql(query, connection)
        return df
    except Exception as e:
        st.error(f"Query error: {e}")
        return None
```

### Analytics Dashboard Tab

```python
with tab2:
    st.header("SQL Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("🔍 Run Query"):
        connection = create_connection({
            'host': db_host,
            'user': db_user,
            'password': db_password,
            'database': db_name
        })
        
        if connection:
            st.code(ANALYTICAL_QUERIES[query_name], language='sql')
            
            result_df = execute_query(connection, ANALYTICAL_QUERIES[query_name])
            
            if result_df is not None and not result_df.empty:
                st.dataframe(result_df, use_container_width=True)
                
                # Auto-generate visualization
                if len(result_df.columns) == 2:
                    import plotly.express as px
                    
                    fig = px.bar(
                        result_df,
                        x=result_df.columns[0],
                        y=result_df.columns[1],
                        title=query_name
                    )
                    st.plotly_chart(fig, use_container_width=True)
            
            connection.close()
```

## Visualization Examples

### Plotly Chart Generation

```python
import plotly.express as px
import plotly.graph_objects as go

def create_bar_chart(df, x_col, y_col, title):
    """Create interactive bar chart"""
    fig = px.bar(
        df,
        x=x_col,
        y=y_col,
        title=title,
        color=y_col,
        color_continuous_scale='viridis'
    )
    fig.update_layout(
        xaxis_title=x_col.replace('_', ' ').title(),
        yaxis_title=y_col.replace('_', ' ').title(),
        hovermode='x unified'
    )
    return fig

def create_pie_chart(df, names_col, values_col, title):
    """Create interactive pie chart"""
    fig = px.pie(
        df,
        names=names_col,
        values=values_col,
        title=title,
        hole=0.4
    )
    return fig

def create_color_distribution(df):
    """Create color distribution visualization"""
    fig = go.Figure(data=[
        go.Bar(
            x=df['color_name'],
            y=df['usage_count'],
            marker=dict(
                color=df['color_name'].map(lambda x: x.lower()),
                line=dict(color='black', width=1)
            )
        )
    ])
    fig.update_layout(title="Color Usage Distribution")
    return fig
```

## Common Patterns

### Complete ETL Workflow

```python
def run_complete_etl(api_key, db_config, num_pages=5):
    """Complete ETL workflow from API to database"""
    
    # Extract
    print("Step 1: Extracting data from API...")
    raw_artifacts = fetch_artifacts(api_key, num_pages)
    
    # Transform
    print("Step 2: Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(raw_artifacts)
    
    # Load
    print("Step 3: Loading into database...")
    connection = create_connection(db_config)
    
    if connection:
        create_tables(connection)
        batch_insert_dataframe(connection, df_metadata, 'artifactmetadata')
        batch_insert_dataframe(connection, df_media, 'artifactmedia')
        batch_insert_dataframe(connection, df_colors, 'artifactcolors')
        connection.close()
        print("ETL Complete!")
        return True
    
    return False
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_api_fetch(api_key, page, page_size=100):
    """Fetch with error handling and retry logic"""
    max_retries = 3
    
    for attempt in range(max_retries):
        try:
            response = requests.get(
                "https://api.harvardartmuseums.org/object",
                params={'apikey': api_key, 'page': page, 'size': page_size},
                timeout=10
            )
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            logger.warning(f"Attempt {attempt + 1} failed: {e}")
            if attempt == max_retries - 1:
                logger.error(f"Failed to fetch page {page} after {max_retries} attempts")
                return None
    
    return None
```

## Troubleshooting

### API Rate Limiting
```python
import time

def fetch_with_rate_limit(api_key, num_pages, delay=1):
    """Fetch data with rate limiting"""
    all_data = []
    
    for page in range(1, num_pages + 1):
        data = safe_api_fetch(api_key, page)
        if data:
            all_data.extend(data.get('records', []))
        time.sleep(delay)  # Respect rate limits
    
    return all_data
```

### Database Connection Issues
```python
def test_database_connection(config):
    """Test database connectivity"""
    try:
        connection = mysql.connector.connect(**config)
        if connection.is_connected():
            db_info = connection.get_server_info()
            print(f"Connected to MySQL Server version {db_info}")
            connection.close()
            return True
    except Error as e:
        print(f"Connection failed: {e}")
        return False
```

### Missing Data Handling
```python
def safe_get(dictionary, key, default='Unknown'):
    """Safely extract values from nested JSON"""
    value = dictionary.get(key, default)
    return value if value else default
```

This skill provides comprehensive guidance for building production-ready ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit.
