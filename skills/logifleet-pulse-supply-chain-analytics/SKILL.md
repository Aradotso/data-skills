---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence platform using MS SQL Server star schema and Power BI for warehouse, fleet, and supply chain analytics
triggers:
  - set up logistics analytics data warehouse
  - create supply chain dashboard with Power BI
  - implement multi-fact star schema for logistics
  - build warehouse and fleet tracking system
  - configure logifleet pulse analytics
  - design cross-modal supply chain reporting
  - implement warehouse gravity zone optimization
  - create real-time fleet telemetry dashboard
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a cohesive analytics engine. Built on MS SQL Server with Power BI visualization, it implements a multi-fact star schema that enables cross-domain KPI analysis (e.g., correlating warehouse dwell time with fleet fuel costs).

**Key capabilities:**
- Multi-fact star schema with time-phased dimensions
- Unified semantic layer linking warehouse, fleet, and supplier data
- Real-time dashboards refreshed every 15 minutes
- Predictive bottleneck detection and fleet maintenance triage
- Warehouse "gravity zone" spatial optimization
- Role-based access control with row-level security

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Admin access to SQL Server instance

### Database Schema Deployment

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance via SSMS
-- Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Execute the main schema script
-- (assumes schema.sql is in the repository)
:r schema.sql
GO
```

3. **Configure data source connections:**
```json
// config.json (create from config_sample.json)
{
  "sql_server": {
    "server": "YOUR_SERVER_NAME",
    "database": "LogiFleetPulse",
    "authentication": "Windows", // or "SQL"
    "username": "${SQL_USERNAME}", // if using SQL auth
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_api_endpoint": "${WMS_API_URL}",
    "wms_api_key": "${WMS_API_KEY}",
    "telematics_endpoint": "${TELEMATICS_API_URL}",
    "telematics_api_key": "${TELEMATICS_API_KEY}",
    "weather_api_key": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

4. **Initialize dimension tables:**
```sql
-- Example: Populate DimTime for 2 years
EXEC sp_PopulateDimTime 
    @StartDate = '2024-01-01', 
    @EndDate = '2025-12-31'
GO

-- Example: Initialize geography hierarchy
INSERT INTO DimGeography (Continent, Country, Region, LocationName, LocationType)
VALUES 
    ('North America', 'USA', 'California', 'LA Warehouse 01', 'Warehouse'),
    ('North America', 'USA', 'California', 'Route LA-SF', 'Route'),
    ('Europe', 'Germany', 'Bavaria', 'Munich Hub', 'Warehouse')
GO
```

### Power BI Setup

1. **Open the template:**
```plaintext
File > Open > LogiFleet_Pulse_Master.pbit
```

2. **Configure data source connection:**
- Enter your SQL Server name and database name
- Choose authentication method (Windows or Database)
- Load data

3. **Set up scheduled refresh (Power BI Service):**
- Publish to workspace
- Configure gateway for on-premises SQL Server
- Set refresh schedule (recommended: every 15-30 minutes)

## Core Data Model

### Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity metrics
CREATE TABLE FactWarehouseOperations (
    OperationID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProduct(ProductKey),
    LocationKey INT FOREIGN KEY REFERENCES DimGeography(LocationKey),
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity DECIMAL(18,2),
    DwellTimeMinutes INT,
    LaborHours DECIMAL(10,2),
    CostUSD DECIMAL(18,2),
    GravityZone VARCHAR(20) -- 'High', 'Medium', 'Low'
)
GO

-- FactFleetTrips: Vehicle and route telemetry
CREATE TABLE FactFleetTrips (
    TripID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    RouteKey INT FOREIGN KEY REFERENCES DimGeography(LocationKey),
    DistanceMiles DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelGallons DECIMAL(10,2),
    FuelCostUSD DECIMAL(10,2),
    LoadWeightLbs DECIMAL(12,2),
    OnTimeDelivery BIT,
    WeatherDelayMinutes INT
)
GO

-- FactCrossDock: Direct transfer operations
CREATE TABLE FactCrossDock (
    CrossDockID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProduct(ProductKey),
    InboundTripID INT FOREIGN KEY REFERENCES FactFleetTrips(TripID),
    OutboundTripID INT FOREIGN KEY REFERENCES FactFleetTrips(TripID),
    TransferTimeMinutes INT,
    Quantity DECIMAL(18,2)
)
GO
```

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME,
    Date DATE,
    Year INT,
    Quarter INT,
    Month INT,
    DayOfMonth INT,
    DayOfWeek INT,
    HourOfDay INT,
    MinuteInterval INT, -- 0, 15, 30, 45
    IsWeekend BIT,
    FiscalYear INT,
    FiscalQuarter INT
)
GO

-- DimProduct: Product hierarchy with gravity scoring
CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50),
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    UnitWeight DECIMAL(10,2),
    IsFragile BIT,
    IsPerishable BIT,
    GravityScore DECIMAL(5,2), -- Calculated metric
    AvgDemandPerDay DECIMAL(10,2),
    UnitValueUSD DECIMAL(10,2)
)
GO

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    LocationKey INT PRIMARY KEY IDENTITY(1,1),
    Continent VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'Route', 'CrossDock'
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
)
GO

-- DimVehicle: Fleet asset information
CREATE TABLE DimVehicle (
    VehicleKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID VARCHAR(50),
    VehicleType VARCHAR(50), -- 'Box Truck', 'Refrigerated', 'Flatbed'
    CapacityLbs DECIMAL(12,2),
    FuelType VARCHAR(50),
    YearManufactured INT,
    MaintenanceScore DECIMAL(5,2),
    CurrentStatus VARCHAR(50) -- 'Active', 'Maintenance', 'Retired'
)
GO
```

## Key SQL Patterns

### Cross-Fact KPI Query: Warehouse Dwell Time vs Fleet Idle Cost

```sql
-- Find products with high warehouse dwell time AND high fleet idle costs
WITH WarehouseDwell AS (
    SELECT 
        p.SKU,
        p.ProductName,
        AVG(w.DwellTimeMinutes) AS AvgDwellMinutes,
        SUM(w.CostUSD) AS TotalWarehouseCost
    FROM FactWarehouseOperations w
    JOIN DimProduct p ON w.ProductKey = p.ProductKey
    JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY p.SKU, p.ProductName
),
FleetIdle AS (
    SELECT 
        p.SKU,
        AVG(f.IdleTimeMinutes) AS AvgIdleMinutes,
        SUM(f.IdleTimeMinutes * 0.85) AS IdleCostUSD -- $0.85 per idle minute
    FROM FactFleetTrips f
    JOIN FactWarehouseOperations w ON f.RouteKey = w.LocationKey
    JOIN DimProduct p ON w.ProductKey = p.ProductKey
    WHERE f.IdleTimeMinutes > 0
    GROUP BY p.SKU
)
SELECT 
    wd.SKU,
    wd.ProductName,
    wd.AvgDwellMinutes,
    fi.AvgIdleMinutes,
    wd.TotalWarehouseCost,
    fi.IdleCostUSD,
    (wd.TotalWarehouseCost + fi.IdleCostUSD) AS TotalLogisticsCost
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.SKU = fi.SKU
WHERE wd.AvgDwellMinutes > 72 AND fi.AvgIdleMinutes > 30
ORDER BY TotalLogisticsCost DESC
GO
```

### Warehouse Gravity Zone Analysis

```sql
-- Calculate and update product gravity scores
UPDATE p
SET p.GravityScore = (
    (p.AvgDemandPerDay * 0.4) + 
    (p.UnitValueUSD * 0.3) + 
    (CASE WHEN p.IsFragile = 1 THEN 20 ELSE 0 END) +
    (CASE WHEN p.IsPerishable = 1 THEN 30 ELSE 0 END)
)
FROM DimProduct p
GO

-- Assign optimal storage zones based on gravity
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    CASE 
        WHEN p.GravityScore >= 80 THEN 'High-Gravity (Near Dock)'
        WHEN p.GravityScore >= 50 THEN 'Medium-Gravity (Mid-Zone)'
        ELSE 'Low-Gravity (Deep Storage)'
    END AS RecommendedZone,
    AVG(w.DwellTimeMinutes) AS CurrentAvgDwell
FROM DimProduct p
LEFT JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
GROUP BY p.SKU, p.ProductName, p.GravityScore
ORDER BY p.GravityScore DESC
GO
```

### Predictive Bottleneck Detection

```sql
-- Identify warehouse operations with risk of bottleneck
CREATE PROCEDURE sp_DetectBottlenecks
    @ThresholdMultiplier DECIMAL(3,2) = 1.5
AS
BEGIN
    WITH BaselineMetrics AS (
        SELECT 
            LocationKey,
            OperationType,
            AVG(DwellTimeMinutes) AS AvgDwell,
            STDEV(DwellTimeMinutes) AS StdDevDwell
        FROM FactWarehouseOperations
        WHERE TimeKey IN (
            SELECT TimeKey FROM DimTime 
            WHERE Date >= DATEADD(DAY, -30, GETDATE())
        )
        GROUP BY LocationKey, OperationType
    ),
    CurrentMetrics AS (
        SELECT 
            LocationKey,
            OperationType,
            AVG(DwellTimeMinutes) AS CurrentAvgDwell
        FROM FactWarehouseOperations
        WHERE TimeKey IN (
            SELECT TimeKey FROM DimTime 
            WHERE Date >= DATEADD(DAY, -1, GETDATE())
        )
        GROUP BY LocationKey, OperationType
    )
    SELECT 
        g.LocationName,
        c.OperationType,
        b.AvgDwell AS HistoricalAvg,
        c.CurrentAvgDwell,
        CASE 
            WHEN c.CurrentAvgDwell > (b.AvgDwell + (b.StdDevDwell * @ThresholdMultiplier))
            THEN 'HIGH RISK'
            WHEN c.CurrentAvgDwell > b.AvgDwell 
            THEN 'ELEVATED'
            ELSE 'NORMAL'
        END AS RiskLevel
    FROM CurrentMetrics c
    INNER JOIN BaselineMetrics b 
        ON c.LocationKey = b.LocationKey 
        AND c.OperationType = b.OperationType
    INNER JOIN DimGeography g ON c.LocationKey = g.LocationKey
    WHERE c.CurrentAvgDwell > b.AvgDwell
    ORDER BY RiskLevel DESC, c.CurrentAvgDwell DESC
END
GO
```

## Data Loading & ETL Patterns

### Incremental Load Stored Procedure

```sql
-- Incremental load from external WMS system
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Stage data from external source
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, LocationKey, OperationType,
        Quantity, DwellTimeMinutes, LaborHours, CostUSD
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        l.LocationKey,
        ext.OperationType,
        ext.Quantity,
        DATEDIFF(MINUTE, ext.StartTime, ext.EndTime) AS DwellTimeMinutes,
        ext.LaborHours,
        ext.LaborHours * 25.00 AS CostUSD -- $25/hour labor rate
    FROM ExternalWMS.dbo.Operations ext
    INNER JOIN DimTime t ON 
        DATEPART(YEAR, ext.StartTime) = t.Year AND
        DATEPART(MONTH, ext.StartTime) = t.Month AND
        DATEPART(DAY, ext.StartTime) = t.DayOfMonth AND
        DATEPART(HOUR, ext.StartTime) = t.HourOfDay AND
        (DATEPART(MINUTE, ext.StartTime) / 15) * 15 = t.MinuteInterval
    INNER JOIN DimProduct p ON ext.SKU = p.SKU
    INNER JOIN DimGeography l ON ext.LocationName = l.LocationName
    WHERE ext.LastModified > @LastLoadDateTime
    
    -- Update last load timestamp
    UPDATE ETLControl
    SET LastLoadDateTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations'
END
GO
```

### Fleet Telemetry Stream Processing

```sql
-- Process streaming telemetry data (called from external integration)
CREATE PROCEDURE sp_ProcessFleetTelemetry
    @TelemetryJSON NVARCHAR(MAX)
AS
BEGIN
    -- Parse JSON telemetry feed
    INSERT INTO FactFleetTrips (
        TimeKey, VehicleKey, RouteKey, DistanceMiles,
        DurationMinutes, IdleTimeMinutes, FuelGallons, FuelCostUSD,
        LoadWeightLbs, OnTimeDelivery
    )
    SELECT 
        t.TimeKey,
        v.VehicleKey,
        r.LocationKey,
        JSON_VALUE(tel.value, '$.distance') AS DistanceMiles,
        JSON_VALUE(tel.value, '$.duration') AS DurationMinutes,
        JSON_VALUE(tel.value, '$.idle_time') AS IdleTimeMinutes,
        JSON_VALUE(tel.value, '$.fuel_consumed') AS FuelGallons,
        JSON_VALUE(tel.value, '$.fuel_consumed') * 3.50 AS FuelCostUSD,
        JSON_VALUE(tel.value, '$.load_weight') AS LoadWeightLbs,
        CAST(JSON_VALUE(tel.value, '$.on_time') AS BIT) AS OnTimeDelivery
    FROM OPENJSON(@TelemetryJSON) tel
    INNER JOIN DimVehicle v ON JSON_VALUE(tel.value, '$.vehicle_id') = v.VehicleID
    INNER JOIN DimGeography r ON JSON_VALUE(tel.value, '$.route') = r.LocationName
    INNER JOIN DimTime t ON 
        CAST(JSON_VALUE(tel.value, '$.timestamp') AS DATETIME) BETWEEN t.FullDateTime AND DATEADD(MINUTE, 15, t.FullDateTime)
END
GO
```

## Power BI DAX Measures

### Cross-Fact Composite KPIs

```dax
// Measure: Total Logistics Cost Per SKU
Total Logistics Cost = 
VAR WarehouseCost = SUM(FactWarehouseOperations[CostUSD])
VAR FleetCost = CALCULATE(
    SUM(FactFleetTrips[FuelCostUSD]),
    USERELATIONSHIP(FactFleetTrips[RouteKey], DimGeography[LocationKey])
)
RETURN WarehouseCost + FleetCost

// Measure: Dwell Time Impact Score
Dwell Impact Score = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR Threshold = 48 // hours converted to minutes
VAR ImpactMultiplier = 
    IF(AvgDwell > Threshold * 60, 
       (AvgDwell - (Threshold * 60)) / 60 * 100,
       0
    )
RETURN ImpactMultiplier

// Measure: Fleet Efficiency Ratio
Fleet Efficiency = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
)

// Measure: Gravity Zone Compliance
Gravity Zone Compliance % = 
VAR HighGravityItems = CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    DimProduct[GravityScore] >= 80
)
VAR HighGravityInOptimalZone = CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    DimProduct[GravityScore] >= 80,
    FactWarehouseOperations[GravityZone] = "High"
)
RETURN DIVIDE(HighGravityInOptimalZone, HighGravityItems, 0)
```

### Time Intelligence Patterns

```dax
// Measure: Rolling 7-Day Average Dwell Time
Dwell 7-Day MA = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATESINPERIOD(
        DimTime[Date],
        LASTDATE(DimTime[Date]),
        -7,
        DAY
    )
)

// Measure: Month-Over-Month Fleet Cost Change
Fleet Cost MoM % = 
VAR CurrentMonth = SUM(FactFleetTrips[FuelCostUSD])
VAR PreviousMonth = CALCULATE(
    SUM(FactFleetTrips[FuelCostUSD]),
    DATEADD(DimTime[Date], -1, MONTH)
)
RETURN DIVIDE(CurrentMonth - PreviousMonth, PreviousMonth, 0)
```

## Configuration & Optimization

### Indexing Strategy

```sql
-- Critical indexes for fact tables
CREATE NONCLUSTERED INDEX IX_WarehouseOps_TimeProduct 
ON FactWarehouseOperations (TimeKey, ProductKey) 
INCLUDE (DwellTimeMinutes, CostUSD)
GO

CREATE NONCLUSTERED INDEX IX_FleetTrips_TimeVehicle 
ON FactFleetTrips (TimeKey, VehicleKey) 
INCLUDE (IdleTimeMinutes, FuelCostUSD)
GO

-- Covering index for cross-fact queries
CREATE NONCLUSTERED INDEX IX_WarehouseOps_Location 
ON FactWarehouseOperations (LocationKey, ProductKey)
INCLUDE (TimeKey, DwellTimeMinutes, CostUSD)
GO
```

### Table Partitioning for Scale

```sql
-- Partition FactWarehouseOperations by month
CREATE PARTITION FUNCTION PF_MonthlyData (INT)
AS RANGE RIGHT FOR VALUES (
    202401, 202402, 202403, 202404, 202405, 202406,
    202407, 202408, 202409, 202410, 202411, 202412
)
GO

CREATE PARTITION SCHEME PS_MonthlyData
AS PARTITION PF_MonthlyData
ALL TO ([PRIMARY])
GO

-- Apply partition scheme
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationID INT IDENTITY(1,1),
    TimeKey INT,
    YearMonth INT, -- Computed column for partitioning
    -- ... other columns
) ON PS_MonthlyData(YearMonth)
GO
```

### Row-Level Security

```sql
-- Create security policy for regional access
CREATE FUNCTION fn_SecurityPredicate(@Region NVARCHAR(100))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS AccessAllowed
WHERE @Region IN (
    SELECT RegionAccess FROM Users WHERE Username = USER_NAME()
)
GO

CREATE SECURITY POLICY RegionalAccessPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(DimGeography.Region) 
ON FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(DimGeography.Region) 
ON FactFleetTrips
WITH (STATE = ON)
GO
```

## Automated Alerting

### Email Alert for Critical Thresholds

```sql
-- Alert when warehouse dwell exceeds 72 hours
CREATE PROCEDURE sp_AlertHighDwell
AS
BEGIN
    DECLARE @HTML NVARCHAR(MAX)
    
    SET @HTML = 
    N'<H3>High Dwell Time Alert</H3>' +
    N'<table border="1">' +
    N'<tr><th>Location</th><th>SKU</th><th>Dwell Hours</th><th>Gravity Zone</th></tr>' +
    CAST((
        SELECT 
            td = g.LocationName, '',
            td = p.SKU, '',
            td = w.DwellTimeMinutes / 60.0, '',
            td = p.GravityScore
        FROM FactWarehouseOperations w
        JOIN DimProduct p ON w.ProductKey = p.ProductKey
        JOIN DimGeography g ON w.LocationKey = g.LocationKey
        WHERE w.DwellTimeMinutes > 4320 -- 72 hours
        AND w.TimeKey IN (
            SELECT TimeKey FROM DimTime 
            WHERE Date >= DATEADD(HOUR, -24, GETDATE())
        )
        ORDER BY w.DwellTimeMinutes DESC
        FOR XML PATH('tr'), TYPE
    ) AS NVARCHAR(MAX)) +
    N'</table>'
    
    IF @HTML IS NOT NULL
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'LogiFleet Alert: High Dwell Time Detected',
            @body = @HTML,
            @body_format = 'HTML'
    END
END
GO

-- Schedule as SQL Agent job (every 4 hours)
```

## Troubleshooting

### Common Issues

**Issue: Power BI refresh fails with "column not found"**
```sql
-- Verify schema matches PBIT expectations
SELECT TABLE_NAME, COLUMN_NAME, DATA_TYPE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME IN ('FactWarehouseOperations', 'FactFleetTrips', 'DimProduct')
ORDER BY TABLE_NAME, ORDINAL_POSITION
GO
```

**Issue: Cross-fact queries are slow**
```sql
-- Check for missing indexes
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    s.avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(
    DB_ID(), NULL, NULL, NULL, 'LIMITED'
) s
JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE s.avg_fragmentation_in_percent > 30
ORDER BY s.avg_fragmentation_in_percent DESC
GO

-- Rebuild fragmented indexes
ALTER INDEX ALL ON FactWarehouseOperations REBUILD
GO
```

**Issue: Time dimension lookup fails**
```sql
-- Verify time dimension coverage
SELECT 
    MIN(Date) AS EarliestDate, 
    MAX(Date) AS LatestDate,
    COUNT(*) AS TotalRecords
FROM DimTime
GO

-- Extend time dimension if needed
EXEC sp_PopulateDimTime 
    @StartDate = '2026-01-01', 
    @EndDate = '2027-12-31'
GO
```

**Issue: Gravity scores not calculating**
```sql
-- Recalculate gravity scores with diagnostic output
UPDATE DimProduct
SET 
    AvgDemandPerDay = (
        SELECT AVG(Quantity) 
        FROM FactWarehouseOperations 
        WHERE ProductKey = DimProduct.ProductKey
    ),
    GravityScore = (
        (COALESCE(AvgDemandPerDay, 0) * 0.4) + 
        (UnitValueUSD * 0.3) + 
        (CASE WHEN IsFragile = 1 THEN 20 ELSE 0 END) +
        (CASE WHEN IsPerishable = 1 THEN 30 ELSE 0 END)
    )
OUTPUT 
    inserted.SKU, 
    inserted.AvgDemandPerDay, 
    inserted.GravityScore
GO
```

## Advanced Patterns

### Temporal Elasticity Simulation

```sql
-- Scenario modeling: What-if warehouse capacity increases
CREATE PROCEDURE sp_SimulateCapacityChange
    @CapacityIncreasePct DECIMAL(5,2),
    @SimulationDays INT = 30
AS
BEGIN
    WITH SimulatedMetrics AS (
        SELECT 
            LocationKey,
            OperationType,
            -- Simulate reduced dwell time
            AVG(DwellTimeMinutes * (1 - (@CapacityIncreasePct / 100))) AS ProjectedDwellMinutes,
            AVG(DwellTimeMinutes) AS CurrentDwellMinutes,
            SUM(CostUSD) AS CurrentCost,
            SUM(CostUSD * (1 - (@CapacityIncreasePct / 200))) AS ProjectedCost
        FROM FactWarehouseOperations
        WHERE TimeKey IN (
            SELECT TimeKey FROM DimTime 
            WHERE Date >= DATEADD(DAY, -@SimulationDays, GETDATE())
        )
        GROUP BY LocationKey, OperationType
    )
    SELECT 
        g.LocationName,
        s.OperationType,
        s.CurrentDwellMinutes,
        s.ProjectedDwellMinutes,
        s.CurrentCost,
        s.ProjectedCost,
        (s.CurrentCost - s.ProjectedCost) AS EstimatedSavings
    FROM SimulatedMetrics s
    JOIN DimGeography g ON s.LocationKey = g.LocationKey
    ORDER BY EstimatedSavings DESC
END
GO
```

### Machine Learning Integration Hook

```sql
-- Export training data for predictive bottleneck ML model
CREATE VIEW vw_ML_BottleneckTraining AS
SELECT 
    w.TimeKey,
    t.DayOfWeek,
    t.HourOfDay,
    g.LocationName,
    w.OperationType,
    p.GravityScore,
    w.DwellTimeMinutes,
    w.Quantity,
    -- Label: 1 if bottleneck, 0 otherwise
    CASE 
        WHEN w.DwellTimeMinutes > (
            SELECT AVG(DwellTimeMinutes) * 1.5 
            FROM FactWarehouseOperations
        ) THEN 1
        ELSE 0
    END AS IsBottleneck
FROM FactWarehouseOperations w
JOIN DimTime t ON w.TimeKey = t.TimeKey
JOIN DimProduct p ON w.ProductKey = p.ProductKey
JOIN DimGeography g ON w.LocationKey = g.LocationKey
WHERE t.Date >= DATEADD(MONTH, -6, GETDATE())
GO
```

This skill provides comprehensive coverage of the LogiFleet Pulse platform's architecture, implementation patterns, and operational procedures for AI coding agents assisting developers in deploying and utilizing this supply chain analytics system.
