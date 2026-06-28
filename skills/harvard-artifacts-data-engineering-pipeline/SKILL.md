---
name: harvard-artifacts-data-engineering-pipeline
description: Build end-to-end ETL pipelines for Harvard Art Museums data with SQL analytics and Streamlit visualization
triggers:
  - build a data pipeline for Harvard Art Museums API
  - create ETL pipeline with Streamlit dashboard
  - analyze Harvard artifacts data with SQL
  - set up museum data engineering workflow
  - extract and visualize Harvard art collection data
  - build museum artifact analytics application
  - create interactive art data dashboard
  - implement Harvard API ETL process
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering solution for the Harvard Art Museums API, featuring ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations. It demonstrates production-ready patterns for API integration, data transformation, and analytics dashboards.

## What It Does

- **API Integration**: Collects artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into relational database tables
- **SQL Analytics**: Stores data in MySQL/TiDB with proper foreign key relationships
- **Interactive Dashboard**: Streamlit-based UI for querying and visualizing data
- **Data Visualization**: Plotly charts for analytical insights

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
# Harvard API Configuration
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Setup

The application uses three main tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    department VARCHAR(255),
    classification VARCHAR(255),
    dated VARCHAR(255),
    description TEXT,
    url VARCHAR(500),
    creditline TEXT,
    rank INT,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    publiccaption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, num_records=100, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        api_key: Harvard API key
        num_records: Total records to fetch
        page_size: Records per API call (max 100)
    
    Returns:
        List of artifact records
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    page = 1
    
    while len(all_artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': page_size,
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

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts = fetch_artifacts(api_key, num_records=500)
```

### 2. ETL Transform Operations

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform raw API data into structured dataframes
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'department': artifact.get('department'),
            'classification': artifact.get('classification'),
            'dated': artifact.get('dated'),
            'description': artifact.get('description'),
            'url': artifact.get('url'),
            'creditline': artifact.get('creditline'),
            'rank': artifact.get('rank')
        })
        
        # Extract media information
        images = artifact.get('images', [])
        for img in images:
            media_records.append({
                'artifact_id': artifact.get('id'),
                'iiifbaseuri': img.get('iiifbaseuri'),
                'baseimageurl': img.get('baseimageurl'),
                'publiccaption': img.get('publiccaption')
            })
        
        # Extract color information
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    metadata_df = pd.DataFrame(metadata_records)
    media_df = pd.DataFrame(media_records)
    colors_df = pd.DataFrame(color_records)
    
    return metadata_df, media_df, colors_df
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def create_db_connection():
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata(df, connection):
    """Batch insert metadata records"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, department, classification, 
         dated, description, url, creditline, rank)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture), 
        century=VALUES(century), department=VALUES(department)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    
    return cursor.rowcount

def load_media(df, connection):
    """Batch insert media records"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, iiifbaseuri, baseimageurl, publiccaption)
        VALUES (%s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    
    return cursor.rowcount

def load_colors(df, connection):
    """Batch insert color records"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    
    return cursor.rowcount
```

### 4. Analytical SQL Queries

```python
# Sample analytical queries for the Streamlit dashboard

ANALYTICAL_QUERIES = {
    "Artifacts by Department": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Top Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN media_count > 0 THEN 'With Media' ELSE 'No Media' END as media_status,
            COUNT(*) as artifact_count
        FROM (
            SELECT a.id, COUNT(m.media_id) as media_count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY a.id
        ) media_summary
        GROUP BY media_status
    """,
    
    "Top Color Spectrums": """
        SELECT spectrum, COUNT(*) as usage_count
        FROM artifactcolors
        WHERE spectrum IS NOT NULL
        GROUP BY spectrum
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts with Multiple Images": """
        SELECT a.title, a.culture, COUNT(m.media_id) as image_count
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY a.id, a.title, a.culture
        HAVING image_count > 1
        ORDER BY image_count DESC
        LIMIT 10
    """
}

def execute_query(query, connection):
    """Execute SQL query and return results as DataFrame"""
    try:
        df = pd.read_sql(query, connection)
        return df
    except Exception as e:
        print(f"Query execution error: {e}")
        return None
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Data Analytics")
    st.markdown("---")
    
    # Sidebar for navigation
    st.sidebar.header("Navigation")
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    # Database connection
    conn = create_db_connection()
    
    if page == "Data Collection":
        st.header("📥 Data Collection from Harvard API")
        
        num_records = st.number_input(
            "Number of records to fetch",
            min_value=10,
            max_value=1000,
            value=100,
            step=10
        )
        
        if st.button("Fetch and Load Data"):
            with st.spinner("Fetching data from API..."):
                api_key = os.getenv('HARVARD_API_KEY')
                artifacts = fetch_artifacts(api_key, num_records)
                
                st.success(f"Fetched {len(artifacts)} artifacts")
                
                # Transform
                metadata_df, media_df, colors_df = transform_artifacts(artifacts)
                
                # Load
                metadata_count = load_metadata(metadata_df, conn)
                media_count = load_media(media_df, conn)
                colors_count = load_colors(colors_df, conn)
                
                st.success(f"""
                ✅ Loaded {metadata_count} metadata records
                ✅ Loaded {media_count} media records
                ✅ Loaded {colors_count} color records
                """)
    
    elif page == "SQL Analytics":
        st.header("📊 SQL Analytics Dashboard")
        
        query_name = st.selectbox(
            "Select Analysis",
            list(ANALYTICAL_QUERIES.keys())
        )
        
        query = ANALYTICAL_QUERIES[query_name]
        
        st.code(query, language='sql')
        
        if st.button("Run Query"):
            result_df = execute_query(query, conn)
            
            if result_df is not None and not result_df.empty:
                st.dataframe(result_df, use_container_width=True)
                
                # Auto-generate visualization
                if len(result_df.columns) == 2:
                    fig = px.bar(
                        result_df,
                        x=result_df.columns[0],
                        y=result_df.columns[1],
                        title=query_name
                    )
                    st.plotly_chart(fig, use_container_width=True)
    
    elif page == "Visualizations":
        st.header("📈 Data Visualizations")
        
        # Example: Culture distribution
        query = ANALYTICAL_QUERIES["Top Cultures by Artifact Count"]
        df = execute_query(query, conn)
        
        if df is not None:
            fig = px.bar(
                df,
                x='culture',
                y='artifact_count',
                title='Top 10 Cultures by Artifact Count',
                color='artifact_count',
                color_continuous_scale='viridis'
            )
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### ETL Pipeline Pattern

```python
def run_etl_pipeline(api_key, num_records=100):
    """Complete ETL pipeline execution"""
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_artifacts(api_key, num_records)
    
    # Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(artifacts)
    
    # Clean data
    metadata_df = metadata_df.fillna('')
    media_df = media_df.fillna('')
    colors_df = colors_df.fillna(0)
    
    # Load
    print("Loading data to database...")
    conn = create_db_connection()
    
    try:
        load_metadata(metadata_df, conn)
        load_media(media_df, conn)
        load_colors(colors_df, conn)
        print("ETL pipeline completed successfully")
    finally:
        conn.close()
```

### Incremental Data Loading

```python
def get_latest_artifact_id(connection):
    """Get the most recent artifact ID in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def fetch_new_artifacts(api_key, last_id):
    """Fetch only artifacts newer than last_id"""
    # Implementation depends on API filtering capabilities
    pass
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff on rate limit"""
    for attempt in range(max_retries):
        response = requests.get(url, params=params)
        
        if response.status_code == 200:
            return response
        elif response.status_code == 429:  # Rate limited
            wait_time = 2 ** attempt
            print(f"Rate limited. Waiting {wait_time}s...")
            time.sleep(wait_time)
        else:
            raise Exception(f"API error: {response.status_code}")
    
    raise Exception("Max retries exceeded")
```

### Database Connection Pooling

```python
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_connection_from_pool():
    return db_pool.get_connection()
```

### Handling Missing Data

```python
def safe_transform(artifact, field, default=''):
    """Safely extract field with default value"""
    return artifact.get(field, default) or default

# Usage in transform
metadata_records.append({
    'id': safe_transform(artifact, 'id', 0),
    'title': safe_transform(artifact, 'title', 'Untitled'),
    'culture': safe_transform(artifact, 'culture', 'Unknown')
})
```

## Best Practices

1. **Always use environment variables** for API keys and database credentials
2. **Implement batch inserts** for performance with large datasets
3. **Use ON DUPLICATE KEY UPDATE** for idempotent ETL runs
4. **Add indexes** on foreign keys and frequently queried columns
5. **Log ETL operations** for monitoring and debugging
6. **Validate data** before database insertion
7. **Use connection pooling** for production deployments
