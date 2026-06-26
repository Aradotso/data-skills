---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - set up ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit
  - query Harvard museum collection data
  - build data engineering pipeline with API
  - analyze Harvard artifacts with SQL
  - visualize museum data with Plotly
  - configure TiDB for artifact storage
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an end-to-end data engineering and analytics application that extracts artifact data from the Harvard Art Museums API, transforms it into relational tables, loads it into SQL databases (MySQL/TiDB), and visualizes insights through Streamlit dashboards.

## What It Does

- **API Integration**: Fetches artifact metadata, media, and color data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON responses into normalized relational database tables
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper schema design and foreign keys
- **Analytics**: Runs 20+ predefined SQL queries for artifact insights
- **Visualization**: Interactive Plotly charts rendered in Streamlit dashboards

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file or use Streamlit secrets:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database credentials
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Schema

The application creates three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    department VARCHAR(255),
    classification VARCHAR(255),
    dated VARCHAR(255),
    description TEXT,
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    media_url TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core API Integration

### Fetching Artifacts from Harvard API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key
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

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=50)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Records fetched: {len(data['records'])}")
```

### Handling Pagination

```python
def fetch_all_artifacts(api_key, max_records=500):
    """Fetch multiple pages of artifacts"""
    all_records = []
    page = 1
    size = 100
    
    while len(all_records) < max_records:
        data = fetch_artifacts(api_key, page=page, size=size)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_records.extend(records)
        page += 1
        
        # Respect rate limits
        import time
        time.sleep(0.5)
    
    return all_records[:max_records]
```

## ETL Pipeline Implementation

### Extract and Transform

```python
import pandas as pd

def transform_artifacts(records):
    """
    Transform raw API records into structured dataframes
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in records:
        # Extract metadata
        metadata = {
            'id': record.get('id'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'department': record.get('department'),
            'classification': record.get('classification'),
            'dated': record.get('dated'),
            'description': record.get('description'),
            'url': record.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media (images)
        for image in record.get('images', []):
            media_list.append({
                'artifact_id': record.get('id'),
                'media_type': 'image',
                'media_url': image.get('baseimageurl')
            })
        
        # Extract colors
        for color in record.get('colors', []):
            colors_list.append({
                'artifact_id': record.get('id'),
                'color': color.get('color'),
                'percentage': color.get('percent')
            })
    
    metadata_df = pd.DataFrame(metadata_list)
    media_df = pd.DataFrame(media_list)
    colors_df = pd.DataFrame(colors_list)
    
    return metadata_df, media_df, colors_df
```

### Load to Database

```python
import mysql.connector
from mysql.connector import Error

def create_connection():
    """Create database connection"""
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

def load_metadata(df, connection):
    """Batch insert artifact metadata"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, department, classification, dated, description, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    # Convert dataframe to list of tuples
    data = [tuple(row) for row in df.to_numpy()]
    
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} metadata records")
    cursor.close()

def load_media(df, connection):
    """Batch insert media records"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, media_type, media_url)
    VALUES (%s, %s, %s)
    """
    
    data = [tuple(row) for row in df.to_numpy()]
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} media records")
    cursor.close()
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Artifacts Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    api_key = st.sidebar.text_input(
        "Harvard API Key", 
        type="password",
        value=os.getenv('HARVARD_API_KEY', '')
    )
    
    # Data collection section
    if st.sidebar.button("Fetch New Data"):
        with st.spinner("Fetching artifacts..."):
            records = fetch_all_artifacts(api_key, max_records=100)
            metadata_df, media_df, colors_df = transform_artifacts(records)
            
            conn = create_connection()
            if conn:
                load_metadata(metadata_df, conn)
                load_media(media_df, conn)
                load_colors(colors_df, conn)
                conn.close()
                st.success(f"Loaded {len(records)} artifacts!")
    
    # Analytics section
    st.header("📊 Analytics")
    
    query_option = st.selectbox(
        "Select Analysis",
        [
            "Artifacts by Culture",
            "Artifacts by Century",
            "Top Departments",
            "Color Distribution"
        ]
    )
    
    run_analysis(query_option)

