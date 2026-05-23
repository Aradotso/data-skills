---
name: harvard-art-museums-etl-analytics
description: ETL pipeline and analytics application for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and analyze Harvard art collection data
  - set up museum artifact data engineering pipeline
  - query Harvard Art Museums API and store in SQL
  - visualize museum artifact data with Streamlit
  - implement art collection data pipeline
  - analyze museum metadata with SQL queries
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build and work with end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL workflows, SQL analytics, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection application:
- Extracts artifact data from Harvard Art Museums API with pagination
- Transforms nested JSON into relational database tables
- Loads structured data into MySQL/TiDB Cloud
- Provides 20+ analytical SQL queries for insights
- Visualizes results through interactive Plotly charts in Streamlit

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

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

**Requirements:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Running the Application

```bash
# Launch Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Harvard Art Museums API Integration

### API Authentication

```python
import requests
import os

API_KEY = os.getenv("HARVARD_API_KEY")
BASE_URL = "https://api.harvardartmuseums.org"

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts with pagination"""
    url = f"{BASE_URL}/object"
    params = {
        "apikey": API_KEY,
        "page": page,
        "size": size
    }
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()
```

### Handling Pagination

```python
def fetch_all_artifacts(max_pages=10):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(page=page, size=100)
        artifacts = data.get("records", [])
        
        if not artifacts:
            break
            
        all_artifacts.extend(artifacts)
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return all_artifacts
```

## ETL Pipeline Implementation

### Extract Phase

```python
import pandas as pd

def extract_artifact_metadata(artifacts):
    """Extract core metadata from artifacts"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            "artifact_id": artifact.get("id"),
            "title": artifact.get("title"),
            "culture": artifact.get("culture"),
            "century": artifact.get("century"),
            "classification": artifact.get("classification"),
            "department": artifact.get("department"),
            "dated": artifact.get("dated"),
            "medium": artifact.get("medium"),
            "technique": artifact.get("technique"),
            "provenance": artifact.get("provenance")
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)
```

### Transform Phase

```python
def extract_media_data(artifacts):
    """Extract and flatten media/image data"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get("id")
        images = artifact.get("images", [])
        
        for image in images:
            media = {
                "artifact_id": artifact_id,
                "image_id": image.get("imageid"),
                "base_url": image.get("baseimageurl"),
                "format": image.get("format"),
                "width": image.get("width"),
                "height": image.get("height"),
                "renditionnumber": image.get("renditionnumber")
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def extract_color_data(artifacts):
    """Extract color palette information"""
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get("id")
        colors = artifact.get("colors", [])
        
        for color in colors:
            color_data = {
                "artifact_id": artifact_id,
                "color_hex": color.get("hex"),
                "color_name": color.get("color"),
                "spectrum": color.get("spectrum"),
                "percentage": color.get("percent")
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### Load Phase

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Create MySQL connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv("DB_HOST"),
            user=os.getenv("DB_USER"),
            password=os.getenv("DB_PASSWORD"),
            database=os.getenv("DB_NAME")
        )
        return connection
    except Error as e:
        print(f"Error connecting to MySQL: {e}")
        return None

def create_tables(connection):
    """Create database schema"""
    cursor = connection.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            dated VARCHAR(255),
            medium TEXT,
            technique TEXT,
            provenance TEXT
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_id VARCHAR(255),
            base_url TEXT,
            format VARCHAR(50),
            width INT,
            height INT,
            renditionnumber VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_hex VARCHAR(7),
            color_name VARCHAR(100),
            spectrum VARCHAR(100),
            percentage DECIMAL(5,2),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    connection.commit()
    cursor.close()

def batch_insert_metadata(connection, df):
    """Batch insert artifact metadata"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (artifact_id, title, culture, century, classification, 
         department, dated, medium, technique, provenance)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    cursor.close()
```

## Streamlit Dashboard Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("Data Engineering & Analytics Pipeline")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    elif page == "Visualizations":
        show_visualizations_page()

def show_etl_page():
    """ETL execution interface"""
    st.header("📊 ETL Pipeline Execution")
    
    num_pages = st.slider("Number of pages to fetch", 1, 50, 10)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            artifacts = fetch_all_artifacts(max_pages=num_pages)
            st.success(f"Extracted {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df = extract_artifact_metadata(artifacts)
            media_df = extract_media_data(artifacts)
            colors_df = extract_color_data(artifacts)
            st.success("Data transformation complete")
        
        with st.spinner("Loading to database..."):
            conn = create_database_connection()
            create_tables(conn)
            batch_insert_metadata(conn, metadata_df)
            st.success("Data loaded successfully!")
```

