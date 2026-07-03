---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server data warehouse and Power BI dashboard system for unified logistics, fleet, and warehouse intelligence
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy the LogiFleet warehouse and fleet data model"
  - "configure Power BI logistics dashboards"
  - "implement multi-fact star schema for supply chain"
  - "create warehouse gravity zones analytics"
  - "build fleet optimization data warehouse"
  - "integrate logistics telemetry with SQL Server"
  - "set up cross-modal supply chain KPI tracking"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a multi-modal logistics intelligence platform that combines warehouse operations, fleet telemetry, inventory management, and external data into a unified MS SQL Server data warehouse with Power BI visualizations. It uses a multi-fact star schema with time-phased dimensions to enable cross-functional analytics.

**Core capabilities:**
- Unified semantic layer linking warehouse, fleet, and supplier data
- Multi-fact star schema with role-playing dimensions
- Real-time Power BI dashboards (15-minute refresh cycles)
- Predictive bottleneck detection
- Warehouse gravity zone optimization
- Adaptive fleet maintenance prioritization

## Installation & Setup

### 1. SQL Server Deployment

**Requirements:**
- MS SQL Server 2019 or later
- Sufficient storage for fact tables (estimate 10GB+ for moderate operations)
- SQL Server Agent enabled for scheduled jobs

**Deploy the schema:**

```sql
-- Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTimeUTC DATETIME2 NOT NULL,
    FifteenMinuteBucket INT,
    HourOfDay INT,
    DayOfWeek INT,
    DayOfMonth INT,
    WeekOfYear INT,
    FiscalPeriod NVARCHAR(10),
    IsBusinessHour BIT,
    INDEX IX_DimTime_DateTime (DateTimeUTC)
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    NodeID NVARCHAR(50) UNIQUE NOT NULL,
    NodeType NVARCHAR(20), -- 'Warehouse', 'RoutePoint', 'Supplier', 'Customer'
    NodeName NVARCHAR(100),
    Address NVARCHAR(255),
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZoneOffset INT,
    INDEX IX_DimGeography_NodeID (NodeID)
);

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,2),
    UnitValue DECIMAL(10,2),
    FragilityScore INT, -- 1-10
    VelocityScore INT, -- Calculated: picks per day
    GravityScore DECIMAL(5,2), -- Composite: velocity * value / fragility
    RecommendedZone NVARCHAR(20), -- 'HighGravity', 'MediumGravity', 'LowGravity'
    INDEX IX_DimProduct_SKU (SKU),
    INDEX IX_DimProduct_Gravity (GravityScore DESC)
);

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID NVARCHAR(50) UNIQUE NOT NULL,
    SupplierName NVARCHAR(200),
    AverageLeadTimeDays DECIMAL(5,1),
    LeadTimeVariance DECIMAL(5,1),
    DefectPercentage DECIMAL(5,2),
    OnTimeDeliveryRate DECIMAL(5,2),
    ComplianceScore INT, -- 0-100
    ReliabilityRating NVARCHAR(20), -- 'Excellent', 'Good', 'Fair', 'Poor'
    INDEX IX_DimSupplier_ID (SupplierID)
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType NVARCHAR(20), -- 'Receiving', 'Putaway', 'Pick', 'Pack', 'Ship'
    OperationStartTime DATETIME2,
    OperationEndTime DATETIME2,
    DurationMinutes DECIMAL(10,2),
    QuantityHandled INT,
    DwellTimeHours DECIMAL(10,2), -- For inventory
    StorageZone NVARCHAR(50),
    EmployeeID NVARCHAR(50),
    BatchNumber NVARCHAR(100),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    INDEX IX_FactWarehouse_Time (TimeKey),
    INDEX IX_FactWarehouse_Product (ProductKey),
    INDEX IX_FactWarehouse_Operation (OperationType, OperationStartTime)
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    TripDurationMinutes DECIMAL(10,2),
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadWeightKG DECIMAL(10,2),
    AvgSpeedKMH DECIMAL(5,2),
    DelayMinutes DECIMAL(10,2),
    DelayReason NVARCHAR(100), -- 'Weather', 'Traffic', 'Mechanical', 'Loading', NULL
    EngineDiagnosticFlag BIT,
    TirePressureWarning BIT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey),
    INDEX IX_FactFleet_Time (TimeKey),
    INDEX IX_FactFleet_Vehicle (VehicleID, TripStartTime)
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    ReceiptTime DATETIME2,
    ShipmentTime DATETIME2,
    DwellMinutes DECIMAL(10,2),
    QuantityTransferred INT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    INDEX IX_FactCrossDock_Time (TimeKey),
    INDEX IX_FactCrossDock_Dwell (DwellMinutes DESC)
);
```

