---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and transform Harvard API data into SQL
  - analyze art museum collection with SQL queries
  - visualize museum artifact data with Streamlit
  - set up data engineering pipeline for art collections
  - query Harvard Art Museums database
  - create interactive art analytics dashboard
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact collections.

## What It Does

The Harvard Artifacts Collection application:
- Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- Performs ETL operations to transform nested JSON into relational database schema
- Stores structured data in MySQL/TiDB with proper foreign key relationships
- Executes 20+ predefined analytical SQL queries
- Visualizes results using interactive Plotly charts in Streamlit dashboards

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests plotly mysql-connector-python sqlalchemy
```

## Configuration

### Environment Variables

Set up your API and database credentials:

```python
# API Configuration
HARVARD_API_KEY = os.getenv('HARVARD_API_KEY')  # Get from https://harvardartmuseums.org/collections/api

# Database Configuration
DB_HOST = os.getenv('DB_HOST', 'localhost')
DB_PORT = os.getenv('DB_PORT', '3306')
DB_USER = os.getenv('DB_USER')
DB_PASSWORD = os.getenv('DB_PASSWORD')
DB_NAME = os.getenv('DB_NAME', 'harvard_artifacts')
```

### Streamlit Secrets (recommended)

Create `.streamlit/secrets.toml`:

```toml
[api]
harvard_api_key = "your-api-key-here"

[database]
host = "your-db-host"
port = 3306
user = "your-db-user"
password = "your-db-password"
database = "harvard_artifacts"
```

## Database Schema

The project uses three main tables with relational structure:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    credit_line TEXT,
    accession_number VARCHAR(100),
    PRIMARY KEY (id)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    thumbnail_url VARCHAR(1000),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core API Usage

### Extract Data from Harvard API

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
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
    
    data = response.json()
    return data['records'], data['info']

def extract_all_artifacts(api_key, max_pages=10):
    """
    Extract artifacts with pagination
    """
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        records, info = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(records)
        
        if page >= info['pages']:
            break
    
    return all_artifacts
```

### Transform and Load into SQL

```python
import mysql.connector
from sqlalchemy import create_engine

def create_database_connection(host, user, password, database):
    """
    Create SQLAlchemy engine for database operations
    """
    connection_string = f"mysql+mysqlconnector://{user}:{password}@{host}/{database}"
    engine = create_engine(connection_string)
    return engine

def transform_metadata(artifacts):
    """
    Transform artifact JSON to metadata DataFrame
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'period': artifact.get('period', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'division': artifact.get('division', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'medium': artifact.get('medium', '')[:500],
            'dimensions': artifact.get('dimensions', '')[:500],
            'credit_line': artifact.get('creditline', ''),
            'accession_number': artifact.get('accessionyear', '')
        })
    
    return pd.DataFrame(metadata_list)

def transform_media(artifacts):
    """
    Transform artifact media/images to DataFrame
    """
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        if 'images' in artifact and artifact['images']:
            for image in artifact['images']:
                media_list.append({
                    'artifact_id': artifact_id,
                    'image_url': image.get('baseimageurl', ''),
                    'thumbnail_url': image.get('thumbnailurl', '')
                })
        elif 'primaryimageurl' in artifact:
            media_list.append({
                'artifact_id': artifact_id,
                'image_url': artifact.get('primaryimageurl', ''),
                'thumbnail_url': ''
            })
    
    return pd.DataFrame(media_list)

def transform_colors(artifacts):
    """
    Transform artifact color data to DataFrame
    """
    colors_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                colors_list.append({
                    'artifact_id': artifact_id,
                    'color': color.get('color', ''),
                    'spectrum': color.get('spectrum', ''),
                    'percent': color.get('percent', 0)
                })
    
    return pd.DataFrame(colors_list)

def load_to_database(engine, metadata_df, media_df, colors_df):
    """
    Load transformed DataFrames to database
    """
    # Load metadata (parent table first)
    metadata_df.to_sql('artifactmetadata', engine, if_exists='append', index=False)
    
    # Load media
    if not media_df.empty:
        media_df.to_sql('artifactmedia', engine, if_exists='append', index=False)
    
    # Load colors
    if not colors_df.empty:
        colors_df.to_sql('artifactcolors', engine, if_exists='append', index=False)
```

## Streamlit Application Structure

### Main Dashboard

