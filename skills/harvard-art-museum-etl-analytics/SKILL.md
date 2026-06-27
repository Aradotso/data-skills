---
name: harvard-art-museum-etl-analytics
description: Build end-to-end data pipelines using Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create a data engineering pipeline for museum artifacts
  - set up Harvard Art Museums data collection and analytics
  - build a Streamlit dashboard for art museum data
  - implement ETL for Harvard Art Museums collection
  - create SQL analytics for museum artifact data
  - how to visualize Harvard Art Museums API data
  - set up museum artifacts data pipeline with Python
---

# Harvard Art Museum ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Collects artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color data into SQL databases
- **SQL Analytics**: Executes predefined analytical queries on structured museum data
- **Interactive Dashboard**: Visualizes query results using Streamlit and Plotly
- **Database Design**: Implements normalized relational schema with proper foreign key relationships

## Installation

### Prerequisites

- Python 3.8+
- MySQL or TiDB Cloud account
- Harvard Art Museums API key (get from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api))

### Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

### Required Dependencies

```txt
streamlit>=1.28.0
pandas>=2.0.0
requests>=2.31.0
mysql-connector-python>=8.0.33
plotly>=5.17.0
python-dotenv>=1.0.0
```

## Key Components

### 1. API Data Collection

Extract artifact data from Harvard Art Museums API with pagination:

```python
import requests
import os
import time

def fetch_artifacts(api_key, page=1, size=100, max_pages=10):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for current_page in range(1, max_pages + 1):
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
            
            records = data.get('records', [])
            all_artifacts.extend(records)
            
            # Check if more pages exist
            if current_page >= data.get('info', {}).get('pages', 0):
                break
                
            # Rate limiting
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {current_page}: {e}")
            break
    
    return all_artifacts

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts = fetch_artifacts(api_key, max_pages=5)
print(f"Collected {len(artifacts)} artifacts")
```

### 2. ETL Pipeline

Transform and load data into SQL database:

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def create_tables(connection):
    """Create normalized database schema"""
    cursor = connection.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            technique VARCHAR(500),
            medium VARCHAR(500),
            dated VARCHAR(255),
            period VARCHAR(255),
            url TEXT,
            creditline TEXT
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_url TEXT,
            media_type VARCHAR(100),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_hex VARCHAR(10),
            color_percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()
    cursor.close()

def transform_artifacts(artifacts):
    """Transform raw API data into structured DataFrames"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'dated': artifact.get('dated', '')[:255],
            'period': artifact.get('period', '')[:255],
            'url': artifact.get('url', ''),
            'creditline': artifact.get('creditline', '')
        })
        
        # Media
        images = artifact.get('images', [])
        for image in images:
            media_records.append({
                'artifact_id': artifact.get('id'),
                'image_url': image.get('baseimageurl', ''),
                'media_type': 'image'
            })
        
        # Colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex', ''),
                'color_percent': color.get('percent', 0.0)
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

def load_data(connection, metadata_df, media_df, colors_df):
    """Batch insert data into database"""
    cursor = connection.cursor()
    
    # Insert metadata
    metadata_sql = """
        INSERT IGNORE INTO artifactmetadata 
        (id, title, culture, century, classification, department, 
         technique, medium, dated, period, url, creditline)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    cursor.executemany(metadata_sql, metadata_df.values.tolist())
    
    # Insert media
    media_sql = """
        INSERT INTO artifactmedia (artifact_id, image_url, media_type)
        VALUES (%s, %s, %s)
    """
    cursor.executemany(media_sql, media_df.values.tolist())
    
    # Insert colors
    colors_sql = """
        INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
        VALUES (%s, %s, %s)
    """
    cursor.executemany(colors_sql, colors_df.values.tolist())
    
    connection.commit()
    cursor.close()
    print(f"Loaded {len(metadata_df)} artifacts with media and color data")
