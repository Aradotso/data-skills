---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform for warehouse operations, fleet management, and multi-fact supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy warehouse and fleet logistics dashboard"
  - "configure multi-fact star schema for logistics"
  - "create Power BI supply chain KPIs"
  - "implement warehouse gravity zone optimization"
  - "build cross-modal logistics analytics"
  - "integrate fleet telemetry with warehouse data"
  - "setup real-time logistics intelligence dashboard"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that unifies warehouse operations, fleet management, and supply chain analytics into a single semantic layer. It uses MS SQL Server for data warehousing with a multi-fact star schema architecture, and Power BI for visualization and reporting.

**Core capabilities:**
- Multi-fact star schema linking warehouse operations, fleet telemetry, and supply chain events
- Time-phased dimensions with 15-minute granularity
- Cross-fact KPI harmonization (e.g., inventory dwell time vs. fleet idle costs)
- Warehouse gravity zone optimization based on pick frequency and item value
- Predictive bottleneck detection using historical patterns
- Role-based access control with row-level security
- Real-time dashboards with automated alerting

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Appropriate permissions to create databases and tables

### Step 1: Deploy SQL Schema

```sql
-- Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create time dimension with 15-minute granularity
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    FullDateTime DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeSlot15Min VARCHAR(5), -- e.g., "14:45"
    HourOfDay INT,
    DayOfWeek INT,
    DayName VARCHAR(20),
    WeekOfYear INT,
    MonthNumber INT,
    MonthName VARCHAR(20),
    QuarterNumber INT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT
)
GO

CREATE CLUSTERED COLUMNSTORE INDEX CCI_DimTime ON DimTime
GO

-- Create geography dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationCode VARCHAR(50) NOT NULL UNIQUE,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'Route Node', 'Cross-Dock'
    Address VARCHAR(500),
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50),
    IsActive BIT DEFAULT 1
)
GO

CREATE NONCLUSTERED INDEX IX_DimGeography_Type ON DimGeography(LocationType, IsActive)
GO

-- Create product gravity dimension
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(500),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    Fragility VARCHAR(20), -- 'Low', 'Medium', 'High'
    ValuePerUnit DECIMAL(12,2),
    VelocityScore DECIMAL(5,2), -- Picks per day average
    GravityScore DECIMAL(5,2), -- Composite score: velocity * value / fragility
    OptimalZone VARCHAR(50), -- 'High-Gravity', 'Medium-Gravity', 'Low-Gravity'
    LastRecalculated DATETIME
)
GO

CREATE NONCLUSTERED INDEX IX_DimProductGravity_Zone ON DimProductGravity(OptimalZone, GravityScore DESC)
GO

-- Create warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OrderID VARCHAR(100),
    BatchID VARCHAR(100),
    QuantityHandled INT,
    DwellTimeMinutes INT,
    CycleTimeSeconds INT,
    WorkerID VARCHAR(50),
    ZoneAssigned VARCHAR(50),
    ActualZone VARCHAR(50),
    ExceptionFlag BIT DEFAULT 0,
    ExceptionReason VARCHAR(500)
)
GO

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey, OperationType)
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey, DwellTimeMinutes)
GO

-- Create fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TripID VARCHAR(100) NOT NULL,
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    StartTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    EndTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    PlannedDistanceKm DECIMAL(10,2),
    ActualDistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DelayMinutes INT,
    DelayReason VARCHAR(500),
    AvgSpeedKmh DECIMAL(6,2),
    MaxSpeedKmh DECIMAL(6,2),
    HarshBrakingCount INT,
    HarshAccelerationCount INT,
    LoadWeightKg DECIMAL(10,2),
    RevenueValue DECIMAL(12,2)
)
GO

CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(StartTimeKey, VehicleID)
CREATE NONCLUSTERED INDEX IX_FactFleet_Route ON FactFleetTrips(OriginGeographyKey, DestinationGeographyKey)
GO

-- Create cross-dock fact table
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    CrossDockTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    QuantityTransferred INT,
    DwellAtDockMinutes INT,
    TemperatureCompliance BIT,
    QualityCheckPassed BIT
)
GO

CREATE NONCLUSTERED INDEX IX_FactCrossDock_Time ON FactCrossDock(CrossDockTimeKey, DwellAtDockMinutes)
GO
```

### Step 2: Populate Dimension Tables

