---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform for warehouse operations, fleet tracking, and cross-modal supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure warehouse and fleet data model"
  - "create Power BI logistics dashboard"
  - "deploy SQL Server supply chain schema"
  - "implement cross-fact KPI harmonization"
  - "build real-time fleet optimization reports"
  - "connect warehouse management system to analytics"
  - "set up logistics bottleneck detection"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive **multi-modal logistics intelligence engine** that unifies:

- **Warehouse operations analytics** (receiving, putaway, picking, packing, shipping, dwell time)
- **Fleet telemetry and GPS tracking** (routes, fuel consumption, idle time, maintenance)
- **Inventory management** (turnover, aging curves, gravity zone optimization)
- **Cross-dock operations** (transfer efficiency between inbound/outbound)
- **Predictive bottleneck detection** (ML-powered congestion forecasting)

Built on **MS SQL Server** for data warehousing and **Power BI** for visualization, it uses a custom multi-fact star schema with time-phased dimensions to enable cross-functional KPI analysis (e.g., "How does warehouse dwell time correlate with fleet idle costs?").

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telematics APIs, or sample data files

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    FullDateTime DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeSlot15Min TIME NOT NULL,
    HourOfDay INT NOT NULL,
    DayOfWeek VARCHAR(10) NOT NULL,
    FiscalPeriod VARCHAR(10) NOT NULL,
    IsWeekend BIT NOT NULL
);
CREATE CLUSTERED INDEX IX_DimTime_DateTime ON DimTime(FullDateTime);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    Continent VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    City VARCHAR(100),
    WarehouseCode VARCHAR(20),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimezoneOffset INT
);

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Higher = faster moving
    ValuePerUnit DECIMAL(10,2),
    FragilityIndex INT, -- 1-10 scale
    TemperatureZone VARCHAR(20) -- Ambient, Chilled, Frozen
);
CREATE UNIQUE INDEX IX_DimProduct_SKU ON DimProductGravity(SKU);

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode VARCHAR(50) NOT NULL,
    SupplierName VARCHAR(200),
    LeadTimeDays INT,
    LeadTimeVarianceDays DECIMAL(5,2),
    DefectPercentage DECIMAL(5,2),
    ComplianceScore DECIMAL(3,2), -- 0-1 scale
    LastAuditDate DATE
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    QuantityHandled INT,
    DwellTimeMinutes INT,
    PickRateUnitsPerHour DECIMAL(8,2),
    LabourCost DECIMAL(10,2),
    StorageZone VARCHAR(20),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    DistanceKM DECIMAL(10,2),
    FuelConsumedLitres DECIMAL(8,2),
    IdleTimeMinutes INT,
    TripDurationMinutes INT,
    LoadWeightKG DECIMAL(10,2),
    DeliveryStatus VARCHAR(20), -- OnTime, Delayed, Cancelled
    DelayReasonCode VARCHAR(50),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);
CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    TransferTimeMinutes INT,
    UnitsTransferred INT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);
```

### Step 2: Create Sample Data (Testing)

```sql
-- Populate DimTime with 15-minute intervals for 2026
DECLARE @StartDate DATETIME = '2026-01-01 00:00:00';
DECLARE @EndDate DATETIME = '2026-12-31 23:45:00';

WHILE @StartDate <= @EndDate
BEGIN
    INSERT INTO DimTime (FullDateTime, DateKey, TimeSlot15Min, HourOfDay, DayOfWeek, FiscalPeriod, IsWeekend)
    VALUES (
        @StartDate,
        CAST(FORMAT(@StartDate, 'yyyyMMdd') AS INT),
        CAST(@StartDate AS TIME),
        DATEPART(HOUR, @StartDate),
        DATENAME(WEEKDAY, @StartDate),
        'Q' + CAST(DATEPART(QUARTER, @StartDate) AS VARCHAR(1)) + '-' + CAST(YEAR(@StartDate) AS VARCHAR(4)),
        CASE WHEN DATEPART(WEEKDAY, @StartDate) IN (1,7) THEN 1 ELSE 0 END
    );
    SET @StartDate = DATEADD(MINUTE, 15, @StartDate);
END;

