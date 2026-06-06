---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API data with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifacts data
  - create a data engineering project with Harvard Art Museums API
  - set up artifact collection analytics with Streamlit
  - implement SQL analytics for museum collections
  - build a data visualization dashboard for art artifacts
  - extract and analyze Harvard museum data
  - create an end-to-end data pipeline with API to dashboard
  - design a relational database for artifact metadata
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build and work with end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL workflows, SQL database design, analytical queries, and interactive Streamlit dashboards for artifact collections.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts nested JSON, transforms into relational format, loads into SQL databases
- **Database Schema**: Three-table design (metadata, media, colors) with proper foreign keys
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Visualization**: Interactive Streamlit dashboards with Plotly charts

**Architecture Flow**: API → ETL → SQL (MySQL/TiDB) → Analytics → Streamlit Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_db_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Setup

```python
import os
import mysql.connector
from dotenv import load_dotenv

load_dotenv()

def create_database_connection():
    """Establish MySQL connection"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    return connection

def create_tables(connection):
    """Create the artifact tables schema"""
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            medium VARCHAR(500),
            department VARCHAR(200),
            division VARCHAR(200),
            dated VARCHAR(200),
            dimensions VARCHAR(500),
            creditline TEXT,
            technique VARCHAR(500),
            provenance TEXT,
            description TEXT,
            url VARCHAR(500),
            totalpageviews INT,
            totaluniquepageviews INT
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            baseimageurl VARCHAR(500),
            primaryimageurl VARCHAR(500),
            imagepermissionlevel INT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent DECIMAL(5,2),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()
    cursor.close()
```

