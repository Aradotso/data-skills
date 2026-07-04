---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI template for multi-modal logistics intelligence with warehouse operations, fleet tracking, and predictive supply chain analytics
triggers:
  - set up LogiFleet Pulse supply chain analytics
  - deploy logistics data warehouse with Power BI
  - configure warehouse and fleet telemetry tracking
  - create multi-fact star schema for logistics
  - build real-time supply chain dashboard
  - implement warehouse gravity zone optimization
  - set up predictive fleet maintenance alerts
  - analyze cross-modal logistics KPIs
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and business intelligence template for logistics operations. It provides:

- **Multi-fact star schema** combining warehouse operations, fleet telemetry, and supply chain data
- **Power BI dashboards** for real-time logistics visualization
- **Predictive analytics** for bottleneck detection and fleet maintenance
- **Cross-fact KPI harmonization** linking inventory, routes, and external factors
- **Warehouse Gravity Zones™** for spatial optimization based on pick frequency and item characteristics

The system integrates data from WMS (Warehouse Management Systems), telematics/GPS feeds, supplier portals, weather/traffic APIs, and customer order history into a unified semantic layer.

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- Access to warehouse management system, fleet telemetry, or similar data sources

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance and create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the complete schema script from the repository
-- This creates all fact tables, dimensions, views, and stored procedures
-- The main schema files should be executed in this order:
-- 1. DimensionTables.sql
-- 2. FactTables.sql
-- 3. BridgeTables.sql
-- 4. Views.sql
-- 5. StoredProcedures.sql
```

### Step 2: Core Schema Components

**Dimension Tables:**

```sql
-- Time dimension with 15-minute granularity
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    [Date] DATE NOT NULL,
    [Year] INT NOT NULL,
    [Quarter] INT NOT NULL,
    [Month] INT NOT NULL,
    [Day] INT NOT NULL,
    DayOfWeek INT NOT NULL,
    [Hour] INT NOT NULL,
    [Minute] INT NOT NULL,
    FiscalYear INT NOT NULL,
    FiscalQuarter INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
);

-- Product dimension with gravity scoring
CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    ProductID NVARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity + value + fragility
    IsPerishable BIT DEFAULT 0,
    WeightKg DECIMAL(10,2),
    VolumeM3 DECIMAL(10,4),
    UnitValue DECIMAL(10,2)
);

-- Geography dimension (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID NVARCHAR(50) NOT NULL UNIQUE,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50), -- Warehouse, Route Node, Distribution Center
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50)
);

-- Supplier reliability dimension
CREATE TABLE DimSupplier (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierID NVARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200) NOT NULL,
    LeadTimeDaysAvg DECIMAL(5,1),
    LeadTimeVariance DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,2),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2) -- 0-100 scale
);
```

**Fact Tables:**

```sql
-- Warehouse operations fact
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProduct(ProductKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OperationType NVARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    OperationStartTime DATETIME NOT NULL,
    OperationEndTime DATETIME,
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime),
    UnitsHandled INT,
    EmployeeID NVARCHAR(50),
    ZoneID NVARCHAR(50), -- Storage zone identifier
    DwellTimeHours DECIMAL(10,2), -- Time in storage
    PickPathMeters DECIMAL(10,2) -- Distance traveled for pick operation
);

-- Fleet trips fact
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TripStartTime DATETIME NOT NULL,
    TripEndTime DATETIME,
    TripDurationMinutes AS DATEDIFF(MINUTE, TripStartTime, TripEndTime),
    DistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    LoadVolumeM3 DECIMAL(10,4),
    AvgSpeedKmh DECIMAL(5,2),
    DelayMinutes DECIMAL(10,2), -- Actual vs planned
    DelayReason NVARCHAR(200) -- Weather, Traffic, Mechanical, etc.
);

-- Cross-dock operations fact
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProduct(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    CrossDockLocationKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    UnitsTransferred INT,
    DockDwellMinutes DECIMAL(10,2),
    TemperatureCompliant BIT DEFAULT 1
);
```

### Step 3: Configure Data Connections

Create a configuration file (do not commit to version control):

```sql
-- Create credential store for external data sources
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '${DB_MASTER_KEY_PASSWORD}';

