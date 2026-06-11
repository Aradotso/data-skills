---
name: car-sales-data-engineering-analytics
description: Process, clean, and analyze car sales data with statistical modeling and interactive Streamlit dashboards for business insights.
triggers:
  - analyze car sales data with statistics
  - build a car sales analytics dashboard
  - process automotive sales dataset
  - create data engineering pipeline for sales
  - run statistical analysis on vehicle sales
  - visualize car dealership revenue trends
  - build streamlit dashboard for sales data
  - perform regression analysis on car prices
---

# Car Sales Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive data engineering and analytics framework for processing ~24K car sales records with ETL pipelines, statistical modeling, and interactive Streamlit dashboards. Provides 15 pre-built analyses covering pricing trends, regional patterns, demographic insights, and feature correlations.

## Installation

This project uses `uv` for package management:

```bash
# Clone the repository
git clone https://github.com/Abdumalik-ProDev/Car-Sales-Data-Engineering.git
cd Car-Sales-Data-Engineering

# Install dependencies
uv sync
```

**Dependencies:** Python 3.10+, pandas, numpy, matplotlib, scipy, streamlit

## Quick Start

### Launch Interactive Dashboard

```bash
# Start Streamlit web UI
uv run streamlit run src/ui.py

# Alternative via entry point
uv run python -m src.main
```

### Run Full Pipeline

```bash
# Execute all 15 analyses and generate figures
uv run python -m src.main --pipeline
```

This will:
- Load and clean `data/Car sales.csv`
- Generate `outputs/cleaned_data.csv`
- Create 15 PNG charts in `outputs/figures/`

## Core Module: `src/analysis.py`

The main analysis engine provides ETL, statistics, and modeling capabilities.

### Data Loading & Cleaning

```python
from src.analysis import CarSalesAnalysis

# Initialize analyzer
analyzer = CarSalesAnalysis('data/Car sales.csv')

# Access cleaned data
df = analyzer.data
print(f"Total records: {len(df)}")
print(f"Columns: {df.columns.tolist()}")

# Save cleaned dataset
analyzer.save_cleaned_data('outputs/cleaned_data.csv')
```

**Key Columns:**
- `car_id`, `date`, `customer_name`, `dealer_name`, `company`, `model`
- `year`, `price`, `body_style`, `transmission`, `color`
- `dealer_no`, `dealer_region`, `phone`, `gender`, `annual_income`

### Statistical Summaries

```python
# Get descriptive statistics
stats = analyzer.describe_data()
print(stats)

# Revenue metrics
total_revenue = analyzer.data['price'].sum()
avg_price = analyzer.data['price'].mean()
median_price = analyzer.data['price'].median()

print(f"Total Revenue: ${total_revenue:,.0f}")
print(f"Avg Price: ${avg_price:,.0f}")
print(f"Median Price: ${median_price:,.0f}")
```

### Generate Individual Analyses

```python
# Q1: Price distribution
analyzer.plot_price_distribution(save_path='outputs/figures/q1_price_dist.png')

# Q2: Monthly sales trend
analyzer.plot_monthly_sales_trend(save_path='outputs/figures/q2_monthly_trend.png')

# Q3: Sales by region
analyzer.plot_sales_by_region(save_path='outputs/figures/q3_regional_sales.png')

# Q6: Income vs Price regression
analyzer.plot_income_vs_price(save_path='outputs/figures/q6_income_price.png')

# Q9: Automatic vs Manual transmission comparison (t-test)
analyzer.compare_transmission_prices(save_path='outputs/figures/q9_transmission.png')
```

### Statistical Modeling

```python
# Q12: Multiple linear regression
# Predicts price from year, annual_income, transmission
analyzer.multiple_regression_analysis(save_path='outputs/figures/q12_regression.png')

# Q13: Detect outliers using Z-scores
analyzer.detect_outliers_zscore(save_path='outputs/figures/q13_outliers.png')

# Q15: Test price normality with Shapiro-Wilk
analyzer.test_normality(save_path='outputs/figures/q15_normality.png')
```

## Streamlit Dashboard (`src/ui.py`)

### Page Structure

The dashboard provides 6 interactive sections:

1. **📊 Overview** - Data summary, sample rows, statistics
2. **💰 Sales & Revenue** - Price trends, regional analysis
3. **👥 Demographics** - Gender, income patterns
4. **🔧 Product Insights** - Brand, body style, transmission
5. **📈 Statistical Modeling** - Regression, outliers, normality
6. **🔍 Filter & Explore** - Custom filters with CSV export
7. **⚖️ Compare Segments** - Side-by-side comparison with t-tests

### Custom Filtering Example

```python
# Users can filter via sidebar widgets
# Example: Filter cars by price range and region

# In ui.py, the filter logic:
filtered = analyzer.data.copy()

if price_range:
    filtered = filtered[
        (filtered['price'] >= price_range[0]) & 
        (filtered['price'] <= price_range[1])
    ]

if selected_regions:
    filtered = filtered[filtered['dealer_region'].isin(selected_regions)]

if selected_companies:
    filtered = filtered[filtered['company'].isin(selected_companies)]

# Display and export
st.dataframe(filtered)
st.download_button(
    "Download CSV",
    filtered.to_csv(index=False),
    "filtered_sales.csv"
)
```

## Common Analysis Patterns

### Price Analysis by Category