```sql
-- Generate time dimension (example for 2026)
DECLARE @StartDate DATETIME = '2026-01-01 00:00:00'
DECLARE @EndDate DATETIME = '2026-12-31 23:45:00'
DECLARE @CurrentDate DATETIME = @StartDate

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO DimTime (
        FullDateTime, DateKey, TimeSlot15Min, HourOfDay, 
        DayOfWeek, DayName, WeekOfYear, MonthNumber, 
        MonthName, QuarterNumber, FiscalYear, IsWeekend, IsHoliday
    )
    VALUES (
        @CurrentDate,
        CAST(FORMAT(@CurrentDate, 'yyyyMMdd') AS INT),
        FORMAT(@CurrentDate, 'HH:mm'),
        DATEPART(HOUR, @CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        DATENAME(WEEKDAY, @CurrentDate),
        DATEPART(WEEK, @CurrentDate),
        DATEPART(MONTH, @CurrentDate),
        DATENAME(MONTH, @CurrentDate),
        DATEPART(QUARTER, @CurrentDate),
        YEAR(@CurrentDate),
        CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1, 7) THEN 1 ELSE 0 END,
        0 -- Update with holiday logic as needed
    )
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate)
END
GO
```

### Step 3: Create Data Loading Procedures

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from staging table or external source
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OrderID, QuantityHandled, DwellTimeMinutes, 
        CycleTimeSeconds, ZoneAssigned, ActualZone
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        stg.OperationType,
        stg.OrderID,
        stg.Quantity,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DwellTimeMinutes,
        DATEDIFF(SECOND, stg.StartTime, stg.EndTime) AS CycleTimeSeconds,
        stg.AssignedZone,
        stg.ActualZone
    FROM StagingWarehouseOps stg
    INNER JOIN DimTime t ON CAST(stg.OperationTime AS DATETIME) = t.FullDateTime
    INNER JOIN DimGeography g ON stg.LocationCode = g.LocationCode
    INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
    WHERE stg.OperationTime > @LastLoadDateTime
    AND stg.IsProcessed = 0
    
    -- Mark staging records as processed
    UPDATE StagingWarehouseOps
    SET IsProcessed = 1
    WHERE OperationTime > @LastLoadDateTime
END
GO

-- Stored procedure to recalculate product gravity scores
CREATE PROCEDURE usp_RecalculateGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    ;WITH RecentActivity AS (
        SELECT 
            ProductKey,
            AVG(CAST(DwellTimeMinutes AS DECIMAL(10,2))) AS AvgDwellTime,
            COUNT(*) AS PickCount
        FROM FactWarehouseOperations
        WHERE TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()))
        AND OperationType = 'Picking'
        GROUP BY ProductKey
    )
    UPDATE p
    SET 
        VelocityScore = ISNULL(ra.PickCount / 30.0, 0),
        GravityScore = (ISNULL(ra.PickCount / 30.0, 0) * p.ValuePerUnit) / 
                       CASE p.Fragility 
                           WHEN 'Low' THEN 1 
                           WHEN 'Medium' THEN 2 
                           WHEN 'High' THEN 3 
                           ELSE 1 
                       END,
        OptimalZone = CASE 
            WHEN (ISNULL(ra.PickCount / 30.0, 0) * p.ValuePerUnit) / 
                 CASE p.Fragility WHEN 'Low' THEN 1 WHEN 'Medium' THEN 2 WHEN 'High' THEN 3 ELSE 1 END > 1000 
            THEN 'High-Gravity'
            WHEN (ISNULL(ra.PickCount / 30.0, 0) * p.ValuePerUnit) / 
                 CASE p.Fragility WHEN 'Low' THEN 1 WHEN 'Medium' THEN 2 WHEN 'High' THEN 3 ELSE 1 END > 200 
            THEN 'Medium-Gravity'
            ELSE 'Low-Gravity'
        END,
        LastRecalculated = GETDATE()
    FROM DimProductGravity p
    LEFT JOIN RecentActivity ra ON p.ProductKey = ra.ProductKey
END
GO
```

### Step 4: Configure Power BI Connection

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Connect to your SQL Server instance and `LogiFleetPulse` database
4. Import the following tables:
   - DimTime
   - DimGeography
   - DimProductGravity
   - FactWarehouseOperations
   - FactFleetTrips
   - FactCrossDock

## Key DAX Measures for Power BI

```dax
// Average Dwell Time (Cross-Fact)
Avg Dwell Time = 
VAR WarehouseDwell = 
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR CrossDockDwell = 
    AVERAGE(FactCrossDock[DwellAtDockMinutes])
RETURN
    DIVIDE(WarehouseDwell + CrossDockDwell, 2)

