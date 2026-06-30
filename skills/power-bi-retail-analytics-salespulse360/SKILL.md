---
name: power-bi-retail-analytics-salespulse360
description: Create and customize SalesPulse 360 Power BI dashboards for retail sales analysis, profit tracking, and regional performance monitoring
triggers:
  - build a Power BI retail sales dashboard
  - create sales analytics visualization with regional breakdown
  - set up Power BI dashboard for profit analysis
  - configure SalesPulse 360 dashboard
  - implement Power BI sales performance tracking
  - customize retail analytics dashboard with Power BI
  - analyze global superstore data in Power BI
  - create interactive sales dashboard with forecasting
---

# Power BI Retail Analytics (SalesPulse 360) Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the SalesPulse 360 Power BI retail analytics dashboard project. It provides expertise in setting up, customizing, and extending interactive sales dashboards for retail data analysis, profit tracking, regional performance monitoring, and predictive analytics.

## What This Project Does

SalesPulse 360 is a comprehensive Power BI dashboard template designed for retail analytics. It transforms raw sales data (Global Superstore dataset) into an interactive command center with:

- **Multi-dimensional sales analysis** (region, product, segment, time)
- **Profit architecture mapping** with geographical heatmaps
- **Predictive forecasting** for 12-month future trends
- **Exception alerts** for margin dips and anomaly detection
- **Multilingual support** (English, Spanish, French, German, Mandarin)
- **Responsive design** adaptable to any screen size
- **Automated refresh** capabilities for live data feeds

The dashboard is built as a `.pbix` file (Power BI Desktop format) with pre-configured data models, DAX measures, and visual layouts.

## Installation & Setup

### Prerequisites

```bash
# Required software
# - Power BI Desktop (version 2.120 or higher)
# - Windows 10/11 or Power BI Service access
# - Optional: Git for cloning the repository
```

### Getting Started

1. **Clone the repository:**

```bash
git clone https://github.com/MahbubNibir/power-bi-retail-analytics-viz.git
cd power-bi-retail-analytics-viz
```

2. **Extract the data:**

```bash
# The Global Superstore dataset should be in the repository
# Look for files like: data/GlobalSuperstore.csv or similar
# Extract if compressed
```

3. **Open in Power BI Desktop:**

```bash
# Double-click the .pbix file or
# Open Power BI Desktop → File → Open → select the .pbix file
```

4. **Initial Data Load:**

The first load triggers automatic data transformation. The Power Query Editor will:
- Clean and normalize raw data
- Create calculated columns
- Build date hierarchies
- Establish relationships

## Key Components & Architecture

### Data Model Structure

The dashboard expects data with these core tables:

**Orders (Fact Table):**
- Order ID
- Order Date, Ship Date
- Customer ID, Customer Name
- Segment (Consumer, Corporate, Home Office)
- Country, City, State, Region, Postal Code
- Product ID, Category, Sub-Category, Product Name
- Sales, Quantity, Discount, Profit
- Shipping Cost

**Calendar (Dimension Table):**
- Date
- Year, Quarter, Month, Week, Day
- Fiscal Period (if applicable)

### Essential DAX Measures

Create these measures in Power BI Desktop:

