---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, MySQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with Harvard museum artifacts
  - extract and analyze Harvard Art Museums API data
  - set up data engineering pipeline for museum collections
  - visualize Harvard artifacts data with Streamlit
  - query and analyze museum artifact metadata with SQL
  - implement batch data ingestion from Harvard API
  - create interactive museum data analytics app
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering and analytics application that demonstrates:
- **API Integration**: Collecting artifact data from the Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracting, transforming, and loading nested JSON into relational database tables
- **SQL Analytics**: Running analytical queries on artifact metadata, media, and color data
- **Interactive Visualization**: Building Streamlit dashboards with Plotly charts

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

### Required Dependencies

```python
# requirements.txt
streamlit>=1.20.0
pandas>=1.5.0
requests>=2.28.0
mysql-connector-python>=8.0.0
plotly>=5.11.0
python-dotenv>=0.21.0
```

## Database Schema

The ETL pipeline creates three main tables with foreign key relationships:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    period VARCHAR(200),
    classification VARCHAR(200),
    medium VARCHAR(300),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(300),
    dated VARCHAR(100),
    url VARCHAR(500),
    accession_number VARCHAR(100),
    object_number VARCHAR(100)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(500),
    image_width INT,
    image_height INT,
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

## API Integration

### Fetching Data from Harvard Art Museums API

```python
import requests
import os
from typing import Dict, List

def fetch_artifacts(api_key: str, page: int = 1, size: int = 100) -> Dict:
    """
    Fetch artifacts from Harvard Art Museums API with pagination.
    
    Args:
        api_key: Harvard API key from environment
        page: Page number for pagination
        size: Number of records per page (max 100)
    
    Returns:
        JSON response with artifact data
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

def collect_all_artifacts(api_key: str, max_records: int = 1000) -> List[Dict]:
    """
    Collect multiple pages of artifact data with rate limiting.
    
    Args:
        api_key: Harvard API key
        max_records: Maximum number of records to fetch
    
    Returns:
        List of artifact dictionaries
    """
    import time
    
    artifacts = []
    page = 1
    size = 100
    
    while len(artifacts) < max_records:
        try:
            data = fetch_artifacts(api_key, page=page, size=size)
            records = data.get('records', [])
            
            if not records:
                break
                
            artifacts.extend(records)
            page += 1
            
            # Rate limiting: 2500 requests/day
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return artifacts[:max_records]
```

## ETL Pipeline Implementation

### Extract and Transform

```python
import pandas as pd
from typing import Tuple

def transform_artifacts_to_dataframes(artifacts: List[Dict]) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
    """
    Transform nested JSON artifacts into relational dataframes.
    
    Args:
        artifacts: List of artifact dictionaries from API
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'period': artifact.get('period'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'accession_number': artifact.get('accessionyear'),
            'object_number': artifact.get('objectnumber')
        }
        metadata_records.append(metadata)
        
        # Extract media/images
        images = artifact.get('images', [])
        for img in images:
            media = {
                'artifact_id': artifact.get('id'),
                'image_url': img.get('baseimageurl'),
                'image_width': img.get('width'),
                'image_height': img.get('height')
            }
            media_records.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent')
            }
            color_records.append(color_data)
    
    metadata_df = pd.DataFrame(metadata_records)
    media_df = pd.DataFrame(media_records)
    colors_df = pd.DataFrame(color_records)
    
    return metadata_df, media_df, colors_df
```

### Load to Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection(host: str, user: str, password: str, database: str):
    """Create MySQL database connection."""
    try:
        connection = mysql.connector.connect(
            host=host,
            user=user,
            password=password,
            database=database
        )
        return connection
    except Error as e:
        print(f"Error connecting to MySQL: {e}")
        return None

def batch_insert_metadata(connection, df: pd.DataFrame):
    """
    Batch insert artifact metadata with conflict handling.
    
    Args:
        connection: MySQL connection object
        df: DataFrame with artifact metadata
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, period, classification, medium, 
     department, division, technique, dated, url, accession_number, object_number)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture), century=VALUES(century)
    """
    
    # Convert DataFrame to list of tuples
    records = df.to_records(index=False).tolist()
    
    try:
        cursor.executemany(insert_query, records)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error inserting metadata: {e}")
        connection.rollback()
    finally:
        cursor.close()