```

### 3. SQL Analytics Queries

Example analytical queries for the dashboard:

```python
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY artifact_count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND department != ''
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN media_count > 0 THEN 'With Media' ELSE 'No Media' END as media_status,
            COUNT(*) as artifact_count
        FROM (
            SELECT m.id, COUNT(im.media_id) as media_count
            FROM artifactmetadata m
            LEFT JOIN artifactmedia im ON m.id = im.artifact_id
            GROUP BY m.id
        ) as media_stats
        GROUP BY media_status
    """,
    
    "Top Color Usage": """
        SELECT color_hex, COUNT(*) as usage_count, 
               AVG(color_percent) as avg_percent
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY usage_count DESC
        LIMIT 20
    """,
    
    "Artifacts by Classification": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Medium Analysis": """
        SELECT medium, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE medium IS NOT NULL AND medium != ''
        GROUP BY medium
        ORDER BY artifact_count DESC
        LIMIT 15
    """
}

def execute_query(connection, query_name):
    """Execute analytical query and return results as DataFrame"""
    cursor = connection.cursor()
    query = ANALYTICAL_QUERIES[query_name]
    cursor.execute(query)
    
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results, columns=columns)
```

### 4. Streamlit Dashboard

Build interactive analytics dashboard:

```python
import streamlit as st
import plotly.express as px
import plotly.graph_objects as go

def create_streamlit_app():
    """Main Streamlit application"""
    st.set_page_config(
        page_title="Harvard Art Museum Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Data Analytics")
    st.markdown("---")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    connection = create_database_connection()
    
    if not connection:
        st.error("Failed to connect to database. Check credentials.")
        return
    
    # Data collection section
    st.sidebar.subheader("Data Collection")
    if st.sidebar.button("Fetch New Data"):
        with st.spinner("Fetching artifacts from API..."):
            api_key = os.getenv('HARVARD_API_KEY')
            artifacts = fetch_artifacts(api_key, max_pages=3)
            
            create_tables(connection)
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            load_data(connection, metadata_df, media_df, colors_df)
            
            st.sidebar.success(f"Loaded {len(artifacts)} artifacts!")
    
    # Analytics section
    st.header("📊 Analytics Dashboard")
    
    # Query selector
    query_options = list(ANALYTICAL_QUERIES.keys())
    selected_query = st.selectbox("Select Analysis", query_options)
    
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            results_df = execute_query(connection, selected_query)
            
            # Display results
            st.subheader(f"Results: {selected_query}")
            st.dataframe(results_df, use_container_width=True)
            
            # Visualization
            if len(results_df) > 0:
                st.subheader("Visualization")
                
                # Auto-generate chart based on data
                x_col = results_df.columns[0]
                y_col = results_df.columns[1]
                
                fig = px.bar(
                    results_df,
                    x=x_col,
                    y=y_col,
                    title=selected_query,
                    color=y_col,
                    color_continuous_scale='Viridis'
                )
                
                fig.update_layout(
                    xaxis_title=x_col,
                    yaxis_title=y_col,
                    showlegend=False
                )
                
                st.plotly_chart(fig, use_container_width=True)
    
    connection.close()

# Run the app
if __name__ == "__main__":
    create_streamlit_app()
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def get_last_artifact_id(connection):
    """Get the highest artifact ID in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def fetch_new_artifacts_only(api_key, last_id):
    """Fetch only artifacts newer than last_id"""
    # Implement incremental loading logic
    pass
```

### Pattern 2: Error Handling for API Calls

```python
def safe_api_request(url, params, max_retries=3):
    """API request with retry logic"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
    return None
```

### Pattern 3: Batch Processing

```python
def process_in_batches(artifacts, batch_size=100):
    """Process large datasets in batches"""
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i + batch_size]
        metadata_df, media_df, colors_df = transform_artifacts(batch)
        load_data(connection, metadata_df, media_df, colors_df)
        print(f"Processed batch {i//batch_size + 1}")
```

## Configuration

### Environment Variables

```bash
# Required
HARVARD_API_KEY=your_api_key
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts

# Optional
MAX_API_PAGES=10
BATCH_SIZE=100
API_RATE_LIMIT_DELAY=0.5
```

### Database Configuration for TiDB Cloud

```python
connection = mysql.connector.connect(
    host=os.getenv('DB_HOST'),  # TiDB Cloud endpoint
    port=4000,
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME'),
    ssl_ca='/path/to/ca.pem',  # For TiDB Cloud SSL
    ssl_verify_cert=True
)
```

## Troubleshooting

### API Rate Limiting

```python
# Add delay between requests
import time
time.sleep(0.5)  # 500ms between calls

# Handle 429 Too Many Requests
if response.status_code == 429:
    retry_after = int(response.headers.get('Retry-After', 60))
    time.sleep(retry_after)
```

### Database Connection Issues

```python
# Test connection
try:
    connection.ping(reconnect=True, attempts=3, delay=5)
except Error:
    connection = create_database_connection()
```

### Large Dataset Memory Issues

```python
# Use chunked processing
chunks = pd.read_sql(query, connection, chunksize=1000)
for chunk in chunks:
    process_chunk(chunk)
```

### Missing Data Handling

```python
# Handle None values in transformation
def safe_get(dictionary, key, default='', max_length=None):
    value = dictionary.get(key, default)
    if max_length and value:
        return str(value)[:max_length]
    return value or default
```

This skill provides everything needed to build production-ready data engineering pipelines with museum API data, SQL analytics, and interactive visualizations.