-- Sample geography data
INSERT INTO DimGeography (Continent, Country, Region, City, WarehouseCode, Latitude, Longitude, TimezoneOffset)
VALUES 
    ('North America', 'USA', 'West', 'Los Angeles', 'WH-LAX-01', 34.0522, -118.2437, -8),
    ('North America', 'USA', 'East', 'New York', 'WH-NYC-01', 40.7128, -74.0060, -5),
    ('Europe', 'Germany', 'Central', 'Berlin', 'WH-BER-01', 52.5200, 13.4050, 1);

-- Sample product data with gravity scores
INSERT INTO DimProductGravity (SKU, ProductName, Category, Subcategory, GravityScore, ValuePerUnit, FragilityIndex, TemperatureZone)
VALUES
    ('SKU-FAST-001', 'High-Velocity Widget', 'Electronics', 'Mobile', 9.5, 299.99, 7, 'Ambient'),
    ('SKU-SLOW-002', 'Bulk Cardboard', 'Packaging', 'Consumables', 2.3, 0.50, 2, 'Ambient'),
    ('SKU-PERI-003', 'Fresh Produce Box', 'Food', 'Perishables', 8.1, 45.00, 9, 'Chilled');
```

### Step 3: Connect Power BI

1. Open Power BI Desktop
2. **Get Data** → **SQL Server**
3. Enter your server name and database: `LogiFleetPulse`
4. Authentication: Windows or SQL Server credentials (use env vars for connection strings)
5. Select tables: All `Fact*` and `Dim*` tables
6. Power BI will auto-detect relationships based on foreign keys

## Key SQL Queries & Stored Procedures

### Cross-Fact KPI: Dwell Time vs Fleet Idle Cost

```sql
-- Correlate warehouse dwell time with fleet idle time per product
CREATE PROCEDURE sp_GetDwellVsFleetCorrelation
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SELECT 
        p.ProductName,
        p.Category,
        AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwellMin,
        AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMin,
        SUM(ft.FuelConsumedLitres) AS TotalFuelWasted,
        COUNT(DISTINCT ft.TripKey) AS TotalTrips
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimTime t1 ON wo.TimeKey = t1.TimeKey
    INNER JOIN FactFleetTrips ft ON ft.TimeKey = wo.TimeKey -- Same time bucket correlation
    INNER JOIN DimTime t2 ON ft.TimeKey = t2.TimeKey
    WHERE t1.FullDateTime BETWEEN @StartDate AND @EndDate
      AND wo.OperationType = 'Shipping'
    GROUP BY p.ProductName, p.Category
    ORDER BY AvgWarehouseDwellMin DESC;
END;
GO

-- Execute
EXEC sp_GetDwellVsFleetCorrelation '2026-06-01', '2026-06-30';
```

### Predictive Bottleneck Index

```sql
-- Identify high-risk time slots based on historical congestion patterns
CREATE VIEW vw_BottleneckRiskIndex AS
SELECT 
    t.HourOfDay,
    t.DayOfWeek,
    COUNT(DISTINCT wo.OperationKey) AS WarehouseOperations,
    COUNT(DISTINCT ft.TripKey) AS FleetTrips,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    -- Risk score: higher when both dwell and idle are above median
    CASE 
        WHEN AVG(wo.DwellTimeMinutes) > (SELECT AVG(DwellTimeMinutes) FROM FactWarehouseOperations)
         AND AVG(ft.IdleTimeMinutes) > (SELECT AVG(IdleTimeMinutes) FROM FactFleetTrips)
        THEN 'HIGH'
        WHEN AVG(wo.DwellTimeMinutes) > (SELECT AVG(DwellTimeMinutes) FROM FactWarehouseOperations)
          OR AVG(ft.IdleTimeMinutes) > (SELECT AVG(IdleTimeMinutes) FROM FactFleetTrips)
        THEN 'MEDIUM'
        ELSE 'LOW'
    END AS BottleneckRisk
FROM DimTime t
LEFT JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.TimeKey
GROUP BY t.HourOfDay, t.DayOfWeek;
GO

SELECT * FROM vw_BottleneckRiskIndex WHERE BottleneckRisk = 'HIGH';
```

### Automated Alert Stored Procedure

```sql
-- Alert when fleet idling exceeds threshold
CREATE PROCEDURE sp_CheckFleetIdleAlert
    @IdleThresholdPercent DECIMAL(5,2) = 15.0,
    @EmailRecipient VARCHAR(200) = NULL