```dax
-- Total Sales
Total Sales = SUM(Orders[Sales])

-- Total Profit
Total Profit = SUM(Orders[Profit])

-- Profit Margin %
Profit Margin = 
DIVIDE(
    [Total Profit],
    [Total Sales],
    0
) * 100

-- Sales Growth (Year over Year)
Sales YoY Growth = 
VAR CurrentYearSales = [Total Sales]
VAR PreviousYearSales = 
    CALCULATE(
        [Total Sales],
        DATEADD('Calendar'[Date], -1, YEAR)
    )
RETURN
    DIVIDE(
        CurrentYearSales - PreviousYearSales,
        PreviousYearSales,
        0
    ) * 100

-- Moving Annual Total (MAT)
Sales MAT = 
CALCULATE(
    [Total Sales],
    DATESINPERIOD(
        'Calendar'[Date],
        LASTDATE('Calendar'[Date]),
        -12,
        MONTH
    )
)

-- Profit Margin Alert Flag
Profit Alert = 
VAR Threshold = 15
VAR CurrentMargin = [Profit Margin]
RETURN
    IF(
        CurrentMargin < Threshold,
        "⚠️ Below Target",
        "✓ On Target"
    )

-- Top N Products by Sales
Top Products Sales = 
CALCULATE(
    [Total Sales],
    TOPN(
        10,
        ALL(Orders[Product Name]),
        [Total Sales],
        DESC
    )
)

-- Customer Retention Index
Retention Index = 
VAR CurrentPeriodCustomers = DISTINCTCOUNT(Orders[Customer ID])
VAR PreviousPeriodCustomers = 
    CALCULATE(
        DISTINCTCOUNT(Orders[Customer ID]),
        DATEADD('Calendar'[Date], -1, QUARTER)
    )
VAR ReturningCustomers = 
    COUNTROWS(
        FILTER(
            VALUES(Orders[Customer ID]),
            CALCULATE(
                COUNTROWS(Orders),
                DATEADD('Calendar'[Date], -1, QUARTER)
            ) > 0
        )
    )
RETURN
    DIVIDE(ReturningCustomers, PreviousPeriodCustomers, 0) * 100
```

## Configuration & Customization

### Adjusting Alert Thresholds

Create a configuration table:

```dax
// In Power Query Editor (M language)
let
    Source = #table(
        {"Parameter", "Value"},
        {
            {"ProfitMarginThreshold", 15},
            {"ReturnRateWarning", 8},
            {"RollingAveragePeriod", 90},
            {"ForecastMonths", 12}
        }
    ),
    ChangedType = Table.TransformColumnTypes(Source,{{"Value", type number}})
in
    ChangedType
```

Reference in DAX:

```dax
Dynamic Profit Alert = 
VAR Threshold = 
    SELECTEDVALUE(
        ConfigTable[Value],
        ConfigTable[Parameter] = "ProfitMarginThreshold"
    )
VAR CurrentMargin = [Profit Margin]
RETURN
    IF(CurrentMargin < Threshold, "⚠️", "✓")
```

### Setting Up Regional Filters

Create a slicer with cascading filters:

```dax
-- Regional Filter Context
Regional Sales = 
CALCULATE(
    [Total Sales],
    ALLSELECTED(Orders[Region])
)

-- Cross-filter measure
Selected Region = 
IF(
    ISFILTERED(Orders[Region]),
    SELECTEDVALUE(Orders[Region], "All Regions"),
    "All Regions"
)
```

### Predictive Forecasting Setup

Power BI has built-in forecasting for line charts:

1. Create a line chart with Date on X-axis and Sales on Y-axis
2. Click on the visualization
3. Analytics pane → Forecast → Add
4. Configure:
   - Forecast length: 12 months
   - Confidence interval: 95%
   - Seasonality: Auto-detect or set to 12 for monthly data

For custom forecasting with DAX:

```dax
-- Simple linear regression forecast
Sales Forecast = 
VAR MaxDate = MAX('Calendar'[Date])
VAR HistoricalData = 
    CALCULATETABLE(
        ADDCOLUMNS(
            'Calendar',
            "Sales", [Total Sales],
            "DateKey", 'Calendar'[Date]
        ),
        'Calendar'[Date] <= MaxDate
    )
VAR Slope = LINESTX(HistoricalData, [Sales], [DateKey])
VAR Intercept = AVERAGEX(HistoricalData, [Sales]) - 
    Slope * AVERAGEX(HistoricalData, [DateKey])
VAR CurrentDate = SELECTEDVALUE('Calendar'[Date])
RETURN
    IF(
        CurrentDate > MaxDate,
        Slope * CurrentDate + Intercept,
        BLANK()
    )
```

## Common Patterns & Use Cases

### Pattern 1: Creating a Regional Performance Dashboard Page

