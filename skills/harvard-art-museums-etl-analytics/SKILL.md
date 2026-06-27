---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for museum data
  - create a data engineering project with Harvard Art Museums API
  - set up Streamlit analytics dashboard for artifact data
  - extract and analyze art museum collection data
  - build a data pipeline from API to SQL database
  - create visualizations for museum artifact collections
  - implement ETL with Pandas and SQL for art data
  - develop an analytics app using Harvard Art Museums
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is a complete data pipeline that:
- Fetches artifact data from the Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into relational database tables
- Loads data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries for insights
- Visualizes results through interactive Streamlit dashboards

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Core Dependencies**:
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
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    accessionyear INT,
    dated VARCHAR(255),
    url VARCHAR(1000)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(1000),
    primaryimageurl VARCHAR(1000),
    iiifbaseuri VARCHAR(1000),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
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

def fetch_artifacts(num_artifacts=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    all_artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(all_artifacts) < num_artifacts:
        params = {
            'apikey': api_key,
            'size': min(size, num_artifacts - len(all_artifacts)),
            'page': page,
            'hasimage': 1  # Only artifacts with images
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
    
    return all_artifacts[:num_artifacts]
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector

def transform_artifacts(raw_data):
    """
    Transform nested JSON into relational dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'accessionyear': artifact.get('accessionyear'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        if artifact.get('primaryimageurl'):
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl'),
                'primaryimageurl': artifact.get('primaryimageurl'),
                'iiifbaseuri': artifact.get('iiifbaseuri')
            }
            media_list.append(media)
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            colors_list.append(color_record)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

def load_to_database(metadata_df, media_df, colors_df):
    """
    Load transformed data into MySQL database
    """
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    cursor = conn.cursor()
    
    # Load metadata (batch insert)
    metadata_tuples = [tuple(row) for row in metadata_df.values]
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, 
         technique, accessionyear, dated, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    cursor.executemany(metadata_query, metadata_tuples)
    
    # Load media
    if not media_df.empty:
        media_tuples = [tuple(row) for row in media_df.values]
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, primaryimageurl, iiifbaseuri)
            VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(media_query, media_tuples)
    
    # Load colors
    if not colors_df.empty:
        colors_tuples = [tuple(row) for row in colors_df.values]
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(colors_query, colors_tuples)
    
    conn.commit()
    cursor.close()
    conn.close()
```

### 3. Analytical SQL Queries

```python
# Sample analytical queries for the application

ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
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
    
    "Most Common Colors": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Artifacts with Images": """
        SELECT 
            CASE 
                WHEN m.artifact_id IS NOT NULL THEN 'Has Image'
                ELSE 'No Image'
            END as image_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY image_status
    """,
    
    "Top Techniques": """
        SELECT technique, COUNT(*) as count
        FROM artifactmetadata
        WHERE technique IS NOT NULL
        GROUP BY technique
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Accession Year Trends": """
        SELECT accessionyear, COUNT(*) as acquisitions
        FROM artifactmetadata
        WHERE accessionyear IS NOT NULL AND accessionyear > 1900
        GROUP BY accessionyear
        ORDER BY accessionyear
    """,
    
    "Color Spectrum Analysis": """
        SELECT spectrum, COUNT(DISTINCT artifact_id) as artifacts,
               AVG(percent) as avg_coverage
        FROM artifactcolors
        WHERE spectrum IS NOT NULL
        GROUP BY spectrum
        ORDER BY artifacts DESC
    """
}

def execute_query(query_name):
    """Execute analytical query and return results"""
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    query = ANALYTICAL_QUERIES[query_name]
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Data Analytics")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Module",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    else:
        show_visualizations_page()

def show_etl_page():
    st.header("📥 ETL Pipeline")
    
    num_artifacts = st.number_input(
        "Number of artifacts to fetch",
        min_value=10,
        max_value=1000,
        value=100,
        step=10
    )
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            raw_data = fetch_artifacts(num_artifacts)
            st.success(f"Fetched {len(raw_data)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            st.success("Data transformed successfully")
            
            col1, col2, col3 = st.columns(3)
            col1.metric("Metadata Records", len(metadata_df))
            col2.metric("Media Records", len(media_df))
            col3.metric("Color Records", len(colors_df))
        
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
            st.success("✅ ETL Pipeline completed successfully!")

def show_analytics_page():
    st.header("📊 SQL Analytics Dashboard")
    
    query_name = st.selectbox(
        "Select Analysis Query",
        list(ANALYTICAL_QUERIES.keys())
    )
    
    if st.button("Execute Query"):
        with st.spinner("Running query..."):
            results_df = execute_query(query_name)
            
            st.subheader("Query Results")
            st.dataframe(results_df, use_container_width=True)
            
            # Auto-generate visualization
            if len(results_df.columns) == 2:
                fig = px.bar(
                    results_df,
                    x=results_df.columns[0],
                    y=results_df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)

def show_visualizations_page():
    st.header("📈 Interactive Visualizations")
    
    # Example: Culture distribution pie chart
    df = execute_query("Artifacts by Culture")
    
    col1, col2 = st.columns(2)
    
    with col1:
        fig_bar = px.bar(
            df.head(10),
            x='culture',
            y='artifact_count',
            title="Top 10 Cultures by Artifact Count"
        )
        st.plotly_chart(fig_bar, use_container_width=True)
    
    with col2:
        fig_pie = px.pie(
            df.head(10),
            values='artifact_count',
            names='culture',
            title="Culture Distribution"
        )
        st.plotly_chart(fig_pie, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will open in your browser at http://localhost:8501
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def incremental_fetch(last_id=0):
    """Fetch only new artifacts since last run"""
    query = f"""
        SELECT MAX(id) as last_id 
        FROM artifactmetadata
    """
    # Get last ID from database
    # Fetch only artifacts with id > last_id
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'after': last_id
    }
    # Continue with fetch...
```

### Pattern 2: Error Handling in ETL

```python
def safe_etl_pipeline(num_artifacts):
    """ETL with comprehensive error handling"""
    try:
        raw_data = fetch_artifacts(num_artifacts)
    except requests.RequestException as e:
        st.error(f"API Error: {str(e)}")
        return False
    
    try:
        metadata_df, media_df, colors_df = transform_artifacts(raw_data)
    except Exception as e:
        st.error(f"Transformation Error: {str(e)}")
        return False
    
    try:
        load_to_database(metadata_df, media_df, colors_df)
    except mysql.connector.Error as e:
        st.error(f"Database Error: {str(e)}")
        return False
    
    return True
```

### Pattern 3: Data Quality Checks

```python
def validate_data(df, required_columns):
    """Validate dataframe before loading"""
    # Check for required columns
    missing = set(required_columns) - set(df.columns)
    if missing:
        raise ValueError(f"Missing columns: {missing}")
    
    # Check for null primary keys
    if df['id'].isnull().any():
        raise ValueError("Null values found in primary key")
    
    # Check for duplicates
    duplicates = df['id'].duplicated().sum()
    if duplicates > 0:
        st.warning(f"Found {duplicates} duplicate IDs, will be handled by UPSERT")
    
    return True
```

## Troubleshooting

### API Rate Limiting
```python
import time

def fetch_with_rate_limit(url, params, delay=1):
    """Add delay between requests"""
    response = requests.get(url, params=params)
    time.sleep(delay)  # Wait 1 second between requests
    return response
```

### Database Connection Issues
```python
def get_db_connection(retries=3):
    """Retry database connection"""
    for attempt in range(retries):
        try:
            conn = mysql.connector.connect(
                host=os.getenv('DB_HOST'),
                port=int(os.getenv('DB_PORT', 3306)),
                user=os.getenv('DB_USER'),
                password=os.getenv('DB_PASSWORD'),
                database=os.getenv('DB_NAME')
            )
            return conn
        except mysql.connector.Error as e:
            if attempt < retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise e
```

### Memory Management for Large Datasets
```python
def chunk_load(df, chunk_size=1000):
    """Load data in chunks to avoid memory issues"""
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i+chunk_size]
        # Process chunk
        yield chunk
```

## Best Practices

1. **Always use environment variables** for sensitive data (API keys, DB credentials)
2. **Implement batch inserts** for performance with large datasets
3. **Use ON DUPLICATE KEY UPDATE** to handle data refreshes
4. **Add indexes** on frequently queried columns (culture, century, department)
5. **Log ETL runs** with timestamps and record counts for monitoring
6. **Validate data types** before database insertion
7. **Use connection pooling** for multiple database operations