AS
BEGIN
    DECLARE @AlertMessage VARCHAR(MAX);
    
    SELECT 
        ft.VehicleID,
        ft.TripDurationMinutes,
        ft.IdleTimeMinutes,
        (ft.IdleTimeMinutes * 100.0 / NULLIF(ft.TripDurationMinutes, 0)) AS IdlePercent
    INTO #HighIdleTrips
    FROM FactFleetTrips ft
    WHERE (ft.IdleTimeMinutes * 100.0 / NULLIF(ft.TripDurationMinutes, 0)) > @IdleThresholdPercent
      AND CAST(ft.TimeKey AS DATE) = CAST(GETDATE() AS DATE); -- Today only
    
    IF EXISTS (SELECT 1 FROM #HighIdleTrips)
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet vehicles exceeding ' + CAST(@IdleThresholdPercent AS VARCHAR) + '% idle time threshold detected today.';
        
        -- Log to audit table (create if needed)
        -- In production, integrate with email service or Teams webhook
        PRINT @AlertMessage;
        
        SELECT * FROM #HighIdleTrips;
    END
    ELSE
    BEGIN
        PRINT 'No high idle time alerts for today.';
    END
    
    DROP TABLE #HighIdleTrips;
END;
GO

-- Schedule this via SQL Server Agent job to run every hour
EXEC sp_CheckFleetIdleAlert @IdleThresholdPercent = 12.0;
```

## Power BI Configuration

### DAX Measures for Cross-Fact Analysis

```dax
-- Composite KPI: Warehouse Efficiency x Fleet Utilization
WarehouseFleetEfficiency = 
VAR WarehouseEff = 
    DIVIDE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        SUM(FactWarehouseOperations[DwellTimeMinutes]),
        0
    )
VAR FleetUtil = 
    DIVIDE(
        SUM(FactFleetTrips[TripDurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
        SUM(FactFleetTrips[TripDurationMinutes]),
        0
    )
RETURN WarehouseEff * FleetUtil * 100

-- Gravity Zone Compliance
GravityZoneCompliance = 
CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FILTER(
        FactWarehouseOperations,
        (FactWarehouseOperations[StorageZone] = "High-Gravity" && 
         RELATED(DimProductGravity[GravityScore]) >= 7) ||
        (FactWarehouseOperations[StorageZone] = "Low-Gravity" && 
         RELATED(DimProductGravity[GravityScore]) < 7)
    )
) / COUNTROWS(FactWarehouseOperations)

-- Fuel Cost Per Delivery (requires price parameter)
FuelCostPerDelivery = 
VAR FuelPricePerLitre = 1.85 -- Parameterize in production
RETURN
DIVIDE(
    SUM(FactFleetTrips[FuelConsumedLitres]) * FuelPricePerLitre,
    COUNTROWS(FactFleetTrips),
    0
)

-- Predictive Dwell Time Trend (moving average)
DwellTimeTrend_7Day = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATESINPERIOD(
        DimTime[FullDateTime],
        LASTDATE(DimTime[FullDateTime]),
        -7,
        DAY
    )
)
```

### Role-Based Security (RLS)

```dax
-- In Power BI, Manage Roles → Create role "RegionalManager_West"
-- DAX filter on DimGeography:
[Region] = "West"

-- Create role "SupplierView_SpecificSupplier"
-- DAX filter on DimSupplierReliability:
[SupplierCode] = USERNAME()
```

## Common Patterns

### Pattern 1: Time-Phased Dimension Query

```sql
-- Analyze operations by 15-minute time buckets
SELECT 
    t.HourOfDay,
    t.TimeSlot15Min,
    t.DayOfWeek,
    COUNT(*) AS OperationCount,
    AVG(wo.PickRateUnitsPerHour) AS AvgPickRate
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.DateKey BETWEEN 20260601 AND 20260630
  AND wo.OperationType = 'Picking'
