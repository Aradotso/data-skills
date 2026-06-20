---
name: harvard-art-museum-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - create a data pipeline for Harvard Art Museums API
  - build an ETL workflow with museum artifact data
  - set up analytics dashboard for Harvard art collection
  - extract and transform Harvard museum data into SQL
  - visualize art museum collection data with Streamlit
  - design a data engineering pipeline for museum artifacts
  - query and analyze Harvard art museum metadata
  - build museum data analytics application
---

# Harvard Art Museum ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It covers ETL pipeline development, SQL database design, analytical queries, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App demonstrates:
- **API Integration**: Fetch artifact metadata from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipelines**: Extract, transform, and load nested JSON data into relational SQL tables
- **SQL Analytics**: Store structured data in MySQL/TiDB Cloud with proper schema design
- **Interactive Dashboards**: Build Streamlit applications with dynamic query execution and Plotly visualizations
- **Real-world Patterns**: Simulate production data workflows used in analytics and data engineering roles

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

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Obtaining Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free for educational/non-commercial use)
3. Add the key to your `.env` file

## Database Schema Design

### Core Tables

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(300),
    period VARCHAR(200),
    dated VARCHAR(200),
    url TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    caption TEXT,
    technique VARCHAR(200),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(7),
    color_name VARCHAR(100),
    color_percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetching Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        page: Page number (default: 1)
        size: Results per page (max 100)
    
    Returns:
        dict: JSON response with artifact data
    """
    api_key = os.getenv('HARVARD_API_KEY')
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

def collect_multiple_pages(num_pages=5):
    """Collect data from multiple pages with rate limiting"""
    import time
    
    all_artifacts = []
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page)
        all_artifacts.extend(data.get('records', []))
        time.sleep(1)  # Rate limiting
    
    return all_artifacts
```

### Transform: Processing Nested JSON

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform nested JSON into flat DataFrames for SQL insertion
    
    Args:
        raw_data: List of artifact dictionaries
    
    Returns:
        tuple: (metadata_df, media_df, colors_df)
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'period': artifact.get('period'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url')
        })
        
        # Extract media information
        if artifact.get('images'):
            for img in artifact['images']:
                media_records.append({
                    'artifact_id': artifact.get('id'),
                    'image_url': img.get('baseimageurl'),
                    'caption': img.get('caption'),
                    'technique': img.get('technique')
                })
        
        # Extract color data
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_records.append({
                    'artifact_id': artifact.get('id'),
                    'color_hex': color.get('hex'),
                    'color_name': color.get('color'),
                    'color_percentage': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load: Batch Insertion into SQL

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection using environment variables"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df):
    """Load artifact metadata into SQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, 
         division, technique, period, dated, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    # Batch insert
    data_tuples = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data_tuples)
    
    conn.commit()
    cursor.close()
    conn.close()
    print(f"Inserted {len(df)} metadata records")

def load_media(df):
    """Load artifact media into SQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, image_url, caption, technique)
        VALUES (%s, %s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data_tuples)
    
    conn.commit()
    cursor.close()
    conn.close()
    print(f"Inserted {len(df)} media records")

def load_colors(df):
    """Load artifact colors into SQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color_hex, color_name, color_percentage)
        VALUES (%s, %s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data_tuples)
    
    conn.commit()
    cursor.close()
    conn.close()
    print(f"Inserted {len(df)} color records")
```

## Streamlit Dashboard Implementation

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museum Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museum Collection Analytics")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        data_collection_page()
    elif page == "SQL Analytics":
        sql_analytics_page()
    elif page == "Visualizations":
        visualization_page()

if __name__ == "__main__":
    main()
```

### Data Collection Page

```python
def data_collection_page():
    st.header("📥 Data Collection from API")
    
    col1, col2 = st.columns(2)
    with col1:
        num_pages = st.number_input("Number of pages to fetch", 
                                     min_value=1, max_value=20, value=5)
    with col2:
        page_size = st.selectbox("Records per page", [10, 25, 50, 100])
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            # Extract
            raw_data = collect_multiple_pages(num_pages)
            st.success(f"✓ Extracted {len(raw_data)} artifacts")
            
            # Transform
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            st.success(f"✓ Transformed into {len(metadata_df)} metadata, "
                      f"{len(media_df)} media, {len(colors_df)} color records")
            
            # Load
            load_metadata(metadata_df)
            load_media(media_df)
            load_colors(colors_df)
            st.success("✓ Loaded all data into SQL database")
            
        st.balloons()
```

### SQL Analytics Page

