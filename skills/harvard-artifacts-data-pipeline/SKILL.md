---
name: harvard-artifacts-data-pipeline
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, Streamlit, and SQL
triggers:
  - build data pipeline for harvard art museums
  - create etl workflow for museum artifacts
  - analyze harvard art collection data
  - build streamlit dashboard for museum data
  - query harvard artifacts with sql
  - visualize art museum data
  - set up museum artifact analytics
  - extract harvard art api data
---

# Harvard Artifacts Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering & Analytics App is an end-to-end data pipeline that demonstrates real-world ETL patterns. It extracts artifact data from the Harvard Art Museums API, transforms nested JSON into relational tables, loads data into SQL databases (MySQL/TiDB), and provides interactive analytics dashboards using Streamlit.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

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

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Get Harvard Art Museums API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Register for a free API key
3. Add to `.env` file

### Database Setup

The application uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    url TEXT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl TEXT,
    renditionnumber VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py
```

The app will be available at `http://localhost:8501`

## Code Examples

### API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
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
    
    data = response.json()
    return data['records'], data['info']

# Fetch first 100 artifacts
artifacts, info = fetch_artifacts(page=1, size=100)
print(f"Total artifacts: {info['totalrecords']}")
print(f"Pages: {info['pages']}")
```

### ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifact_data(raw_artifacts):
    """Transform nested JSON to relational format"""
    metadata = []
    media = []
    colors = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url')
        })
        
        # Extract media information
        if artifact.get('images'):
            for img in artifact['images']:
                media.append({
                    'artifact_id': artifact['id'],
                    'media_type': 'image',
                    'baseimageurl': img.get('baseimageurl'),
                    'renditionnumber': img.get('renditionnumber')
                })
        
        # Extract color information
        if artifact.get('colors'):
            for color_data in artifact['colors']:
                colors.append({
                    'artifact_id': artifact['id'],
                    'color': color_data.get('color'),
                    'spectrum': color_data.get('spectrum'),
                    'percentage': color_data.get('percent')
                })
    
    return (
        pd.DataFrame(metadata),
        pd.DataFrame(media),
        pd.DataFrame(colors)
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Load transformed data into SQL database"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        cursor = connection.cursor()
        
        # Insert metadata (using batch insert for performance)
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, dated, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        metadata_values = metadata_df.values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, media_type, baseimageurl, renditionnumber)
            VALUES (%s, %s, %s, %s)
        """
        if not media_df.empty:
            cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, percentage)
            VALUES (%s, %s, %s, %s)
        """
        if not colors_df.empty:
            cursor.executemany(colors_query, colors_df.values.tolist())
        
        connection.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### Streamlit Dashboard Components

```python
import streamlit as st
import plotly.express as px

