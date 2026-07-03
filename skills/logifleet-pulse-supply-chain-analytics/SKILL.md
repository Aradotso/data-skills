---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for fleet logistics and supply chain analytics with multi-fact star schema
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "create warehouse operations data model"
  - "build fleet management SQL schema"
  - "implement cross-modal supply chain KPIs"
  - "deploy logistics intelligence warehouse"
  - "connect WMS data to Power BI"
  - "optimize warehouse gravity zone queries"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a multi-modal logistics intelligence platform that combines:
- **MS SQL Server data warehouse** with multi-fact star schema for warehouse operations, fleet telemetry, and supply chain metrics
- **Power BI dashboards** for real-time visualization of cross-functional logistics KPIs
- **Semantic layer** that harmonizes warehouse velocity, fleet performance, and inventory metrics
- **Predictive analytics** for bottleneck detection and maintenance prioritization

The platform integrates data from WMS, telematics, supplier portals, weather APIs, and customer orders into a unified analytical layer.

## Installation & Setup

### Prerequisites
- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, TMS, ERP)

### Step 1: Deploy SQL Schema

```sql
-- Create database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateValue DATE NOT NULL,
    Hour TINYINT NOT NULL,
    Minute TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayOfMonth TINYINT NOT NULL,
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT DEFAULT 0,
    TimeSlot DATETIME NOT NULL
)
GO

CREATE NONCLUSTERED INDEX IX_DimTime_DateValue ON DimTime(DateValue)
CREATE NONCLUSTERED INDEX IX_DimTime_TimeSlot ON DimTime(TimeSlot)
GO

CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    Continent VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    City VARCHAR(100),
    FacilityType VARCHAR(50), -- 'Warehouse', 'Distribution Center', 'Route Node'
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    ActiveFrom DATE,
    ActiveTo DATE
)
GO

CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Velocity × Value × (1/Fragility)
    Fragility TINYINT, -- 1-10 scale
    UnitValue DECIMAL(10,2),
    WeightKg DECIMAL(8,3),
    VolumeM3 DECIMAL(8,4),
    IsPerishable BIT DEFAULT 0,
    ShelfLifeDays INT
)
GO

CREATE TABLE DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierCode VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200),
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4), -- Percentage
    ComplianceScore DECIMAL(3,2), -- 0-1 scale
    LastAuditDate DATE
)
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(20), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity INT NOT NULL,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes INT,
    StorageZone VARCHAR(20),
    PickerID VARCHAR(20),
    BatchNumber VARCHAR(50)
)
GO

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey)
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey)
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Operation ON FactWarehouseOperations(OperationType)
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(20) NOT NULL,
    DriverID VARCHAR(20),
    DistanceKm DECIMAL(8,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    TripDurationMinutes INT,
    LoadWeightKg DECIMAL(10,2),
    WeatherCondition VARCHAR(50),
    DelayReasonCode VARCHAR(20)
)
GO

CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey)
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID)
CREATE NONCLUSTERED INDEX IX_FactFleet_Origin ON FactFleetTrips(OriginGeographyKey)
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    Quantity INT NOT NULL,
    DockDwellMinutes INT,
    TransferTimeMinutes INT
)
GO
```

### Step 2: Create Incremental Load Procedures

```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example loading from external WMS table
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType, 
        Quantity, DwellTimeMinutes, ProcessingTimeMinutes, 
        StorageZone, PickerID, BatchNumber
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        wms.operation_type,
        wms.quantity,
        DATEDIFF(MINUTE, wms.start_time, wms.end_time) AS DwellTime,
        wms.processing_minutes,
        wms.storage_zone,
        wms.picker_id,
        wms.batch_number
    FROM ExternalWMS.dbo.operations wms
    INNER JOIN DimTime t ON t.TimeSlot = DATEADD(MINUTE, 
        (DATEDIFF(MINUTE, 0, wms.start_time) / 15) * 15, 0)
    INNER JOIN DimGeography g ON g.FacilityType = 'Warehouse' 
        AND g.City = wms.warehouse_city
    INNER JOIN DimProductGravity p ON p.SKU = wms.sku
    WHERE wms.created_at > @LastLoadTimestamp
    
    RETURN @@ROWCOUNT
END
GO

CREATE PROCEDURE usp_CalculateGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update gravity scores based on recent activity
    UPDATE p
    SET p.GravityScore = (
        (velocity.picks_per_day * 10) * 
        (p.UnitValue / 100) * 
        (11 - p.Fragility) / 10.0
    )
    FROM DimProductGravity p
    INNER JOIN (
        SELECT 
            ProductKey,
            COUNT(*) * 1.0 / DATEDIFF(DAY, MIN(t.DateValue), MAX(t.DateValue)) AS picks_per_day
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE wo.OperationType = 'Picking'
            AND t.DateValue >= DATEADD(DAY, -30, GETDATE())
        GROUP BY ProductKey
    ) velocity ON p.ProductKey = velocity.ProductKey
END
GO
```