```python
def sql_analytics_page():
    st.header("📊 SQL Analytics Dashboard")
    
    # Predefined analytical queries
    queries = {
        "Top 10 Cultures by Artifact Count": """
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
            ORDER BY century
        """,
        "Most Common Classifications": """
            SELECT classification, COUNT(*) as count
            FROM artifactmetadata
            WHERE classification IS NOT NULL
            GROUP BY classification
            ORDER BY count DESC
            LIMIT 15
        """,
        "Color Distribution Across Artifacts": """
            SELECT color_name, COUNT(*) as frequency,
                   AVG(color_percentage) as avg_percentage
            FROM artifactcolors
            GROUP BY color_name
            ORDER BY frequency DESC
            LIMIT 20
        """,
        "Departments with Most Artifacts": """
            SELECT department, COUNT(*) as total_artifacts
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY total_artifacts DESC
        """,
        "Artifacts with Multiple Images": """
            SELECT a.title, a.culture, COUNT(m.media_id) as image_count
            FROM artifactmetadata a
            JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY a.id, a.title, a.culture
            HAVING image_count > 3
            ORDER BY image_count DESC
            LIMIT 20
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        conn = get_db_connection()
        df = pd.read_sql(queries[selected_query], conn)
        conn.close()
        
        st.subheader("Query Results")
        st.dataframe(df, use_container_width=True)
        
        # Auto-generate visualization
        if len(df) > 0:
            st.subheader("Visualization")
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)
```

## Common Analytical Queries

### Time-based Analysis

```python
# Artifacts by period with classification breakdown
query = """
    SELECT period, classification, COUNT(*) as count
    FROM artifactmetadata
    WHERE period IS NOT NULL AND classification IS NOT NULL
    GROUP BY period, classification
    ORDER BY period, count DESC
"""
```

### Color Analysis

```python
# Dominant colors in ancient vs modern art
query = """
    SELECT 
        CASE 
            WHEN century < '1000' THEN 'Ancient'
            WHEN century >= '1000' AND century < '1900' THEN 'Historical'
            ELSE 'Modern'
        END as era,
        c.color_name,
        AVG(c.color_percentage) as avg_percentage
    FROM artifactcolors c
    JOIN artifactmetadata m ON c.artifact_id = m.id
    WHERE m.century IS NOT NULL
    GROUP BY era, c.color_name
    ORDER BY era, avg_percentage DESC
"""
```

### Media Availability

```python
# Artifacts with and without images by department
query = """
    SELECT 
        m.department,
        COUNT(DISTINCT m.id) as total_artifacts,
        COUNT(DISTINCT med.artifact_id) as artifacts_with_images,
        ROUND(COUNT(DISTINCT med.artifact_id) * 100.0 / COUNT(DISTINCT m.id), 2) as image_coverage_pct
    FROM artifactmetadata m
    LEFT JOIN artifactmedia med ON m.id = med.artifact_id
    GROUP BY m.department
    ORDER BY image_coverage_pct DESC
"""
```

## Running the Application

```bash
# Run the Streamlit app
streamlit run app.py

# Run with custom port
streamlit run app.py --server.port 8080

# Run with auto-reload disabled
streamlit run app.py --server.runOnSave false
```

## Troubleshooting

### API Rate Limiting

```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page)
        except HTTPError as e:
            if e.response.status_code == 429:  # Too many requests
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def safe_db_connection():
    """Create database connection with error handling"""
    try:
        conn = get_db_connection()
        conn.ping(reconnect=True, attempts=3, delay=1)
        return conn
    except Error as e:
        st.error(f"Database connection failed: {e}")
        return None
```

### Memory Management for Large Datasets

```python
def batch_load_large_dataset(df, batch_size=1000):
    """Load large datasets in batches to avoid memory issues"""
    total_rows = len(df)
    
    for i in range(0, total_rows, batch_size):
        batch = df.iloc[i:i+batch_size]
        load_metadata(batch)
        print(f"Loaded batch {i//batch_size + 1}: {len(batch)} records")
```

### Handling Missing Data

```python
def clean_artifact_data(df):
    """Clean and validate artifact data before loading"""
    # Replace NaN with None for SQL compatibility
    df = df.where(pd.notnull(df), None)
    
    # Truncate long strings
    df['title'] = df['title'].str[:500]
    df['culture'] = df['culture'].str[:200]
    
    # Remove duplicates
    df = df.drop_duplicates(subset=['id'], keep='first')
    
    return df
```

This skill provides comprehensive guidance for building production-ready data engineering pipelines with the Harvard Art Museums API, suitable for analytics, visualization, and data science applications.