```python
# Average price by car company
company_prices = analyzer.data.groupby('company')['price'].mean().sort_values(ascending=False)
print(company_prices.head(10))

# Price by body style
body_prices = analyzer.data.groupby('body_style')['price'].agg(['mean', 'median', 'count'])
print(body_prices)

# Price by transmission type
trans_prices = analyzer.data.groupby('transmission')['price'].describe()
print(trans_prices)
```

### Regional & Temporal Analysis

```python
# Sales volume by region
regional_sales = analyzer.data['dealer_region'].value_counts()
print(regional_sales)

# Monthly revenue trend
analyzer.data['month'] = pd.to_datetime(analyzer.data['date']).dt.to_period('M')
monthly_revenue = analyzer.data.groupby('month')['price'].sum()
print(monthly_revenue)

# Year-over-year comparison
yearly_sales = analyzer.data.groupby('year').agg({
    'price': ['sum', 'mean', 'count']
})
print(yearly_sales)
```

### Statistical Tests

```python
from scipy import stats

# Compare prices: Automatic vs Manual transmission
auto_prices = analyzer.data[analyzer.data['transmission'] == 'Automatic']['price']
manual_prices = analyzer.data[analyzer.data['transmission'] == 'Manual']['price']

t_stat, p_value = stats.ttest_ind(auto_prices, manual_prices)
print(f"T-statistic: {t_stat:.4f}, P-value: {p_value:.4f}")

# Correlation between income and price
correlation = analyzer.data['annual_income'].corr(analyzer.data['price'])
print(f"Income-Price Correlation: {correlation:.4f}")
```

## Configuration

### File Paths

Default paths are defined in `src/analysis.py`:

```python
# Customize data paths
analyzer = CarSalesAnalysis('custom_path/sales_data.csv')

# Custom output directory
analyzer.save_cleaned_data('custom_output/cleaned.csv')

# Figures directory
os.makedirs('custom_figures', exist_ok=True)
analyzer.plot_price_distribution(save_path='custom_figures/prices.png')
```

### Streamlit Configuration

Create `.streamlit/config.toml` for dashboard customization:

```toml
[theme]
primaryColor = "#FF4B4B"
backgroundColor = "#FFFFFF"
secondaryBackgroundColor = "#F0F2F6"
textColor = "#262730"

[server]
port = 8501
headless = true
enableCORS = false
```

## Running Full Pipeline Programmatically

```python
from src.analysis import CarSalesAnalysis
import os

# Initialize
analyzer = CarSalesAnalysis('data/Car sales.csv')

# Create output directories
os.makedirs('outputs/figures', exist_ok=True)

# Save cleaned data
analyzer.save_cleaned_data('outputs/cleaned_data.csv')

# Generate all 15 analyses
analyses = [
    ('q1_price_dist.png', analyzer.plot_price_distribution),
    ('q2_monthly_trend.png', analyzer.plot_monthly_sales_trend),
    ('q3_regional_sales.png', analyzer.plot_sales_by_region),
    ('q4_gender_split.png', analyzer.plot_gender_distribution),
    ('q5_income_region.png', analyzer.plot_income_by_region),
    ('q6_income_price.png', analyzer.plot_income_vs_price),
    ('q7_company_prices.png', analyzer.plot_avg_price_by_company),
    ('q8_body_style.png', analyzer.plot_price_by_body_style),
    ('q9_transmission.png', analyzer.compare_transmission_prices),
    ('q10_colors.png', analyzer.plot_popular_colors),
    ('q11_heatmap.png', analyzer.plot_body_transmission_heatmap),
    ('q12_regression.png', analyzer.multiple_regression_analysis),
    ('q13_outliers.png', analyzer.detect_outliers_zscore),
    ('q14_dealer_prices.png', analyzer.plot_dealer_prices),
    ('q15_normality.png', analyzer.test_normality),
]

for filename, func in analyses:
    func(save_path=f'outputs/figures/{filename}')
    print(f"✓ Generated {filename}")
```

## Troubleshooting

### Missing Data Issues

```python
# Check for missing values
missing = analyzer.data.isnull().sum()
print(missing[missing > 0])

# Handle missing values
analyzer.data = analyzer.data.dropna(subset=['price', 'year'])
analyzer.data['annual_income'].fillna(analyzer.data['annual_income'].median(), inplace=True)
```

### Date Parsing Errors

```python
# Ensure proper date format
analyzer.data['date'] = pd.to_datetime(analyzer.data['date'], errors='coerce')
analyzer.data = analyzer.data.dropna(subset=['date'])
```

### Memory Issues with Large Datasets

```python
# Load only required columns
usecols = ['price', 'company', 'body_style', 'dealer_region', 'year']
df = pd.read_csv('data/Car sales.csv', usecols=usecols)

# Use dtype optimization
df['price'] = df['price'].astype('float32')
df['year'] = df['year'].astype('int16')
```

### Streamlit Port Conflicts

```bash
# Specify custom port
uv run streamlit run src/ui.py --server.port 8502

# Or in config
echo "[server]\nport = 8502" > .streamlit/config.toml
```

## Key Insights Reference

- **Total Records:** 23,906 sales
- **Revenue:** $655.6M total
- **Pricing:** $27,426 avg, $23,000 median
- **Top Body Style:** SUV (27%)
- **Top Region:** Austin (17%)
- **Premium Brand:** Cadillac ($37,557 avg)
- **Demographics:** 79% Male, 21% Female
- **Transmission:** 53% Automatic, 47% Manual
