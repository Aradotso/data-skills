---
name: harvard-art-museum-data-pipeline
description: Build ETL pipelines and analytics apps using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build a data pipeline for Harvard Art Museums API
  - create ETL workflow for museum artifact data
  - analyze Harvard museum collections with SQL
  - visualize museum artifacts with Streamlit
  - extract and transform Harvard Art Museums data
  - build museum data analytics dashboard
  - query Harvard artifacts database
  - create museum collection data engineering pipeline
---

# Harvard Art Museum Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL analytics, and interactive visualization using Streamlit, Python, and MySQL/TiDB Cloud.

## What This Project Does

The Harvard Art Museum Data Pipeline:
- Extracts artifact data from the Harvard Art Museums API with pagination support
- Transforms nested JSON into normalized relational database tables
- Loads data into MySQL/TiDB Cloud with proper foreign key relationships
- Provides 20+ pre-built analytical SQL queries
- Visualizes query results through an interactive Streamlit dashboard

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
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

### API Key Setup

Obtain an API key from the Harvard Art Museums:
1. Visit https://www.harvardartmuseums.org/collections/api
2. Register for a free API key
3. Store in environment variable or `.env` file:

```python
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Configuration

```python
import os
from dotenv import load_dotenv
import mysql.connector

load_dotenv()

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'port': int(os.getenv('DB_PORT', 3306))
}

connection = mysql.connector.connect(**db_config)
```

## Database Schema

The project uses three main tables:

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
    medium VARCHAR(500),
    technique VARCHAR(500),
    dimensions VARCHAR(500)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(1000),
    renditionnumber VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
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

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Extract artifacts from Harvard Art Museums API"""
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

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts, info = fetch_artifacts(api_key, page=1, size=100)
print(f"Total artifacts: {info['totalrecords']}")
print(f"Pages: {info['pages']}")
```

