---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an art museum data pipeline
  - create ETL pipeline for Harvard Art Museums API
  - analyze Harvard artifacts with SQL
  - visualize museum collection data with Streamlit
  - set up data engineering pipeline for art collections
  - extract and transform museum API data
  - build analytics dashboard for art museum data
  - create interactive museum artifact visualizations
---

# Harvard Art Museums Data Pipeline & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering pipelines using the Harvard Art Museums API. Extract artifact metadata, transform it into relational tables, load into SQL databases, and create interactive analytics dashboards with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App demonstrates real-world ETL patterns for cultural data:

- **Extract**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transform**: Convert nested JSON into normalized relational tables (metadata, media, colors)
- **Load**: Batch insert into MySQL/TiDB with proper schema design
- **Analyze**: Execute SQL queries for insights on collections, cultures, centuries, and media
- **Visualize**: Interactive Streamlit dashboards with Plotly charts

## Installation

### Prerequisites

```bash
# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv

# Or use requirements.txt if provided
pip install -r requirements.txt
```

### Environment Setup

Create a `.env` file with your credentials:

```bash
# Harvard API
HARVARD_API_KEY=your_api_key_here

# Database connection
DB_HOST=gateway01.your-region.prod.aws.tidbcloud.com
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=your_database_name
```

### Get Harvard API Key

1. Visit https://docs.api.harvardartmuseums.org/
2. Request an API key (free)
3. Add to `.env` file

## Project Architecture

```
API → Extract → Transform → Load → SQL → Query → Visualize
```