-- Example: Connect to external WMS via linked server
EXEC sp_addlinkedserver 
    @server = 'WMS_SERVER',
    @srvproduct = '',
    @provider = 'SQLNCLI',
    @datasrc = '${WMS_SERVER_HOST}',
    @catalog = 'WMS_Database';

EXEC sp_addlinkedsrvlogin 
    @rmtsrvname = 'WMS_SERVER',
    @useself = 'FALSE',
    @rmtuser = '${WMS_USERNAME}',
    @rmtpassword = '${WMS_PASSWORD}';
```

### Step 4: Data Loading Stored Procedures

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last 24 hours if not specified
    IF @LastLoadTime IS NULL
        SET @LastLoadTime = DATEADD(HOUR, -24, GETDATE());
    
    -- Insert new records from WMS
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, GeographyKey, OperationType,
        OperationStartTime, OperationEndTime, UnitsHandled,
        EmployeeID, ZoneID, DwellTimeHours, PickPathMeters
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        wms.OperationType,
        wms.StartTime,
        wms.EndTime,
        wms.Quantity,
        wms.EmployeeID,
        wms.ZoneID,
        wms.DwellHours,
        wms.PathMeters
    FROM WMS_SERVER.WMS_Database.dbo.Operations wms
    INNER JOIN DimTime t ON CAST(wms.StartTime AS DATE) = t.[Date] 
        AND DATEPART(HOUR, wms.StartTime) = t.[Hour]
        AND (DATEPART(MINUTE, wms.StartTime) / 15) * 15 = t.[Minute]
    INNER JOIN DimProduct p ON wms.ProductID = p.ProductID
    INNER JOIN DimGeography g ON wms.LocationID = g.LocationID
    WHERE wms.StartTime > @LastLoadTime;
    
    -- Log load completion
    INSERT INTO LoadLog (TableName, LoadTime, RowsInserted)
    VALUES ('FactWarehouseOperations', GETDATE(), @@ROWCOUNT);
END;
GO

-- Calculate warehouse gravity zones
CREATE PROCEDURE sp_UpdateWarehouseGravityZones
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update product gravity scores based on recent activity
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            COUNT(*) AS PickFrequency,
            AVG(wo.DwellTimeHours) AS AvgDwellTime,
            SUM(wo.UnitsHandled) AS TotalUnits
        FROM FactWarehouseOperations wo
        INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
        WHERE wo.TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime) -- Last 30 days
            AND wo.OperationType = 'Picking'
        GROUP BY p.ProductKey
    )
    UPDATE p
    SET GravityScore = 
        (pm.PickFrequency * 0.4) + -- Velocity weight
        ((p.UnitValue / 100) * 0.3) + -- Value weight
        ((1.0 / NULLIF(pm.AvgDwellTime, 0)) * 0.3) -- Turnover weight
    FROM DimProduct p
    INNER JOIN ProductMetrics pm ON p.ProductKey = pm.ProductKey;
    
    -- Recommend zone reassignments
    SELECT 
        p.ProductID,
        p.ProductName,
        p.GravityScore,
        CASE 
            WHEN p.GravityScore > 75 THEN 'High-Gravity (Near Shipping)'
            WHEN p.GravityScore > 40 THEN 'Medium-Gravity (Mid-Warehouse)'
            ELSE 'Low-Gravity (Deep Storage)'
        END AS RecommendedZone,
        wo.ZoneID AS CurrentZone
    FROM DimProduct p
    INNER JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
    WHERE wo.OperationKey IN (
        SELECT MAX(OperationKey) 
        FROM FactWarehouseOperations 
        GROUP BY ProductKey
    )
    ORDER BY p.GravityScore DESC;
END;
GO
```

### Step 5: Set Up Power BI

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Connect to your LogiFleetPulse database using:
   - Server: `${SQL_SERVER_HOST}`
   - Database: `LogiFleetPulse`
   - Authentication: Windows or SQL Server (use environment variables)