## Analytical SQL Queries

### Sample Queries

```python
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Artifacts with Images": """
        SELECT 
            CASE WHEN m.artifact_id IS NOT NULL THEN 'With Images' 
                 ELSE 'No Images' END as image_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.artifact_id = m.artifact_id
        GROUP BY image_status
    """,
    
    "Top Colors Used": """
        SELECT color_name, COUNT(*) as usage_count,
               AVG(percentage) as avg_percentage
        FROM artifactcolors
        WHERE color_name IS NOT NULL
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """
}

def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)

def show_analytics_page():
    """Display analytics queries"""
    st.header("📈 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Run Query"):
        conn = create_database_connection()
        
        with st.spinner("Executing query..."):
            df = execute_query(conn, ANALYTICAL_QUERIES[query_name])
            
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                            title=query_name)
                st.plotly_chart(fig, use_container_width=True)
```

## Visualization Patterns

```python
def create_culture_chart(df):
    """Create interactive culture distribution chart"""
    fig = px.bar(
        df, 
        x='culture', 
        y='artifact_count',
        title='Artifact Distribution by Culture',
        labels={'artifact_count': 'Number of Artifacts'},
        color='artifact_count',
        color_continuous_scale='Viridis'
    )
    fig.update_layout(xaxis_tickangle=-45)
    return fig

def create_color_pie_chart(df):
    """Create color usage pie chart"""
    fig = px.pie(
        df, 
        values='usage_count', 
        names='color_name',
        title='Color Distribution in Artifacts'
    )
    return fig

def create_timeline_chart(df):
    """Create century timeline visualization"""
    fig = px.line(
        df, 
        x='century', 
        y='count',
        title='Artifacts Across Centuries',
        markers=True
    )
    return fig
```

## Configuration Best Practices

### Environment Variables (.env)

```bash
# Harvard API
HARVARD_API_KEY=your_key_from_harvard_api

# Database Configuration
DB_HOST=your_tidb_host.com
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_secure_password
DB_NAME=harvard_artifacts

# Streamlit Configuration
STREAMLIT_SERVER_PORT=8501
STREAMLIT_SERVER_ADDRESS=localhost
```

### Loading Configuration

```python
from dotenv import load_dotenv
import os

load_dotenv()

CONFIG = {
    "api_key": os.getenv("HARVARD_API_KEY"),
    "db_host": os.getenv("DB_HOST"),
    "db_port": int(os.getenv("DB_PORT", 3306)),
    "db_user": os.getenv("DB_USER"),
    "db_password": os.getenv("DB_PASSWORD"),
    "db_name": os.getenv("DB_NAME")
}
```

## Common Troubleshooting

### API Rate Limiting

```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            response.raise_for_status()
            return response.json()
        except HTTPError as e:
            if e.response.status_code == 429:  # Too Many Requests
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def ensure_connection(connection):
    """Check and reconnect if needed"""
    try:
        connection.ping(reconnect=True, attempts=3, delay=5)
    except Error:
        connection = create_database_connection()
    return connection
```

### Memory Management for Large Datasets

```python
def process_in_chunks(artifacts, chunk_size=1000):
    """Process large datasets in chunks"""
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        
        metadata_df = extract_artifact_metadata(chunk)
        conn = create_database_connection()
        batch_insert_metadata(conn, metadata_df)
        conn.close()
        
        # Clear memory
        del metadata_df
```

## Testing the Pipeline

```python
def test_etl_pipeline():
    """Test complete ETL workflow"""
    # Test API extraction
    artifacts = fetch_artifacts(page=1, size=10)
    assert len(artifacts.get("records", [])) > 0
    
    # Test transformation
    metadata_df = extract_artifact_metadata(artifacts["records"])
    assert not metadata_df.empty
    
    # Test database load
    conn = create_database_connection()
    assert conn is not None
    
    create_tables(conn)
    batch_insert_metadata(conn, metadata_df.head(5))
    
    # Verify data
    result = pd.read_sql("SELECT COUNT(*) as count FROM artifactmetadata", conn)
    assert result['count'][0] > 0
    
    conn.close()
    print("✓ ETL pipeline test passed")
```

This skill enables comprehensive data engineering workflows with museum artifact data, from API integration through SQL analytics to interactive visualization.