```python
import streamlit as st
import pandas as pd
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("Explore artifact collections through data analytics")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    else:
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection")
    
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=st.secrets.get("api", {}).get("harvard_api_key", ""))
    
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            artifacts = extract_all_artifacts(api_key, max_pages=num_pages)
            st.success(f"Extracted {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df = transform_metadata(artifacts)
            media_df = transform_media(artifacts)
            colors_df = transform_colors(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            engine = create_database_connection(
                st.secrets["database"]["host"],
                st.secrets["database"]["user"],
                st.secrets["database"]["password"],
                st.secrets["database"]["database"]
            )
            load_to_database(engine, metadata_df, media_df, colors_df)
            st.success("✅ ETL Pipeline completed successfully!")

if __name__ == "__main__":
    main()
```

## Analytics Queries

### Common SQL Queries

```python
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY artifact_count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as usage_count, 
               AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Artifacts with Media": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as with_media,
            COUNT(DISTINCT a.id) as total_artifacts,
            ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / COUNT(DISTINCT a.id), 2) as percentage
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
    """,
    
    "Color Spectrum Analysis": """
        SELECT spectrum, COUNT(*) as count,
               ROUND(AVG(percent), 2) as avg_dominance
        FROM artifactcolors
        WHERE spectrum IS NOT NULL
        GROUP BY spectrum
        ORDER BY count DESC
    """
}

def execute_query(engine, query):
    """
    Execute SQL query and return DataFrame
    """
    return pd.read_sql(query, engine)

def show_sql_analytics():
    st.header("📊 SQL Analytics")
    
    # Connect to database
    engine = create_database_connection(
        st.secrets["database"]["host"],
        st.secrets["database"]["user"],
        st.secrets["database"]["password"],
        st.secrets["database"]["database"]
    )
    
    # Query selector
    query_name = st.selectbox("Select Query", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Execute Query"):
        query = ANALYTICAL_QUERIES[query_name]
        
        st.code(query, language="sql")
        
        with st.spinner("Executing query..."):
            results_df = execute_query(engine, query)
            
            st.dataframe(results_df)
            
            # Auto-generate visualization
            if len(results_df.columns) >= 2:
                fig = px.bar(results_df, 
                            x=results_df.columns[0], 
                            y=results_df.columns[1],
                            title=query_name)
                st.plotly_chart(fig, use_container_width=True)
```

## Visualization Examples

```python
import plotly.graph_objects as go

def create_culture_chart(df):
    """
    Create interactive bar chart for culture distribution
    """
    fig = px.bar(df, 
                 x='culture', 
                 y='artifact_count',
                 title='Artifact Distribution by Culture',
                 labels={'artifact_count': 'Number of Artifacts', 'culture': 'Culture'},
                 color='artifact_count',
                 color_continuous_scale='Viridis')
    
    fig.update_layout(xaxis_tickangle=-45)
    return fig

def create_color_pie_chart(df):
    """
    Create pie chart for color usage
    """
    fig = px.pie(df, 
                 values='usage_count', 
                 names='color',
                 title='Color Usage Distribution',
                 hole=0.3)
    
    return fig

def create_timeline_chart(df):
    """
    Create timeline visualization for centuries
    """
    fig = px.line(df, 
                  x='century', 
                  y='artifact_count',
                  title='Artifacts Over Centuries',
                  markers=True)
    
    return fig
```

## Running the Application

```bash
# Run the Streamlit dashboard
streamlit run app.py

# Access the dashboard at http://localhost:8501
```

## Troubleshooting

**API Rate Limiting:**
```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise
```

**Database Connection Issues:**
```python
def test_database_connection(engine):
    try:
        with engine.connect() as conn:
            result = conn.execute("SELECT 1")
            return True
    except Exception as e:
        st.error(f"Database connection failed: {e}")
        return False
```

**Handling Null Values:**
```python
def clean_dataframe(df):
    # Replace None with empty strings for VARCHAR columns
    string_columns = df.select_dtypes(include=['object']).columns
    df[string_columns] = df[string_columns].fillna('')
    
    # Replace None with 0 for numeric columns
    numeric_columns = df.select_dtypes(include=['float64', 'int64']).columns
    df[numeric_columns] = df[numeric_columns].fillna(0)
    
    return df
```

This skill enables AI agents to build complete data engineering pipelines for museum artifact analysis, from API extraction through SQL analytics to interactive dashboards.
