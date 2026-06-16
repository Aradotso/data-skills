---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - set up ETL pipeline for museum artifact data
  - create analytics dashboard with Harvard Art Museums data
  - query Harvard Art Museums API and store in SQL
  - build Streamlit app for art museum data visualization
  - process and transform Harvard Art Museums JSON data
  - design SQL schema for museum artifacts
  - analyze art museum collections with Python
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application for the Harvard Art Museums API. It demonstrates ETL pipeline development, SQL database design, and interactive visualization using Streamlit. The application extracts artifact metadata, transforms nested JSON into relational tables, loads data into SQL databases, and provides analytical dashboards with 20+ predefined queries.

## Project Structure

The application follows this architecture:
**API → ETL → SQL → Analytics → Visualization**

Key components:
- **Data Collection**: Harvard Art Museums API integration with pagination
- **ETL Pipeline**: Python-based extraction, transformation, and loading
- **Database**: MySQL/TiDB Cloud with relational schema
- **Analytics**: SQL query engine with predefined analytical queries
- **Visualization**: Streamlit frontend with Plotly charts

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

Required packages:
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

Create a `.env` file or set environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_art_museums
```

Get your API key from: https://www.harvardartmuseums.org/collections/api

### Database Schema

The application uses three main tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    objectnumber VARCHAR(100),
    dated VARCHAR(255),
    division VARCHAR(255),
    technique VARCHAR(500),
    period VARCHAR(255)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    artifact_id INT,
    media_id INT,
    baseimageurl VARCHAR(500),
    height INT,
    width INT,
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## API Integration

### Fetching Data from Harvard Art Museums

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        api_key: Your Harvard API key
        page: Page number (default 1)
        size: Results per page (max 100)
    
    Returns:
        dict: JSON response with records and pagination info
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)

print(f"Total records: {data['info']['totalrecords']}")
print(f"Pages: {data['info']['pages']}")
print(f"Fetched: {len(data['records'])} artifacts")
```

### Handling Pagination

```python
def fetch_all_artifacts(api_key, max_pages=10):
    """
    Fetch multiple pages of artifacts
    
    Args:
        api_key: Your Harvard API key
        max_pages: Maximum number of pages to fetch
    
    Returns:
        list: Combined list of all artifact records
    """
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page, size=100)
            all_artifacts.extend(data['records'])
            
            print(f"Fetched page {page}/{max_pages}")
            
            # Respect rate limits
            import time
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

## ETL Pipeline

### Extract and Transform

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform artifact records into metadata dataframe
    
    Args:
        artifacts: List of artifact records from API
    
    Returns:
        pd.DataFrame: Cleaned metadata
    """
    metadata = []
    
    for artifact in artifacts:
        record = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'objectnumber': artifact.get('objectnumber'),
            'dated': artifact.get('dated'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'period': artifact.get('period')
        }
        metadata.append(record)
    
    return pd.DataFrame(metadata)

def transform_artifact_media(artifacts):
    """
    Extract media information from artifacts
    
    Args:
        artifacts: List of artifact records from API
    
    Returns:
        pd.DataFrame: Media data with artifact relationships
    """
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media = {
                'artifact_id': artifact_id,
                'media_id': img.get('imageid'),
                'baseimageurl': img.get('baseimageurl'),
                'height': img.get('height'),
                'width': img.get('width'),
                'format': img.get('format')
            }
            media_records.append(media)
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """
    Extract color data from artifacts
    
    Args:
        artifacts: List of artifact records from API
    
    Returns:
        pd.DataFrame: Color analysis data
    """
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(color_data)
    
    return pd.DataFrame(color_records)
```

