---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering and analytics application for Harvard Art Museums API with ETL pipelines, SQL storage, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - set up Harvard artifacts data engineering project
  - create analytics dashboard with Harvard Art Museums API
  - implement SQL data warehouse for museum artifacts
  - visualize Harvard Art Museums collection data
  - extract and transform art museum API data
  - build Streamlit app for museum data analytics
  - design relational schema for artifacts collection
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill provides expertise in using the Harvard-Artifacts-Collection-Data-Engineering-Analytics-App, an end-to-end data engineering solution that demonstrates real-world ETL pipelines, SQL analytics, and interactive visualization using the Harvard Art Museums API.

## What This Project Does

The Harvard Art Museums ETL Analytics application implements a complete data engineering pipeline:

- **Data Collection**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Processing**: Transforms nested JSON into normalized relational tables
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper schema design
- **Analytics**: Provides 20+ predefined analytical SQL queries
- **Visualization**: Interactive Streamlit dashboard with Plotly charts

**Architecture Flow**: `API → ETL → SQL → Analytics → Visualization`

## Installation

### Prerequisites

- Python 3.8+
- MySQL or TiDB Cloud database
- Harvard Art Museums API key (obtain from https://www.harvardartmuseums.org/collections/api)

### Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

```text
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

### Environment Configuration

Create a `.env` file with your credentials:

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

## Database Schema Design

The project uses three main tables with relational integrity:

```sql
-- Artifact Metadata (main table)
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    period VARCHAR(200),
    department VARCHAR(200),
    classification VARCHAR(200),
    dated VARCHAR(200),
    description TEXT,
    provenance TEXT,
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    imageurl VARCHAR(1000),
    iiifbaseuri VARCHAR(1000),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py
```

The app will be available at `http://localhost:8501`

## Key Components and Usage

### 1. API Data Collection

The ETL pipeline extracts data from the Harvard Art Museums API with pagination:

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key from environment
        page: Page number for pagination
        size: Number of records per page (max 100)
    """
    url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv("HARVARD_API_KEY")
data = fetch_artifacts(api_key, page=1, size=100)

print(f"Total records: {data['info']['totalrecords']}")
print(f"Total pages: {data['info']['pages']}")
```

### 2. ETL Transform Logic

Transform nested JSON into normalized relational format:

```python
import pandas as pd

def transform_artifact_data(raw_data):
    """
    Transform raw API response into structured dataframes
    
    Returns:
        tuple: (metadata_df, media_df, colors_df)
    """
    records = raw_data.get('records', [])
    
    # Extract metadata
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in records:
        # Metadata
        metadata = {
            'id': record.get('id'),
            'title': record.get('title', '')[:500],
            'culture': record.get('culture', '')[:200],
            'century': record.get('century', '')[:100],
            'period': record.get('period', '')[:200],
            'department': record.get('department', '')[:200],
            'classification': record.get('classification', '')[:200],
            'dated': record.get('dated', '')[:200],
            'description': record.get('description', ''),
            'provenance': record.get('provenance', ''),
            'totalpageviews': record.get('totalpageviews', 0),
            'totaluniquepageviews': record.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
        
        # Media/Images
        images = record.get('images', [])
        for img in images:
            media = {
                'artifact_id': record.get('id'),
                'imageurl': img.get('baseimageurl', ''),
                'iiifbaseuri': img.get('iiifbaseuri', '')
            }
            media_list.append(media)
        
        # Colors
        colors = record.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': record.get('id'),
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### 3. SQL Database Loading

Batch insert data into MySQL/TiDB with error handling:

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection from environment variables"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            database=os.getenv('DB_NAME'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata(df, connection):
    """Batch insert artifact metadata"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, period, department, classification, 
     dated, description, provenance, totalpageviews, totaluniquepageviews)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data_tuples = [tuple(x) for x in df.to_numpy()]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error inserting metadata: {e}")
        connection.rollback()
    finally:
        cursor.close()

def load_media(df, connection):
    """Batch insert media data"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, imageurl, iiifbaseuri)
    VALUES (%s, %s, %s)
    """
    
    data_tuples = [tuple(x) for x in df.to_numpy()]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

### 4. Analytics Queries

Pre-built SQL queries for common analytics use cases:

```python
# Sample analytical queries from the project

ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
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
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Top Viewed Artifacts": """
        SELECT title, culture, totalpageviews
        FROM artifactmetadata
        WHERE totalpageviews > 0
        ORDER BY totalpageviews DESC
        LIMIT 20
    """,
    
    "Color Usage Analysis": """
        SELECT color, COUNT(*) as usage_count, 
               AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Artifacts with Media": """
        SELECT am.department, 
               COUNT(DISTINCT am.id) as total_artifacts,
               COUNT(DISTINCT media.artifact_id) as with_media,
               ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / 
                     COUNT(DISTINCT am.id), 2) as media_percentage
        FROM artifactmetadata am
        LEFT JOIN artifactmedia media ON am.id = media.artifact_id
        GROUP BY am.department
        ORDER BY media_percentage DESC
    """
}

def run_analytics_query(query_name, connection):
    """Execute an analytics query and return results as DataFrame"""
    cursor = connection.cursor(dictionary=True)
    
    query = ANALYTICS_QUERIES.get(query_name)
    if not query:
        print(f"Query '{query_name}' not found")
        return None
    
    try:
        cursor.execute(query)
        results = cursor.fetchall()
        df = pd.DataFrame(results)
        return df
    except Error as e:
        print(f"Query execution error: {e}")
        return None
    finally:
        cursor.close()
```