GROUP BY t.HourOfDay, t.TimeSlot15Min, t.DayOfWeek
ORDER BY t.DayOfWeek, t.HourOfDay, t.TimeSlot15Min;
```

### Pattern 2: Gravity Zone Optimization Query

```sql
-- Identify products in wrong gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    wo.StorageZone,
    COUNT(*) AS PickCount,
    AVG(wo.DwellTimeMinutes) AS AvgDwell,
    CASE 
        WHEN p.GravityScore >= 7 AND wo.StorageZone NOT LIKE '%High-Gravity%' THEN 'MOVE TO HIGH'
        WHEN p.GravityScore < 4 AND wo.StorageZone NOT LIKE '%Low-Gravity%' THEN 'MOVE TO LOW'
        ELSE 'OPTIMAL'
    END AS Recommendation
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationType = 'Picking'
GROUP BY p.SKU, p.ProductName, p.GravityScore, wo.StorageZone
HAVING COUNT(*) > 50 -- Only for frequently picked items
ORDER BY AvgDwell DESC;
```

### Pattern 3: Cross-Dock Efficiency

```sql
-- Measure cross-dock speed (inbound to outbound transfer time)
SELECT 
    g.WarehouseCode,
    p.Category,
    AVG(cd.TransferTimeMinutes) AS AvgTransferTime,
    SUM(cd.UnitsTransferred) AS TotalUnits,
    COUNT(*) AS TransferEvents
FROM FactCrossDock cd
INNER JOIN DimGeography g ON cd.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON cd.ProductKey = p.ProductKey
GROUP BY g.WarehouseCode, p.Category
HAVING AVG(cd.TransferTimeMinutes) > 30 -- Flag slow transfers
ORDER BY AvgTransferTime DESC;
```

## Configuration Files

### config.json (Connection Strings)

```json
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "ActiveDirectoryIntegrated",
    "timeout": 30
  },
  "dataRefresh": {
    "intervalMinutes": 15,
    "batchSize": 10000
  },
  "externalAPIs": {
    "weatherService": {
      "endpoint": "https://api.weatherservice.example/v2",
      "apiKey": "${WEATHER_API_KEY}"
    },
    "telematicsProvider": {
      "endpoint": "https://fleet-telemetry.example/api",
      "apiKey": "${TELEMATICS_API_KEY}"
    }
  },
  "alerting": {
    "emailService": "SendGrid",
    "apiKey": "${SENDGRID_API_KEY}",
    "defaultRecipients": ["logistics-ops@company.com"]
  }
}
```

### Power BI Incremental Refresh Policy

In Power BI Desktop:

1. Create parameters `RangeStart` and `RangeEnd` (DateTime type)
2. Filter `FactWarehouseOperations` and `FactFleetTrips` tables:
   ```m
   = Table.SelectRows(Source, each [FullDateTime] >= RangeStart and [FullDateTime] < RangeEnd)
   ```
3. Enable incremental refresh: Store 2 years, refresh last 30 days

## Troubleshooting

### Issue: Power BI Relationship Detection Fails

**Cause**: Foreign key constraints missing or incorrect data types

**Solution**:
```sql
-- Verify relationships
SELECT 
    fk.name AS ForeignKeyName,
    tp.name AS ParentTable,
    cp.name AS ParentColumn,
    tr.name AS ReferencedTable,
    cr.name AS ReferencedColumn
FROM sys.foreign_keys AS fk
INNER JOIN sys.tables AS tp ON fk.parent_object_id = tp.object_id
INNER JOIN sys.tables AS tr ON fk.referenced_object_id = tr.object_id
INNER JOIN sys.foreign_key_columns AS fkc ON fk.object_id = fkc.constraint_object_id
INNER JOIN sys.columns AS cp ON fkc.parent_object_id = cp.object_id AND fkc.parent_column_id = cp.column_id
INNER JOIN sys.columns AS cr ON fkc.referenced_object_id = cr.object_id AND fkc.referenced_column_id = cr.column_id
WHERE tp.name LIKE 'Fact%' OR tr.name LIKE 'Dim%';

-- Manually recreate if needed
ALTER TABLE FactWarehouseOperations
ADD CONSTRAINT FK_Warehouse_Product
FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey);
```

### Issue: DAX Measure Returns BLANK

**Cause**: Missing context or filter propagation issue

**Solution**:
```dax
-- Use CALCULATE to override filters
CorrectedMeasure = 
CALCULATE(
    SUM(FactWarehouseOperations[QuantityHandled]),
    REMOVEFILTERS(DimTime[IsWeekend]) -- Remove specific filter
)

