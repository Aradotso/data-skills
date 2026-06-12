---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline using Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline with Harvard Art Museums API
  - create ETL workflow for museum artifacts data
  - set up analytics dashboard for Harvard art collection
  - extract and transform museum API data to SQL
  - visualize Harvard artifacts with Streamlit
  - implement museum data engineering pipeline
  - analyze Harvard art museums collection data
  - build artifact metadata analytics app
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to build and work with the Harvard Art Museums data engineering and analytics application. The project demonstrates real-world ETL pipelines, SQL database design, and interactive data visualization using the Harvard Art Museums API.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is an end-to-end data pipeline that:

1. **Extracts** artifact data from the Harvard Art Museums API with pagination and rate limiting
2. **Transforms** nested JSON into normalized relational tables
3. **Loads** data into SQL databases (MySQL/TiDB Cloud) with proper schema design
4. **Analyzes** data using 20+ predefined SQL queries
5. **Visualizes** results through interactive Streamlit dashboards with Plotly charts

**Architecture Flow**: `API → ETL → SQL → Analytics → Visualization`

## Installation

### Prerequisites

```bash
# Python 3.8+ required
python --version

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

### Environment Setup

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

**Get API Key**: Register at [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Run on specific port
streamlit run app.py --server.port 8501

# Run with custom config
streamlit run app.py --server.headless true
```

## Database Schema Design

### Core Tables

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    technique VARCHAR(500),
    period VARCHAR(255),
    dated VARCHAR(255),
    description TEXT,
    dimensions VARCHAR(500),
    medium VARCHAR(500),
    credit_line TEXT,
    provenance TEXT,
    url VARCHAR(500),
    total_unique_pages INT,
    total_page_views INT,
    object_number VARCHAR(100),
    accession_year INT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    image_height INT,
    image_width INT,
    format VARCHAR(50),
    caption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    color_percentage FLOAT,
    hue VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);
```

## ETL Pipeline Implementation

### Extract: Fetching Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        api_key: Harvard API key
        page: Page number (default: 1)
        size: Results per page (max: 100)
    
    Returns:
        dict: JSON response with artifact records
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params, timeout=30)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return None

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=50)
print(f"Total records: {data['info']['totalrecords']}")
```

### Transform: Processing Nested JSON

```python
import pandas as pd

def transform_artifact_metadata(records):
    """
    Transform API records into normalized metadata structure
    
    Args:
        records: List of artifact records from API
    
    Returns:
        DataFrame: Cleaned artifact metadata
    """
    metadata_list = []
    
    for record in records:
        metadata = {
            'artifact_id': record.get('id'),
            'title': record.get('title', 'Unknown'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'division': record.get('division'),
            'technique': record.get('technique'),
            'period': record.get('period'),
            'dated': record.get('dated'),
            'description': record.get('description'),
            'dimensions': record.get('dimensions'),
            'medium': record.get('medium'),
            'credit_line': record.get('creditline'),
            'provenance': record.get('provenance'),
            'url': record.get('url'),
            'total_unique_pages': record.get('totaluniquepageviews', 0),
            'total_page_views': record.get('totalpageviews', 0),
            'object_number': record.get('objectnumber'),
            'accession_year': record.get('accessionyear')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(records):
    """
    Extract media/image information from artifacts
    """
    media_list = []
    
    for record in records:
        artifact_id = record.get('id')
        images = record.get('images', [])
        
        for image in images:
            media = {
                'artifact_id': artifact_id,
                'image_url': image.get('baseimageurl'),
                'image_height': image.get('height'),
                'image_width': image.get('width'),
                'format': image.get('format'),
                'caption': image.get('caption')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(records):
    """
    Extract color data from artifacts
    """
    color_list = []
    
    for record in records:
        artifact_id = record.get('id')
        colors = record.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_name': color.get('color'),
                'color_percentage': color.get('percent'),
                'hue': color.get('hue')
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### Load: Inserting into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def get_database_connection():
    """
    Create database connection using environment variables
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

def batch_insert_metadata(df, connection):
    """
    Batch insert artifact metadata with conflict handling
    
    Args:
        df: DataFrame with artifact metadata
        connection: Active database connection
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata (
        artifact_id, title, culture, century, classification, 
        department, division, technique, period, dated,
        description, dimensions, medium, credit_line, provenance,
        url, total_unique_pages, total_page_views, 
        object_number, accession_year
    ) VALUES (
        %s, %s, %s, %s, %s, %s, %s, %s, %s, %s,
        %s, %s, %s, %s, %s, %s, %s, %s, %s, %s
    ) ON DUPLICATE KEY UPDATE
        title = VALUES(title),
        culture = VALUES(culture),
        century = VALUES(century)
    """
    
    # Convert DataFrame to list of tuples
    records = df.to_records(index=False).tolist()
    
    try:
        cursor.executemany(insert_query, records)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Insert error: {e}")
        connection.rollback()
    finally:
        cursor.close()

def batch_insert_media(df, connection):
    """Batch insert artifact media/images"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (
        artifact_id, image_url, image_height, 
        image_width, format, caption
    ) VALUES (%s, %s, %s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    
    try:
        cursor.executemany(insert_query, records)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Media insert error: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

## Complete ETL Workflow

```python
def run_etl_pipeline(num_pages=5, page_size=50):
    """
    Execute complete ETL pipeline
    
    Args:
        num_pages: Number of API pages to fetch
        page_size: Records per page
    """
    api_key = os.getenv('HARVARD_API_KEY')
    connection = get_database_connection()
    
    if not connection:
        print("Failed to connect to database")
        return
    
    all_records = []
    
    # Extract
    print("Starting data extraction...")
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}/{num_pages}")
        data = fetch_artifacts(api_key, page=page, size=page_size)
        
        if data and 'records' in data:
            all_records.extend(data['records'])
    
    print(f"Extracted {len(all_records)} total records")
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(all_records)
    media_df = transform_artifact_media(all_records)
    colors_df = transform_artifact_colors(all_records)
    
    # Load
    print("Loading data into database...")
    batch_insert_metadata(metadata_df, connection)
    batch_insert_media(media_df, connection)
    batch_insert_colors(colors_df, connection)
    
    connection.close()
    print("ETL pipeline completed successfully!")