4. Import key tables:
```
FactWarehouseOperations
FactFleetTrips
FactCrossDock
DimTime
DimProduct
DimGeography
DimSupplier
```

5. Power BI will auto-detect relationships. Verify:
   - All fact tables → DimTime on TimeKey
   - All fact tables → DimProduct on ProductKey
   - All fact tables → DimGeography on GeographyKey (or specific FK)

## Key Analytics Patterns

### Cross-Fact KPI: Dwell Time vs Fleet Idle Time

```sql
-- DAX Measure in Power BI
Avg Dwell Time per SKU = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeHours]),
    FactWarehouseOperations[OperationType] = "Picking"
)

Fleet Idle Time % = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes]),
    0
) * 100

-- Combined correlation metric
Dwell-Idle Correlation Index = 
VAR DwellScore = [Avg Dwell Time per SKU] / 24 -- Normalize to days
VAR IdleScore = [Fleet Idle Time %] / 100
RETURN (DwellScore * 0.6) + (IdleScore * 0.4)
```

### Predictive Bottleneck Detection

```sql
-- SQL View for bottleneck analysis
CREATE VIEW vw_PredictiveBottlenecks AS
WITH HourlyMetrics AS (
    SELECT 
        t.[Date],
        t.[Hour],
        g.LocationName,
        COUNT(*) AS OperationCount,
        AVG(wo.DurationMinutes) AS AvgDuration,
        STDEV(wo.DurationMinutes) AS StdDevDuration
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    WHERE t.[Date] >= DATEADD(DAY, -30, GETDATE())
    GROUP BY t.[Date], t.[Hour], g.LocationName
),
BottleneckScores AS (
    SELECT 
        *,
        CASE 
            WHEN AvgDuration > (SELECT AVG(AvgDuration) * 1.5 FROM HourlyMetrics) 
                AND StdDevDuration > (SELECT AVG(StdDevDuration) * 1.5 FROM HourlyMetrics)
            THEN 'High Risk'
            WHEN AvgDuration > (SELECT AVG(AvgDuration) * 1.2 FROM HourlyMetrics)
            THEN 'Medium Risk'
            ELSE 'Normal'
        END AS BottleneckRisk,
        PERCENT_RANK() OVER (ORDER BY AvgDuration DESC) AS DurationPercentile
    FROM HourlyMetrics
)
SELECT 
    LocationName,
    [Hour],
    AVG(AvgDuration) AS AvgOperationMinutes,
    MAX(BottleneckRisk) AS RiskLevel,
    AVG(DurationPercentile) AS Percentile
FROM BottleneckScores
WHERE BottleneckRisk <> 'Normal'
GROUP BY LocationName, [Hour];
```

### Fleet Maintenance Triage

```sql
-- Weighted priority scoring for vehicle maintenance
CREATE VIEW vw_FleetMaintenancePriority AS
WITH VehicleMetrics AS (
    SELECT 
        VehicleID,
        AVG(FuelConsumedLiters / NULLIF(DistanceKm, 0)) AS AvgFuelEfficiency,
        AVG(IdleTimeMinutes / NULLIF(TripDurationMinutes, 0)) AS IdleRatio,
        SUM(CASE WHEN DelayMinutes > 30 THEN 1 ELSE 0 END) AS DelayCount,
        SUM(LoadWeightKg * DistanceKm) AS TotalTonKm
    FROM FactFleetTrips
    WHERE TripStartTime >= DATEADD(DAY, -90, GETDATE())
    GROUP BY VehicleID
),
LoadedTrips AS (
    SELECT 
        ft.VehicleID,
        SUM(p.UnitValue * wo.UnitsHandled) AS RevenueAtRisk
    FROM FactFleetTrips ft
    INNER JOIN FactWarehouseOperations wo ON wo.GeographyKey = ft.OriginGeographyKey
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    WHERE ft.TripStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY ft.VehicleID
)
SELECT 
    vm.VehicleID,
    vm.AvgFuelEfficiency,
    vm.IdleRatio,
    vm.DelayCount,
    lt.RevenueAtRisk,
    -- Priority score (0-100)
    (
        (vm.IdleRatio * 25) + -- Higher idle = higher score
        (vm.DelayCount / 10.0 * 25) + -- More delays = higher score
        ((1.0 / NULLIF(vm.AvgFuelEfficiency, 0)) * 25) + -- Lower efficiency = higher score
        ((lt.RevenueAtRisk / 100000.0) * 25) -- Higher revenue risk = higher score
    ) AS MaintenancePriorityScore
FROM VehicleMetrics vm
LEFT JOIN LoadedTrips lt ON vm.VehicleID = lt.VehicleID
ORDER BY MaintenancePriorityScore DESC;
```