**Tables Created:**
- `artifactmetadata` - Core artifact information (id, title, culture, century, division)
- `artifactmedia` - Media URLs and image data
- `artifactcolors` - Color analysis from artifact images

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    all_artifacts = []
    page = 1
    
    while len(all_artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': min(page_size, num_records - len(all_artifacts)),
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            all_artifacts.extend(records)
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_artifacts[:num_records]
```

### 2. Data Transformation

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform nested JSON into normalized dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        
        # Extract metadata
        metadata_list.append({
            'id': artifact_id,
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'division': artifact.get('division'),
            'classification': artifact.get('classification'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'url': artifact.get('url')
        })
        
        # Extract media
        for media in artifact.get('images', []):
            media_list.append({
                'artifact_id': artifact_id,
                'baseimageurl': media.get('baseimageurl'),
                'iiifbaseuri': media.get('iiifbaseuri'),
                'width': media.get('width'),
                'height': media.get('height')
            })
        
        # Extract colors
        for color in artifact.get('colors', []):
            colors_list.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return {
        'metadata': pd.DataFrame(metadata_list),
        'media': pd.DataFrame(media_list),
        'colors': pd.DataFrame(colors_list)
    }
```

### 3. Database Schema & Loading

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
        print(f"Database connection error: {e}")
        return None

def create_tables(connection):
    """Create schema for artifact data"""
    cursor = connection.cursor()
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            century VARCHAR(100),
            division VARCHAR(255),
            classification VARCHAR(255),
            dated VARCHAR(255),
            medium TEXT,
            url TEXT
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            baseimageurl TEXT,
            iiifbaseuri TEXT,
            width INT,
            height INT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()
    cursor.close()

def load_data(connection, dataframes):
    """Batch insert data into SQL tables"""
    cursor = connection.cursor()
    
    # Insert metadata
    for _, row in dataframes['metadata'].iterrows():
        cursor.execute("""
            INSERT IGNORE INTO artifactmetadata 
            (id, title, culture, century, division, classification, dated, medium, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert media
    for _, row in dataframes['media'].iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, iiifbaseuri, width, height)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in dataframes['colors'].iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    connection.commit()
    cursor.close()
```

### 4. SQL Analytics Queries

```python
ANALYTICAL_QUERIES = {
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
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency 
        FROM artifactcolors 
        GROUP BY color 
        ORDER BY frequency DESC 
        LIMIT 10
    """,
    
    "Artifacts with Media": """
        SELECT 
            COUNT(DISTINCT m.id) as total_artifacts,
            COUNT(DISTINCT med.artifact_id) as with_media,
            ROUND(COUNT(DISTINCT med.artifact_id) * 100.0 / COUNT(DISTINCT m.id), 2) as percentage
        FROM artifactmetadata m
        LEFT JOIN artifactmedia med ON m.id = med.artifact_id
    """,
    
    "Division Distribution": """
        SELECT division, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE division IS NOT NULL 
        GROUP BY division 
        ORDER BY count DESC
    """
}

def execute_query(connection, query):
    """Execute SQL query and return results as DataFrame"""
    return pd.read_sql(query, connection)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # ETL Section
    st.header("📥 Data Collection & ETL")
    num_records = st.number_input("Number of records to fetch", 
                                   min_value=10, max_value=1000, 
                                   value=100)
    
    if st.button("🚀 Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(num_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            dataframes = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            conn = create_connection()
            create_tables(conn)
            load_data(conn, dataframes)
            conn.close()
            st.success("Data loaded to SQL database")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", 
                              list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Execute Query"):
        conn = create_connection()
        df = execute_query(conn, ANALYTICAL_QUERIES[query_name])
        conn.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Complete ETL Pipeline

```python
from dotenv import load_dotenv
import os

load_dotenv()

def run_full_pipeline(num_records=100):
    """Execute complete ETL pipeline"""
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_artifacts(num_records)
    
    # Transform
    print("Transforming data...")
    dataframes = transform_artifacts(artifacts)
    
    # Load
    print("Loading to database...")
    conn = create_connection()
    create_tables(conn)
    load_data(conn, dataframes)
    conn.close()
    
    print(f"Pipeline complete: {num_records} artifacts processed")
    return dataframes

# Run pipeline
data = run_full_pipeline(200)
```

### Custom Analytics Query

```python
def analyze_culture_by_century(connection):
    """Custom analysis: culture distribution across centuries"""
    query = """
        SELECT 
            century, 
            culture, 
            COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND culture IS NOT NULL
        GROUP BY century, culture
        ORDER BY century, count DESC
    """
    return pd.read_sql(query, connection)

conn = create_connection()
results = analyze_culture_by_century(conn)
conn.close()
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        response = requests.get(url, params=params)
        
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 429:  # Rate limited
            wait_time = 2 ** attempt
            print(f"Rate limited. Waiting {wait_time}s...")
            time.sleep(wait_time)
        else:
            raise Exception(f"API error: {response.status_code}")
    
    return None
```

### Database Connection Issues

```python
def create_connection_with_retry(max_attempts=3):
    """Create database connection with retries"""
    for attempt in range(max_attempts):
        try:
            connection = mysql.connector.connect(
                host=os.getenv('DB_HOST'),
                port=int(os.getenv('DB_PORT', 3306)),
                user=os.getenv('DB_USER'),
                password=os.getenv('DB_PASSWORD'),
                database=os.getenv('DB_NAME'),
                connect_timeout=10
            )
            return connection
        except Error as e:
            print(f"Connection attempt {attempt + 1} failed: {e}")
            if attempt < max_attempts - 1:
                time.sleep(2)
    
    return None
```

### Handling Missing Data

```python
def safe_transform(artifacts):
    """Transform with null handling"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Untitled'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'division': artifact.get('division', 'Unknown'),
            'classification': artifact.get('classification'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'url': artifact.get('url')
        })
    
    return pd.DataFrame(metadata_list)
```

## Best Practices

1. **Always use environment variables** for credentials
2. **Implement pagination** for large datasets
3. **Use batch inserts** for performance (commit every 100-1000 records)
4. **Add indexes** on foreign keys and frequently queried columns
5. **Handle API rate limits** with exponential backoff
6. **Validate data** before loading to SQL
7. **Use connection pooling** for production deployments

This skill provides everything needed to build production-ready data pipelines for cultural heritage data using modern ETL and analytics patterns.
