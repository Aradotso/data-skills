---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create a data analytics app using museum artifacts data
  - set up SQL analytics for Harvard Art Museums collection
  - build a Streamlit dashboard for art museum data
  - extract and visualize Harvard artifacts metadata
  - implement data engineering pipeline for museum API
  - query and analyze Harvard Art Museums data
  - create interactive visualizations for museum artifacts
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations.

## What It Does

The Harvard Art Museums ETL Analytics project provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into SQL databases
- **SQL Analytics**: Execute predefined analytical queries on structured artifact data
- **Interactive Dashboards**: Visualize query results using Streamlit and Plotly
- **Database Design**: Relational schema with proper foreign key relationships

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

1. Get your API key from [Harvard Art Museums API](https://docs.harvardartmuseums.org/):
   - Register at Harvard Art Museums
   - Navigate to API access section
   - Generate your API key

2. Create a `.env` file in the project root:
```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Setup

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

# Create database connection
connection = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

# Create tables
cursor = connection.cursor()

# Artifact metadata table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    url VARCHAR(500)
)
""")

# Artifact media table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
)
""")

# Artifact colors table
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
connection.close()
```

## API Data Collection

### Fetch Artifacts with Pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_artifacts=100, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    
    while len(artifacts) < num_artifacts:
        params = {
            'apikey': api_key,
            'size': min(page_size, num_artifacts - len(artifacts)),
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            artifacts.extend(records)
            page += 1
            
            # Rate limiting
            import time
            time.sleep(0.5)
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_artifacts]
```

## ETL Pipeline Implementation

### Extract and Transform

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform raw API data into structured dataframes
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear'),
            'url': artifact.get('url')
        }
        metadata_records.append(metadata)
        
        # Extract media information
        images = artifact.get('images', [])
        for img in images:
            media = {
                'artifact_id': artifact.get('id'),
                'media_type': 'image',
                'baseimageurl': img.get('baseimageurl'),
                'iiifbaseuri': img.get('iiifbaseuri')
            }
            media_records.append(media)
        
        # Extract color information
        colors = artifact.get('colors', [])
        for color in colors:
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load to Database

```python
def load_to_database(metadata_df, media_df, colors_df):
    """
    Batch insert dataframes into SQL database
    """
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    cursor = connection.cursor()
    
    # Insert metadata
    metadata_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, division, 
     dated, technique, medium, dimensions, creditline, accessionyear, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    for _, row in metadata_df.iterrows():
        cursor.execute(metadata_query, tuple(row))
    
    # Insert media
    media_query = """
    INSERT INTO artifactmedia (artifact_id, media_type, baseimageurl, iiifbaseuri)
    VALUES (%s, %s, %s, %s)
    """
    
    for _, row in media_df.iterrows():
        cursor.execute(media_query, tuple(row))
    
    # Insert colors
    colors_query = """
    INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    for _, row in colors_df.iterrows():
        cursor.execute(colors_query, tuple(row))
    
    connection.commit()
    cursor.close()
    connection.close()
```

## SQL Analytics Queries

### Common Analytical Patterns

```python
# Query 1: Artifact distribution by culture
query_by_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 20
"""

# Query 2: Artifacts by century
query_by_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Query 3: Media availability analysis
query_media_stats = """
SELECT 
    am.classification,
    COUNT(DISTINCT am.id) as total_artifacts,
    COUNT(DISTINCT ame.artifact_id) as artifacts_with_media,
    ROUND(COUNT(DISTINCT ame.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as media_percentage
FROM artifactmetadata am
LEFT JOIN artifactmedia ame ON am.id = ame.artifact_id
GROUP BY am.classification
ORDER BY total_artifacts DESC
LIMIT 15
"""

# Query 4: Color distribution analysis
query_color_distribution = """
SELECT 
    color,
    COUNT(*) as usage_count,
    ROUND(AVG(percent), 2) as avg_percentage
FROM artifactcolors
WHERE color IS NOT NULL
GROUP BY color
ORDER BY usage_count DESC
LIMIT 20
"""

# Query 5: Department insights
query_department_stats = """
SELECT 
    department,
    COUNT(*) as artifact_count,
    COUNT(DISTINCT classification) as unique_classifications,
    MIN(accessionyear) as earliest_accession,
    MAX(accessionyear) as latest_accession
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY artifact_count DESC
"""
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")

st.title("🎨 Harvard Art Museums Collection Analytics")

# Sidebar for navigation
page = st.sidebar.selectbox(
    "Select Page",
    ["Data Collection", "SQL Analytics", "Visualizations"]
)

def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Data Collection Page
if page == "Data Collection":
    st.header("📥 Collect Artifact Data")
    
    num_artifacts = st.slider("Number of artifacts to collect", 10, 500, 100)
    
    if st.button("Fetch Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(num_artifacts)
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            load_to_database(metadata_df, media_df, colors_df)
            
            st.success(f"Successfully loaded {len(metadata_df)} artifacts!")
            st.dataframe(metadata_df.head())

# SQL Analytics Page
elif page == "SQL Analytics":
    st.header("📊 SQL Analytics Dashboard")
    
    queries = {
        "Artifacts by Culture": query_by_culture,
        "Artifacts by Century": query_by_century,
        "Media Availability": query_media_stats,
        "Color Distribution": query_color_distribution,
        "Department Statistics": query_department_stats
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        connection = get_db_connection()
        df = pd.read_sql(queries[selected_query], connection)
        connection.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

# Visualizations Page
elif page == "Visualizations":
    st.header("📈 Interactive Visualizations")
    
    connection = get_db_connection()
    
    # Culture distribution pie chart
    culture_df = pd.read_sql(query_by_culture, connection)
    fig1 = px.pie(culture_df, values='artifact_count', names='culture',
                  title='Artifact Distribution by Culture (Top 20)')
    st.plotly_chart(fig1, use_container_width=True)
    
    # Color usage bar chart
    color_df = pd.read_sql(query_color_distribution, connection)
    fig2 = px.bar(color_df, x='color', y='usage_count',
                  title='Color Usage in Artifacts',
                  color='avg_percentage')
    st.plotly_chart(fig2, use_container_width=True)
    
    connection.close()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Complete ETL Workflow

```python
from dotenv import load_dotenv
import os

load_dotenv()

def run_etl_pipeline(num_artifacts=100):
    """
    Execute complete ETL pipeline
    """
    print("Step 1: Extracting data from API...")
    artifacts = fetch_artifacts(num_artifacts)
    
    print("Step 2: Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(artifacts)
    
    print("Step 3: Loading to database...")
    load_to_database(metadata_df, media_df, colors_df)
    
    print(f"ETL complete! Processed {len(artifacts)} artifacts.")
    return metadata_df, media_df, colors_df

# Execute
if __name__ == "__main__":
    run_etl_pipeline(200)
```

## Troubleshooting

### API Rate Limiting
```python
# Add retry logic with exponential backoff
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session():
    session = requests.Session()
    retry = Retry(total=3, backoff_factor=1, status_forcelist=[429, 500, 502, 503, 504])
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    return session
```

### Database Connection Issues
```python
# Test database connection
def test_db_connection():
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        print("✅ Database connection successful")
        connection.close()
        return True
    except Exception as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Handling Missing Data
```python
# Clean and validate data before insertion
def clean_dataframe(df):
    # Replace None with appropriate defaults
    df = df.fillna({
        'culture': 'Unknown',
        'century': 'Unknown',
        'accessionyear': 0
    })
    
    # Truncate long strings
    if 'title' in df.columns:
        df['title'] = df['title'].str[:500]
    
    return df
```