### Step 3: Populate Time Dimension

```sql
-- Populate 15-minute granularity time dimension for 2 years
DECLARE @StartDate DATE = '2025-01-01'
DECLARE @EndDate DATE = '2026-12-31'
DECLARE @CurrentDateTime DATETIME = @StartDate

WHILE @CurrentDateTime <= @EndDate
BEGIN
    INSERT INTO DimTime (
        TimeKey, DateValue, Hour, Minute, 
        DayOfWeek, DayOfMonth, FiscalPeriod, 
        IsWeekend, TimeSlot
    )
    VALUES (
        CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT),
        CAST(@CurrentDateTime AS DATE),
        DATEPART(HOUR, @CurrentDateTime),
        DATEPART(MINUTE, @CurrentDateTime),
        DATEPART(WEEKDAY, @CurrentDateTime),
        DATEPART(DAY, @CurrentDateTime),
        CONCAT('FY', YEAR(@CurrentDateTime), '-Q', DATEPART(QUARTER, @CurrentDateTime)),
        CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1,7) THEN 1 ELSE 0 END,
        @CurrentDateTime
    )
    
    SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime)
END
GO
```

### Step 4: Configure Power BI Connection

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Connect using these parameters:

```powerbi
Server: your-sql-server.database.windows.net
Database: LogiFleetPulse
Data Connectivity mode: Import (for smaller datasets) or DirectQuery (for real-time)
```

4. Load all dimension and fact tables
5. Verify relationships in Model view (should be auto-detected)

## Key SQL Queries & Patterns

### Cross-Fact KPI: Dwell Time vs Fleet Idle Cost

```sql
-- Correlate warehouse dwell time with fleet idling costs
SELECT 
    p.Category,
    g.Region,
    AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwellMin,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMin,
    SUM(ft.FuelConsumedLiters * 1.5) AS IdleFuelCostUSD, -- Assuming $1.5/liter
    COUNT(DISTINCT wo.ProductKey) AS UniqueProducts
FROM FactWarehouseOperations wo
INNER JOIN FactFleetTrips ft ON ft.TimeKey = wo.TimeKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.DateValue >= DATEADD(MONTH, -3, GETDATE())
    AND wo.OperationType = 'Shipping'
GROUP BY p.Category, g.Region
HAVING AVG(wo.DwellTimeMinutes) > 72
ORDER BY IdleFuelCostUSD DESC
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products that should be relocated to high-gravity zones
WITH ProductVelocity AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        p.GravityScore,
        wo.StorageZone,
        COUNT(*) AS PickFrequency,
        AVG(wo.ProcessingTimeMinutes) AS AvgPickTime
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE wo.OperationType = 'Picking'
        AND t.DateValue >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.ProductKey, p.SKU, p.ProductName, p.GravityScore, wo.StorageZone
)
SELECT 
    SKU,
    ProductName,
    GravityScore,
    StorageZone AS CurrentZone,
    PickFrequency,
    AvgPickTime,
    CASE 
        WHEN GravityScore > 75 AND StorageZone NOT IN ('A1', 'A2', 'A3') THEN 'Move to High-Gravity Zone'
        WHEN GravityScore < 25 AND StorageZone IN ('A1', 'A2', 'A3') THEN 'Move to Low-Gravity Zone'
        ELSE 'Optimally Placed'
    END AS Recommendation
FROM ProductVelocity
WHERE PickFrequency > 10
ORDER BY GravityScore DESC, PickFrequency DESC
```

### Predictive Bottleneck Detection

