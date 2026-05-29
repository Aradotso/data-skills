---
name: harvard-art-museums-data-pipeline
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build a data pipeline for museum artifacts
  - create ETL workflow with Harvard Art Museums API
  - set up artifact data analytics dashboard
  - process and visualize museum collection data
  - implement SQL-based art museum analytics
  - design streamlit app for cultural heritage data
  - extract and analyze Harvard museum artifacts
  - build end-to-end data engineering pipeline
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates an end-to-end data engineering and analytics application using the Harvard Art Museums API. It implements ETL pipelines, SQL analytics, and interactive Streamlit dashboards for artifact data visualization.

## Architecture

**API → ETL → SQL → Analytics → Visualization**

- **Data Source**: Harvard Art Museums API (paginated, rate-limited)
- **ETL**: Python with requests and pandas
- **Storage**: MySQL/TiDB Cloud with relational schema
- **Analytics**: SQL queries with 20+ predefined insights
- **Frontend**: Streamlit with Plotly visualizations

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Run the application
streamlit run app.py
```

### Dependencies

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

# MySQL/TiDB Database
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Schema

The application uses three main tables:

```sql
-- Artifact metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    credit_line TEXT,
    accession_number VARCHAR(100),
    url VARCHAR(500)
);

-- Artifact media/images
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    base_url VARCHAR(500),
    alt_text TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact color analysis
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key API Patterns

### Fetching Artifacts from Harvard API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts with pagination support"""
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

# Usage
artifacts, info = fetch_artifacts(page=1, size=50)
print(f"Total records: {info['totalrecords']}")
print(f"Total pages: {info['totalpages']}")
```

### Handling Rate Limits

```python
import time

def fetch_all_artifacts(max_pages=10, delay=1):
    """Fetch multiple pages with rate limiting"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            artifacts, info = fetch_artifacts(page=page, size=100)
            all_artifacts.extend(artifacts)
            print(f"Fetched page {page}/{info['totalpages']}")
            
            # Respect rate limits
            time.sleep(delay)
            
        except requests.exceptions.RequestException as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

## ETL Pipeline Implementation

### Extract: Parse Artifact Data

```python
def extract_artifact_metadata(artifact):
    """Extract metadata from API response"""
    return {
        'id': artifact.get('id'),
        'title': artifact.get('title'),
        'culture': artifact.get('culture'),
        'century': artifact.get('century'),
        'classification': artifact.get('classification'),
        'department': artifact.get('department'),
        'division': artifact.get('division'),
        'dated': artifact.get('dated'),
        'period': artifact.get('period'),
        'technique': artifact.get('technique'),
        'medium': artifact.get('medium'),
        'dimensions': artifact.get('dimensions'),
        'credit_line': artifact.get('creditline'),
        'accession_number': artifact.get('accessionyear'),
        'url': artifact.get('url')
    }

def extract_media_data(artifact):
    """Extract media/image data"""
    media_list = []
    if 'images' in artifact and artifact['images']:
        for image in artifact['images']:
            media_list.append({
                'artifact_id': artifact['id'],
                'image_url': image.get('iiifbaseuri'),
                'base_url': image.get('baseimageurl'),
                'alt_text': image.get('alttext')
            })
    return media_list

def extract_color_data(artifact):
    """Extract color analysis data"""
    colors = []
    if 'colors' in artifact and artifact['colors']:
        for color in artifact['colors']:
            colors.append({
                'artifact_id': artifact['id'],
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent')
            })
    return colors
```

### Transform: Clean and Normalize

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """Transform raw API data into structured DataFrames"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_artifacts:
        # Extract each component
        metadata_list.append(extract_artifact_metadata(artifact))
        media_list.extend(extract_media_data(artifact))
        colors_list.extend(extract_color_data(artifact))
    
    # Create DataFrames
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    # Clean null values
    df_metadata = df_metadata.fillna('')
    df_media = df_media.fillna('')
    df_colors = df_colors.dropna()
    
    return df_metadata, df_media, df_colors