def create_analytics_dashboard():
    """Main dashboard with SQL analytics"""
    st.title("🎨 Harvard Artifacts Analytics Dashboard")
    
    # Sidebar for query selection
    queries = {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE culture IS NOT NULL 
            GROUP BY culture 
            ORDER BY count DESC 
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
            SELECT department, COUNT(*) as count 
            FROM artifactmetadata 
            GROUP BY department 
            ORDER BY count DESC
        """,
        "Most Common Colors": """
            SELECT color, COUNT(*) as frequency, AVG(percentage) as avg_percentage
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY frequency DESC 
            LIMIT 20
        """,
        "Media Availability": """
            SELECT 
                COUNT(DISTINCT am.artifact_id) as artifacts_with_media,
                (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts
            FROM artifactmedia am
        """
    }
    
    selected_query = st.sidebar.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        df = execute_query(queries[selected_query])
        
        # Display results
        st.subheader(f"Results: {selected_query}")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2 and df.shape[0] > 0:
            fig = px.bar(
                df, 
                x=df.columns[0], 
                y=df.columns[1],
                title=selected_query
            )
            st.plotly_chart(fig, use_container_width=True)

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```

### Complete ETL Workflow

```python
def run_etl_pipeline(num_pages=5):
    """Complete ETL pipeline execution"""
    all_metadata = []
    all_media = []
    all_colors = []
    
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}/{num_pages}")
        
        # Extract
        artifacts, info = fetch_artifacts(page=page, size=100)
        
        # Transform
        metadata_df, media_df, colors_df = transform_artifact_data(artifacts)
        
        all_metadata.append(metadata_df)
        all_media.append(media_df)
        all_colors.append(colors_df)
        
        # Rate limiting
        import time
        time.sleep(1)
    
    # Combine all data
    final_metadata = pd.concat(all_metadata, ignore_index=True)
    final_media = pd.concat(all_media, ignore_index=True)
    final_colors = pd.concat(all_colors, ignore_index=True)
    
    # Load
    load_to_database(final_metadata, final_media, final_colors)
    
    return {
        'artifacts': len(final_metadata),
        'media_records': len(final_media),
        'color_records': len(final_colors)
    }

# Execute pipeline
stats = run_etl_pipeline(num_pages=10)
print(f"ETL Complete: {stats}")
```

## Common Patterns

### Pagination Handling

```python
def fetch_all_artifacts(max_pages=None):
    """Fetch all artifacts with pagination"""
    all_artifacts = []
    page = 1
    
    while True:
        artifacts, info = fetch_artifacts(page=page)
        all_artifacts.extend(artifacts)
        
        if page >= info['pages'] or (max_pages and page >= max_pages):
            break
        
        page += 1
    
    return all_artifacts
```

### Error Handling for API Requests

```python
import time

def safe_api_request(page, max_retries=3):
    """API request with retry logic"""
    for attempt in range(max_retries):
        try:
            artifacts, info = fetch_artifacts(page=page)
            return artifacts, info
        except requests.exceptions.RequestException as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
                time.sleep(wait_time)
            else:
                raise e
```

### Data Quality Checks

```python
def validate_artifact_data(df):
    """Validate data quality before loading"""
    issues = []
    
    # Check for duplicates
    if df['id'].duplicated().any():
        issues.append(f"Found {df['id'].duplicated().sum()} duplicate IDs")
    
    # Check for missing critical fields
    if df['id'].isnull().any():
        issues.append("Missing artifact IDs")
    
    # Check data types
    if not pd.api.types.is_integer_dtype(df['id']):
        issues.append("ID column is not integer type")
    
    return issues if issues else None
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limiting errors:

```python
# Add exponential backoff
import time

def fetch_with_backoff(page, initial_wait=1):
    wait_time = initial_wait
    max_wait = 32
    
    while wait_time <= max_wait:
        try:
            return fetch_artifacts(page=page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:
                time.sleep(wait_time)
                wait_time *= 2
            else:
                raise
```

### Database Connection Issues

```python
def test_database_connection():
    """Test database connectivity"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            connect_timeout=10
        )
        print("✓ Database connection successful")
        connection.close()
        return True
    except Error as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Memory Management for Large Datasets

```python
def process_in_chunks(num_pages, chunk_size=5):
    """Process and load data in chunks to manage memory"""
    for start_page in range(1, num_pages + 1, chunk_size):
        end_page = min(start_page + chunk_size, num_pages + 1)
        
        stats = run_etl_pipeline(
            start_page=start_page, 
            end_page=end_page
        )
        
        print(f"Processed pages {start_page}-{end_page}: {stats}")
```

## Advanced Analytics Queries

### Time-based Analysis

```python
time_analysis_query = """
    SELECT 
        century,
        COUNT(*) as total_artifacts,
        COUNT(DISTINCT culture) as cultures,
        COUNT(DISTINCT classification) as classifications
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY century
"""
```

### Color Spectrum Analysis

```python
color_spectrum_query = """
    SELECT 
        ac.spectrum,
        ac.color,
        COUNT(*) as frequency,
        AVG(ac.percentage) as avg_percentage,
        am.classification
    FROM artifactcolors ac
    JOIN artifactmetadata am ON ac.artifact_id = am.id
    GROUP BY ac.spectrum, ac.color, am.classification
    HAVING frequency > 10
    ORDER BY frequency DESC
"""
```

This skill provides comprehensive guidance for building data pipelines with the Harvard Art Museums API, implementing ETL workflows, and creating analytics dashboards with Streamlit.
