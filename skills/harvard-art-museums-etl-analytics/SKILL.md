---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - set up data engineering project with museum artifacts
  - create analytics dashboard for Harvard art collection
  - extract and load Harvard museum data to SQL database
  - build Streamlit app for art museum data visualization
  - implement data pipeline for artifact metadata analysis
  - query and analyze Harvard Art Museums collection data
  - design relational database schema for museum artifacts
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering solution for the Harvard Art Museums API, demonstrating ETL pipelines, SQL analytics, and interactive visualization. It extracts artifact metadata, transforms nested JSON into relational tables, loads data into SQL databases, and presents analytics through a Streamlit dashboard.

## What This Project Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables (metadata, media, colors)
- **SQL Storage**: Stores data in MySQL/TiDB with proper foreign key relationships
- **Analytics Queries**: Provides 20+ predefined SQL queries for artifact analysis
- **Visualization**: Interactive Plotly charts in Streamlit dashboard

Architecture: `API → ETL → SQL → Analytics → Visualization`

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
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://harvardartmuseums.org/collections/api
2. Register for a free API key
3. Add to `.env` file

## Database Schema

The project uses three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    accession_number VARCHAR(100),
    dated VARCHAR(200),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    url TEXT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100):
    """Extract artifacts from Harvard API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    per_page = 100
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': per_page,
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            if len(data.get('records', [])) < per_page:
                break
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### 2. ETL Transform Functions

```python
import pandas as pd

def transform_metadata(artifacts):
    """Transform artifact metadata into DataFrame"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'accession_number': artifact.get('accessionyear', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'medium': artifact.get('medium', 'Unknown'),
            'dimensions': artifact.get('dimensions', 'Unknown'),
            'url': artifact.get('url', '')
        })
    
    return pd.DataFrame(metadata)

def transform_media(artifacts):
    """Extract media/images from artifacts"""
    media = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media.append({
                'artifact_id': artifact_id,
                'image_url': img.get('baseimageurl', ''),
                'media_type': 'image'
            })
    
    return pd.DataFrame(media)

def transform_colors(artifacts):
    """Extract color data from artifacts"""
    colors = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        color_data = artifact.get('colors', [])
        
        for color in color_data:
            colors.append({
                'artifact_id': artifact_id,
                'color': color.get('color', 'Unknown'),
                'percentage': color.get('percent', 0.0)
            })
    
    return pd.DataFrame(colors)
```

### 3. Load Data to SQL

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT', 3306),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata(df_metadata):
    """Batch insert metadata into database"""
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, 
     accession_number, dated, medium, dimensions, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE 
    title=VALUES(title), culture=VALUES(culture)
    """
    
    records = df_metadata.values.tolist()
    
    try:
        cursor.executemany(insert_query, records)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
        return True
    except Error as e:
        print(f"Error inserting metadata: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()

def load_media(df_media):
    """Batch insert media records"""
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, image_url, media_type)
    VALUES (%s, %s, %s)
    """
    
    records = df_media.values.tolist()
    
    try:
        cursor.executemany(insert_query, records)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
        return True
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museum Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Data Pipeline")
    
    # Sidebar for ETL controls
    st.sidebar.header("ETL Pipeline")
    
    num_records = st.sidebar.number_input(
        "Number of records to fetch",
        min_value=10,
        max_value=1000,
        value=100
    )
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(num_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata = transform_metadata(artifacts)
            df_media = transform_media(artifacts)
            df_colors = transform_colors(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            load_metadata(df_metadata)
            load_media(df_media)
            load_colors(df_colors)
            st.success("Data loaded to SQL database")
    
    # Analytics Section
    st.header("📊 Analytics Dashboard")
    
    query_options = {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as count 
            FROM artifactmetadata 
            GROUP BY culture 
            ORDER BY count DESC 
            LIMIT 10
        """,
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count 
            FROM artifactmetadata 
            GROUP BY century 
            ORDER BY count DESC 
            LIMIT 10
        """,
        "Department Distribution": """
            SELECT department, COUNT(*) as count 
            FROM artifactmetadata 
            GROUP BY department 
            ORDER BY count DESC
        """,
        "Color Analysis": """
            SELECT color, SUM(percentage) as total_percentage 
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY total_percentage DESC 
            LIMIT 10
        """,
        "Media Availability": """
            SELECT 
                CASE WHEN EXISTS(
                    SELECT 1 FROM artifactmedia m 
                    WHERE m.artifact_id = a.id
                ) THEN 'Has Media' ELSE 'No Media' END as media_status,
                COUNT(*) as count
            FROM artifactmetadata a
            GROUP BY media_status
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        connection = get_db_connection()
        if connection:
            df_result = pd.read_sql(query_options[selected_query], connection)
            connection.close()
            
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) == 2:
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

## Running the Application

```bash
# Run Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Analytical Queries

### Top Cultures by Artifact Count

```sql
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture != 'Unknown'
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 15;
```

### Artifacts with Most Color Diversity

```sql
SELECT 
    m.title,
    m.culture,
    COUNT(DISTINCT c.color) as color_count
FROM artifactmetadata m
JOIN artifactcolors c ON m.id = c.artifact_id
GROUP BY m.id, m.title, m.culture
ORDER BY color_count DESC
LIMIT 10;
```

### Department-wise Media Availability

```sql
SELECT 
    m.department,
    COUNT(DISTINCT m.id) as total_artifacts,
    COUNT(DISTINCT med.artifact_id) as with_media,
    ROUND(COUNT(DISTINCT med.artifact_id) * 100.0 / COUNT(DISTINCT m.id), 2) as media_percentage
FROM artifactmetadata m
LEFT JOIN artifactmedia med ON m.id = med.artifact_id
GROUP BY m.department
ORDER BY media_percentage DESC;
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_artifacts_with_delay(num_records=100, delay=0.5):
    """Add delay between API calls to avoid rate limiting"""
    artifacts = []
    page = 1
    
    while len(artifacts) < num_records:
        # ... fetch logic ...
        time.sleep(delay)  # Wait between requests
        page += 1
    
    return artifacts
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            connect_timeout=10
        )
        if connection.is_connected():
            print("Database connection successful")
            connection.close()
            return True
    except Error as e:
        print(f"Connection failed: {e}")
        return False
```

### Handling Missing Data

```python
def safe_transform(artifacts):
    """Handle missing or null values during transformation"""
    metadata = []
    
    for artifact in artifacts:
        try:
            metadata.append({
                'id': artifact.get('id', -1),
                'title': artifact.get('title') or 'Untitled',
                'culture': artifact.get('culture') or 'Unknown',
                # Use .get() with defaults for all fields
            })
        except Exception as e:
            print(f"Error processing artifact {artifact.get('id')}: {e}")
            continue
    
    return pd.DataFrame(metadata)
```

## Best Practices

1. **Batch Processing**: Load data in batches for better performance
2. **Error Handling**: Wrap API calls and DB operations in try-except blocks
3. **Data Validation**: Check for null values and data types before loading
4. **Indexing**: Add indexes on frequently queried columns (culture, century, department)
5. **Environment Variables**: Never hardcode credentials
6. **Logging**: Implement logging for production deployments