```dax
-- KPI Card Measures
Region Sales Rank = 
RANKX(
    ALL(Orders[Region]),
    [Total Sales],
    ,
    DESC,
    DENSE
)

Region Performance Score = 
VAR SalesScore = [Sales YoY Growth] * 0.4
VAR ProfitScore = [Profit Margin] * 0.3
VAR RetentionScore = [Retention Index] * 0.3
RETURN
    SalesScore + ProfitScore + RetentionScore

-- Traffic Light Indicator
Region Status = 
SWITCH(
    TRUE(),
    [Region Performance Score] >= 80, "🟢 Excellent",
    [Region Performance Score] >= 60, "🟡 Good",
    [Region Performance Score] >= 40, "🟠 Fair",
    "🔴 Needs Attention"
)
```

### Pattern 2: Product Category Deep Dive

```dax
-- Category Contribution %
Category Sales % = 
DIVIDE(
    [Total Sales],
    CALCULATE([Total Sales], ALL(Orders[Category])),
    0
) * 100

-- Pareto Analysis (80/20 rule)
Cumulative Sales % = 
VAR CurrentCategory = SELECTEDVALUE(Orders[Category])
VAR CurrentSales = [Total Sales]
VAR AllCategories = 
    ADDCOLUMNS(
        ALL(Orders[Category]),
        "Sales", [Total Sales]
    )
VAR TopCategories = 
    FILTER(
        AllCategories,
        [Sales] >= CurrentSales
    )
VAR CumulativeSales = SUMX(TopCategories, [Sales])
VAR TotalSales = SUMX(AllCategories, [Sales])
RETURN
    DIVIDE(CumulativeSales, TotalSales, 0) * 100

-- ABC Classification
Product Class = 
VAR CumPct = [Cumulative Sales %]
RETURN
    SWITCH(
        TRUE(),
        CumPct <= 70, "A - High Value",
        CumPct <= 90, "B - Medium Value",
        "C - Low Value"
    )
```

### Pattern 3: Time Intelligence Analysis

```dax
-- Quarter-to-Date (QTD)
Sales QTD = 
TOTALQTD([Total Sales], 'Calendar'[Date])

-- Previous Quarter
Sales PQ = 
CALCULATE(
    [Total Sales],
    PREVIOUSQUARTER('Calendar'[Date])
)

-- Quarter-over-Quarter Growth
Sales QoQ % = 
DIVIDE(
    [Sales QTD] - [Sales PQ],
    [Sales PQ],
    0
) * 100

-- Same Period Last Year
Sales SPLY = 
CALCULATE(
    [Total Sales],
    SAMEPERIODLASTYEAR('Calendar'[Date])
)

-- Year-to-Date vs Last Year
Sales YTD vs LY = 
VAR CurrentYTD = TOTALYTD([Total Sales], 'Calendar'[Date])
VAR LastYearYTD = 
    CALCULATE(
        TOTALYTD([Total Sales], 'Calendar'[Date]),
        SAMEPERIODLASTYEAR('Calendar'[Date])
    )
RETURN
    CurrentYTD - LastYearYTD
```

### Pattern 4: Customer Segmentation Analysis

```dax
-- Customer Lifetime Value
Customer LTV = 
SUMX(
    VALUES(Orders[Customer ID]),
    CALCULATE([Total Sales])
)

-- Customer Frequency
Order Frequency = 
DIVIDE(
    DISTINCTCOUNT(Orders[Order ID]),
    DISTINCTCOUNT(Orders[Customer ID]),
    0
)

-- RFM Scoring (Recency, Frequency, Monetary)
Customer Recency = 
DATEDIFF(
    MAX(Orders[Order Date]),
    TODAY(),
    DAY
)

RFM Score = 
VAR R = [Customer Recency]
VAR F = [Order Frequency]
VAR M = [Customer LTV]
VAR RScore = SWITCH(TRUE(), R <= 30, 5, R <= 90, 4, R <= 180, 3, R <= 365, 2, 1)
VAR FScore = SWITCH(TRUE(), F >= 10, 5, F >= 7, 4, F >= 4, 3, F >= 2, 2, 1)
VAR MScore = SWITCH(TRUE(), M >= 5000, 5, M >= 2000, 4, M >= 1000, 3, M >= 500, 2, 1)
RETURN
    RScore & FScore & MScore
```

## Data Refresh Configuration