### 2. Populate Dimension Tables

```sql
-- Generate time dimension (15-minute buckets for one year)
DECLARE @StartDate DATETIME2 = '2026-01-01 00:00:00';
DECLARE @EndDate DATETIME2 = '2026-12-31 23:59:59';
DECLARE @CurrentDate DATETIME2 = @StartDate;
DECLARE @TimeKey INT = 1;

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO DimTime (TimeKey, DateTimeUTC, FifteenMinuteBucket, HourOfDay, DayOfWeek, DayOfMonth, WeekOfYear, FiscalPeriod, IsBusinessHour)
    VALUES (
        @TimeKey,
        @CurrentDate,
        DATEPART(MINUTE, @CurrentDate) / 15,
        DATEPART(HOUR, @CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        DATEPART(DAY, @CurrentDate),
        DATEPART(WEEK, @CurrentDate),
        CONCAT('FY26-Q', DATEPART(QUARTER, @CurrentDate)),
        CASE WHEN DATEPART(HOUR, @CurrentDate) BETWEEN 8 AND 17 AND DATEPART(WEEKDAY, @CurrentDate) BETWEEN 2 AND 6 THEN 1 ELSE 0 END
    );
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate);
    SET @TimeKey = @TimeKey + 1;
END;
```

### 3. Configure Data Ingestion

**Stored procedure for incremental loading from WMS:**

```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OperationStartTime, OperationEndTime, DurationMinutes,
        QuantityHandled, DwellTimeHours, StorageZone, EmployeeID, BatchNumber
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        wms.OperationType,
        wms.StartTime,
        wms.EndTime,
        DATEDIFF(MINUTE, wms.StartTime, wms.EndTime),
        wms.Quantity,
        CASE WHEN wms.OperationType = 'Putaway' 
            THEN DATEDIFF(HOUR, wms.EndTime, COALESCE(next_op.StartTime, GETUTCDATE()))
            ELSE NULL END,
        wms.Zone,
        wms.EmployeeID,
        wms.BatchNumber
    FROM ExternalWMS.dbo.Operations wms
    INNER JOIN DimTime t ON DATEADD(MINUTE, -(DATEPART(MINUTE, wms.StartTime) % 15), wms.StartTime) = t.DateTimeUTC
    INNER JOIN DimGeography g ON wms.WarehouseID = g.NodeID
    INNER JOIN DimProductGravity p ON wms.SKU = p.SKU
    LEFT JOIN ExternalWMS.dbo.Operations next_op 
        ON wms.BatchNumber = next_op.BatchNumber 
        AND next_op.OperationType = 'Pick'
        AND next_op.StartTime > wms.EndTime
    WHERE wms.StartTime > @LastLoadTime
        AND wms.StartTime <= GETUTCDATE();
END;
GO
```

**SQL Server Agent job for scheduled refresh:**

