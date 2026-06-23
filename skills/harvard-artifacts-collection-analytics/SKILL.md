```markdown
---
name: harvard-artifacts-collection-analytics
description: ETL pipeline and analytics app for Harvard Art Museums API using Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering project with Harvard API
  - analyze Harvard artifacts collection with SQL
  - visualize museum artifacts data with Streamlit
  - set up Harvard Art Museums analytics dashboard
  - fetch and transform Harvard museum data
  - query Harvard artifacts database
  - build museum collection analytics app
---

# Harvard Artifacts Collection Data Engineering & Analytics App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates an end-to-end data engineering pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational structures, loads it into SQL databases, and provides interactive analytics dashboards using Streamlit.

## What It Does

- **Data Collection**: Fetches artifact metadata, media details, and color information from Harvard Art Museums API
- **ETL Pipeline**: Transforms nested JSON responses into normalized relational tables
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper foreign key relationships
- **Analytics**: Executes predefined SQL queries for insights on artifacts, cultures, centuries, and media
- **Visualization**: Creates interactive dashboards with Plotly charts based on query results

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

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

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```env
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Request an API key (free for research/educational use)
3. Add to `.env` file

## Database Schema

The application creates three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    people TEXT,
    url VARCHAR(500)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_pages=5):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = 'https://api.harvardartmuseums.org/object'
    
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': 100,  # Max results per page
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return all_artifacts
```

### 2. Data Transformation

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON into relational dataframes"""
    
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'division': artifact.get('division', ''),
            'dated': artifact.get('dated', ''),
            'period': artifact.get('period', ''),
            'technique': artifact.get('technique', ''),
            'medium': artifact.get('medium', ''),
            'dimensions': artifact.get('dimensions', ''),
            'people': ', '.join([p.get('name', '') for p in artifact.get('people', [])]),
            'url': artifact.get('url', '')
        }
        metadata_records.append(metadata)
        
        # Extract media
        for image in artifact.get('images', []):
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': image.get('baseimageurl', ''),
                'format': image.get('format', ''),
                'height': image.get('height', 0),
                'width': image.get('width', 0)
            }
            media_records.append(media)
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color', ''),
                'percent': color.get('percent', 0.0)
            }
            color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=os.getenv('DB_PORT'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Batch insert dataframes into SQL database"""
    
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Load metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, 
         division, dated, period, technique, medium, dimensions, people, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Load media
        media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, format, height, width)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
        
        # Load colors
        colors_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, percent)
        VALUES (%s, %s, %s)
        """
        cursor.executemany(colors_query, colors_df.values.tolist())
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def run_analytics_query(query):
    """Execute SQL query and return results"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Sample queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Top Colors Used": """
        SELECT color, SUM(percent) as total_percent, COUNT(*) as occurrence
        FROM artifactcolors
        GROUP BY color
        ORDER BY total_percent DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN EXISTS (
                SELECT 1 FROM artifactmedia m 
                WHERE m.artifact_id = a.id
            ) THEN 'Has Images' ELSE 'No Images' END as media_status,
            COUNT(*) as count
        FROM artifactmetadata a
        GROUP BY media_status
    """
}

# Streamlit app
st.title("Harvard Artifacts Analytics Dashboard")

# Sidebar for query selection
query_name = st.sidebar.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))

if st.button("Run Analysis"):
    with st.spinner("Executing query..."):
        df = run_analytics_query(ANALYTICS_QUERIES[query_name])
        
        # Display results
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=query_name)
            st.plotly_chart(fig)
```

## Running the Application

```bash
# Run the Streamlit app
streamlit run app.py

# The app will open in your browser at http://localhost:8501
```

## Common Patterns

### Full ETL Pipeline Execution

```python
def run_etl_pipeline(num_pages=5):
    """Complete ETL pipeline from API to database"""
    
    # Extract
    print("Extracting data from API...")
    raw_artifacts = fetch_artifacts(num_pages)
    
    # Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_artifacts)
    
    # Load
    print("Loading to database...")
    load_to_database(metadata_df, media_df, colors_df)
    
    print("ETL pipeline completed successfully!")

# Execute
run_etl_pipeline(num_pages=10)
```

### Custom Analytics Query

```python
def get_artifacts_by_department():
    """Get artifact distribution by department"""
    query = """
        SELECT 
            department,
            COUNT(*) as total_artifacts,
            COUNT(DISTINCT culture) as unique_cultures
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """
    return run_analytics_query(query)
```

## Troubleshooting

### API Rate Limiting

If you hit rate limits, add delays between requests:

```python
import time

def fetch_artifacts_with_delay(num_pages=5, delay=1):
    all_artifacts = []
    for page in range(1, num_pages + 1):
        # ... fetch logic ...
        time.sleep(delay)  # Wait between requests
    return all_artifacts
```

### Database Connection Errors

Verify environment variables are set:

```python
import os
from dotenv import load_dotenv

load_dotenv()

required_vars = ['DB_HOST', 'DB_USER', 'DB_PASSWORD', 'DB_NAME']
missing = [v for v in required_vars if not os.getenv(v)]

if missing:
    raise EnvironmentError(f"Missing environment variables: {missing}")
```

### Empty Results

Check if data exists in database:

```python
def verify_data():
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT COUNT(*) FROM artifactmetadata")
    count = cursor.fetchone()[0]
    print(f"Total artifacts in database: {count}")
    cursor.close()
    conn.close()
```

## Advanced Usage

### Incremental Data Loading

```python
def get_max_artifact_id():
    """Get the highest artifact ID already in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    max_id = cursor.fetchone()[0] or 0
    cursor.close()
    conn.close()
    return max_id

def fetch_new_artifacts():
    """Fetch only artifacts newer than what we have"""
    max_id = get_max_artifact_id()
    # Modify API call to filter by ID > max_id
    # Implementation depends on API capabilities
```

This skill enables AI agents to help developers build complete data engineering pipelines using the Harvard Art Museums API, from data extraction through visualization.
```