### Power Query (M Language) for Data Loading

```m
let
    // Load source data
    Source = Csv.Document(
        File.Contents("C:\Data\GlobalSuperstore.csv"),
        [Delimiter=",", Columns=21, Encoding=65001, QuoteStyle=QuoteStyle.None]
    ),
    
    // Promote headers
    PromotedHeaders = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    
    // Change types
    ChangedType = Table.TransformColumnTypes(
        PromotedHeaders,
        {
            {"Order Date", type date},
            {"Ship Date", type date},
            {"Sales", type number},
            {"Quantity", Int64.Type},
            {"Discount", Percentage.Type},
            {"Profit", type number}
        }
    ),
    
    // Add calculated columns
    AddedMargin = Table.AddColumn(
        ChangedType,
        "Profit Margin",
        each [Profit] / [Sales],
        Percentage.Type
    ),
    
    // Clean data
    RemovedErrors = Table.RemoveRowsWithErrors(AddedMargin),
    ReplacedNull = Table.ReplaceValue(
        RemovedErrors,
        null,
        0,
        Replacer.ReplaceValue,
        {"Discount", "Profit"}
    )
in
    ReplacedNull
```

### Scheduled Refresh Setup (Power BI Service)

Once published to Power BI Service:

1. Navigate to Workspace → Datasets
2. Click on your dataset → Settings
3. Configure Gateway (if using on-premises data)
4. Set refresh schedule:
   ```
   Frequency: Daily
   Time zone: UTC
   Time: 02:00 AM
   ```
5. Enable "Refresh failure notifications"

### Dynamic Data Source Configuration

```m
let
    // Use parameter for flexible data paths
    DataPath = Excel.CurrentWorkbook(){[Name="ConfigPath"]}[Content]{0}[Value],
    
    Source = Csv.Document(
        File.Contents(DataPath),
        [Delimiter=",", Encoding=65001]
    )
in
    Source
```

## Row-Level Security (RLS) Implementation

### Creating Security Roles

In Power BI Desktop:

1. Modeling tab → Manage Roles → New
2. Create role "RegionalManager"
3. Add filter to Orders table:

```dax
[Region] = USERNAME()
```

Or for email-based filtering:

```dax
[Region] = LOOKUPVALUE(
    UserTable[Region],
    UserTable[Email],
    USERPRINCIPALNAME()
)
```

### Testing RLS

```dax
-- Test as different users
-- Modeling tab → View as → Role: RegionalManager
-- Then specify username: "East" or email

-- Dynamic RLS with parameter table
VAR UserRegion = 
    LOOKUPVALUE(
        UserMapping[Region],
        UserMapping[Email],
        USERPRINCIPALNAME()
    )
RETURN
    Orders[Region] = UserRegion || UserRegion = "All"
```

## Multilingual Support

### Creating Translation Tables

```m
// In Power Query
let
    Source = #table(
        {"English", "Spanish", "French", "German", "Mandarin"},
        {
            {"Total Sales", "Ventas Totales", "Ventes Totales", "Gesamtumsatz", "总销售额"},
            {"Profit", "Beneficio", "Bénéfice", "Gewinn", "利润"},
            {"Region", "Región", "Région", "Region", "地区"}
        }
    )
in
    Source
```

### Dynamic Label Generation

```dax
-- Create language selector
Selected Language = SELECTEDVALUE(Language[Language], "English")

-- Dynamic measure label
Sales Label = 
SWITCH(
    [Selected Language],
    "English", "Total Sales",
    "Spanish", "Ventas Totales",
    "French", "Ventes Totales",
    "German", "Gesamtumsatz",
    "Mandarin", "总销售额",
    "Total Sales"
)

-- Use in card visuals or tooltips
Dynamic Sales = 
[Total Sales] & " (" & [Sales Label] & ")"
```

## Troubleshooting

### Common Issues

**Issue: Data refresh fails**
```dax
-- Check data source connection in Power Query
-- Verify file paths are accessible
-- Ensure gateway is online (for cloud refresh)

-- Debug query:
= try 
    Csv.Document(File.Contents(FilePath))
  otherwise 
    #table({"Error"}, {{"File not found"}})
```