```sql
-- Create job to run every 15 minutes
USE msdb;
GO

EXEC dbo.sp_add_job
    @job_name = N'LogiFleet_IncrementalLoad',
    @enabled = 1,
    @description = N'Incremental load from WMS and telematics systems';

EXEC sp_add_jobstep
    @job_name = N'LogiFleet_IncrementalLoad',
    @step_name = N'Load Warehouse Operations',
    @subsystem = N'TSQL',
    @database_name = N'LogiFleetPulse',
    @command = N'
        DECLARE @LastLoad DATETIME2;
        SELECT @LastLoad = MAX(OperationStartTime) FROM FactWarehouseOperations;
        EXEC sp_LoadWarehouseOperations @LastLoadTime = @LastLoad;
    ';

EXEC sp_add_schedule
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC sp_attach_schedule
    @job_name = N'LogiFleet_IncrementalLoad',
    @schedule_name = N'Every15Minutes';

EXEC sp_add_jobserver
    @job_name = N'LogiFleet_IncrementalLoad';
GO
```

### 4. Power BI Configuration

**Connection string format:**

```
Server=YOUR_SQL_SERVER;Database=LogiFleetPulse;Trusted_Connection=True;
```

Use environment variables for production:

```
Server=$env:LOGIFLEET_SQL_SERVER;Database=$env:LOGIFLEET_DB;User ID=$env:LOGIFLEET_USER;Password=$env:LOGIFLEET_PASSWORD;
```

## Key Analytics Patterns

### Cross-Fact KPI: Dwell Time vs Fleet Idle Cost

```sql
-- Find products with high warehouse dwell time that correlate with fleet delays
WITH WarehouseDwell AS (
    SELECT 
        p.SKU,
        p.ProductName,
        AVG(w.DwellTimeHours) AS AvgDwellHours,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE w.OperationType = 'Putaway'
        AND w.OperationStartTime >= DATEADD(DAY, -30, GETUTCDATE())
    GROUP BY p.SKU, p.ProductName
    HAVING AVG(w.DwellTimeHours) > 48
),
FleetImpact AS (
    SELECT 
        p.SKU,
        SUM(f.IdleTimeMinutes * 0.5) AS EstimatedIdleCostUSD, -- $0.5 per idle minute
        COUNT(f.TripKey) AS AffectedTrips
    FROM FactFleetTrips f
    INNER JOIN FactWarehouseOperations w 
        ON f.TripStartTime > w.OperationEndTime
        AND f.TripStartTime < DATEADD(HOUR, 2, w.OperationEndTime)
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE f.DelayReason = 'Loading'
    GROUP BY p.SKU
)
SELECT 
    d.SKU,
    d.ProductName,
    d.AvgDwellHours,
    d.OperationCount,
    COALESCE(fi.EstimatedIdleCostUSD, 0) AS IdleCostImpact,
    COALESCE(fi.AffectedTrips, 0) AS DelayedTrips
FROM WarehouseDwell d
LEFT JOIN FleetImpact fi ON d.SKU = fi.SKU
ORDER BY COALESCE(fi.EstimatedIdleCostUSD, 0) DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Recalculate gravity scores based on recent velocity
UPDATE DimProductGravity
SET 
    VelocityScore = recent.PicksPerDay,
    GravityScore = (recent.PicksPerDay * UnitValue) / NULLIF(FragilityScore, 0),
    RecommendedZone = CASE 
        WHEN (recent.PicksPerDay * UnitValue) / NULLIF(FragilityScore, 0) > 100 THEN 'HighGravity'
        WHEN (recent.PicksPerDay * UnitValue) / NULLIF(FragilityScore, 0) > 50 THEN 'MediumGravity'
        ELSE 'LowGravity'
    END
FROM DimProductGravity p
INNER JOIN (
    SELECT 
        ProductKey,
        COUNT(*) / 30.0 AS PicksPerDay
    FROM FactWarehouseOperations
    WHERE OperationType = 'Pick'
        AND OperationStartTime >= DATEADD(DAY, -30, GETUTCDATE())
    GROUP BY ProductKey
) recent ON p.ProductKey = recent.ProductKey;

-- Find misplaced products (high gravity in low gravity zones)
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.RecommendedZone,
    w.StorageZone AS CurrentZone,
    COUNT(*) AS PicksLast7Days
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE w.OperationType = 'Pick'
    AND w.OperationStartTime >= DATEADD(DAY, -7, GETUTCDATE())
    AND p.RecommendedZone = 'HighGravity'
    AND w.StorageZone NOT LIKE '%High%'
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.RecommendedZone, w.StorageZone
ORDER BY COUNT(*) DESC;
```