### 5. Streamlit Dashboard Integration

Build interactive visualization dashboard:

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """Main Streamlit dashboard application"""
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Options")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Database connection
    connection = get_db_connection()
    
    if connection:
        if st.sidebar.button("Run Analysis"):
            with st.spinner("Executing query..."):
                df = run_analytics_query(query_name, connection)
                
                if df is not None and not df.empty:
                    # Display results table
                    st.subheader(f"Results: {query_name}")
                    st.dataframe(df, use_container_width=True)
                    
                    # Auto-generate visualization
                    if len(df.columns) >= 2:
                        fig = px.bar(
                            df.head(20),
                            x=df.columns[0],
                            y=df.columns[1],
                            title=query_name,
                            labels={df.columns[0]: df.columns[0].title(),
                                   df.columns[1]: df.columns[1].title()}
                        )
                        st.plotly_chart(fig, use_container_width=True)
                else:
                    st.warning("No data returned from query")
        
        connection.close()
    else:
        st.error("Database connection failed. Check credentials.")

if __name__ == "__main__":
    create_dashboard()
```

## Common Patterns

### Complete ETL Pipeline Execution

```python
def run_complete_etl(num_pages=5):
    """
    Execute full ETL pipeline for specified number of pages
    """
    api_key = os.getenv("HARVARD_API_KEY")
    connection = get_db_connection()
    
    if not connection:
        print("Failed to connect to database")
        return
    
    try:
        for page in range(1, num_pages + 1):
            print(f"Processing page {page}/{num_pages}")
            
            # Extract
            raw_data = fetch_artifacts(api_key, page=page, size=100)
            
            # Transform
            metadata_df, media_df, colors_df = transform_artifact_data(raw_data)
            
            # Load
            load_metadata(metadata_df, connection)
            load_media(media_df, connection)
            load_colors(colors_df, connection)
            
            print(f"Page {page} completed successfully")
            
    except Exception as e:
        print(f"ETL pipeline error: {e}")
    finally:
        connection.close()

# Execute ETL for 10 pages (1000 artifacts)
run_complete_etl(num_pages=10)
```

### Incremental Data Updates

```python
def get_latest_artifact_id(connection):
    """Get the highest artifact ID currently in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) as max_id FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_update():
    """Fetch only new artifacts since last update"""
    connection = get_db_connection()
    latest_id = get_latest_artifact_id(connection)
    
    # Fetch artifacts with ID greater than latest
    # Implementation depends on API filtering capabilities
    pass
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limiting errors:

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff on rate limit"""
    for attempt in range(max_retries):
        try:
            data = fetch_artifacts(api_key, page)
            return data
        except Exception as e:
            if "429" in str(e):  # Too Many Requests
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity and credentials"""
    try:
        connection = get_db_connection()
        if connection and connection.is_connected():
            cursor = connection.cursor()
            cursor.execute("SELECT DATABASE()")
            db_name = cursor.fetchone()[0]
            print(f"✓ Connected to database: {db_name}")
            cursor.close()
            connection.close()
            return True
    except Error as e:
        print(f"✗ Connection failed: {e}")
        return False
```

### Memory Management for Large Datasets

```python
def batch_process_artifacts(total_pages, batch_size=5):
    """Process large datasets in batches to manage memory"""
    connection = get_db_connection()
    
    for batch_start in range(1, total_pages + 1, batch_size):
        batch_end = min(batch_start + batch_size, total_pages + 1)
        
        print(f"Processing batch: pages {batch_start}-{batch_end-1}")
        
        for page in range(batch_start, batch_end):
            # Process single page
            pass
        
        # Clear memory between batches
        import gc
        gc.collect()
    
    connection.close()
```

### Missing or Null Data Handling

```python
def clean_dataframe(df):
    """Clean and validate dataframe before loading"""
    # Replace empty strings with None for NULL in database
    df = df.replace({'': None, 'null': None})
    
    # Handle missing required fields
    if 'id' in df.columns:
        df = df[df['id'].notna()]
    
    # Truncate long text fields
    text_columns = ['title', 'description', 'provenance']
    for col in text_columns:
        if col in df.columns:
            df[col] = df[col].astype(str).str[:500]
    
    return df
```

## Best Practices

1. **Always use environment variables** for API keys and database credentials
2. **Implement pagination** when fetching large datasets from the API
3. **Use batch inserts** for database operations to improve performance
4. **Add error handling** around API calls and database operations
5. **Clean and validate data** before loading into the database
6. **Create indexes** on frequently queried columns (culture, century, department)
7. **Use ON DUPLICATE KEY UPDATE** to handle re-runs without duplicating data
8. **Monitor API rate limits** and implement retry logic with exponential backoff