```sql
-- Flag potential bottlenecks based on trend analysis
WITH HourlyMetrics AS (
    SELECT 
        t.DateValue,
        t.Hour,
        g.City,
        wo.StorageZone,
        COUNT(*) AS OperationCount,
        AVG(wo.ProcessingTimeMinutes) AS AvgProcessingTime,
        MAX(wo.ProcessingTimeMinutes) AS MaxProcessingTime
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    WHERE t.DateValue >= DATEADD(DAY, -7, GETDATE())
    GROUP BY t.DateValue, t.Hour, g.City, wo.StorageZone
),
Baseline AS (
    SELECT 
        Hour,
        City,
        StorageZone,
        AVG(AvgProcessingTime) AS BaselineProcessingTime,
        STDEV(AvgProcessingTime) AS StdDevProcessingTime
    FROM HourlyMetrics
    GROUP BY Hour, City, StorageZone
)
SELECT 
    hm.DateValue,
    hm.Hour,
    hm.City,
    hm.StorageZone,
    hm.AvgProcessingTime,
    b.BaselineProcessingTime,
    CASE 
        WHEN hm.AvgProcessingTime > b.BaselineProcessingTime + (2 * b.StdDevProcessingTime) 
        THEN 'HIGH RISK'
        WHEN hm.AvgProcessingTime > b.BaselineProcessingTime + b.StdDevProcessingTime 
        THEN 'MODERATE RISK'
        ELSE 'NORMAL'
    END AS BottleneckRisk
FROM HourlyMetrics hm
INNER JOIN Baseline b ON hm.Hour = b.Hour 
    AND hm.City = b.City 
    AND hm.StorageZone = b.StorageZone
WHERE hm.DateValue = CAST(GETDATE() AS DATE)
    AND hm.AvgProcessingTime > b.BaselineProcessingTime + b.StdDevProcessingTime
ORDER BY hm.AvgProcessingTime DESC
```

## Power BI DAX Measures

### Fleet Efficiency Score

```dax
Fleet Efficiency Score = 
VAR TotalDistance = SUM(FactFleetTrips[DistanceKm])
VAR TotalFuel = SUM(FactFleetTrips[FuelConsumedLiters])
VAR FuelEfficiency = DIVIDE(TotalDistance, TotalFuel, 0)
VAR TotalTime = SUM(FactFleetTrips[TripDurationMinutes])
VAR IdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
VAR IdleRatio = DIVIDE(IdleTime, TotalTime, 0)
VAR EfficiencyScore = (FuelEfficiency * 10) * (1 - IdleRatio)
RETURN EfficiencyScore
```

### Warehouse Throughput Rate

```dax
Throughput Rate (Units/Hour) = 
VAR TotalUnits = SUM(FactWarehouseOperations[Quantity])
VAR TotalMinutes = SUM(FactWarehouseOperations[ProcessingTimeMinutes])
VAR TotalHours = DIVIDE(TotalMinutes, 60, 1)
RETURN DIVIDE(TotalUnits, TotalHours, 0)
```

### Cross-Dock Efficiency

```dax
Cross-Dock Efficiency % = 
VAR DirectTransfer = 
    CALCULATE(
        SUM(FactCrossDock[Quantity]),
        FactCrossDock[DockDwellMinutes] < 120
    )
VAR TotalCrossDock = SUM(FactCrossDock[Quantity])
RETURN DIVIDE(DirectTransfer, TotalCrossDock, 0)
```

### Time-Intelligence: Rolling 30-Day Average

```dax
Rolling 30-Day Avg Dwell Time = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATESINPERIOD(DimTime[DateValue], LASTDATE(DimTime[DateValue]), -30, DAY)
)
```

## Configuration Files

### Connection Configuration (JSON)

Create `config/connections.json`:

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "ActiveDirectoryIntegrated",
    "connection_timeout": 30,
    "command_timeout": 600
  },
  "external_sources": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}",
      "refresh_interval_minutes": 15
    },
    "telematics_api": {
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "api_key": "${TELEMATICS_API_KEY}",
      "refresh_interval_minutes": 5
    },
    "weather_api": {
      "endpoint": "https://api.openweathermap.org/data/2.5",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "power_bi": {
    "workspace_id": "${POWERBI_WORKSPACE_ID}",
    "dataset_id": "${POWERBI_DATASET_ID}",
    "refresh_schedule": "0 */15 * * *"
  }
}
```

### Alert Thresholds Configuration

```sql
CREATE TABLE Config_AlertThresholds (
    AlertID INT IDENTITY(1,1) PRIMARY KEY,
    MetricName VARCHAR(100),
    ThresholdType VARCHAR(20), -- 'Absolute', 'Percentage', 'Standard Deviation'
    ThresholdValue DECIMAL(10,2),
    Severity VARCHAR(20), -- 'Critical', 'Warning', 'Info'
    NotificationChannels VARCHAR(200), -- Comma-separated: 'Email,SMS,Teams'
    IsActive BIT DEFAULT 1
)
GO

