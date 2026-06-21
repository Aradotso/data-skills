---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with MySQL and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and analyze Harvard museum collection data
  - set up data engineering pipeline with Streamlit
  - query Harvard Art Museums API and store in SQL
  - visualize museum artifact data with Plotly
  - build end-to-end data pipeline for art collections
  - analyze artifact metadata with SQL queries
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Collects artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color data into relational databases
- **SQL Analytics**: Executes predefined analytical queries on structured artifact data
- **Interactive Visualization**: Displays query results through Streamlit dashboards with Plotly charts

**Architecture**: API → ETL → SQL (MySQL/TiDB) → Analytics → Visualization (Streamlit + Plotly)

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup

```bash
# Clone repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install requirements
pip install -r requirements.txt

# Set up environment variables
cat > .env << EOF
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
EOF
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Register for a free API key
3. Add key to `.env` file as `HARVARD_API_KEY`

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Key Components

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(num_pages=5, page_size=100):
    """Fetch artifacts with pagination"""
    artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': API_KEY,
            'size': page_size,
            'page': page
        }
        
        response = requests.get(BASE_URL, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}: {len(data.get('records', []))} artifacts")
        else:
            print(f"Error on page {page}: {response.status_code}")
            break
    
    return artifacts

# Collect data
artifacts_data = fetch_artifacts(num_pages=3, page_size=50)
print(f"Total artifacts collected: {len(artifacts_data)}")
```

### 2. ETL Pipeline - Transform

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform raw API data into structured dataframes"""
    
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        artifact_id = artifact.get('id')
        
        # Extract metadata
        metadata = {
            'artifact_id': artifact_id,
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'accession_year': artifact.get('accessionyear'),
            'provenance': artifact.get('provenance'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        images = artifact.get('images', [])
        for img in images:
            media = {
                'artifact_id': artifact_id,
                'image_url': img.get('baseimageurl'),
                'image_width': img.get('width'),
                'image_height': img.get('height'),
                'format': img.get('format')
            }
            media_list.append(media)
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_name': color.get('color'),
                'percentage': color.get('percent')
            }
            colors_list.append(color_data)
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

### 3. Database Connection and Loading

```python
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
        print(f"Error connecting to MySQL: {e}")
        return None

def create_tables(connection):
    """Create database schema"""
    cursor = connection.cursor()
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            dated VARCHAR(255),
            accession_year INT,
            provenance TEXT,
            url TEXT
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_url TEXT,
            image_width INT,
            image_height INT,
            format VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_hex VARCHAR(10),
            color_name VARCHAR(100),
            percentage FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    connection.commit()
    cursor.close()

def load_dataframes_to_db(df_metadata, df_media, df_colors, connection):
    """Batch insert dataframes into database"""
    cursor = connection.cursor()
    
    # Insert metadata
    for _, row in df_metadata.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (artifact_id, title, culture, century, classification, department, 
             dated, accession_year, provenance, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """, tuple(row))
    
    # Insert media
    for _, row in df_media.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (artifact_id, image_url, image_width, image_height, format)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in df_colors.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (artifact_id, color_hex, color_name, percentage)
            VALUES (%s, %s, %s, %s)
        """, tuple(row))
    
    connection.commit()
    cursor.close()
```

### 4. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICAL_QUERIES = {
    "Total Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
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
    
    "Artifacts with Images": """
        SELECT 
            CASE WHEN EXISTS (
                SELECT 1 FROM artifactmedia WHERE artifactmedia.artifact_id = artifactmetadata.artifact_id
            ) THEN 'Has Images' ELSE 'No Images' END as image_status,
            COUNT(*) as count
        FROM artifactmetadata
        GROUP BY image_status
    """,
    
    "Most Common Colors": """
        SELECT color_name, COUNT(*) as frequency, AVG(percentage) as avg_percentage
        FROM artifactcolors
        WHERE color_name IS NOT NULL
        GROUP BY color_name
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Accession Year Trends": """
        SELECT accession_year, COUNT(*) as artifacts_acquired
        FROM artifactmetadata
        WHERE accession_year IS NOT NULL
        GROUP BY accession_year
        ORDER BY accession_year DESC
        LIMIT 20
    """
}

def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    cursor = connection.cursor()
    cursor.execute(query)
    
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    
    cursor.close()
    
    return pd.DataFrame(results, columns=columns)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar
    st.sidebar.header("Configuration")
    
    # Connect to database
    connection = create_database_connection()
    
    if not connection:
        st.error("Failed to connect to database. Check credentials.")
        return
    
    # Query selection
    st.sidebar.subheader("Select Analysis")
    query_name = st.sidebar.selectbox(
        "Choose Query",
        list(ANALYTICAL_QUERIES.keys())
    )
    
    # Execute query
    if st.sidebar.button("Run Query"):
        with st.spinner("Executing query..."):
            query = ANALYTICAL_QUERIES[query_name]
            df_result = execute_query(connection, query)
            
            st.subheader(f"Results: {query_name}")
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) == 2:
                fig = px.bar(
                    df_result,
                    x=df_result.columns[0],
                    y=df_result.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)
    
    # ETL Section
    st.sidebar.header("ETL Operations")
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(num_pages=2, page_size=50)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata, df_media, df_colors = transform_artifacts(artifacts)
            st.success("Data transformed")
        
        with st.spinner("Loading to database..."):
            create_tables(connection)
            load_dataframes_to_db(df_metadata, df_media, df_colors, connection)
            st.success("Data loaded successfully")
    
    connection.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Workflow

```python
def run_complete_etl_pipeline(num_pages=5):
    """Execute full ETL pipeline"""
    
    # Extract
    print("Extracting data from API...")
    raw_data = fetch_artifacts(num_pages=num_pages, page_size=100)
    
    # Transform
    print("Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(raw_data)
    
    # Load
    print("Loading to database...")
    connection = create_database_connection()
    create_tables(connection)
    load_dataframes_to_db(df_metadata, df_media, df_colors, connection)
    connection.close()
    
    print("ETL pipeline completed successfully")

# Execute
run_complete_etl_pipeline(num_pages=10)
```

### Custom Query Execution

```python
def run_custom_analytics(custom_query):
    """Execute custom SQL analytics"""
    connection = create_database_connection()
    
    try:
        df_result = execute_query(connection, custom_query)
        
        # Visualize if applicable
        if len(df_result.columns) == 2:
            fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1])
            fig.show()
        
        return df_result
    
    finally:
        connection.close()

# Example usage
query = """
    SELECT classification, COUNT(*) as count
    FROM artifactmetadata
    GROUP BY classification
    HAVING count > 10
    ORDER BY count DESC