// Fleet Idle Time Percentage
Fleet Idle % = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[IdleTimeMinutes]) + 
    SUM(FactFleetTrips[LoadingTimeMinutes]) + 
    SUM(FactFleetTrips[UnloadingTimeMinutes]) + 
    DATEDIFF(
        MIN(DimTime[FullDateTime]),
        MAX(DimTime[FullDateTime]),
        MINUTE
    ),
    0
) * 100

// Gravity Zone Compliance
Gravity Zone Compliance % = 
VAR TotalOps = COUNTROWS(FactWarehouseOperations)
VAR CompliantOps = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[ActualZone] = RELATED(DimProductGravity[OptimalZone])
    )
RETURN
    DIVIDE(CompliantOps, TotalOps, 0) * 100

// Cross-Fact KPI: Cost Per Unit Delivered
Cost Per Unit Delivered = 
VAR FleetCost = 
    SUMX(
        FactFleetTrips,
        [FuelConsumedLiters] * 1.5 // Assume $1.50 per liter
    )
VAR UnitsDelivered = 
    SUM(FactWarehouseOperations[QuantityHandled])
RETURN
    DIVIDE(FleetCost, UnitsDelivered, 0)

// Predictive Bottleneck Score (simplified)
Bottleneck Risk Score = 
VAR HighDwellCount = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[DwellTimeMinutes] > 120
    )
VAR HighIdleCount = 
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FactFleetTrips[IdleTimeMinutes] > 30
    )
VAR TotalOps = COUNTROWS(FactWarehouseOperations) + COUNTROWS(FactFleetTrips)
RETURN
    DIVIDE(HighDwellCount + HighIdleCount, TotalOps, 0) * 100

// Time Intelligence: MoM Dwell Time Change
Dwell Time MoM Change = 
VAR CurrentMonthDwell = [Avg Dwell Time]
VAR PreviousMonthDwell = 
    CALCULATE(
        [Avg Dwell Time],
        DATEADD(DimTime[FullDateTime], -1, MONTH)
    )
RETURN
    DIVIDE(CurrentMonthDwell - PreviousMonthDwell, PreviousMonthDwell, 0) * 100
```

## Configuration Files

### Connection Configuration (JSON format for ETL tools)

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "ActiveDirectory",
    "connection_timeout": 30,
    "command_timeout": 300
  },
  "data_sources": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "auth_token": "${WMS_API_TOKEN}",
      "refresh_interval_minutes": 15
    },
    "fleet_telemetry": {
      "endpoint": "${FLEET_TELEMETRY_ENDPOINT}",
      "auth_token": "${FLEET_API_TOKEN}",
      "refresh_interval_minutes": 5
    },
    "weather_api": {
      "endpoint": "https://api.weatherapi.com/v1/current.json",
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

## Common Query Patterns

### Cross-Fact Analysis: Warehouse → Fleet Impact

```sql
-- Find products with high dwell time that also cause fleet delays
SELECT 
    p.SKU,
    p.ProductName,
    p.OptimalZone,
    AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwell,
    AVG(ft.DelayMinutes) AS AvgFleetDelay,
    COUNT(DISTINCT wo.OrderID) AS OrderCount
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN FactFleetTrips ft ON wo.OrderID = CAST(ft.TripID AS VARCHAR(100))
WHERE wo.TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()))
GROUP BY p.SKU, p.ProductName, p.OptimalZone
HAVING AVG(wo.DwellTimeMinutes) > 60
AND AVG(ft.DelayMinutes) > 15
ORDER BY AVG(wo.DwellTimeMinutes) DESC
```

### Time-Phased Bottleneck Detection

```sql
-- Identify hourly patterns of warehouse congestion
SELECT 
    t.HourOfDay,
    t.DayName,
    COUNT(*) AS OperationCount,
    AVG(wo.CycleTimeSeconds) AS AvgCycleTime,
    STDEV(wo.CycleTimeSeconds) AS StdDevCycleTime,
    CASE 
        WHEN AVG(wo.CycleTimeSeconds) > 180 THEN 'High Risk'
        WHEN AVG(wo.CycleTimeSeconds) > 120 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS BottleneckRisk
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.FiscalYear = 2026
GROUP BY t.HourOfDay, t.DayName
ORDER BY BottleneckRisk DESC, AvgCycleTime DESC
```

### Gravity Zone Optimization Query

```sql
-- Identify products in wrong zones causing inefficiency
SELECT 
    p.SKU,
    p.ProductName,
    p.OptimalZone,
    wo.ActualZone,
    COUNT(*) AS MisplacedPickCount,
    AVG(wo.CycleTimeSeconds) AS AvgCycleTime,
    (AVG(wo.CycleTimeSeconds) - 
     (SELECT AVG(CycleTimeSeconds) 
      FROM FactWarehouseOperations wo2 
      WHERE wo2.ProductKey = p.ProductKey 
      AND wo2.ActualZone = p.OptimalZone)) AS CycleTimePenalty
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE wo.ActualZone <> p.OptimalZone
AND wo.OperationType = 'Picking'
GROUP BY p.SKU, p.ProductName, p.OptimalZone, wo.ActualZone
HAVING COUNT(*) > 10
ORDER BY CycleTimePenalty DESC
```

### Fleet Fuel Efficiency by Route

```sql
-- Calculate fuel efficiency and identify optimization opportunities
SELECT 
    og.LocationName AS Origin,
    dg.LocationName AS Destination,
    COUNT(*) AS TripCount,
    AVG(ft.ActualDistanceKm) AS AvgDistance,
    AVG(ft.FuelConsumedLiters) AS AvgFuel,
    AVG(ft.FuelConsumedLiters / NULLIF(ft.ActualDistanceKm, 0)) AS AvgFuelPerKm,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    AVG(ft.LoadWeightKg) AS AvgLoadWeight,
    MIN(ft.FuelConsumedLiters / NULLIF(ft.ActualDistanceKm, 0)) AS BestFuelPerKm