# Execute pipeline
if __name__ == "__main__":
    run_etl_pipeline(num_pages=10, page_size=100)
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Query 1: Top 10 cultures by artifact count
query_cultures = """
SELECT 
    culture,
    COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Query 2: Artifacts by century distribution
query_centuries = """
SELECT 
    century,
    COUNT(*) as count,
    ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Query 3: Departments with most images
query_dept_media = """
SELECT 
    a.department,
    COUNT(DISTINCT a.artifact_id) as artifacts_with_images,
    COUNT(m.media_id) as total_images
FROM artifactmetadata a
JOIN artifactmedia m ON a.artifact_id = m.artifact_id
WHERE a.department IS NOT NULL
GROUP BY a.department
ORDER BY total_images DESC
LIMIT 10
"""

# Query 4: Most common colors in collection
query_top_colors = """
SELECT 
    color_name,
    COUNT(*) as usage_count,
    ROUND(AVG(color_percentage), 2) as avg_percentage
FROM artifactcolors
WHERE color_name IS NOT NULL
GROUP BY color_name
ORDER BY usage_count DESC
LIMIT 15
"""

# Query 5: Artifacts with highest page views
query_popular = """
SELECT 
    title,
    culture,
    century,
    total_page_views,
    url
FROM artifactmetadata
WHERE total_page_views > 0
ORDER BY total_page_views DESC
LIMIT 20
"""
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px
import plotly.graph_objects as go

def execute_query(query, connection):
    """Execute SQL query and return DataFrame"""
    try:
        return pd.read_sql(query, connection)
    except Exception as e:
        st.error(f"Query execution error: {e}")
        return pd.DataFrame()