**Issue: Relationships not working**
```dax
-- Verify cardinality and cross-filter direction
-- Check for duplicate keys
-- Use RELATED() to test relationships:

Test Relationship = 
RELATED(DimTable[Column])

-- If BLANK(), relationship is broken
```

**Issue: Performance degradation with large datasets**
```dax
-- Reduce cardinality with SUMMARIZE
Aggregated Sales = 
SUMMARIZE(
    Orders,
    Orders[Region],
    Orders[Category],
    "Total", [Total Sales]
)

-- Use variables to avoid recalculation
Optimized Measure = 
VAR CurrentSales = [Total Sales]
VAR PreviousSales = [Sales PQ]
RETURN
    DIVIDE(CurrentSales - PreviousSales, PreviousSales, 0)
```

**Issue: Forecast not appearing**
```
1. Ensure date column is continuous (no gaps)
2. Minimum data points: 2 cycles (24 months for monthly)
3. Check Analytics pane → Forecast is enabled
4. Verify date hierarchy is not expanded
```

## Advanced Customization

### Creating Custom Visuals with Bookmarks

```dax
-- Page navigation measures
Current Page = SELECTEDVALUE(Navigation[Page], "Overview")

Show Detail = 
IF([Current Page] = "Detail", 1, 0)

-- Use in filter to show/hide visuals
```

### Implementing What-If Parameters

In Power BI Desktop:
1. Modeling → New Parameter
2. Name: "Discount Scenario"
3. Min: 0, Max: 0.5, Increment: 0.05

```dax
-- Projected Sales with Discount
Projected Sales = 
[Total Sales] * (1 + [Discount Scenario Value])

-- Impact Analysis
Discount Impact = 
[Projected Sales] - [Total Sales]
```

### Custom Tooltips

Create a separate report page:
1. Set page size to Tooltip
2. Add visuals (e.g., trend line, KPI card)
3. On main page, select visual → Format → Tooltip → Report page

```dax
-- Tooltip-specific measure
Tooltip Detail = 
"Sales: " & FORMAT([Total Sales], "$#,##0") & UNICHAR(10) &
"Growth: " & FORMAT([Sales YoY Growth], "0.0%") & UNICHAR(10) &
"Rank: #" & [Region Sales Rank]
```

## Exporting & Sharing

### Publish to Power BI Service

```bash
# From Power BI Desktop
# File → Publish → Publish to Power BI
# Select workspace → Publish

# Verify publish:
# - Open browser to app.powerbi.com
# - Navigate to workspace
# - Confirm dataset and report appear
```

### Embedding in Web Applications

```html
<!-- Power BI Embed iframe -->
<iframe 
  width="800" 
  height="600" 
  src="https://app.powerbi.com/view?r=REPORT_ID_HERE" 
  frameborder="0" 
  allowFullScreen="true">
</iframe>
```

```javascript
// Using Power BI JavaScript API
const embedConfig = {
    type: 'report',
    tokenType: models.TokenType.Embed,
    accessToken: process.env.POWERBI_EMBED_TOKEN,
    embedUrl: process.env.POWERBI_EMBED_URL,
    id: process.env.POWERBI_REPORT_ID,
    permissions: models.Permissions.Read,
    settings: {
        filterPaneEnabled: false,
        navContentPaneEnabled: true
    }
};

const reportContainer = document.getElementById('reportContainer');
const report = powerbi.embed(reportContainer, embedConfig);
```

## Best Practices

1. **Always use explicit measures** instead of implicit calculations
2. **Create a date table** with CALENDAR() or CALENDARAUTO()
3. **Use variables (VAR)** to improve readability and performance
4. **Test RLS** before deploying to production
5. **Document complex DAX** with comments
6. **Optimize data model**: Remove unused columns, use proper data types
7. **Version control**: Export to PBIX format regularly, commit to Git
8. **Monitor performance**: Use Performance Analyzer in Power BI Desktop

This skill covers the complete lifecycle of working with the SalesPulse 360 Power BI dashboard, from setup through advanced customization and deployment.