### Load to SQL Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """
    Create MySQL database connection
    
    Returns:
        connection: MySQL connection object
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT', 3306),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def load_metadata_to_sql(df, connection):
    """
    Load artifact metadata to SQL database
    
    Args:
        df: Pandas DataFrame with metadata
        connection: MySQL connection object
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, 
     objectnumber, dated, division, technique, period)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    # Batch insert
    records = df.to_records(index=False)
    data = [tuple(r) for r in records]
    
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} metadata records")
    cursor.close()

def load_media_to_sql(df, connection):
    """
    Load artifact media to SQL database
    
    Args:
        df: Pandas DataFrame with media data
        connection: MySQL connection object
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, media_id, baseimageurl, height, width, format)
    VALUES (%s, %s, %s, %s, %s, %s)
    """
    
    records = df.to_records(index=False)
    data = [tuple(r) for r in records]
    
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} media records")
    cursor.close()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    st.markdown("An end-to-end data engineering and analytics application")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # API Key input
    api_key = st.sidebar.text_input(
        "Harvard API Key",
        type="password",
        value=os.getenv('HARVARD_API_KEY', '')
    )
    
    # Database connection
    db_connected = check_database_connection()
    
    if db_connected:
        st.sidebar.success("✅ Database Connected")
    else:
        st.sidebar.error("❌ Database Not Connected")
    
    # Main tabs
    tab1, tab2, tab3 = st.tabs(["Data Collection", "SQL Analytics", "Visualizations"])
    
    with tab1:
        show_data_collection(api_key)
    
    with tab2:
        show_sql_analytics()
    
    with tab3:
        show_visualizations()

def show_data_collection(api_key):
    """Data collection interface"""
    st.header("📥 Collect Artifact Data")
    
    num_pages = st.slider("Number of pages to fetch", 1, 50, 5)
    
    if st.button("Fetch Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_all_artifacts(api_key, max_pages=num_pages)
            
            st.success(f"Fetched {len(artifacts)} artifacts")
            
            # Transform data
            metadata_df = transform_artifact_metadata(artifacts)
            media_df = transform_artifact_media(artifacts)
            colors_df = transform_artifact_colors(artifacts)
            
            # Load to database
            conn = create_database_connection()
            if conn:
                load_metadata_to_sql(metadata_df, conn)
                load_media_to_sql(media_df, conn)
                # Load colors similarly
                conn.close()
                
                st.success("✅ Data loaded to database")

if __name__ == "__main__":
    main()
```

## SQL Analytics Queries

### Predefined Analytical Queries

```python
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY century
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Top Colors in Collection": """
        SELECT color, COUNT(*) as occurrence, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY occurrence DESC
        LIMIT 15
    """,
    
    "Media Format Distribution": """
        SELECT format, COUNT(*) as count
        FROM artifactmedia
        WHERE format IS NOT NULL
        GROUP BY format
        ORDER BY count DESC
    """,
    
    "Artifacts with Most Images": """
        SELECT m.id, m.title, COUNT(media.media_id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia media ON m.id = media.artifact_id
        GROUP BY m.id, m.title
        ORDER BY image_count DESC
        LIMIT 20
    """,
    
    "Color Diversity by Artifact": """
        SELECT artifact_id, COUNT(DISTINCT color) as color_count
        FROM artifactcolors
        GROUP BY artifact_id
        ORDER BY color_count DESC
        LIMIT 10
    """
}

def execute_query(query_name):
    """
    Execute SQL query and return results
    
    Args:
        query_name: Name of the query from ANALYTICAL_QUERIES
    
    Returns:
        pd.DataFrame: Query results
    """
    conn = create_database_connection()
    
    if not conn:
        st.error("Database connection failed")
        return None
    
    query = ANALYTICAL_QUERIES[query_name]
    
    try:
        df = pd.read_sql(query, conn)
        conn.close()
        return df
    except Error as e:
        st.error(f"Query error: {e}")
        conn.close()
        return None

def show_sql_analytics():
    """SQL Analytics interface"""
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICAL_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            results = execute_query(query_name)
            
            if results is not None and not results.empty:
                st.dataframe(results)
                
                # Auto-generate visualization
                if len(results.columns) == 2:
                    fig = px.bar(
                        results,
                        x=results.columns[0],
                        y=results.columns[1],
                        title=query_name
                    )
                    st.plotly_chart(fig)
```

## Visualization

```python
import plotly.express as px
import plotly.graph_objects as go

def create_culture_distribution_chart(df):
    """
    Create interactive bar chart for culture distribution
    
    Args:
        df: DataFrame with culture and count columns
    
    Returns:
        plotly figure
    """
    fig = px.bar(
        df,
        x='culture',
        y='count',
        title='Artifact Distribution by Culture',
        labels={'culture': 'Culture', 'count': 'Number of Artifacts'},
        color='count',
        color_continuous_scale='viridis'
    )
    
    fig.update_layout(
        xaxis_tickangle=-45,
        height=500
    )
    
    return fig

def create_color_spectrum_chart(df):
    """
    Create pie chart for color spectrum distribution
    
    Args:
        df: DataFrame with color and occurrence columns
    
    Returns:
        plotly figure
    """
    fig = px.pie(
        df,
        values='occurrence',
        names='color',
        title='Color Distribution in Collection',
        hole=0.3
    )
    
    return fig

def create_timeline_chart(df):
    """
    Create timeline visualization for centuries
    
    Args:
        df: DataFrame with century and count columns
    
    Returns:
        plotly figure
    """
    fig = px.line(
        df,
        x='century',
        y='count',
        title='Artifacts Across Centuries',
        markers=True
    )
    
    fig.update_layout(
        xaxis_title='Century',
        yaxis_title='Number of Artifacts',
        hovermode='x unified'
    )
    
    return fig
```

## Common Patterns

### Complete ETL Workflow

```python
def run_etl_pipeline(api_key, num_pages=10):
    """
    Complete ETL pipeline from API to database
    
    Args:
        api_key: Harvard API key
        num_pages: Number of pages to process
    """
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_all_artifacts(api_key, max_pages=num_pages)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(artifacts)
    media_df = transform_artifact_media(artifacts)
    colors_df = transform_artifact_colors(artifacts)
    
    # Load
    print("Loading to database...")
    conn = create_database_connection()
    
    if conn:
        load_metadata_to_sql(metadata_df, conn)
        load_media_to_sql(media_df, conn)
        # Load colors table
        conn.close()
        print("ETL pipeline completed successfully")
    else:
        print("ETL pipeline failed - database connection error")
```

## Troubleshooting

### API Rate Limiting

```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(api_key, page, retries=3, delay=1):
    """
    Fetch with exponential backoff retry
    
    Args:
        api_key: Harvard API key
        page: Page number
        retries: Number of retry attempts
        delay: Initial delay in seconds
    
    Returns:
        dict: API response
    """
    for attempt in range(retries):
        try:
            return fetch_artifacts(api_key, page=page)
        except HTTPError as e:
            if e.response.status_code == 429:  # Rate limited
                wait_time = delay * (2 ** attempt)
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    
    raise Exception(f"Failed after {retries} retries")
```

### Database Connection Issues

```python
def check_database_connection():
    """
    Verify database connection and schema
    
    Returns:
        bool: True if connection successful
    """
    try:
        conn = create_database_connection()
        if not conn:
            return False
        
        cursor = conn.cursor()
        cursor.execute("SHOW TABLES")
        tables = cursor.fetchall()
        
        required_tables = {'artifactmetadata', 'artifactmedia', 'artifactcolors'}
        existing_tables = {table[0] for table in tables}
        
        if not required_tables.issubset(existing_tables):
            print(f"Missing tables: {required_tables - existing_tables}")
            return False
        
        conn.close()
        return True
        
    except Error as e:
        print(f"Database check failed: {e}")
        return False
```

### Handling Null Values

```python
def clean_artifact_data(df):
    """
    Clean and handle null values in artifact data
    
    Args:
        df: Raw DataFrame
    
    Returns:
        pd.DataFrame: Cleaned DataFrame
    """
    # Replace None with 'Unknown' for string columns
    string_cols = ['title', 'culture', 'century', 'classification']
    for col in string_cols:
        if col in df.columns:
            df[col] = df[col].fillna('Unknown')
    
    # Drop rows where ID is null
    df = df.dropna(subset=['id'])
    
    # Convert numeric columns
    numeric_cols = ['height', 'width', 'percent']
    for col in numeric_cols:
        if col in df.columns:
            df[col] = pd.to_numeric(df[col], errors='coerce')
    
    return df
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

The application provides a complete workflow for collecting museum data, performing ETL operations, running SQL analytics, and visualizing insights through an interactive dashboard.