def create_bar_chart(df, x_col, y_col, title):
    """Create interactive bar chart with Plotly"""
    fig = px.bar(
        df, 
        x=x_col, 
        y=y_col,
        title=title,
        labels={x_col: x_col.replace('_', ' ').title(), 
                y_col: y_col.replace('_', ' ').title()}
    )
    fig.update_layout(xaxis_tickangle=-45)
    return fig

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("End-to-end ETL pipeline and analytics for museum artifacts")
    
    # Sidebar navigation
    st.sidebar.title("Navigation")
    page = st.sidebar.selectbox(
        "Choose a section",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    # Database connection
    connection = get_database_connection()
    
    if page == "ETL Pipeline":
        st.header("📥 ETL Pipeline Execution")
        
        col1, col2 = st.columns(2)
        with col1:
            num_pages = st.number_input("Number of pages", 1, 50, 5)
        with col2:
            page_size = st.number_input("Records per page", 10, 100, 50)
        
        if st.button("🚀 Run ETL Pipeline"):
            with st.spinner("Running ETL pipeline..."):
                run_etl_pipeline(num_pages, page_size)
                st.success("ETL pipeline completed!")
    
    elif page == "SQL Analytics":
        st.header("📊 SQL Analytics")
        
        queries = {
            "Top Cultures": query_cultures,
            "Century Distribution": query_centuries,
            "Department Media": query_dept_media,
            "Top Colors": query_top_colors,
            "Popular Artifacts": query_popular
        }
        
        selected_query = st.selectbox("Select Analysis", list(queries.keys()))
        
        if st.button("Execute Query"):
            df = execute_query(queries[selected_query], connection)
            
            st.subheader("Query Results")
            st.dataframe(df, use_container_width=True)
            
            # Auto-visualization
            if len(df.columns) >= 2:
                fig = create_bar_chart(df, df.columns[0], df.columns[1], selected_query)
                st.plotly_chart(fig, use_container_width=True)
    
    elif page == "Visualizations":
        st.header("📈 Data Visualizations")
        
        # Example: Color distribution pie chart
        color_query = """
        SELECT color_name, COUNT(*) as count
        FROM artifactcolors
        GROUP BY color_name
        ORDER BY count DESC
        LIMIT 10
        """
        
        df_colors = execute_query(color_query, connection)
        
        if not df_colors.empty:
            fig_pie = px.pie(
                df_colors, 
                values='count', 
                names='color_name',
                title='Top 10 Colors in Collection'
            )
            st.plotly_chart(fig_pie, use_container_width=True)
    
    if connection:
        connection.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(api_key, pages, delay=1):
    """Fetch data with rate limiting to avoid API throttling"""
    results = []
    
    for page in range(1, pages + 1):
        data = fetch_artifacts(api_key, page=page)
        if data:
            results.extend(data['records'])
        time.sleep(delay)  # Wait between requests
    
    return results
```

### Incremental ETL Updates

```python
def incremental_etl(last_updated_date):
    """
    Fetch only artifacts updated after a certain date
    """
    api_key = os.getenv('HARVARD_API_KEY')
    
    params = {
        'apikey': api_key,
        'lastupdate': last_updated_date,  # Format: YYYY-MM-DD
        'size': 100
    }
    
    response = requests.get(
        "https://api.harvardartmuseums.org/object",
        params=params
    )
    
    return response.json()
```

## Troubleshooting

### API Connection Issues

```python
# Test API connectivity
def test_api_connection(api_key):
    try:
        response = requests.get(
            f"https://api.harvardartmuseums.org/object?apikey={api_key}&size=1"
        )
        if response.status_code == 200:
            print("✓ API connection successful")
            return True
        elif response.status_code == 401:
            print("✗ Invalid API key")
            return False
    except Exception as e:
        print(f"✗ Connection error: {e}")
        return False
```

### Database Connection Errors

```python
# Test database connection
def test_db_connection():
    try:
        conn = get_database_connection()
        if conn and conn.is_connected():
            print("✓ Database connection successful")
            conn.close()
            return True
        else:
            print("✗ Failed to connect to database")
            return False
    except Error as e:
        print(f"✗ Database error: {e}")
        return False
```

### Handling NULL Values

```python
# Clean DataFrame before inserting
def clean_dataframe(df):
    """Replace None/NaN with appropriate defaults"""
    df = df.fillna({
        'culture': 'Unknown',
        'century': 'Undated',
        'description': '',
        'total_page_views': 0
    })
    return df
```

### Memory Management for Large Datasets

```python
def chunked_etl(total_pages, chunk_size=5):
    """Process data in chunks to manage memory"""
    for start_page in range(1, total_pages + 1, chunk_size):
        end_page = min(start_page + chunk_size, total_pages + 1)
        print(f"Processing pages {start_page}-{end_page}")
        
        # Process chunk
        run_etl_pipeline(num_pages=chunk_size, page_size=50)
        
        # Clear memory
        import gc
        gc.collect()
```

## Performance Optimization

### Batch Inserts with Transaction Control

```python
def optimized_batch_insert(df, table_name, connection, batch_size=1000):
    """Insert data in batches with transaction management"""
    cursor = connection.cursor()
    
    try:
        connection.start_transaction()
        
        for i in range(0, len(df), batch_size):
            batch = df.iloc[i:i+batch_size]
            records = batch.to_records(index=False).tolist()
            
            # Dynamic query generation based on table
            placeholders = ', '.join(['%s'] * len(batch.columns))
            columns = ', '.join(batch.columns)
            query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
            
            cursor.executemany(query, records)
            print(f"Inserted batch {i//batch_size + 1}")
        
        connection.commit()
        print(f"Successfully inserted {len(df)} records")
        
    except Exception as e:
        connection.rollback()
        print(f"Error: {e}")
    finally:
        cursor.close()
```

This skill provides comprehensive guidance for building and extending the Harvard Art Museums ETL pipeline, enabling AI agents to assist developers with data engineering tasks, analytics, and visualization workflows.
