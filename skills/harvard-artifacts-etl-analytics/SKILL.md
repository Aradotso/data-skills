---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for the Harvard Art Museums API using Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data analytics dashboard with Streamlit
  - query Harvard Art Museums API and store in SQL
  - visualize museum artifact data with Plotly
  - set up data engineering pipeline for museum collections
  - analyze Harvard Art Museums artifacts with SQL
  - create museum data visualization app
  - build end-to-end analytics for art museum data
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL analytics, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:
- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color data into relational databases
- **SQL Analytics**: Executes analytical queries on structured artifact data
- **Interactive Dashboards**: Visualizes insights using Streamlit and Plotly

Architecture: `API → ETL → SQL → Analytics → Visualization`

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
DB_NAME=harvard_artifacts
```

### Database Setup

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Establish MySQL/TiDB connection"""
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
        print(f"Error connecting to database: {e}")
        return None

def create_tables(connection):
    """Create artifact tables"""
    cursor = connection.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            century VARCHAR(255),
            classification VARCHAR(255),
            division VARCHAR(255),
            department VARCHAR(255),
            dated VARCHAR(255),
            period VARCHAR(255),
            technique VARCHAR(500),
            medium VARCHAR(500),
            dimensions VARCHAR(500),
            creditline TEXT,
            url TEXT
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            primary_image_url TEXT,
            image_count INT DEFAULT 0,
            total_unique_pages INT DEFAULT 0,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_hex VARCHAR(7),
            color_percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()
    cursor.close()
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100, max_pages=10):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        api_key: Harvard API key from environment
        page: Starting page number
        size: Records per page
        max_pages: Maximum pages to fetch
    
    Returns:
        List of artifact records
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for current_page in range(page, page + max_pages):
        params = {
            'apikey': api_key,
            'page': current_page,
            'size': size,
            'hasimage': 1  # Only artifacts with images
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            print(f"Fetched page {current_page}: {len(artifacts)} artifacts")
            
            # Respect rate limits
            time.sleep(1)
            
            # Check if last page
            if len(artifacts) < size:
                break
                
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {current_page}: {e}")
            break
    
    return all_artifacts
```

### Transform: Process JSON to Relational Format

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform nested JSON artifacts into relational dataframes
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:255],
            'classification': artifact.get('classification', '')[:255],
            'division': artifact.get('division', '')[:255],
            'department': artifact.get('department', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'period': artifact.get('period', '')[:255],
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'dimensions': artifact.get('dimensions', '')[:500],
            'creditline': artifact.get('creditline', ''),
            'url': artifact.get('url', '')
        })
        
        # Extract media information
        media_records.append({
            'artifact_id': artifact.get('id'),
            'primary_image_url': artifact.get('primaryimageurl', ''),
            'image_count': len(artifact.get('images', [])),
            'total_unique_pages': artifact.get('totalpageviews', 0)
        })
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex', ''),
                'color_percent': color.get('percent', 0.0)
            })
    
    metadata_df = pd.DataFrame(metadata_records)
    media_df = pd.DataFrame(media_records)
    colors_df = pd.DataFrame(color_records)
    
    return metadata_df, media_df, colors_df
```

### Load: Batch Insert into SQL

```python
def load_to_database(connection, metadata_df, media_df, colors_df):
    """
    Load transformed data into SQL database using batch inserts
    """
    cursor = connection.cursor()
    
    # Insert metadata
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, division, department, 
         dated, period, technique, medium, dimensions, creditline, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    metadata_values = metadata_df.values.tolist()
    cursor.executemany(metadata_query, metadata_values)
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, primary_image_url, image_count, total_unique_pages)
        VALUES (%s, %s, %s, %s)
    """
    
    media_values = media_df.values.tolist()
    cursor.executemany(media_query, media_values)
    
    # Insert colors
    if not colors_df.empty:
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color_hex, color_percent)
            VALUES (%s, %s, %s)
        """
        
        colors_values = colors_df.values.tolist()
        cursor.executemany(colors_query, colors_values)
    
    connection.commit()
    cursor.close()
    print(f"Loaded {len(metadata_df)} artifacts to database")
```

## Streamlit Analytics Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px
from dotenv import load_dotenv
import os

load_dotenv()

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    elif page == "Visualizations":
        show_visualization_page()

def show_data_collection_page():
    st.header("📥 Data Collection")
    
    api_key = st.text_input("Harvard API Key", 
                            value=os.getenv('HARVARD_API_KEY', ''),
                            type="password")
    
    col1, col2 = st.columns(2)
    with col1:
        num_pages = st.number_input("Pages to Fetch", 1, 50, 10)
    with col2:
        page_size = st.number_input("Records per Page", 10, 100, 100)
    
    if st.button("🚀 Start ETL Pipeline"):
        if not api_key:
            st.error("Please provide an API key")
            return
        
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(api_key, page=1, 
                                       size=page_size, 
                                       max_pages=num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            conn = create_database_connection()
            if conn:
                load_to_database(conn, metadata_df, media_df, colors_df)
                conn.close()
                st.success("✅ ETL Pipeline completed!")

if __name__ == "__main__":
    main()
```

### SQL Analytics Queries