FROM FactFleetTrips ft
INNER JOIN DimGeography og ON ft.OriginGeographyKey = og.GeographyKey
INNER JOIN DimGeography dg ON ft.DestinationGeographyKey = dg.GeographyKey
WHERE ft.StartTimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -90, GETDATE()))
GROUP BY og.LocationName, dg.LocationName
HAVING COUNT(*) >= 5
ORDER BY AvgFuelPerKm DESC
```

## Automated Alerting Setup

```sql
-- Create alert thresholds table
CREATE TABLE AlertThresholds (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName VARCHAR(200),
    MetricType VARCHAR(100), -- 'DwellTime', 'IdleTime', 'DelayMinutes', etc.
    ThresholdValue DECIMAL(10,2),
    ComparisonOperator VARCHAR(10), -- '>', '<', '>=', '<=', '='
    NotificationEmails VARCHAR(MAX), -- Comma-separated
    IsActive BIT DEFAULT 1
)
GO

-- Stored procedure to check alerts and send notifications
CREATE PROCEDURE usp_CheckAlertsAndNotify
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Check warehouse dwell time alerts
    DECLARE @AlertResults TABLE (
        AlertName VARCHAR(200),
        MetricValue DECIMAL(10,2),
        ThresholdValue DECIMAL(10,2),
        LocationName VARCHAR(200),
        ProductName VARCHAR(500)
    )
    
    INSERT INTO @AlertResults
    SELECT 
        'High Warehouse Dwell Time' AS AlertName,
        AVG(wo.DwellTimeMinutes) AS MetricValue,
        at.ThresholdValue,
        g.LocationName,
        p.ProductName
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    CROSS JOIN AlertThresholds at
    WHERE at.MetricType = 'DwellTime'
    AND at.IsActive = 1
    AND t.FullDateTime >= DATEADD(HOUR, -1, GETDATE())
    GROUP BY g.LocationName, p.ProductName, at.ThresholdValue
    HAVING AVG(wo.DwellTimeMinutes) > at.ThresholdValue
    
    -- Check fleet idle time alerts
    INSERT INTO @AlertResults
    SELECT 
        'High Fleet Idle Time' AS AlertName,
        ft.IdleTimeMinutes AS MetricValue,
        at.ThresholdValue,
        g.LocationName,
        ft.VehicleID AS ProductName
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
    INNER JOIN DimGeography g ON ft.OriginGeographyKey = g.GeographyKey
    CROSS JOIN AlertThresholds at
    WHERE at.MetricType = 'IdleTime'
    AND at.IsActive = 1
    AND t.FullDateTime >= DATEADD(HOUR, -1, GETDATE())
    AND ft.IdleTimeMinutes > at.ThresholdValue
    
    -- Send alerts (integrate with email service or Teams webhook)
    SELECT * FROM @AlertResults
    
    -- Log alerts to history table
    INSERT INTO AlertHistory (AlertTime, AlertName, MetricValue, ThresholdValue, LocationName, Details)
    SELECT GETDATE(), AlertName, MetricValue, ThresholdValue, LocationName, ProductName
    FROM @AlertResults