## Automated Alerting

```sql
-- Stored procedure for threshold-based alerts
CREATE PROCEDURE sp_GenerateAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertTable TABLE (
        AlertType NVARCHAR(100),
        AlertMessage NVARCHAR(500),
        Severity NVARCHAR(20),
        EntityID NVARCHAR(50),
        MetricValue DECIMAL(10,2)
    );
    
    -- Alert: Fleet idle time exceeds 15%
    INSERT INTO @AlertTable
    SELECT 
        'Fleet Idle Time',
        'Vehicle ' + VehicleID + ' has idle time of ' + CAST(IdleRatio * 100 AS NVARCHAR) + '%',
        'Warning',
        VehicleID,
        IdleRatio * 100
    FROM (
        SELECT 
            VehicleID,
            AVG(IdleTimeMinutes / NULLIF(TripDurationMinutes, 0)) AS IdleRatio
        FROM FactFleetTrips
        WHERE TripStartTime >= DATEADD(DAY, -7, GETDATE())
        GROUP BY VehicleID
    ) metrics
    WHERE IdleRatio > 0.15;
    
    -- Alert: Warehouse dwell time exceeds 72 hours
    INSERT INTO @AlertTable
    SELECT 
        'Warehouse Dwell Time',
        'Product ' + p.ProductName + ' in zone ' + wo.ZoneID + ' has dwell time of ' + CAST(wo.DwellTimeHours AS NVARCHAR) + ' hours',
        CASE WHEN wo.DwellTimeHours > 120 THEN 'Critical' ELSE 'Warning' END,
        p.ProductID,
        wo.DwellTimeHours
    FROM FactWarehouseOperations wo
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    WHERE wo.DwellTimeHours > 72
        AND wo.OperationType = 'Putaway'
        AND wo.TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime); -- Last 24 hours
    
    -- Alert: Cross-dock temperature violations
    INSERT INTO @AlertTable
    SELECT 
        'Temperature Compliance',
        'Cross-dock operation at ' + g.LocationName + ' for product ' + p.ProductName + ' violated temperature compliance',
        'Critical',
        CAST(cd.CrossDockKey AS NVARCHAR),
        0
    FROM FactCrossDock cd
    INNER JOIN DimProduct p ON cd.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON cd.CrossDockLocationKey = g.GeographyKey
    WHERE cd.TemperatureCompliant = 0
        AND p.IsPerishable = 1
        AND cd.TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime);
    
    -- Output all alerts
    SELECT * FROM @AlertTable
    ORDER BY 
        CASE Severity 
            WHEN 'Critical' THEN 1 
            WHEN 'Warning' THEN 2 
            ELSE 3 
        END;
    
    -- Optionally, send to monitoring table
    INSERT INTO AlertLog (AlertType, AlertMessage, Severity, EntityID, MetricValue, AlertTime)
    SELECT AlertType, AlertMessage, Severity, EntityID, MetricValue, GETDATE()
    FROM @AlertTable;
END;
GO

-- Schedule this procedure to run every 15 minutes via SQL Agent
```

## Power BI Dashboard Components

### Key Visualizations

**1. Real-Time Operations Monitor (Card Visuals)**
```
DAX Measures:
- Total Operations Today = CALCULATE(COUNT(FactWarehouseOperations[OperationKey]), DimTime[Date] = TODAY())
- Active Vehicles = DISTINCTCOUNT(FactFleetTrips[VehicleID])
- Avg Dwell Time = AVERAGE(FactWarehouseOperations[DwellTimeHours])
- Fleet Utilization % = DIVIDE(SUM(FactFleetTrips[TripDurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]), SUM(FactFleetTrips[TripDurationMinutes]))
```