## Core ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100):
    """Fetch artifact data from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    records_per_page = 100
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': records_per_page,
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            if len(data.get('records', [])) == 0:
                break
            
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### Transform: Clean and Structure Data

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """Transform raw API data into relational format"""
    
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'technique': artifact.get('technique'),
            'provenance': artifact.get('provenance'),
            'description': artifact.get('description'),
            'url': artifact.get('url'),
            'totalpageviews': artifact.get('totalpageviews'),
            'totaluniquepageviews': artifact.get('totaluniquepageviews')
        }
        metadata_list.append(metadata)
        
        # Extract media
        if artifact.get('primaryimageurl'):
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl'),
                'primaryimageurl': artifact.get('primaryimageurl'),
                'imagepermissionlevel': artifact.get('imagepermissionlevel')
            }
            media_list.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_entry = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            colors_list.append(color_entry)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Insert into SQL Database

```python
def load_to_database(metadata_df, media_df, colors_df, connection):
    """Load transformed data into MySQL tables"""
    cursor = connection.cursor()
    
    # Insert metadata (batch insert)
    metadata_insert = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, medium, 
         department, division, dated, dimensions, creditline, technique, 
         provenance, description, url, totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    metadata_values = metadata_df.values.tolist()
    cursor.executemany(metadata_insert, metadata_values)
    
    # Insert media
    if not media_df.empty:
        media_insert = """
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, primaryimageurl, imagepermissionlevel)
            VALUES (%s, %s, %s, %s)
        """
        media_values = media_df.values.tolist()
        cursor.executemany(media_insert, media_values)
    
    # Insert colors
    if not colors_df.empty:
        colors_insert = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        colors_values = colors_df.values.tolist()
        cursor.executemany(colors_insert, colors_values)
    
    connection.commit()
    cursor.close()
    
    print(f"Loaded {len(metadata_df)} artifacts successfully!")
```

## SQL Analytics Queries

### Common Analytical Queries

```python
# Sample analytical queries for the dashboard
ANALYTICAL_QUERIES = {
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
        ORDER BY count DESC
    """,
    
    "Most Common Classifications": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Artifacts with Media Coverage": """
        SELECT 
            CASE 
                WHEN am.artifact_id IS NOT NULL THEN 'Has Image'
                ELSE 'No Image'
            END as media_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia am ON a.id = am.artifact_id
        GROUP BY media_status
    """,
    
    "Top Colors in Collection": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Most Viewed Artifacts": """
        SELECT title, culture, totalpageviews
        FROM artifactmetadata
        WHERE totalpageviews IS NOT NULL
        ORDER BY totalpageviews DESC
        LIMIT 10
    """
}

def execute_query(connection, query_name):
    """Execute an analytical query and return results"""
    cursor = connection.cursor()
    query = ANALYTICAL_QUERIES[query_name]
    
    cursor.execute(query)
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    
    cursor.close()
    
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard

### Main Application

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "Analytics Dashboard", "Database Info"]
    )
    
    # Database connection
    try:
        conn = create_database_connection()
    except Exception as e:
        st.error(f"Database connection failed: {e}")
        return
    
    if page == "Data Collection":
        show_data_collection_page(conn)
    elif page == "Analytics Dashboard":
        show_analytics_page(conn)
    else:
        show_database_info(conn)
    
    conn.close()

def show_data_collection_page(conn):
    """ETL page for collecting new artifacts"""
    st.header("📥 Artifact Data Collection")
    
    num_records = st.number_input(
        "Number of artifacts to collect",
        min_value=10,
        max_value=1000,
        value=100,
        step=10
    )
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Fetching artifacts from API..."):
            raw_artifacts = fetch_artifacts(num_records)
            st.success(f"Extracted {len(raw_artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(raw_artifacts)
            st.success("Data transformation complete")
        
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df, conn)
            st.success("✅ ETL Pipeline Complete!")
        
        # Show preview
        st.subheader("Preview - Artifact Metadata")
        st.dataframe(metadata_df.head(10))

def show_analytics_page(conn):
    """Analytics dashboard with SQL queries"""
    st.header("📊 Analytics Dashboard")
    
    # Query selector
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICAL_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            results_df = execute_query(conn, query_name)
        
        st.subheader("Query Results")
        st.dataframe(results_df)
        
        # Auto-generate visualization
        if len(results_df.columns) >= 2:
            st.subheader("Visualization")
            
            # Use first column as x, second as y
            x_col = results_df.columns[0]
            y_col = results_df.columns[1]
            
            fig = px.bar(
                results_df,
                x=x_col,
                y=y_col,
                title=query_name,
                labels={x_col: x_col.title(), y_col: y_col.title()}
            )
            
            st.plotly_chart(fig, use_container_width=True)

def show_database_info(conn):
    """Display database statistics"""
    st.header("🗄️ Database Information")
    
    cursor = conn.cursor()
    
    # Table counts
    cursor.execute("SELECT COUNT(*) FROM artifactmetadata")
    metadata_count = cursor.fetchone()[0]
    
    cursor.execute("SELECT COUNT(*) FROM artifactmedia")
    media_count = cursor.fetchone()[0]
    
    cursor.execute("SELECT COUNT(*) FROM artifactcolors")
    colors_count = cursor.fetchone()[0]
    
    col1, col2, col3 = st.columns(3)
    col1.metric("Total Artifacts", metadata_count)
    col2.metric("Media Records", media_count)
    col3.metric("Color Records", colors_count)
    
    cursor.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Workflow

```python
def run_complete_etl(num_records=100):
    """Execute full ETL pipeline"""
    # 1. Connect to database
    conn = create_database_connection()
    create_tables(conn)
    
    # 2. Extract
    print("Extracting data from API...")
    raw_artifacts = fetch_artifacts(num_records)
    
    # 3. Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_artifacts)
    
    # 4. Load
    print("Loading to database...")
    load_to_database(metadata_df, media_df, colors_df, conn)
    
    # 5. Verify
    cursor = conn.cursor()
    cursor.execute("SELECT COUNT(*) FROM artifactmetadata")
    count = cursor.fetchone()[0]
    print(f"Total artifacts in database: {count}")
    
    cursor.close()
    conn.close()
    
    return count
```

### Custom Query Execution

```python
def run_custom_query(sql_query):
    """Execute custom SQL query and return DataFrame"""
    conn = create_database_connection()
    cursor = conn.cursor()
    
    try:
        cursor.execute(sql_query)
        columns = [desc[0] for desc in cursor.description]
        results = cursor.fetchall()
        df = pd.DataFrame(results, columns=columns)
        return df
    except Exception as e:
        print(f"Query error: {e}")
        return None
    finally:
        cursor.close()
        conn.close()
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
import time

def fetch_artifacts_with_retry(num_records=100, delay=1):
    """Fetch with delay between requests"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    
    while len(artifacts) < num_records:
        response = requests.get(
            base_url,
            params={'apikey': api_key, 'size': 100, 'page': page}
        )
        
        if response.status_code == 429:  # Rate limited
            print("Rate limited, waiting 5 seconds...")
            time.sleep(5)
            continue
        
        if response.status_code == 200:
            artifacts.extend(response.json().get('records', []))
            time.sleep(delay)  # Polite delay
            page += 1
        else:
            break
    
    return artifacts[:num_records]
```

### Database Connection Issues

```python
def create_database_connection_with_retry(max_retries=3):
    """Create connection with retry logic"""
    for attempt in range(max_retries):
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
        except mysql.connector.Error as e:
            print(f"Connection attempt {attempt + 1} failed: {e}")
            if attempt < max_retries - 1:
                time.sleep(2)
            else:
                raise
```

### Handling Missing Data

```python
def safe_transform(artifact, field, default=None):
    """Safely extract field with default value"""
    value = artifact.get(field, default)
    
    # Handle None for string fields
    if isinstance(value, str):
        return value[:500] if value else default
    
    return value

# Use in transformation
metadata = {
    'id': artifact.get('id'),
    'title': safe_transform(artifact, 'title', 'Untitled'),
    'culture': safe_transform(artifact, 'culture', 'Unknown'),
    # ... etc
}
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# The app will open at http://localhost:8501

# Or specify port
streamlit run app.py --server.port 8080
```

This skill provides everything needed to build, deploy, and extend the Harvard Artifacts ETL Analytics application with proper data engineering practices.
