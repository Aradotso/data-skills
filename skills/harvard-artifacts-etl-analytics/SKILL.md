---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Streamlit and SQL
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - create a data engineering project with Harvard artifacts
  - build analytics dashboard for museum collection data
  - extract and transform Harvard Art Museums API data
  - set up SQL analytics for artifact collections
  - visualize museum artifacts data with Streamlit
  - create end-to-end data pipeline for art museum data
  - analyze Harvard museum collections with Python
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Collect artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into relational databases
- **SQL Analytics**: Pre-built analytical queries for insight generation
- **Interactive Dashboards**: Streamlit-based visualization with Plotly charts
- **Database Design**: Normalized schema with proper relationships for artifacts, media, and colors

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

### 1. Harvard Art Museums API Key

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file or configure directly in the app:

```bash
# .env
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

Set up MySQL or TiDB Cloud connection:

```python
import os
from dotenv import load_dotenv

load_dotenv()

DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

## Key Components and Usage

### 1. API Data Extraction

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_records=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'page': page,
            'size': min(size, num_records - len(artifacts))
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            if not data.get('info', {}).get('next'):
                break
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### 2. ETL Pipeline - Transform and Load

```python
import mysql.connector
from mysql.connector import Error

def create_database_schema(connection):
    """Create normalized database schema for artifacts"""
    cursor = connection.cursor()
    
    # Artifacts metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            division VARCHAR(200),
            medium VARCHAR(500),
            dated VARCHAR(200),
            accessionyear INT,
            verificationlevel INT,
            totalpageviews INT,
            totaluniquepageviews INT
        )
    """)
    
    # Artifacts media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            primaryimageurl VARCHAR(1000),
            imagepermissionlevel INT,
            has_images BOOLEAN,
            total_images INT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifacts colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
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

def transform_and_load_artifacts(artifacts, connection):
    """Transform API data and load into SQL database"""
    cursor = connection.cursor()
    
    for artifact in artifacts:
        # Extract metadata
        metadata = (
            artifact.get('id'),
            artifact.get('title', '')[:500],
            artifact.get('culture', ''),
            artifact.get('century', ''),
            artifact.get('classification', ''),
            artifact.get('department', ''),
            artifact.get('division', ''),
            artifact.get('medium', ''),
            artifact.get('dated', ''),
            artifact.get('accessionyear'),
            artifact.get('verificationlevel', 0),
            artifact.get('totalpageviews', 0),
            artifact.get('totaluniquepageviews', 0)
        )
        
        # Insert metadata
        cursor.execute("""
            INSERT IGNORE INTO artifactmetadata 
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, metadata)
        
        # Extract and insert media data
        media = (
            artifact.get('id'),
            artifact.get('primaryimageurl', ''),
            artifact.get('imagepermissionlevel', 0),
            1 if artifact.get('images') else 0,
            len(artifact.get('images', []))
        )
        
        cursor.execute("""
            INSERT INTO artifactmedia (artifact_id, primaryimageurl, 
                imagepermissionlevel, has_images, total_images)
            VALUES (%s, %s, %s, %s, %s)
        """, media)
        
        # Extract and insert color data
        for color_data in artifact.get('colors', []):
            color = (
                artifact.get('id'),
                color_data.get('color'),
                color_data.get('spectrum'),
                color_data.get('hue'),
                color_data.get('percent', 0)
            )
            
            cursor.execute("""
                INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """, color)
    
    connection.commit()
    cursor.close()
```

### 3. Analytical SQL Queries