```python
def show_analytics_page():
    st.header("📊 SQL Analytics")
    
    # Predefined analytical queries
    queries = {
        "Top 10 Cultures by Artifact Count": """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century
            ORDER BY count DESC
        """,
        
        "Department Distribution": """
            SELECT department, COUNT(*) as total_artifacts
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY total_artifacts DESC
        """,
        
        "Most Common Colors": """
            SELECT color_hex, COUNT(*) as usage_count,
                   AVG(color_percent) as avg_percent
            FROM artifactcolors
            GROUP BY color_hex
            ORDER BY usage_count DESC
            LIMIT 20
        """,
        
        "Image Availability Analysis": """
            SELECT 
                CASE 
                    WHEN image_count = 0 THEN 'No Images'
                    WHEN image_count BETWEEN 1 AND 5 THEN '1-5 Images'
                    WHEN image_count BETWEEN 6 AND 10 THEN '6-10 Images'
                    ELSE '10+ Images'
                END as image_range,
                COUNT(*) as artifact_count
            FROM artifactmedia
            GROUP BY image_range
            ORDER BY artifact_count DESC
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Execute Query"):
        conn = create_database_connection()
        if conn:
            try:
                df = pd.read_sql(queries[selected_query], conn)
                
                st.subheader("Results")
                st.dataframe(df, use_container_width=True)
                
                # Auto-generate visualization
                if len(df.columns) == 2:
                    fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                                title=selected_query)
                    st.plotly_chart(fig, use_container_width=True)
                
            except Exception as e:
                st.error(f"Query error: {e}")
            finally:
                conn.close()
```

### Visualization Dashboard

```python
def show_visualization_page():
    st.header("📈 Data Visualizations")
    
    conn = create_database_connection()
    if not conn:
        st.error("Cannot connect to database")
        return
    
    # Culture distribution
    culture_df = pd.read_sql("""
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """, conn)
    
    fig1 = px.bar(culture_df, x='culture', y='count',
                  title='Top 15 Cultures in Collection',
                  labels={'count': 'Number of Artifacts', 'culture': 'Culture'})
    st.plotly_chart(fig1, use_container_width=True)
    
    # Color distribution
    color_df = pd.read_sql("""
        SELECT color_hex, COUNT(*) as count
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY count DESC
        LIMIT 10
    """, conn)
    
    fig2 = px.bar(color_df, x='color_hex', y='count',
                  title='Most Common Colors',
                  color='color_hex',
                  color_discrete_map={row['color_hex']: row['color_hex'] 
                                     for _, row in color_df.iterrows()})
    st.plotly_chart(fig2, use_container_width=True)
    
    conn.close()
```

## Common Patterns

### Complete ETL Workflow

```python
import os
from dotenv import load_dotenv

def run_complete_etl():
    """Complete ETL pipeline execution"""
    load_dotenv()
    
    # Step 1: Extract
    api_key = os.getenv('HARVARD_API_KEY')
    artifacts = fetch_artifacts(api_key, page=1, size=100, max_pages=5)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Step 2: Transform
    metadata_df, media_df, colors_df = transform_artifacts(artifacts)
    print(f"Transformed into {len(metadata_df)} metadata records")
    
    # Step 3: Load
    conn = create_database_connection()
    create_tables(conn)
    load_to_database(conn, metadata_df, media_df, colors_df)
    conn.close()
    print("ETL pipeline completed successfully")

if __name__ == "__main__":
    run_complete_etl()
```

### Custom Analytics Query

```python
def run_custom_analytics(query):
    """Execute custom SQL analytics query"""
    conn = create_database_connection()
    try:
        df = pd.read_sql(query, conn)
        return df
    except Exception as e:
        print(f"Error executing query: {e}")
        return None
    finally:
        conn.close()

# Example usage
query = """
    SELECT 
        a.classification,
        COUNT(DISTINCT a.id) as artifact_count,
        AVG(m.image_count) as avg_images
    FROM artifactmetadata a
    JOIN artifactmedia m ON a.id = m.artifact_id
    GROUP BY a.classification
    HAVING artifact_count > 10
    ORDER BY artifact_count DESC
"""

results = run_custom_analytics(query)
```

## Troubleshooting

### API Rate Limiting
```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff on rate limit"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            response.raise_for_status()
            return response.json()
        except HTTPError as e:
            if e.response.status_code == 429:  # Too Many Requests
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues
```python
def ensure_connection(connection):
    """Verify and reconnect if needed"""
    try:
        connection.ping(reconnect=True, attempts=3, delay=5)
    except Error:
        connection = create_database_connection()
    return connection
```

### Memory Optimization for Large Datasets
```python
def load_in_chunks(connection, df, table_name, chunk_size=1000):
    """Load large dataframe in chunks"""
    total_rows = len(df)
    for start in range(0, total_rows, chunk_size):
        end = min(start + chunk_size, total_rows)
        chunk = df.iloc[start:end]
        chunk.to_sql(table_name, connection, 
                    if_exists='append', index=False)
        print(f"Loaded rows {start}-{end}/{total_rows}")
```

### Handling Missing Data
```python
def clean_artifact_data(df):
    """Clean and standardize artifact data"""
    # Fill missing values
    df['culture'] = df['culture'].fillna('Unknown')
    df['century'] = df['century'].fillna('Unknown')
    
    # Truncate long strings
    df['title'] = df['title'].str[:500]
    df['medium'] = df['medium'].str[:500]
    
    # Remove duplicates
    df = df.drop_duplicates(subset=['id'])
    
    return df
```

This skill provides comprehensive guidance for building ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit.
