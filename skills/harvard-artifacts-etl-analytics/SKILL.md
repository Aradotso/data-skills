---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with museum artifacts
  - extract and visualize Harvard museum collection data
  - set up SQL database for art museum analytics
  - implement streaming data pipeline for artifacts
  - analyze Harvard Art Museums API data
  - create interactive museum data visualization
  - build end-to-end data engineering pipeline for art collections
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables building end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection application:
- Extracts artifact data from the Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into normalized relational tables
- Loads data into SQL databases (MySQL/TiDB Cloud)
- Executes analytical SQL queries for insights
- Visualizes results through interactive Streamlit dashboards with Plotly charts

**Architecture Flow:** `API → ETL → SQL → Analytics → Visualization`

## Installation

### Prerequisites
- Python 3.8+
- MySQL or TiDB Cloud account
- Harvard Art Museums API key (get from https://www.harvardartmuseums.org/collections/api)

### Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Environment Configuration

Create a `.env` file or configure environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

## Database Schema

The project uses three main tables with foreign key relationships:

```sql
-- Artifact Metadata (main table)
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    medium VARCHAR(500),
    technique VARCHAR(500),
    people TEXT,
    copyright TEXT,
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact Media (images and media files)
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(1000),
    primaryimageurl VARCHAR(1000),
    iiifbaseuri VARCHAR(1000),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors (extracted color data)
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

## Key API Patterns

### Fetching Artifacts with Pagination

```python
import requests
import os

def fetch_artifacts(api_key, max_records=100, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    page = 1
    
    while len(all_artifacts) < max_records:
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code != 200:
            print(f"Error: {response.status_code}")
            break
            
        data = response.json()
        records = data.get('records', [])
        
        if not records:
            break
            
        all_artifacts.extend(records)
        page += 1
        
        # Respect rate limits
        import time
        time.sleep(0.5)
    
    return all_artifacts[:max_records]
```

### ETL Transform Pipeline

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform raw API data into structured DataFrames
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'medium': artifact.get('medium', '')[:500],
            'technique': artifact.get('technique', '')[:500],
            'people': str(artifact.get('people', []))[:1000],
            'copyright': artifact.get('copyright', ''),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
        
        # Extract media information
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl', ''),
            'primaryimageurl': artifact.get('primaryimageurl', ''),
            'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri', '') if artifact.get('images') else ''
        }
        media_list.append(media)
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_data)
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

### Loading Data into SQL

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(df_metadata, df_media, df_colors, config):
    """
    Load transformed DataFrames into SQL database
    """
    try:
        connection = mysql.connector.connect(
            host=config['host'],
            port=config['port'],
            user=config['user'],
            password=config['password'],
            database=config['database']
        )
        
        cursor = connection.cursor()
        
        # Insert metadata
        for _, row in df_metadata.iterrows():
            query = """
                INSERT INTO artifactmetadata 
                (id, title, culture, century, classification, department, 
                 dated, medium, technique, people, copyright, 
                 totalpageviews, totaluniquepageviews)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """
            cursor.execute(query, tuple(row))
        
        # Insert media
        for _, row in df_media.iterrows():
            query = """
                INSERT INTO artifactmedia 
                (artifact_id, baseimageurl, primaryimageurl, iiifbaseuri)
                VALUES (%s, %s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        # Insert colors
        for _, row in df_colors.iterrows():
            query = """
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        connection.commit()
        print(f"Loaded {len(df_metadata)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        connection.rollback()
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

## Streamlit Application Pattern

```python
import streamlit as st
import plotly.express as px
import os

# Page configuration
st.set_page_config(
    page_title="Harvard Artifacts Analytics",
    page_icon="🏛️",
    layout="wide"
)

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar configuration
with st.sidebar:
    st.header("⚙️ Configuration")
    
    api_key = st.text_input(
        "Harvard API Key",
        type="password",
        value=os.getenv("HARVARD_API_KEY", "")
    )
    
    max_records = st.slider(
        "Number of Records to Fetch",
        min_value=10,
        max_value=1000,
        value=100,
        step=10
    )
    
    if st.button("🔄 Run ETL Pipeline"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(api_key, max_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
            st.success("Data transformed")
        
        with st.spinner("Loading to database..."):
            db_config = {
                'host': os.getenv('DB_HOST'),
                'port': int(os.getenv('DB_PORT', 3306)),
                'user': os.getenv('DB_USER'),
                'password': os.getenv('DB_PASSWORD'),
                'database': os.getenv('DB_NAME')
            }
            load_to_database(df_meta, df_media, df_colors, db_config)
            st.success("Data loaded to database")

# Analytics tab
tab1, tab2, tab3 = st.tabs(["📊 Analytics", "🔍 Query Editor", "📈 Visualizations"])

with tab1:
    st.header("Pre-built Analytics Queries")
    
    query_options = {
        "Artifacts by Culture": "SELECT culture, COUNT(*) as count FROM artifactmetadata GROUP BY culture ORDER BY count DESC LIMIT 10",
        "Artifacts by Century": "SELECT century, COUNT(*) as count FROM artifactmetadata GROUP BY century ORDER BY count DESC",
        "Top Departments": "SELECT department, COUNT(*) as count FROM artifactmetadata GROUP BY department ORDER BY count DESC",
        "Most Popular Colors": "SELECT color, COUNT(*) as count FROM artifactcolors GROUP BY color ORDER BY count DESC LIMIT 10",
        "Media Availability": "SELECT COUNT(DISTINCT artifact_id) as with_media FROM artifactmedia"
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Execute Query"):
        query = query_options[selected_query]
        # Execute query and display results
        st.code(query, language="sql")

with tab3:
    st.header("Interactive Visualizations")
    
    # Example: Color distribution chart
    if 'df_colors' in locals():
        color_counts = df_colors['color'].value_counts().head(10)
        fig = px.bar(
            x=color_counts.index,
            y=color_counts.values,
            labels={'x': 'Color', 'y': 'Count'},
            title='Top 10 Colors in Artifacts'
        )
        st.plotly_chart(fig, use_container_width=True)
```

## Common Analytical Queries

```sql
-- Top 10 cultures by artifact count
SELECT culture, COUNT(*) as artifact_count 
FROM artifactmetadata 
WHERE culture IS NOT NULL 
GROUP BY culture 
ORDER BY artifact_count DESC 
LIMIT 10;

-- Artifacts with images vs without
SELECT 
    CASE WHEN primaryimageurl IS NOT NULL THEN 'With Image' ELSE 'Without Image' END as image_status,
    COUNT(*) as count
FROM artifactmedia
GROUP BY image_status;

-- Color spectrum distribution
SELECT spectrum, COUNT(*) as count, AVG(percent) as avg_percent
FROM artifactcolors
GROUP BY spectrum
ORDER BY count DESC;

-- Department pageview analysis
SELECT department, 
       SUM(totalpageviews) as total_views,
       AVG(totalpageviews) as avg_views
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY total_views DESC;

-- Join query: Artifacts with dominant colors
SELECT m.title, m.culture, c.color, c.percent
FROM artifactmetadata m
JOIN artifactcolors c ON m.id = c.artifact_id
WHERE c.percent > 50
ORDER BY c.percent DESC
LIMIT 20;
```

## Troubleshooting

### API Rate Limiting
```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except HTTPError as e:
            if e.response.status_code == 429:
                wait_time = (attempt + 1) * 2
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues
```python
def create_connection_with_retry(config, max_attempts=3):
    """Retry database connection with exponential backoff"""
    for attempt in range(max_attempts):
        try:
            connection = mysql.connector.connect(**config)
            return connection
        except Error as e:
            if attempt < max_attempts - 1:
                wait_time = 2 ** attempt
                print(f"Connection failed. Retrying in {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise e
```

### Memory Management for Large Datasets
```python
def batch_insert(df, connection, table_name, batch_size=1000):
    """Insert data in batches to manage memory"""
    cursor = connection.cursor()
    
    for start in range(0, len(df), batch_size):
        batch = df.iloc[start:start + batch_size]
        # Insert batch
        connection.commit()
        print(f"Inserted batch {start//batch_size + 1}")
    
    cursor.close()
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY=your_api_key
export DB_HOST=your_host
export DB_USER=your_user
export DB_PASSWORD=your_password
export DB_NAME=harvard_artifacts

# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```