**2. Warehouse Gravity Heatmap (Matrix Visual)**
```
Rows: DimGeography[LocationName]
Columns: DimProduct[Category]
Values: SUM(FactWarehouseOperations[UnitsHandled])
Color Scale: Conditional formatting by DimProduct[GravityScore]
```

**3. Fleet Route Efficiency (Map Visual)**
```
Location: DimGeography[Latitude], DimGeography[Longitude]
Size: SUM(FactFleetTrips[FuelConsumedLiters])
Color: AVG(FactFleetTrips[DelayMinutes])
Tooltip: VehicleID, DistanceKm, AvgSpeedKmh
```

**4. Temporal Bottleneck Forecast (Line Chart)**
```
X-Axis: DimTime[Hour]
Y-Axis: CALCULATE(AVERAGE(FactWarehouseOperations[DurationMinutes]))
Legend: DimGeography[LocationName]
Forecast: Enable 30-period prediction with 95% confidence interval
```

## Configuration for Production

### Row-Level Security (RLS)

```sql
-- Create roles in SQL Server
CREATE ROLE WarehouseManager;
CREATE ROLE FleetManager;
CREATE ROLE ExecutiveView;

-- Grant appropriate permissions
GRANT SELECT ON FactWarehouseOperations TO WarehouseManager;
GRANT SELECT ON FactFleetTrips TO FleetManager;
GRANT SELECT ON SCHEMA::dbo TO ExecutiveView;

-- In Power BI: Manage Roles → Create role with DAX filter
-- Example: WarehouseManagerRole
[DimGeography][LocationName] = USERNAME()

-- Example: RegionalManagerRole
[DimGeography][Region] IN VALUES(UserRegions[Region])
```

### Performance Optimization

```sql
-- Create columnstore indexes for fact tables
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouseOps_Analytics
ON FactWarehouseOperations (TimeKey, ProductKey, GeographyKey, DurationMinutes, UnitsHandled);

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleetTrips_Analytics
ON FactFleetTrips (TimeKey, VehicleID, OriginGeographyKey, DestinationGeographyKey, DistanceKm, FuelConsumedLiters);

-- Partition large fact tables by date
ALTER PARTITION SCHEME PS_DailyPartitions
NEXT USED [PRIMARY];

ALTER TABLE FactWarehouseOperations
SWITCH PARTITION 1 TO StagingTable PARTITION 1;
```

### Incremental Refresh in Power BI

1. Add RangeStart and RangeEnd parameters (DateTime)
2. Filter fact tables: `[OperationStartTime] >= RangeStart and [OperationStartTime] < RangeEnd`
3. Configure incremental refresh policy:
   - Archive data: 5 years
   - Incremental refresh: 30 days
   - Refresh frequency: Every 15 minutes

## Common Troubleshooting

### Issue: Slow dashboard refresh times

**Solution:**
```sql
-- Check for missing indexes
SELECT 
    OBJECT_NAME(s.[object_id]) AS TableName,
    i.name AS IndexName,
    s.user_seeks,
    s.user_scans,
    s.user_lookups
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.indexes i ON s.[object_id] = i.[object_id] AND s.index_id = i.index_id
WHERE database_id = DB_ID('LogiFleetPulse')
    AND OBJECTPROPERTY(s.[object_id], 'IsUserTable') = 1
ORDER BY s.user_seeks + s.user_scans + s.user_lookups DESC;

-- Add covering indexes for frequent queries
CREATE INDEX IX_FactWarehouseOps_TimeProduct
ON FactWarehouseOperations (TimeKey, ProductKey)
INCLUDE (DurationMinutes, UnitsHandled);
```

### Issue: Duplicate records in fact tables

**Solution:**
```sql
-- Find duplicates
SELECT 
    TimeKey, ProductKey, GeographyKey, OperationStartTime,
    COUNT(*) AS DuplicateCount
FROM FactWarehouseOperations
GROUP BY TimeKey, ProductKey, GeographyKey, OperationStartTime
HAVING COUNT(*) > 1;

-- Remove duplicates (keep first occurrence)
WITH CTE AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY TimeKey, ProductKey, GeographyKey, OperationStartTime 
            ORDER BY OperationKey
        ) AS RowNum
    FROM FactWarehouseOperations
)
DELETE FROM CTE WHERE RowNum > 1;
```

