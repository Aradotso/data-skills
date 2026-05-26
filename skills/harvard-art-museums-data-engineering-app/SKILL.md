---
name: harvard-art-museums-data-engineering-app
description: End-to-end data engineering and analytics application using Harvard Art Museums API with ETL pipelines, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline with Harvard Art Museums API
  - create ETL workflow for museum artifact data
  - set up Harvard artifacts analytics dashboard
  - implement museum data engineering project
  - develop Streamlit app with Harvard API
  - design SQL schema for art museum data
  - analyze Harvard Art Museums collection data
  - visualize museum artifact metadata with Plotly
---

# Harvard Art Museums Data Engineering App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build and work with the Harvard Art Museums Data Engineering and Analytics Application. The project demonstrates production-grade ETL pipelines, relational database design, SQL analytics, and interactive visualization using the Harvard Art Museums API.

## What This Project Does

The application provides a complete data engineering workflow:

1. **Extract** artifact data from Harvard Art Museums API with pagination and rate limiting
2. **Transform** nested JSON into normalized relational tables
3. **Load** structured data into MySQL/TiDB Cloud databases
4. **Analyze** using 20+ predefined SQL queries
5. **Visualize** results through interactive Streamlit dashboards with Plotly charts

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

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

### Environment Variables

Create a `.env` file in the project root:

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

### Get Harvard API Key

1. Register at https://docs.hms.harvard.edu/
2. Request an API key (free tier available)
3. Add to `.env` file

### Database Setup

```python
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

# Connect to database
conn = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME'),
    port=int(os.getenv('DB_PORT', 3306))
)

# Create tables
cursor = conn.cursor()

# Artifact metadata table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    url TEXT,
    imagepermissionlevel INT,
    totalpageviews INT,
    totaluniquepageviews INT
)
""")

# Artifact media table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri TEXT,
    baseimageurl TEXT,
    publiccaption TEXT,
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

conn.commit()
conn.close()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Key API Patterns

### Fetching Artifacts with Pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_artifacts=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(artifacts) < num_artifacts:
        params = {
            'apikey': api_key,
            'page': page,
            'size': min(size, num_artifacts - len(artifacts))
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            if len(data.get('records', [])) < size:
                break  # No more records
                
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_artifacts]
```

### ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

def extract_artifact_data(artifacts: List[Dict]) -> tuple:
    """Transform nested JSON into normalized dataframes"""
    
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'imagepermissionlevel': artifact.get('imagepermissionlevel', 0),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_records.append(metadata)
        
        # Extract media (images)
        for image in artifact.get('images', []):
            media = {
                'artifact_id': artifact.get('id'),
                'iiifbaseuri': image.get('iiifbaseuri'),
                'baseimageurl': image.get('baseimageurl'),
                'publiccaption': image.get('publiccaption')
            }
            media_records.append(media)
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_rec = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(color_rec)
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors


def load_to_database(df_metadata, df_media, df_colors):
    """Load dataframes into SQL database"""
    
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    cursor = conn.cursor()
    
    # Insert metadata
    for _, row in df_metadata.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, classification, 
             department, dated, url, imagepermissionlevel, 
             totalpageviews, totaluniquepageviews)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
            title=VALUES(title), culture=VALUES(culture)
        """, tuple(row))
    
    # Insert media
    for _, row in df_media.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (artifact_id, iiifbaseuri, baseimageurl, publiccaption)
            VALUES (%s, %s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in df_colors.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    conn.close()
```

## Streamlit Application Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px

st.set_page_config(
    page_title="Harvard Artifacts Analytics",
    page_icon="🏛️",
    layout="wide"
)

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for configuration
with st.sidebar:
    st.header("Configuration")
    num_artifacts = st.number_input(
        "Number of artifacts to fetch",
        min_value=10,
        max_value=1000,
        value=100
    )
    
    if st.button("Fetch & Load Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(num_artifacts)
            df_meta, df_media, df_colors = extract_artifact_data(artifacts)
            load_to_database(df_meta, df_media, df_colors)
            st.success(f"Loaded {len(artifacts)} artifacts!")

# Analytics queries
st.header("📊 Analytics Queries")

queries = {
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
    "Top Departments": """
        SELECT department, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE department IS NOT NULL 
        GROUP BY department 
        ORDER BY count DESC
    """,
    "Color Distribution": """
        SELECT color, COUNT(*) as count 
        FROM artifactcolors 
        GROUP BY color 
        ORDER BY count DESC 
        LIMIT 15
    """,
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as with_media,
            (SELECT COUNT(*) FROM artifactmetadata) - COUNT(DISTINCT m.artifact_id) as without_media
        FROM artifactmedia m
    """
}

query_name = st.selectbox("Select Query", list(queries.keys()))

if st.button("Run Query"):
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df_result = pd.read_sql(queries[query_name], conn)
    conn.close()
    
    st.dataframe(df_result)
    
    # Auto-visualize
    if len(df_result.columns) == 2:
        fig = px.bar(
            df_result,
            x=df_result.columns[0],
            y=df_result.columns[1],
            title=query_name
        )
        st.plotly_chart(fig, use_container_width=True)
```

## Advanced Analytics Queries

```python
# Top viewed artifacts
top_viewed_query = """
SELECT title, culture, totalpageviews
FROM artifactmetadata
WHERE totalpageviews > 0
ORDER BY totalpageviews DESC
LIMIT 20
"""

# Color analysis by culture
color_by_culture = """
SELECT 
    m.culture,
    c.color,
    AVG(c.percent) as avg_percent
FROM artifactmetadata m
JOIN artifactcolors c ON m.id = c.artifact_id
WHERE m.culture IS NOT NULL
GROUP BY m.culture, c.color
ORDER BY avg_percent DESC
LIMIT 30
"""

# Classification distribution
classification_dist = """
SELECT 
    classification,
    COUNT(*) as count,
    AVG(totalpageviews) as avg_views
FROM artifactmetadata
WHERE classification IS NOT NULL
GROUP BY classification
ORDER BY count DESC
"""
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
            time.sleep(wait_time)
        else:
            break
    
    return None
```

### Database Connection Errors

```python
def get_db_connection():
    """Create database connection with error handling"""
    try:
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            connect_timeout=10
        )
        return conn
    except mysql.connector.Error as err:
        st.error(f"Database connection failed: {err}")
        return None
```

### Handling NULL Values

```python
# Clean dataframe before insertion
df_metadata = df_metadata.where(pd.notnull(df_metadata), None)

# Or use default values
df_metadata['totalpageviews'] = df_metadata['totalpageviews'].fillna(0)
df_metadata['culture'] = df_metadata['culture'].fillna('Unknown')
```

## Common Patterns

### Batch Processing for Large Datasets

```python
def batch_insert(df, table_name, batch_size=1000):
    """Insert dataframe in batches"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        # Insert batch
        for _, row in batch.iterrows():
            # Your insert logic here
            pass
        conn.commit()
    
    conn.close()
```

### Export Results

```python
# Export query results to CSV
if st.button("Export to CSV"):
    df_result.to_csv("query_results.csv", index=False)
    st.download_button(
        "Download CSV",
        data=df_result.to_csv(index=False),
        file_name="artifacts_analysis.csv"
    )
```