INSERT INTO Config_AlertThresholds (MetricName, ThresholdType, ThresholdValue, Severity, NotificationChannels)
VALUES 
    ('Fleet Idle Time %', 'Percentage', 15.0, 'Warning', 'Email,Teams'),
    ('Warehouse Dwell Time Hours', 'Absolute', 72.0, 'Critical', 'Email,SMS,Teams'),
    ('Cross-Dock Dwell Minutes', 'Absolute', 120.0, 'Warning', 'Email'),
    ('Fuel Efficiency Km/L', 'Absolute', 5.0, 'Critical', 'Email,Teams'),
    ('Processing Time Standard Deviation', 'Standard Deviation', 2.0, 'Warning', 'Teams')
GO
```

## Data Loading Patterns

### Incremental Load Script (Python Example)

```python
import pyodbc
import os
from datetime import datetime, timedelta

# Connection from environment variables
conn_str = (
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.getenv('SQL_SERVER_HOST')};"
    f"DATABASE=LogiFleetPulse;"
    f"Trusted_Connection=yes;"
)

conn = pyodbc.connect(conn_str)
cursor = conn.cursor()

# Get last load timestamp
cursor.execute("SELECT MAX(TimeSlot) FROM DimTime WHERE TimeKey IN (SELECT TimeKey FROM FactWarehouseOperations)")
last_load = cursor.fetchone()[0] or datetime.now() - timedelta(days=30)

# Call incremental load procedure
cursor.execute("{CALL usp_LoadWarehouseOperations (?)}", last_load)
rows_loaded = cursor.fetchone()[0]

print(f"Loaded {rows_loaded} new warehouse operation records")

# Update gravity scores
cursor.execute("{CALL usp_CalculateGravityScores}")
conn.commit()

cursor.close()
conn.close()
```

### Polybase External Table for Streaming Data

```sql
-- Create external data source for WMS files
CREATE EXTERNAL DATA SOURCE WMS_Files
WITH (
    TYPE = HADOOP,
    LOCATION = 'wasbs://wms-data@yourstorageaccount.blob.core.windows.net'
)
GO

-- Create external file format
CREATE EXTERNAL FILE FORMAT CSV_Format
WITH (
    FORMAT_TYPE = DELIMITEDTEXT,
    FORMAT_OPTIONS (
        FIELD_TERMINATOR = ',',
        STRING_DELIMITER = '"',
        FIRST_ROW = 2,
        USE_TYPE_DEFAULT = FALSE
    )
)
GO

-- Create external table
CREATE EXTERNAL TABLE Staging_WarehouseOps (
    operation_id VARCHAR(50),
    timestamp DATETIME,
    sku VARCHAR(50),
    operation_type VARCHAR(20),
    quantity INT,
    warehouse_city VARCHAR(100),
    storage_zone VARCHAR(20)
)
WITH (
    LOCATION = '/operations/',
    DATA_SOURCE = WMS_Files,
    FILE_FORMAT = CSV_Format
)
GO
```

## Troubleshooting

### Issue: Power BI Refresh Fails

```sql
-- Check for orphaned records in fact tables
SELECT 'Missing Time Keys' AS Issue, COUNT(*) AS Count
FROM FactWarehouseOperations wo
LEFT JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.TimeKey IS NULL

UNION ALL

SELECT 'Missing Geography Keys', COUNT(*)
FROM FactWarehouseOperations wo
LEFT JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
WHERE g.GeographyKey IS NULL

UNION ALL

SELECT 'Missing Product Keys', COUNT(*)
FROM FactWarehouseOperations wo
LEFT JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE p.ProductKey IS NULL
```

### Issue: Slow Cross-Fact Queries

```sql
-- Add covering indexes for common join patterns
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Composite
ON FactWarehouseOperations (TimeKey, GeographyKey, ProductKey)
INCLUDE (Quantity, DwellTimeMinutes, ProcessingTimeMinutes)
GO

-- Enable query store for performance monitoring
ALTER DATABASE LogiFleetPulse SET QUERY_STORE = ON
GO
```

### Issue: Gravity Score Calculation Takes Too Long

```sql
-- Partition fact table by date for faster aggregation
ALTER PARTITION SCHEME PS_Monthly
NEXT USED [PRIMARY]