### Adaptive Fleet Triage Engine

```sql
-- Prioritize vehicle maintenance by revenue impact
WITH VehicleIssues AS (
    SELECT 
        VehicleID,
        COUNT(*) AS TripsWithIssues,
        SUM(CASE WHEN EngineDiagnosticFlag = 1 THEN 1 ELSE 0 END) AS EngineFlagCount,
        SUM(CASE WHEN TirePressureWarning = 1 THEN 1 ELSE 0 END) AS TireFlagCount,
        AVG(LoadWeightKG) AS AvgLoadWeight,
        SUM(DelayMinutes) AS TotalDelayMinutes
    FROM FactFleetTrips
    WHERE TripStartTime >= DATEADD(DAY, -7, GETUTCDATE())
        AND (EngineDiagnosticFlag = 1 OR TirePressureWarning = 1)
    GROUP BY VehicleID
),
RevenueAtRisk AS (
    SELECT 
        f.VehicleID,
        SUM(p.UnitValue * w.QuantityHandled) AS TotalCargoValue
    FROM FactFleetTrips f
    INNER JOIN FactWarehouseOperations w 
        ON f.OriginGeographyKey = w.GeographyKey
        AND w.OperationStartTime BETWEEN DATEADD(HOUR, -2, f.TripStartTime) AND f.TripStartTime
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE f.TripStartTime >= DATEADD(DAY, -7, GETUTCDATE())
    GROUP BY f.VehicleID
)
SELECT 
    vi.VehicleID,
    vi.TripsWithIssues,
    vi.EngineFlagCount,
    vi.TireFlagCount,
    vi.TotalDelayMinutes,
    COALESCE(r.TotalCargoValue, 0) AS RevenueAtRisk,
    CASE 
        WHEN vi.EngineFlagCount > 3 THEN 'Critical'
        WHEN vi.TireFlagCount > 2 AND r.TotalCargoValue > 50000 THEN 'High'
        WHEN vi.TotalDelayMinutes > 120 THEN 'Medium'
        ELSE 'Low'
    END AS MaintenancePriority
FROM VehicleIssues vi
LEFT JOIN RevenueAtRisk r ON vi.VehicleID = r.VehicleID
ORDER BY 
    CASE 
        WHEN vi.EngineFlagCount > 3 THEN 1
        WHEN vi.TireFlagCount > 2 AND r.TotalCargoValue > 50000 THEN 2
        WHEN vi.TotalDelayMinutes > 120 THEN 3
        ELSE 4
    END,
    COALESCE(r.TotalCargoValue, 0) DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify time periods with converging warehouse and fleet pressure
WITH WarehousePressure AS (
    SELECT 
        t.HourOfDay,
        t.DayOfWeek,
        COUNT(*) AS OperationCount,
        AVG(DurationMinutes) AS AvgDuration
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE w.OperationStartTime >= DATEADD(DAY, -30, GETUTCDATE())
    GROUP BY t.HourOfDay, t.DayOfWeek
),
FleetPressure AS (
    SELECT 
        t.HourOfDay,
        t.DayOfWeek,
        COUNT(*) AS TripCount,
        AVG(DelayMinutes) AS AvgDelay,
        AVG(IdleTimeMinutes) AS AvgIdleTime
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE f.TripStartTime >= DATEADD(DAY, -30, GETUTCDATE())
    GROUP BY t.HourOfDay, t.DayOfWeek
)
SELECT 
    wp.HourOfDay,
    wp.DayOfWeek,
    wp.OperationCount AS WarehouseOps,
    wp.AvgDuration AS WarehouseAvgMinutes,
    fp.TripCount AS FleetTrips,
    fp.AvgDelay AS FleetAvgDelayMinutes,
    fp.AvgIdleTime AS FleetAvgIdleMinutes,
    (wp.OperationCount * 0.4 + fp.TripCount * 0.3 + fp.AvgDelay * 0.3) AS BottleneckIndex
FROM WarehousePressure wp
INNER JOIN FleetPressure fp ON wp.HourOfDay = fp.HourOfDay AND wp.DayOfWeek = fp.DayOfWeek
WHERE (wp.OperationCount * 0.4 + fp.TripCount * 0.3 + fp.AvgDelay * 0.3) > 50
ORDER BY (wp.OperationCount * 0.4 + fp.TripCount * 0.3 + fp.AvgDelay * 0.3) DESC;
```