-- Debug with HASONEVALUE
DebugMeasure = 
IF(
    HASONEVALUE(DimProductGravity[SKU]),
    SUM(FactWarehouseOperations[QuantityHandled]),
    BLANK()
)
```

### Issue: Slow Query Performance

**Cause**: Missing indexes or table scans

**Solution**:
```sql
-- Create covering indexes for frequent queries
CREATE NONCLUSTERED INDEX IX_Warehouse_Composite
ON FactWarehouseOperations(TimeKey, ProductKey, OperationType)
INCLUDE (QuantityHandled, DwellTimeMinutes);

-- Check execution plan
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- Your query here
SELECT ...

-- Review missing index recommendations
SELECT 
    migs.avg_user_impact,
    mid.statement,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns
FROM sys.dm_db_missing_index_group_stats AS migs
INNER JOIN sys.dm_db_missing_index_groups AS mig ON migs.group_handle = mig.index_group_handle
INNER JOIN sys.dm_db_missing_index_details AS mid ON mig.index_handle = mid.index_handle
WHERE migs.avg_user_impact > 50
ORDER BY migs.avg_user_impact DESC;
```

### Issue: Power BI Refresh Fails with Timeout

**Cause**: Large dataset or inefficient queries

**Solution**:
1. Implement partitioning:
```sql
-- Partition large fact tables by month
CREATE PARTITION FUNCTION pf_MonthlyPartition (INT)
AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301, ...);

CREATE PARTITION SCHEME ps_MonthlyPartition
AS PARTITION pf_MonthlyPartition ALL TO ([PRIMARY]);

-- Rebuild table on partition scheme
CREATE TABLE FactWarehouseOperations_Partitioned (
    -- columns same as original
) ON ps_MonthlyPartition(TimeKey);
```

2. Use aggregated views for Power BI:
```sql
CREATE VIEW vw_WarehouseOperations_Daily AS
SELECT 
    CAST(t.FullDateTime AS DATE) AS OperationDate,
    wo.ProductKey,
    wo.GeographyKey,
    wo.OperationType,
    SUM(wo.QuantityHandled) AS TotalQuantity,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
GROUP BY CAST(t.FullDateTime AS DATE), wo.ProductKey, wo.GeographyKey, wo.OperationType;
```

## Advanced: External Data Integration

### Python Script for Weather API Enrichment

```python
import pyodbc
import requests
import os
from datetime import datetime

# Connection using environment variables
conn_str = (
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.getenv('SQL_SERVER_HOST')};"
    f"DATABASE=LogiFleetPulse;"
    f"Trusted_Connection=yes;"
)

conn = pyodbc.connect(conn_str)
cursor = conn.cursor()

# Fetch trips needing weather data
cursor.execute("""
    SELECT TripKey, TimeKey, OriginGeographyKey 
    FROM FactFleetTrips 
    WHERE WeatherCondition IS NULL
    LIMIT 100
""")

weather_api_key = os.getenv('WEATHER_API_KEY')

for row in cursor.fetchall():
    trip_key, time_key, geo_key = row
    
    # Get coordinates from geography dimension
    cursor.execute("SELECT Latitude, Longitude FROM DimGeography WHERE GeographyKey = ?", geo_key)
    lat, lon = cursor.fetchone()
    
    # Call weather API (example)
    response = requests.get(
        f"https://api.weatherservice.example/v2/history",
        params={
            'lat': lat,
            'lon': lon,
            'apikey': weather_api_key,
            'time': time_key  # Convert to ISO format in production
        }
    )
    
    if response.ok:
        weather_data = response.json()
        cursor.execute("""
            UPDATE FactFleetTrips 
            SET WeatherCondition = ?, Temperature = ?
            WHERE TripKey = ?
        """, weather_data['condition'], weather_data['temp_c'], trip_key)

conn.commit()
conn.close()
```

## Production Deployment Checklist

- [ ] Configure SQL Server backup schedule (daily full, hourly differential)
- [ ] Set up Power BI Premium capacity or Pro licenses for all users
- [ ] Implement row-level security based on organizational hierarchy
- [ ] Create SQL Server Agent jobs for incremental data loads
- [ ] Configure alert email delivery (SendGrid, SMTP, or Teams webhook)
- [ ] Document data lineage and transformation logic
- [ ] Set up monitoring for query performance (Extended Events or Query Store)
- [ ] Establish data retention policies (archive data older than 2 years)
- [ ] Test disaster recovery procedures
- [ ] Train end users on dashboard navigation and self-service capabilities