CREATE PARTITION FUNCTION PF_Monthly (DATE)
AS RANGE RIGHT FOR VALUES (
    '2025-01-01', '2025-02-01', '2025-03-01', '2025-04-01',
    '2025-05-01', '2025-06-01', '2025-07-01', '2025-08-01',
    '2025-09-01', '2025-10-01', '2025-11-01', '2025-12-01'
)
GO

-- Rebuild table with partitioning (requires migration)
-- Use filtered statistics for better query plans
CREATE STATISTICS Stats_Recent_Operations 
ON FactWarehouseOperations (ProductKey, OperationType)
WHERE TimeKey >= CAST(FORMAT(DATEADD(MONTH, -1, GETDATE()), 'yyyyMMddHHmm') AS INT)
GO
```

### Issue: Real-Time Dashboard Lag

Enable DirectQuery mode in Power BI:
1. Model → Connection Settings → DirectQuery
2. Optimize DAX measures to use aggregation tables:

```sql
-- Create aggregation table
CREATE TABLE Agg_HourlyWarehouseMetrics (
    DateHour DATETIME,
    GeographyKey INT,
    ProductKey INT,
    OperationType VARCHAR(20),
    TotalQuantity INT,
    AvgDwellTime DECIMAL(10,2),
    OperationCount INT
)
GO

CREATE CLUSTERED INDEX CIX_Agg ON Agg_HourlyWarehouseMetrics (DateHour)
GO

-- Populate via scheduled job
INSERT INTO Agg_HourlyWarehouseMetrics
SELECT 
    DATEADD(HOUR, DATEDIFF(HOUR, 0, t.TimeSlot), 0) AS DateHour,
    wo.GeographyKey,
    wo.ProductKey,
    wo.OperationType,
    SUM(wo.Quantity) AS TotalQuantity,
    AVG(CAST(wo.DwellTimeMinutes AS DECIMAL(10,2))) AS AvgDwellTime,
    COUNT(*) AS OperationCount
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.TimeSlot >= DATEADD(HOUR, -1, GETDATE())
GROUP BY DATEADD(HOUR, DATEDIFF(HOUR, 0, t.TimeSlot), 0),
         wo.GeographyKey, wo.ProductKey, wo.OperationType
GO
```

## Common Workflow Patterns

### Daily Operational Workflow

1. **Morning**: Check overnight alerts and bottleneck predictions
2. **Mid-day**: Review real-time fleet idle time and cross-dock efficiency
3. **Afternoon**: Run gravity zone optimization query for rearrangement recommendations
4. **Evening**: Generate daily KPI report and schedule next-day routes based on predicted demand

### Monthly Strategic Review

```sql
-- Generate executive summary dataset
SELECT 
    YEAR(t.DateValue) AS Year,
    MONTH(t.DateValue) AS Month,
    -- Warehouse KPIs
    SUM(wo.Quantity) AS TotalUnitsProcessed,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMin,
    COUNT(DISTINCT wo.ProductKey) AS ActiveProducts,
    -- Fleet KPIs
    SUM(ft.DistanceKm) AS TotalDistanceDriven,
    SUM(ft.FuelConsumedLiters) AS TotalFuelConsumed,
    AVG(ft.IdleTimeMinutes * 100.0 / NULLIF(ft.TripDurationMinutes, 0)) AS AvgIdleTimePct,
    -- Cross-Dock KPIs
    SUM(cd.Quantity) AS CrossDockUnits,
    AVG(cd.DockDwellMinutes) AS AvgCrossDockDwellMin,
    -- Financial
    SUM(ft.FuelConsumedLiters * 1.5) AS FuelCostUSD,
    SUM(wo.ProcessingTimeMinutes * 25.0 / 60) AS LaborCostUSD -- $25/hour
FROM DimTime t
LEFT JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.TimeKey
LEFT JOIN FactCrossDock cd ON t.TimeKey = cd.TimeKey
WHERE t.DateValue >= DATEADD(MONTH, -6, GETDATE())
GROUP BY YEAR(t.DateValue), MONTH(t.DateValue)
ORDER BY Year DESC, Month DESC
GO
```

This skill provides comprehensive guidance for deploying and using LogiFleet Pulse's SQL Server data warehouse and Power BI analytics for supply chain optimization.