def batch_insert_media(connection, df: pd.DataFrame):
    """Batch insert artifact media/images."""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, image_url, image_width, image_height)
    VALUES (%s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    
    try:
        cursor.executemany(insert_query, records)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
    finally:
        cursor.close()

def batch_insert_colors(connection, df: pd.DataFrame):
    """Batch insert artifact color data."""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
    VALUES (%s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    
    try:
        cursor.executemany(insert_query, records)
        connection.commit()
        print(f"Inserted {cursor.rowcount} color records")
    except Error as e:
        print(f"Error inserting colors: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

## SQL Analytics Queries

### Common Analytical Queries

```python
# Query 1: Artifact distribution by culture
culture_query = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 20
"""

# Query 2: Artifacts by century and classification
century_classification_query = """
SELECT century, classification, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL AND classification IS NOT NULL
GROUP BY century, classification
ORDER BY count DESC
LIMIT 25
"""

# Query 3: Media availability analysis
media_analysis_query = """
SELECT 
    am.department,
    COUNT(DISTINCT am.id) as total_artifacts,
    COUNT(DISTINCT med.artifact_id) as artifacts_with_media,
    ROUND(COUNT(DISTINCT med.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as media_percentage
FROM artifactmetadata am
LEFT JOIN artifactmedia med ON am.id = med.artifact_id
WHERE am.department IS NOT NULL
GROUP BY am.department
ORDER BY media_percentage DESC
"""

# Query 4: Most common colors across artifacts
color_popularity_query = """
SELECT 
    color_hex,
    COUNT(*) as usage_count,
    AVG(color_percent) as avg_percentage
FROM artifactcolors
GROUP BY color_hex
ORDER BY usage_count DESC
LIMIT 15
"""

# Query 5: Artifacts with most images
most_images_query = """
SELECT 
    am.title,
    am.culture,
    am.century,
    COUNT(med.media_id) as image_count
FROM artifactmetadata am
JOIN artifactmedia med ON am.id = med.artifact_id
GROUP BY am.id, am.title, am.culture, am.century
ORDER BY image_count DESC
LIMIT 20
"""

def execute_analytics_query(connection, query: str) -> pd.DataFrame:
    """
    Execute SQL analytics query and return results as DataFrame.
    
    Args:
        connection: MySQL connection
        query: SQL query string
    
    Returns:
        DataFrame with query results
    """
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Error executing query: {e}")
        return pd.DataFrame()
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px
import os
from dotenv import load_dotenv

load_dotenv()

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar configuration
    with st.sidebar:
        st.header("⚙️ Configuration")
        
        api_key = os.getenv('HARVARD_API_KEY')
        db_config = {
            'host': os.getenv('DB_HOST'),
            'user': os.getenv('DB_USER'),
            'password': os.getenv('DB_PASSWORD'),
            'database': os.getenv('DB_NAME')
        }
        
        st.success("✅ Configuration loaded")
    
    # Navigation
    menu = st.sidebar.radio(
        "Navigation",
        ["Data Collection", "Analytics Dashboard", "Custom Query"]
    )
    
    if menu == "Data Collection":
        show_data_collection_page(api_key, db_config)
    elif menu == "Analytics Dashboard":
        show_analytics_dashboard(db_config)
    elif menu == "Custom Query":
        show_custom_query_page(db_config)

def show_data_collection_page(api_key, db_config):
    """Display data collection interface."""
    st.header("📥 Data Collection from Harvard API")
    
    num_records = st.slider("Number of records to fetch", 100, 5000, 1000, 100)
    
    if st.button("🚀 Start ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            # Fetch artifacts
            artifacts = collect_all_artifacts(api_key, max_records=num_records)
            st.success(f"✅ Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            # Transform to dataframes
            metadata_df, media_df, colors_df = transform_artifacts_to_dataframes(artifacts)
            st.success(f"✅ Transformed into {len(metadata_df)} metadata, {len(media_df)} media, {len(colors_df)} color records")
        
        with st.spinner("Loading to database..."):
            # Load to database
            conn = create_database_connection(**db_config)
            if conn:
                batch_insert_metadata(conn, metadata_df)
                batch_insert_media(conn, media_df)
                batch_insert_colors(conn, colors_df)
                conn.close()
                st.success("✅ Data loaded successfully!")
        
        # Show sample data
        st.subheader("Sample Data Preview")
        st.dataframe(metadata_df.head(10))

def show_analytics_dashboard(db_config):
    """Display analytics dashboard with pre-built queries."""
    st.header("📊 Analytics Dashboard")
    
    conn = create_database_connection(**db_config)
    if not conn:
        st.error("❌ Cannot connect to database")
        return
    
    # Query selector
    queries = {
        "Artifacts by Culture": culture_query,
        "Century & Classification": century_classification_query,
        "Media Availability": media_analysis_query,
        "Color Popularity": color_popularity_query,
        "Most Images": most_images_query
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("📈 Run Analysis"):
        with st.spinner("Executing query..."):
            df = execute_analytics_query(conn, queries[selected_query])
        
        if not df.empty:
            # Display table
            st.subheader("Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            st.subheader("Visualization")
            if len(df.columns) >= 2:
                fig = px.bar(
                    df.head(20),
                    x=df.columns[0],
                    y=df.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)
    
    conn.close()

def show_custom_query_page(db_config):
    """Allow users to run custom SQL queries."""
    st.header("🔍 Custom SQL Query")
    
    query = st.text_area(
        "Enter SQL Query",
        height=200,
        placeholder="SELECT * FROM artifactmetadata LIMIT 10;"
    )
    
    if st.button("Execute Query"):
        conn = create_database_connection(**db_config)
        if conn:
            df = execute_analytics_query(conn, query)
            if not df.empty:
                st.dataframe(df)
                st.download_button(
                    "📥 Download CSV",
                    df.to_csv(index=False),
                    "query_results.csv",
                    "text/csv"
                )
            conn.close()

if __name__ == "__main__":
    main()
```

### Running the Dashboard

```bash
# Set environment variables
export HARVARD_API_KEY="your_api_key"
export DB_HOST="localhost"
export DB_USER="root"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"

# Run Streamlit app
streamlit run app.py
```

## Complete ETL Workflow

```python
import os
from dotenv import load_dotenv

def run_complete_etl_pipeline():
    """
    Execute complete ETL pipeline from API to database.
    """
    # Load configuration
    load_dotenv()
    api_key = os.getenv('HARVARD_API_KEY')
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    # Step 1: Extract
    print("Step 1: Extracting data from Harvard API...")
    artifacts = collect_all_artifacts(api_key, max_records=1000)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Step 2: Transform
    print("Step 2: Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts_to_dataframes(artifacts)
    print(f"Created {len(metadata_df)} metadata, {len(media_df)} media, {len(colors_df)} color records")
    
    # Step 3: Load
    print("Step 3: Loading data to database...")
    conn = create_database_connection(**db_config)
    if conn:
        batch_insert_metadata(conn, metadata_df)
        batch_insert_media(conn, media_df)
        batch_insert_colors(conn, colors_df)
        conn.close()
        print("ETL pipeline completed successfully!")
    else:
        print("Failed to connect to database")

if __name__ == "__main__":
    run_complete_etl_pipeline()
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff for rate limiting
import time
from requests.exceptions import HTTPError

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff retry."""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
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
# Test database connection
def test_database_connection(db_config):
    """Verify database connectivity."""
    conn = create_database_connection(**db_config)
    if conn:
        cursor = conn.cursor()
        cursor.execute("SELECT VERSION()")
        version = cursor.fetchone()
        print(f"Connected to MySQL version: {version[0]}")
        cursor.close()
        conn.close()
        return True
    return False
```

### Missing Data Handling
```python
# Handle missing or null values during transformation
def safe_get(dictionary, key, default=None):
    """Safely extract value with default."""
    value = dictionary.get(key, default)
    return value if value else default

# Use in transformation
metadata = {
    'id': artifact.get('id'),
    'title': safe_get(artifact, 'title', 'Unknown'),
    'culture': safe_get(artifact, 'culture', 'Unknown'),
    # ... rest of fields
}
```

### Large Dataset Memory Management
```python
# Process data in chunks for large datasets
def chunked_etl_pipeline(api_key, db_config, total_records=10000, chunk_size=1000):
    """Process ETL in chunks to manage memory."""
    conn = create_database_connection(**db_config)
    
    for offset in range(0, total_records, chunk_size):
        print(f"Processing records {offset} to {offset + chunk_size}...")
        
        # Fetch chunk
        artifacts = collect_all_artifacts(api_key, max_records=chunk_size)
        
        # Transform chunk
        metadata_df, media_df, colors_df = transform_artifacts_to_dataframes(artifacts)
        
        # Load chunk
        batch_insert_metadata(conn, metadata_df)
        batch_insert_media(conn, media_df)
        batch_insert_colors(conn, colors_df)
        
        # Clear memory
        del artifacts, metadata_df, media_df, colors_df
    
    conn.close()
```

## Best Practices

1. **Always use environment variables** for API keys and database credentials
2. **Implement retry logic** for API calls to handle rate limiting
3. **Use batch inserts** for better database performance
4. **Add data validation** before loading to database
5. **Log ETL operations** for debugging and monitoring
6. **Create database indexes** on frequently queried columns
7. **Handle null values** explicitly during transformation
8. **Use connection pooling** for production deployments