END
GO

-- Schedule this procedure to run every 15 minutes via SQL Agent Job
```

## Power BI Report Structure

### Dashboard 1: Executive Overview
- KPI cards: Total shipments, on-time delivery %, avg dwell time, fleet utilization
- Line chart: Daily trend of operations volume
- Map visual: Geographic distribution of warehouses and routes
- Gauge: Bottleneck risk score

### Dashboard 2: Warehouse Operations
- Matrix: Product category × optimal zone compliance
- Bar chart: Top 10 products by dwell time
- Scatter plot: Cycle time vs. quantity handled
- Slicer: Date range, location, operation type

### Dashboard 3: Fleet Management
- Table: Vehicle performance metrics (fuel efficiency, idle time, delays)
- Column chart: Trips by hour of day
- Line + column chart: Distance vs. fuel consumption
- Card visual: Cost per kilometer

### Dashboard 4: Cross-Modal Analysis
- Decomposition tree: Dwell time → product → zone → time of day
- Ribbon chart: Gravity zone compliance over time
- Waterfall chart: Cost breakdown (fuel, labor, delays)

## Troubleshooting

### Issue: Power BI refresh takes too long

**Solution:** Implement incremental refresh in Power BI

```m
// Power Query M code for incremental refresh
let
    Source = Sql.Database(SqlServerHost, "LogiFleetPulse"),
    FactTable = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
    FilteredRows = Table.SelectRows(FactTable, each [TimeKey] >= RangeStart and [TimeKey] < RangeEnd)
in
    FilteredRows
```

Configure incremental refresh policy:
- Archive data: 2 years
- Refresh data: Last 30 days
- Detect data changes: TimeKey column

### Issue: Gravity scores not updating

**Solution:** Schedule the recalculation procedure

```sql
-- Create SQL Agent Job to run usp_RecalculateGravityScores daily
EXEC msdb.dbo.sp_add_job
    @job_name = N'Recalculate_Gravity_Scores',
    @enabled = 1

EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'Recalculate_Gravity_Scores',
    @step_name = N'Run_Procedure',
    @subsystem = N'TSQL',
    @command = N'EXEC usp_RecalculateGravityScores',
    @database_name = N'LogiFleetPulse'

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'Daily_Midnight',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @active_start_time = 000000

EXEC msdb.dbo.sp_attach_schedule
    @job_name = N'Recalculate_Gravity_Scores',
    @schedule_name = N'Daily_Midnight'
```

### Issue: Cross-fact queries are slow

**Solution:** Create indexed views for common joins

```sql
CREATE VIEW vw_WarehouseFleetJoin
WITH SCHEMABINDING
AS
SELECT 
    wo.OperationKey,
    wo.ProductKey,
    wo.DwellTimeMinutes,
    ft.TripKey,
    ft.VehicleID,
    ft.DelayMinutes,
    ft.IdleTimeMinutes,
    t.FullDateTime
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.FactFleetTrips ft ON wo.OrderID = CAST(ft.TripID AS VARCHAR(100))
INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.FiscalYear >= 2026
GO

CREATE UNIQUE CLUSTERED INDEX UCIX_WarehouseFleetJoin 
ON vw_WarehouseFleetJoin(OperationKey, TripKey)
GO
```

### Issue: Row-level security not working in Power BI

**Solution:** Define RLS roles in Power BI dataset

```dax
// Create role: "Warehouse Manager - East"
[LocationName] IN { "New York Warehouse", "Boston Warehouse", "Philadelphia Warehouse" }

// Create role: "Fleet Supervisor - West"
[LocationName] IN { "Los Angeles Hub", "San Francisco Hub", "Seattle Hub" }

// Test RLS in Power BI Desktop: Modeling → View as Roles
```

## Integration with External Systems

### ETL from WMS (using Python)

```python
import pyodbc
import requests
import os
from datetime import datetime

# Database connection
conn = pyodbc.connect(
    f"Driver={{ODBC Driver 17 for SQL Server}};"
    f"Server={os.getenv('SQL_SERVER_HOST')};"
    f"Database=LogiFleetPulse;"
    f"Trusted_Connection=yes;"
)
cursor = conn.cursor()

# Fetch data from WMS API
wms_endpoint = os.getenv('WMS_API_ENDPOINT')
wms_token = os.getenv('WMS_API_TOKEN')

response = requests.get(
    f"{wms_endpoint}/operations/recent",
    headers={"Authorization": f"Bearer
