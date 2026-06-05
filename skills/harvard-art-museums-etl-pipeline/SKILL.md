---
name: harvard-art-museums-etl-pipeline
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering app with Harvard API
  - set up artifact collection analytics dashboard
  - extract and analyze Harvard museum artifacts
  - build a Streamlit app for museum data visualization
  - implement SQL analytics for art collection data
  - create a museum artifact data pipeline
  - analyze Harvard Art Museums collection data
---

# Harvard Art Museums ETL Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build and work with an end-to-end data engineering pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational structures, loads it into SQL databases, and visualizes analytics through Streamlit dashboards.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App demonstrates:
- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper schema design
- **Analytics**: Executes 20+ predefined SQL queries for insights
- **Visualization**: Interactive Plotly charts in a Streamlit dashboard

**Architecture Flow**: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required dependencies**:
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

Create a `.env` file:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit [Harvard Art Museums API](https://harvardartmuseums.org/collections/api)
2. Request an API key (free)
3. Add to `.env` file

### Database Setup

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Establish database connection"""
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
        print(f"Error: {e}")
        return None
```

## Database Schema

### Create Tables

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dated VARCHAR(255),
    description TEXT,
    provenance TEXT,
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    url VARCHAR(500),
    INDEX idx_culture (culture),
    INDEX idx_century (century),
    INDEX idx_department (department)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    format VARCHAR(50),
    caption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE,
    INDEX idx_artifact (artifact_id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE,
    INDEX idx_artifact (artifact_id),
    INDEX idx_spectrum (spectrum)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        page: Page number (pagination)
        size: Number of records per page (max 100)
    
    Returns:
        dict: API response with artifact data
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return None

def fetch_all_artifacts(max_records=1000):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        data = fetch_artifacts(page=page, size=size)
        if not data or 'records' not in data:
            break
            
        all_artifacts.extend(data['records'])
        
        if len(data['records']) < size:
            break  # No more pages
            
        page += 1
    
    return all_artifacts[:max_records]
```

### Transform: Process Nested JSON

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform artifact data into metadata DataFrame
    
    Args:
        artifacts: List of artifact dictionaries from API
    
    Returns:
        pd.DataFrame: Transformed metadata
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dated': artifact.get('dated'),
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """Transform media data into DataFrame"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media = {
                'artifact_id': artifact_id,
                'iiifbaseuri': image.get('iiifbaseuri'),
                'baseimageurl': image.get('baseimageurl'),
                'format': image.get('format'),
                'caption': image.get('caption')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """Transform color data into DataFrame"""
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percentage': color.get('percent')
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### Load: Insert into Database

```python
def load_metadata_to_db(df, connection):
    """
    Load metadata DataFrame to database with batch insert
    
    Args:
        df: pandas DataFrame with metadata
        connection: MySQL connection object
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, classification, department, 
     technique, medium, dated, description, provenance, dimensions, 
     creditline, accessionyear, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    # Convert DataFrame to list of tuples
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error loading metadata: {e}")
        connection.rollback()
    finally:
        cursor.close()

def load_media_to_db(df, connection):
    """Load media DataFrame to database"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, iiifbaseuri, baseimageurl, format, caption)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error loading media: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

## SQL Analytics Queries

### Common Analytical Queries

```python
# Query 1: Top 10 cultures by artifact count
query_cultures = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Query 2: Artifacts by century
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC;
"""

# Query 3: Department distribution
query_departments = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY total_artifacts DESC;
"""

# Query 4: Color spectrum analysis
query_colors = """
SELECT spectrum, COUNT(*) as color_count, AVG(percentage) as avg_percentage
FROM artifactcolors
GROUP BY spectrum
ORDER BY color_count DESC;
"""

# Query 5: Media availability
query_media = """
SELECT 
    COUNT(DISTINCT am.id) as total_artifacts,
    COUNT(DISTINCT media.artifact_id) as artifacts_with_media,
    ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as media_percentage
FROM artifactmetadata am
LEFT JOIN artifactmedia media ON am.id = media.artifact_id;
"""

# Query 6: Top techniques
query_techniques = """
SELECT technique, COUNT(*) as usage_count
FROM artifactmetadata
WHERE technique IS NOT NULL AND technique != ''
GROUP BY technique
ORDER BY usage_count DESC
LIMIT 15;
"""
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px
from dotenv import load_dotenv

load_dotenv()

st.set_page_config(
    page_title="Harvard Artifacts Analytics",
    page_icon="🏛️",
    layout="wide"
)

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        data_collection_page()
    elif page == "SQL Analytics":
        sql_analytics_page()
    else:
        visualizations_page()

def data_collection_page():
    """ETL data collection interface"""
    st.header("📥 Data Collection from API")
    
    num_records = st.number_input(
        "Number of records to fetch",
        min_value=10,
        max_value=1000,
        value=100,
        step=10
    )
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_all_artifacts(max_records=num_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata = transform_artifact_metadata(artifacts)
            df_media = transform_artifact_media(artifacts)
            df_colors = transform_artifact_colors(artifacts)
            st.success("Data transformation complete")
        
        with st.spinner("Loading to database..."):
            connection = create_database_connection()
            if connection:
                load_metadata_to_db(df_metadata, connection)
                load_media_to_db(df_media, connection)
                connection.close()
                st.success("✅ ETL Pipeline Complete!")

def sql_analytics_page():
    """Execute and display SQL queries"""
    st.header("📊 SQL Analytics")
    
    queries = {
        "Top Cultures": query_cultures,
        "Artifacts by Century": query_century,
        "Department Distribution": query_departments,
        "Color Spectrum": query_colors,
        "Media Availability": query_media
    }
    
    selected_query = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        connection = create_database_connection()
        if connection:
            df_result = pd.read_sql(queries[selected_query], connection)
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) >= 2:
                fig = px.bar(
                    df_result,
                    x=df_result.columns[0],
                    y=df_result.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)
            
            connection.close()

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Pattern 1: Incremental Data Load

```python
def get_latest_artifact_id(connection):
    """Get the latest artifact ID in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_load():
    """Load only new artifacts"""
    connection = create_database_connection()
    latest_id = get_latest_artifact_id(connection)
    
    # Fetch artifacts with ID > latest_id
    # Transform and load only new records
    connection.close()
```

### Pattern 2: Data Validation

```python
def validate_artifact_data(df):
    """Validate DataFrame before loading"""
    required_columns = ['id', 'title']
    
    for col in required_columns:
        if col not in df.columns:
            raise ValueError(f"Missing required column: {col}")
    
    # Remove duplicates
    df = df.drop_duplicates(subset=['id'])
    
    # Handle nulls
    df = df.fillna({
        'culture': 'Unknown',
        'century': 'Unknown',
        'department': 'Unknown'
    })
    
    return df
```

### Pattern 3: Query Results Caching

```python
@st.cache_data(ttl=3600)  # Cache for 1 hour
def execute_cached_query(query_string):
    """Execute query with caching"""
    connection = create_database_connection()
    df = pd.read_sql(query_string, connection)
    connection.close()
    return df
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            data = fetch_artifacts(page=page)
            return data
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    return None
```

### Database Connection Issues

```python
def robust_db_connection(max_retries=3):
    """Create connection with retry logic"""
    for attempt in range(max_retries):
        try:
            connection = create_database_connection()
            if connection.is_connected():
                return connection
        except Error as e:
            print(f"Connection attempt {attempt + 1} failed: {e}")
            time.sleep(2)
    
    raise Exception("Could not establish database connection")
```

### Memory Management for Large Datasets

```python
def chunked_data_load(artifacts, chunk_size=100):
    """Load data in chunks to manage memory"""
    connection = create_database_connection()
    
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        df = transform_artifact_metadata(chunk)
        load_metadata_to_db(df, connection)
        print(f"Loaded chunk {i//chunk_size + 1}")
    
    connection.close()
```

### Handling Missing API Key

```python
def validate_api_key():
    """Check if API key is configured"""
    api_key = os.getenv('HARVARD_API_KEY')
    if not api_key or api_key == 'your_api_key_here':
        st.error("⚠️ Harvard API Key not configured!")
        st.info("Please set HARVARD_API_KEY in your .env file")
        st.stop()
    return api_key
```

## Best Practices

1. **Always use environment variables** for sensitive data
2. **Implement error handling** for API and database operations
3. **Use batch inserts** for better database performance
4. **Cache query results** in Streamlit to reduce database load
5. **Validate data** before inserting into database
6. **Index foreign keys** for faster joins
7. **Monitor API rate limits** and implement backoff strategies