def run_analysis(query_type):
    """Execute SQL query and visualize results"""
    conn = create_connection()
    if not conn:
        st.error("Database connection failed")
        return
    
    queries = {
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
            ORDER BY count DESC
        """,
        "Top Departments": """
            SELECT department, COUNT(*) as count 
            FROM artifactmetadata 
            GROUP BY department 
            ORDER BY count DESC 
            LIMIT 10
        """,
        "Color Distribution": """
            SELECT color, AVG(percentage) as avg_percentage 
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY avg_percentage DESC 
            LIMIT 10
        """
    }
    
    df = pd.read_sql(queries[query_type], conn)
    conn.close()
    
    # Display table
    st.dataframe(df)
    
    # Visualize
    if len(df) > 0:
        fig = px.bar(
            df, 
            x=df.columns[0], 
            y=df.columns[1],
            title=query_type
        )
        st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, num_records=100):
    """ETL pipeline with comprehensive error handling"""
    try:
        # Extract
        records = fetch_all_artifacts(api_key, max_records=num_records)
        if not records:
            raise ValueError("No records fetched from API")
        
        # Transform
        metadata_df, media_df, colors_df = transform_artifacts(records)
        
        # Validate
        if metadata_df.empty:
            raise ValueError("No metadata extracted")
        
        # Load
        conn = create_connection()
        if not conn:
            raise ConnectionError("Failed to connect to database")
        
        try:
            load_metadata(metadata_df, conn)
            load_media(media_df, conn)
            load_colors(colors_df, conn)
        finally:
            conn.close()
        
        return True, f"Successfully processed {len(records)} records"
        
    except requests.RequestException as e:
        return False, f"API Error: {str(e)}"
    except Error as e:
        return False, f"Database Error: {str(e)}"
    except Exception as e:
        return False, f"Unexpected Error: {str(e)}"
```

### Incremental Data Loading

```python
def get_max_artifact_id(connection):
    """Get the highest artifact ID already in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT IFNULL(MAX(id), 0) FROM artifactmetadata")
    max_id = cursor.fetchone()[0]
    cursor.close()
    return max_id

def incremental_load(api_key):
    """Only load artifacts newer than what's in database"""
    conn = create_connection()
    max_id = get_max_artifact_id(conn)
    conn.close()
    
    # Fetch records with filtering
    records = fetch_all_artifacts(api_key, max_records=100)
    new_records = [r for r in records if r.get('id', 0) > max_id]
    
    if new_records:
        metadata_df, media_df, colors_df = transform_artifacts(new_records)
        # Load as before...
    
    return len(new_records)
```

## Troubleshooting

### API Rate Limiting

If you encounter 429 errors:

```python
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_retry_session():
    """Create session with automatic retries"""
    session = requests.Session()
    retry = Retry(
        total=5,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504]
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('https://', adapter)
    return session

# Use in fetch function
session = create_retry_session()
response = session.get(base_url, params=params)
```

### Database Connection Issues

```python
# Test connection
def test_db_connection():
    try:
        conn = create_connection()
        if conn and conn.is_connected():
            print("✓ Database connection successful")
            conn.close()
            return True
    except Error as e:
        print(f"✗ Connection failed: {e}")
        return False
```

### Memory Management for Large Datasets

```python
def chunked_load(records, chunk_size=100):
    """Load data in chunks to avoid memory issues"""
    for i in range(0, len(records), chunk_size):
        chunk = records[i:i+chunk_size]
        metadata_df, media_df, colors_df = transform_artifacts(chunk)
        
        conn = create_connection()
        load_metadata(metadata_df, conn)
        load_media(media_df, conn)
        load_colors(colors_df, conn)
        conn.close()
        
        print(f"Processed chunk {i//chunk_size + 1}")
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY=your_key
export DB_HOST=localhost
export DB_USER=root
export DB_PASSWORD=password
export DB_NAME=harvard_artifacts

# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```