## Power BI DAX Measures

### Cross-Fact Average Dwell Time

```dax
Avg Dwell Time (hrs) = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeHours]),
    FactWarehouseOperations[OperationType] = "Putaway"
)
```

### Fleet Utilization Rate

```dax
Fleet Utilization % = 
DIVIDE(
    SUMX(
        FactFleetTrips,
        FactFleetTrips[TripDurationMinutes] - FactFleetTrips[IdleTimeMinutes]
    ),
    SUM(FactFleetTrips[TripDurationMinutes]),
    0
) * 100
```

### Gravity Zone Compliance

```dax
Gravity Compliance % = 
VAR CorrectPlacements = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[StorageZone] = DimProductGravity[RecommendedZone]
    )
VAR TotalPlacements = COUNTROWS(FactWarehouseOperations)
RETURN
    DIVIDE(CorrectPlacements, TotalPlacements, 0) * 100
```

### Delay Cost Impact

```dax
Delay Cost ($) = 
SUMX(
    FactFleetTrips,
    FactFleetTrips[DelayMinutes] * 0.75 // $0.75 per delay minute
)
```

## Automated Alerting

```sql
-- Stored procedure for KPI threshold alerts
CREATE PROCEDURE sp_CheckKPIAlerts
AS
BEGIN
    DECLARE @AlertMessages TABLE (
        AlertType NVARCHAR(50),
        Severity NVARCHAR(20),
        Message NVARCHAR(500)
    );
    
    -- Check fleet idle time
    INSERT INTO @AlertMessages
    SELECT 
        'Fleet Idle Breach' AS AlertType,
        'High' AS Severity,
        CONCAT('Vehicle ', VehicleID, ' idle time ', CAST(IdleTimeMinutes AS VARCHAR), 
               ' minutes exceeds threshold on trip ', CAST(TripKey AS VARCHAR)) AS Message
    FROM FactFleetTrips
    WHERE TripStartTime >= DATEADD(HOUR, -2, GETUTCDATE())
        AND (IdleTimeMinutes / NULLIF(TripDurationMinutes, 0)) > 0.15;
    
    -- Check warehouse dwell time
    INSERT INTO @AlertMessages
    SELECT 
        'Warehouse Dwell Breach' AS AlertType,
        'Medium' AS Severity,
        CONCAT('SKU ', p.SKU, ' dwell time ', CAST(w.DwellTimeHours AS VARCHAR), 
               ' hours in zone ', w.StorageZone) AS Message
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE w.OperationStartTime >= DATEADD(HOUR, -4, GETUTCDATE())
        AND w.DwellTimeHours > 72;
    
    -- Check gravity zone misplacements
    INSERT INTO @AlertMessages
    SELECT 
        'Gravity Misplacement' AS AlertType,
        'Low' AS Severity,
        CONCAT('High-gravity SKU ', p.SKU, ' placed in ', w.StorageZone, 
               ', recommend ', p.RecommendedZone) AS Message
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE w.OperationType = 'Putaway'
        AND w.OperationStartTime >= DATEADD(HOUR, -1, GETUTCDATE())
        AND p.RecommendedZone = 'HighGravity'
        AND w.StorageZone NOT LIKE '%High%';
    
    -- Send alerts (integrate with email/Teams via SQL Server Database Mail or external service)
    SELECT * FROM @AlertMessages
    ORDER BY 
        CASE Severity 
            WHEN 'Critical' THEN 1
            WHEN 'High' THEN 2
            WHEN 'Medium' THEN 3
            ELSE 4
        END;
END;
GO

-- Schedule alert check every hour
EXEC sp_add_job
    @job_name = N'LogiFleet_AlertMonitor',
    @enabled = 1;

EXEC sp_add_jobstep
    @job_name = N'LogiFleet_AlertMonitor',
    @step_name = N'Check KPI Alerts',
    @subsystem = N'TSQL',
    @database_name = N'LogiFleetPulse',
    @command = N'EXEC sp_CheckKPIAlerts';

EXEC sp_add_schedule
    @schedule_name = N'Hourly',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 8, -- Hourly
    @freq_subday_interval = 1;

EXEC sp_attach_schedule
    @job_name = N'LogiFleet_AlertMonitor',
    @schedule_name = N'Hourly';

EXEC sp_add_jobserver
    @job_name = N'LogiFleet_AlertMonitor';
```

