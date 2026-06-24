---
name: harvard-artifacts-etl-streamlit-analytics
description: Build ETL pipelines and interactive analytics dashboards using Harvard Art Museums API with Streamlit, MySQL, and Plotly
triggers:
  - build a data pipeline from Harvard Art Museums API
  - create an ETL workflow for museum artifacts
  - set up Streamlit analytics dashboard with SQL
  - extract and visualize Harvard museum collection data
  - implement artifact data engineering pipeline
  - query and analyze museum metadata with SQL
  - build interactive art collection analytics app
  - process Harvard API data into relational database
---

# Harvard Artifacts ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract nested JSON, transform into relational tables, load into MySQL/TiDB
- **SQL Analytics**: Run predefined analytical queries on artifact metadata, media, and colors
- **Interactive Dashboards**: Visualize query results using Streamlit and Plotly
- **Database Design**: Proper schema with foreign key relationships across artifacts, media, and color tables

**Architecture Flow**: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required Dependencies** (typical `requirements.txt`):
```
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
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Setup

**Schema Design** (three main tables):

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    period VARCHAR(255),
    dated VARCHAR(255),
    url TEXT,
    description TEXT,
    provenance TEXT
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_url TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Data Collection

**Fetching artifacts with pagination:**

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = 'https://api.harvardartmuseums.org/object'
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Collect multiple pages
def collect_artifacts(num_pages=10):
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(page=page)
        artifacts = data.get('records', [])
        all_artifacts.extend(artifacts)
        print(f"Collected page {page}: {len(artifacts)} artifacts")
    
    return all_artifacts
```

### 2. ETL Pipeline

**Extract, Transform, Load process:**

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifacts(raw_data):
    """Transform nested JSON into relational dataframes"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'period': artifact.get('period'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance')
        }
        metadata_list.append(metadata)
        
        # Extract media
        for media in artifact.get('images', []):
            media_entry = {
                'artifact_id': artifact.get('id'),
                'base_url': media.get('baseimageurl'),
                'format': media.get('format'),
                'height': media.get('height'),
                'width': media.get('width')
            }
            media_list.append(media_entry)
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_entry = {
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex'),
                'color_name': color.get('color'),
                'percentage': color.get('percent')
            }
            colors_list.append(color_entry)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Load dataframes into MySQL database"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        cursor = connection.cursor()
        
        # Insert metadata
        for _, row in metadata_df.iterrows():
            sql = """INSERT INTO artifactmetadata 
                     (id, title, culture, century, classification, department, 
                      division, technique, medium, period, dated, url, description, provenance)
                     VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                     ON DUPLICATE KEY UPDATE title=VALUES(title)"""
            cursor.execute(sql, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            sql = """INSERT INTO artifactmedia 
                     (artifact_id, base_url, format, height, width)
                     VALUES (%s, %s, %s, %s, %s)"""
            cursor.execute(sql, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            sql = """INSERT INTO artifactcolors 
                     (artifact_id, color_hex, color_name, percentage)
                     VALUES (%s, %s, %s, %s)"""
            cursor.execute(sql, tuple(row))
        
        connection.commit()
        print(f"Successfully loaded {len(metadata_df)} artifacts")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 3. Streamlit Application

**Main app structure:**

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museum Analytics")
    st.markdown("End-to-end data pipeline and analytics dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    elif page == "Visualizations":
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection")
    
    num_pages = st.number_input("Number of pages to collect", 
                                 min_value=1, max_value=50, value=10)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Collecting artifacts..."):
            artifacts = collect_artifacts(num_pages)
            st.success(f"Collected {len(artifacts)} artifacts")
            
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            
            st.write(f"Metadata records: {len(metadata_df)}")
            st.write(f"Media records: {len(media_df)}")
            st.write(f"Color records: {len(colors_df)}")
            
            load_to_database(metadata_df, media_df, colors_df)
            st.success("✅ ETL Pipeline completed!")

def show_sql_analytics():
    st.header("🔍 SQL Analytics")
    
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
        "Top Colors Used": """
            SELECT color_name, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
            FROM artifactcolors 
            GROUP BY color_name 
            ORDER BY usage_count DESC 
            LIMIT 10
        """,
        "Media Availability": """
            SELECT 
                CASE WHEN COUNT(m.media_id) > 0 THEN 'Has Media' ELSE 'No Media' END as media_status,
                COUNT(*) as artifact_count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY media_status
        """
    }
    
    selected_query = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        results_df = execute_query(queries[selected_query])
        st.dataframe(results_df)
        
        # Auto-generate visualization
        if len(results_df) > 0:
            visualize_results(results_df, selected_query)

def execute_query(sql):
    """Execute SQL query and return DataFrame"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    df = pd.read_sql(sql, connection)
    connection.close()
    return df

def visualize_results(df, title):
    """Generate visualization based on query results"""
    if len(df.columns) >= 2:
        fig = px.bar(df, x=df.columns[0], y=df.columns[1], title=title)
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Analytical Queries

### Artifact Distribution
```sql
-- Artifacts by department
SELECT department, COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY count DESC;

-- Classification distribution
SELECT classification, COUNT(*) as total
FROM artifactmetadata
GROUP BY classification
ORDER BY total DESC
LIMIT 20;
```

### Media Analysis
```sql
-- Artifacts with multiple images
SELECT a.id, a.title, COUNT(m.media_id) as image_count
FROM artifactmetadata a
JOIN artifactmedia m ON a.id = m.artifact_id
GROUP BY a.id, a.title
HAVING image_count > 5
ORDER BY image_count DESC;
```

### Color Patterns
```sql
-- Most common color combinations
SELECT a.culture, c.color_name, COUNT(*) as usage
FROM artifactmetadata a
JOIN artifactcolors c ON a.id = c.artifact_id
WHERE a.culture IS NOT NULL
GROUP BY a.culture, c.color_name
ORDER BY usage DESC;
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Troubleshooting

**API Rate Limiting**:
```python
import time

def fetch_with_retry(page, max_retries=3):
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page)
        except Exception as e:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise e
```

**Database Connection Issues**:
```python
def test_connection():
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        if connection.is_connected():
            print("✅ Database connection successful")
            connection.close()
    except Error as e:
        print(f"❌ Connection failed: {e}")
```

**Handling Missing Data**:
```python
def safe_get(data, key, default=None):
    """Safely extract nested values"""
    return data.get(key) if data.get(key) else default

# Use in transformation
metadata = {
    'id': artifact.get('id'),
    'title': safe_get(artifact, 'title', 'Unknown'),
    'culture': safe_get(artifact, 'culture', 'Unknown')
}
```

## Best Practices

1. **Batch Processing**: Insert records in batches for better performance
2. **Error Handling**: Wrap API calls and DB operations in try-except blocks
3. **Data Validation**: Check for required fields before insertion
4. **Incremental Updates**: Use `ON DUPLICATE KEY UPDATE` to avoid duplicates
5. **Connection Pooling**: Reuse database connections when possible
6. **Logging**: Track ETL progress and errors for debugging