### Transform: Normalize JSON to Relational Format

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform nested JSON into normalized dataframes"""
    
    # Metadata table
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
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique'),
            'dimensions': artifact.get('dimensions')
        })
        
        # Extract media (images)
        if artifact.get('images'):
            for image in artifact['images']:
                media_records.append({
                    'artifact_id': artifact['id'],
                    'baseimageurl': image.get('baseimageurl'),
                    'renditionnumber': image.get('renditionnumber'),
                    'height': image.get('height'),
                    'width': image.get('width')
                })
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_records.append({
                    'artifact_id': artifact['id'],
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load: Insert Data into Database

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """Load transformed data into MySQL database"""
    
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Insert metadata (batch insert)
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, 
         division, dated, medium, technique, dimensions)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
        """
        
        metadata_values = [tuple(row) for row in metadata_df.values]
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, renditionnumber, height, width)
        VALUES (%s, %s, %s, %s, %s)
        """
        
        if not media_df.empty:
            media_values = [tuple(row) for row in media_df.values]
            cursor.executemany(media_query, media_values)
        
        # Insert colors
        color_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        
        if not colors_df.empty:
            color_values = [tuple(row) for row in colors_df.values]
            cursor.executemany(color_query, color_values)
        
        connection.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        connection.rollback()
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

## Analytical SQL Queries

### Example Queries

```python
# Sample analytical queries
analytical_queries = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "top_colors": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "artifacts_with_images": """
        SELECT 
            COUNT(DISTINCT a.id) as total_artifacts,
            COUNT(DISTINCT m.artifact_id) as artifacts_with_images,
            ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / COUNT(DISTINCT a.id), 2) as percentage
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
    """,
    
    "department_distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
}

def execute_query(query, db_config):
    """Execute SQL query and return results as DataFrame"""
    connection = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px
import pandas as pd

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Select Analysis")
    query_choice = st.sidebar.selectbox(
        "Choose Query",
        [
            "Artifacts by Culture",
            "Artifacts by Century",
            "Top Colors",
            "Department Distribution",
            "Media Availability"
        ]
    )
    
    # Database connection
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    # Execute selected query
    if query_choice == "Artifacts by Culture":
        query = analytical_queries["artifacts_by_culture"]
        df = execute_query(query, db_config)
        
        st.subheader("Artifacts Distribution by Culture")
        st.dataframe(df)
        
        fig = px.bar(df, x='culture', y='count', 
                     title="Top 20 Cultures by Artifact Count")
        st.plotly_chart(fig, use_container_width=True)
    
    elif query_choice == "Top Colors":
        query = analytical_queries["top_colors"]
        df = execute_query(query, db_config)
        
        st.subheader("Most Common Colors in Artifacts")
        st.dataframe(df)
        
        fig = px.bar(df, x='color', y='frequency',
                     color='avg_percent',
                     title="Color Frequency and Average Percentage")
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Run the Streamlit dashboard
streamlit run app.py

# Or specify a different port
streamlit run app.py --server.port 8080

# Run in headless mode (server deployment)
streamlit run app.py --server.headless true
```

## Common Patterns

### Pagination Handler for Large Datasets

```python
def fetch_all_artifacts(api_key, max_pages=10):
    """Fetch multiple pages of artifacts with rate limiting"""
    import time
    
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            artifacts, info = fetch_artifacts(api_key, page=page, size=100)
            all_artifacts.extend(artifacts)
            
            print(f"Fetched page {page}/{info['pages']}")
            
            # Rate limiting (API typically allows 2500 requests/day)
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

### Incremental Data Loading

```python
def incremental_load(api_key, db_config, last_artifact_id=None):
    """Load only new artifacts since last ETL run"""
    
    # Get max artifact ID from database
    if last_artifact_id is None:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        cursor.execute("SELECT MAX(id) FROM artifactmetadata")
        result = cursor.fetchone()
        last_artifact_id = result[0] if result[0] else 0
        connection.close()
    
    # Fetch only newer artifacts
    artifacts = []
    page = 1
    
    while True:
        batch, info = fetch_artifacts(api_key, page=page)
        new_artifacts = [a for a in batch if a['id'] > last_artifact_id]
        
        if not new_artifacts:
            break
            
        artifacts.extend(new_artifacts)
        page += 1
    
    return artifacts
```

## Troubleshooting

### API Rate Limiting

```python
# Handle rate limit errors
def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff on rate limit"""
    import time
    
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Too Many Requests
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
# Test database connection
def test_db_connection(db_config):
    """Verify database connectivity"""
    try:
        connection = mysql.connector.connect(**db_config)
        if connection.is_connected():
            db_info = connection.get_server_info()
            print(f"Connected to MySQL Server version {db_info}")
            cursor = connection.cursor()
            cursor.execute("SELECT DATABASE();")
            record = cursor.fetchone()
            print(f"Connected to database: {record}")
            return True
    except Error as e:
        print(f"Error connecting to MySQL: {e}")
        return False
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### Missing Data Handling

```python
# Handle missing/null values in transformation
def safe_transform(artifacts):
    """Transform with null-safe handling"""
    
    metadata_records = []
    
    for artifact in artifacts:
        metadata_records.append({
            'id': artifact.get('id', None),
            'title': artifact.get('title', 'Unknown')[:500],  # Truncate
            'culture': artifact.get('culture', None),
            'century': artifact.get('century', None),
            'classification': artifact.get('classification', None),
            'department': artifact.get('department', None),
            'division': artifact.get('division', None),
            'dated': artifact.get('dated', None),
            'medium': artifact.get('medium', None),
            'technique': artifact.get('technique', None),
            'dimensions': artifact.get('dimensions', None)
        })
    
    return pd.DataFrame(metadata_records)
```

This skill provides AI coding agents with comprehensive knowledge to build, configure, and extend Harvard Art Museum data pipelines for analytics applications.