## Row-Level Security

```sql
-- Create security table
CREATE TABLE SecurityRoles (
    UserPrincipalName NVARCHAR(255) PRIMARY KEY,
    Role NVARCHAR(50), -- 'Executive', 'Supervisor', 'User'
    GeographyFilter NVARCHAR(100), -- NULL for all, specific NodeID for restricted
    CanViewCost BIT
);

-- Add security function
CREATE FUNCTION fn_SecurityFilter(@UserEmail NVARCHAR(255))
RETURNS TABLE
AS
RETURN
(
    SELECT GeographyFilter, CanViewCost
    FROM SecurityRoles
    WHERE UserPrincipalName = @UserEmail
);
GO

-- Apply in Power BI using USERNAME() function
-- DAX filter: LOOKUPVALUE(SecurityRoles[GeographyFilter], SecurityRoles[UserPrincipalName], USERNAME())
```

## Troubleshooting

### Slow Query Performance

**Issue:** Cross-fact queries timeout  
**Solution:** Add filtered indexes on time ranges

```sql
-- Partition fact tables by month
ALTER TABLE FactWarehouseOperations
ADD OperationMonth AS DATEPART(MONTH, OperationStartTime) PERSISTED;

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Month_Product
ON FactWarehouseOperations (OperationMonth, ProductKey)
INCLUDE (DurationMinutes, DwellTimeHours);
```

### Power BI Refresh Failures

**Issue:** Gateway timeout during scheduled refresh  
**Solution:** Implement incremental refresh

```powerquery
// In Power Query Advanced Editor
let
    Source = Sql.Database(ServerName, DatabaseName),
    FilteredRows = Table.SelectRows(Source, 
        each [OperationStartTime] >= RangeStart and [OperationStartTime] < RangeEnd)
in
    FilteredRows
```

Configure incremental refresh policy in Power BI Desktop:
- Store rows in last 3 years
- Refresh rows in last 7 days
- Detect data changes: No

### Dimension Key Mismatches

**Issue:** Products or locations not linking to facts  
**Solution:** Implement SCD Type 2 tracking

```sql
ALTER TABLE DimProductGravity
ADD 
    EffectiveDate DATETIME2 DEFAULT GETUTCDATE(),
    ExpirationDate DATETIME2 DEFAULT '9999-12-31',
    IsCurrent BIT DEFAULT 1;

-- Update procedure to handle changes
CREATE PROCEDURE sp_UpsertProduct
    @SKU NVARCHAR(50),
    @ProductName NVARCHAR(200),
    @Category NVARCHAR(100)
AS
BEGIN
    IF EXISTS (SELECT 1 FROM DimProductGravity WHERE SKU = @SKU AND IsCurrent = 1)
    BEGIN
        UPDATE DimProductGravity
        SET ExpirationDate = GETUTCDATE(), IsCurrent = 0
        WHERE SKU = @SKU AND IsCurrent = 1;
    END
    
    INSERT INTO