```python
# Example analytics queries
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifacts DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            has_images,
            COUNT(*) as count,
            AVG(total_images) as avg_images
        FROM artifactmedia
        GROUP BY has_images
    """,
    
    "Color Distribution": """
        SELECT spectrum, COUNT(*) as usage_count
        FROM artifactcolors
        WHERE spectrum IS NOT NULL
        GROUP BY spectrum
        ORDER BY usage_count DESC
    """,
    
    "Most Viewed Artifacts": """
        SELECT title, culture, totalpageviews, totaluniquepageviews
        FROM artifactmetadata
        WHERE totalpageviews > 0
        ORDER BY totalpageviews DESC
        LIMIT 20
    """,
    
    "Accession Year Distribution": """
        SELECT accessionyear, COUNT(*) as artifacts_acquired
        FROM artifactmetadata
        WHERE accessionyear IS NOT NULL
        GROUP BY accessionyear
        ORDER BY accessionyear DESC
        LIMIT 20
    """
}

def execute_query(connection, query_name):
    """Execute analytical query and return results as DataFrame"""
    cursor = connection.cursor()
    cursor.execute(ANALYTICAL_QUERIES[query_name])
    
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    
    cursor.close()
    return pd.DataFrame(results, columns=columns)
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # API Key input
    api_key = st.text_input("Enter Harvard API Key", type="password")
    
    if not api_key:
        st.warning("Please enter your Harvard Art Museums API key")
        return
    
    # Database connection
    try:
        connection = mysql.connector.connect(**DB_CONFIG)
        st.success("✅ Database connected")
    except Error as e:
        st.error(f"Database connection failed: {e}")
        return
    
    # ETL Section
    st.header("📥 Data Collection (ETL)")
    
    num_records = st.number_input("Number of records to fetch", 
                                   min_value=10, max_value=1000, value=100)
    
    if st.button("Fetch and Load Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(api_key, num_records)
            st.info(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Loading to database..."):
            create_database_schema(connection)
            transform_and_load_artifacts(artifacts, connection)
            st.success("✅ Data loaded successfully")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    
    query_selection = st.selectbox("Select Analysis", 
                                    list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Run Query"):
        df = execute_query(connection, query_selection)
        
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=query_selection)
            st.plotly_chart(fig)
    
    connection.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Batch Processing for Large Datasets

```python
def batch_insert_artifacts(artifacts, connection, batch_size=500):
    """Insert artifacts in batches for better performance"""
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i + batch_size]
        transform_and_load_artifacts(batch, connection)
        print(f"Processed batch {i//batch_size + 1}")
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_api_fetch(api_key, num_records):
    """Fetch with error handling and retry logic"""
    try:
        artifacts = fetch_artifacts(api_key, num_records)
        logger.info(f"Successfully fetched {len(artifacts)} artifacts")
        return artifacts
    except requests.RequestException as e:
        logger.error(f"API fetch failed: {e}")
        return []
    except Exception as e:
        logger.error(f"Unexpected error: {e}")
        return []
```

### Data Validation

```python
def validate_artifact_data(artifact):
    """Validate artifact data before insertion"""
    required_fields = ['id', 'title']
    
    for field in required_fields:
        if field not in artifact or artifact[field] is None:
            return False
    
    return True

def filter_valid_artifacts(artifacts):
    """Filter only valid artifacts"""
    return [a for a in artifacts if validate_artifact_data(a)]
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_rate_limit(api_key, num_records, delay=1):
    """Fetch with rate limiting between requests"""
    artifacts = []
    page = 1
    
    while len(artifacts) < num_records:
        # Fetch page
        response = requests.get(base_url, params={'apikey': api_key, 'page': page})
        
        if response.status_code == 429:
            st.warning("Rate limit hit, waiting...")
            time.sleep(5)
            continue
        
        artifacts.extend(response.json().get('records', []))
        page += 1
        time.sleep(delay)  # Polite delay
    
    return artifacts
```

### Database Connection Issues

```python
def create_connection_with_retry(config, retries=3):
    """Create database connection with retry logic"""
    for attempt in range(retries):
        try:
            connection = mysql.connector.connect(**config)
            return connection
        except Error as e:
            if attempt < retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise e
```

### Handling NULL Values

```python
def safe_get(dictionary, key, default=''):
    """Safely get value with default for NULL handling"""
    value = dictionary.get(key, default)
    return value if value is not None else default
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY=your_api_key
export DB_HOST=localhost
export DB_USER=root
export DB_PASSWORD=your_password
export DB_NAME=harvard_artifacts

# Run Streamlit app
streamlit run app.py
```

The app will open in your browser at `http://localhost:8501` with interactive ETL controls and analytics dashboards.