### Issue: Time dimension mismatches

**Solution:**
```sql
-- Ensure complete time dimension coverage
DECLARE @StartDate DATE = '2025-01-01';
DECLARE @EndDate DATE = '2027-12-31';

;WITH TimeSeries AS (
    SELECT @StartDate AS [Date]
    UNION ALL
    SELECT DATEADD(DAY, 1, [Date])
    FROM TimeSeries
    WHERE [Date] < @EndDate
)
INSERT INTO DimTime (TimeKey, FullDateTime, [Date], [Year], [Quarter], [Month], [Day], DayOfWeek, [Hour], [Minute], FiscalYear, FiscalQuarter, IsWeekend, IsHoliday)
SELECT 
    CONVERT(INT, FORMAT(dt, 'yyyyMMddHHmm')),
    dt,
    CAST(dt AS DATE),
    YEAR(dt),
    DATEPART(QUARTER, dt),
    MONTH(dt),
    DAY(dt),
    DATEPART(WEEKDAY, dt),
    DATEPART(HOUR, dt),
    DATEPART(MINUTE, dt),
    CASE WHEN MONTH(dt) >= 7 THEN YEAR(dt) + 1 ELSE YEAR(dt) END,
    CASE WHEN MONTH(dt) >= 7 THEN DATEPART(QUARTER, dt) - 2 ELSE DATEPART(QUARTER, dt) + 2 END,
    CASE WHEN DATEPART(WEEKDAY, dt) IN (1, 7) THEN 1 ELSE 0 END,
    0
FROM (
    SELECT DATEADD(MINUTE, minutes.number * 15, days.[Date]) AS dt
    FROM TimeSeries days
    CROSS APPLY (SELECT number FROM master..spt_values WHERE type = 'P' AND number BETWEEN 0 AND 95) minutes
) expanded
OPTION (MAXRECURSION 0);
```

## Integration Examples

### Webhook for Real-Time Alerts

```sql
-- Enable external scripts in SQL Server
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'Ole Automation Procedures', 1;
RECONFIGURE;

-- Send HTTP POST to webhook (e.g., Teams, Slack)
CREATE PROCEDURE sp_SendWebhookAlert
    @Message NVARCHAR(MAX),
    @WebhookUrl NVARCHAR(500) = '${WEBHOOK_URL}'
AS
BEGIN
    DECLARE @Object INT;
    DECLARE @ResponseText VARCHAR(8000);
    DECLARE @Json NVARCHAR(MAX) = '{"text":"' + @Message + '"}';
    
    EXEC sp_OACreate 'MSXML2.ServerXMLHTTP', @Object OUT;
    EXEC sp_OAMethod @Object, 'open', NULL, 'POST', @WebhookUrl, 'false';
    EXEC sp_OAMethod @Object, 'setRequestHeader', NULL, 'Content-Type', 'application/json';
    EXEC sp_OAMethod @Object, 'send', NULL, @Json;
    EXEC sp_OAMethod @Object, 'responseText', @ResponseText OUTPUT;
    EXEC sp_OADestroy @Object;
END;
GO

-- Trigger webhook on critical alerts
CREATE TRIGGER trg_CriticalAlert
ON AlertLog
AFTER INSERT
AS
BEGIN
    IF EXISTS (SELECT 1 FROM inserted WHERE Severity = 'Critical')
    BEGIN
        DECLARE @Msg NVARCHAR(MAX);
        SELECT @Msg = AlertMessage FROM inserted WHERE Severity = 'Critical';
        EXEC sp_SendWebhookAlert @Msg;
    END
END;
GO
```

### Python Integration for ML Predictions

```python
import pyodbc
import pandas as pd
from sklearn.ensemble import RandomForestRegressor
import os

# Connect to SQL Server
conn = pyodbc.connect(
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.get