"""

results = run_custom_analytics(query)
print(results)
```

## Configuration

### Environment Variables

```bash
# .env file
HARVARD_API_KEY=your_harvard_api_key
DB_HOST=localhost  # or TiDB Cloud host
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

### Database Configuration (TiDB Cloud)

```python
def create_tidb_connection():
    """Connect to TiDB Cloud"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 4000)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME'),
        ssl_ca='/path/to/ca.pem',  # TiDB Cloud SSL certificate
        ssl_verify_cert=True
    )
    return connection
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_artifacts_with_retry(num_pages=5, page_size=100, delay=1):
    """Fetch with rate limiting and retry logic"""
    artifacts = []
    
    for page in range(1, num_pages + 1):
        max_retries = 3
        for attempt in range(max_retries):
            try:
                response = requests.get(BASE_URL, params={
                    'apikey': API_KEY,
                    'size': page_size,
                    'page': page
                })
                
                if response.status_code == 429:  # Rate limited
                    wait_time = delay * (2 ** attempt)
                    print(f"Rate limited. Waiting {wait_time}s...")
                    time.sleep(wait_time)
                    continue
                
                response.raise_for_status()
                artifacts.extend(response.json().get('records', []))
                break
                
            except requests.exceptions.RequestException as e:
                print(f"Error on page {page}, attempt {attempt + 1}: {e}")
                if attempt == max_retries - 1:
                    raise
        
        time.sleep(delay)  # Polite delay between requests
    
    return artifacts
```

### Database Connection Issues

```python
def safe_db_operation(operation_func, *args, **kwargs):
    """Execute database operation with error handling"""
    connection = None
    try:
        connection = create_database_connection()
        if not connection:
            raise Exception("Failed to establish database connection")
        
        result = operation_func(connection, *args, **kwargs)
        return result
        
    except Error as e:
        print(f"Database error: {e}")
        raise
        
    finally:
        if connection and connection.is_connected():
            connection.close()
```

### Handling NULL Values

```python
def clean_transform_artifacts(raw_data):
    """Transform with NULL handling"""
    df_metadata, df_media, df_colors = transform_artifacts(raw_data)
    
    # Replace None with appropriate defaults
    df_metadata['culture'] = df_metadata['culture'].fillna('Unknown')
    df_metadata['century'] = df_metadata['century'].fillna('Undated')
    df_metadata['accession_year'] = df_metadata['accession_year'].fillna(0)
    
    return df_metadata, df_media, df_colors
```

This skill provides comprehensive guidance for building ETL pipelines and analytics applications with the Harvard Art Museums API, MySQL/TiDB databases, and Streamlit visualization.