```

### Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=os.getenv('DB_PORT'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_sql(df_metadata, df_media, df_colors):
    """Load DataFrames into SQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata (batch)
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, 
         division, dated, period, technique, medium, dimensions, 
         credit_line, accession_number, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, df_metadata.values.tolist())
        
        # Insert media
        if not df_media.empty:
            media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, image_url, base_url, alt_text)
            VALUES (%s, %s, %s, %s)
            """
            cursor.executemany(media_query, df_media.values.tolist())
        
        # Insert colors
        if not df_colors.empty:
            colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color_hex, color_percent)
            VALUES (%s, %s, %s)
            """
            cursor.executemany(colors_query, df_colors.values.tolist())
        
        conn.commit()
        print(f"Loaded {len(df_metadata)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

## SQL Analytics Queries

### Example Analytical Queries

```python
# Query 1: Artifacts by culture
query_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture != ''
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Query 2: Century distribution
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL AND century != ''
GROUP BY century
ORDER BY century
"""

# Query 3: Artifacts with images
query_images = """
SELECT am.department, COUNT(DISTINCT am.id) as artifacts_with_images
FROM artifactmetadata am
INNER JOIN artifactmedia media ON am.id = media.artifact_id
GROUP BY am.department
ORDER BY artifacts_with_images DESC
"""

# Query 4: Most common colors
query_colors = """
SELECT color_hex, 
       COUNT(*) as usage_count,
       AVG(color_percent) as avg_percent
FROM artifactcolors
GROUP BY color_hex
ORDER BY usage_count DESC
LIMIT 15
"""

def execute_analytics_query(query):
    """Execute SQL query and return DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard Integration

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics")
    st.markdown("Data Engineering Pipeline & Visualization")
    
    # Sidebar for query selection
    query_options = {
        "Artifacts by Culture": query_culture,
        "Century Distribution": query_century,
        "Departments with Images": query_images,
        "Popular Colors": query_colors
    }
    
    selected_query = st.sidebar.selectbox(
        "Select Analysis",
        list(query_options.keys())
    )
    
    # Execute query
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            df_result = execute_analytics_query(query_options[selected_query])
            
            # Display results
            st.subheader("Query Results")
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) >= 2:
                fig = px.bar(
                    df_result,
                    x=df_result.columns[0],
                    y=df_result.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Complete ETL Workflow

```python
def run_etl_pipeline(num_pages=5):
    """Execute complete ETL pipeline"""
    st.write("### 🔄 Starting ETL Pipeline")
    
    # Extract
    st.write("**Step 1:** Extracting data from API...")
    raw_artifacts = fetch_all_artifacts(max_pages=num_pages)
    st.success(f"Extracted {len(raw_artifacts)} artifacts")
    
    # Transform
    st.write("**Step 2:** Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(raw_artifacts)
    st.success("Data transformed into structured format")
    
    # Load
    st.write("**Step 3:** Loading to database...")
    load_to_sql(df_metadata, df_media, df_colors)
    st.success("✅ ETL Pipeline Complete!")
    
    # Summary statistics
    st.write("### 📊 Summary")
    col1, col2, col3 = st.columns(3)
    col1.metric("Artifacts", len(df_metadata))
    col2.metric("Images", len(df_media))
    col3.metric("Color Records", len(df_colors))
```

## Troubleshooting

### API Key Issues

```python
# Verify API key is loaded
def check_api_key():
    api_key = os.getenv('HARVARD_API_KEY')
    if not api_key:
        raise ValueError("HARVARD_API_KEY not found in environment")
    return True
```

### Database Connection Errors

```python
# Test database connectivity
def test_db_connection():
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        conn.close()
        return result is not None
    except Error as e:
        print(f"Connection failed: {e}")
        return False
```

### Handling Missing Data

```python
# Safely extract nested fields
def safe_get(data, *keys, default=''):
    """Safely navigate nested dictionaries"""
    for key in keys:
        if isinstance(data, dict):
            data = data.get(key, default)
        else:
            return default
    return data if data is not None else default

# Usage
culture = safe_get(artifact, 'culture', default='Unknown')
```

## Best Practices

1. **Rate Limiting**: Always add delays between API calls (1-2 seconds)
2. **Batch Processing**: Insert data in batches for better performance
3. **Error Handling**: Wrap API calls and DB operations in try-except blocks
4. **Data Validation**: Check for null/empty values before inserting
5. **Incremental Loading**: Use `ON DUPLICATE KEY UPDATE` to handle re-runs
6. **Connection Pooling**: Reuse database connections when possible
