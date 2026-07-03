---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform with multi-fact star schema for warehouse, fleet, and supply chain KPI harmonization
triggers:
  - "set up logifleet pulse logistics dashboard"
  - "configure supply chain analytics warehouse"
  - "deploy power bi fleet optimization template"
  - "build multi-fact star schema for logistics"
  - "integrate warehouse and fleet telemetry data"
  - "create cross-modal supply chain kpi reports"
  - "implement real-time logistics intelligence engine"
  - "optimize warehouse gravity zones with sql"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a **multi-modal logistics intelligence engine** that unifies warehouse operations, fleet telemetry, inventory management, and supply chain analytics into a single semantic layer. Built on MS SQL Server with Power BI visualization, it provides:

- **Multi-fact star schema** linking warehouse operations, fleet trips, and cross-dock activities
- **Time-phased dimensions** with 15-minute granularity for real-time operational awareness
- **Cross-fact KPI harmonization** (e.g., inventory dwell time vs. fleet fuel efficiency)
- **Warehouse Gravity Zones™** spatial optimization based on pick frequency and item value
- **Predictive bottleneck detection** using temporal elasticity modeling
- **Role-based dashboards** with row-level security

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access credentials for source systems (WMS, TMS, telemetry APIs)

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Run the schema deployment script

USE master;
GO

-- Create the LogiFleet database
CREATE DATABASE LogiFleetPulse
ON PRIMARY 
(
    NAME = 'LogiFleetPulse_Data',
    FILENAME = 'D:\SQLData\LogiFleetPulse_Data.mdf',
    SIZE = 10GB,
    FILEGROWTH = 1GB
)
LOG ON
(
    NAME = 'LogiFleetPulse_Log',
    FILENAME = 'D:\SQLData\LogiFleetPulse_Log.ldf',
    SIZE = 2GB,
    FILEGROWTH = 512MB
);
GO

USE LogiFleetPulse;
GO

-- Create core dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    TimeSlot15Min VARCHAR(5),
    HourOfDay TINYINT,
    DayOfWeek VARCHAR(10),
    FiscalPeriod VARCHAR(7),
    IsWeekend BIT,
    INDEX IX_DimTime_DateTime NONCLUSTERED (FullDateTime)
);

CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID VARCHAR(50) NOT NULL,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'Route Node', 'Cross-Dock'
    Continent VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    INDEX IX_DimGeo_LocationID NONCLUSTERED (LocationID)
);

CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(500),
    Category VARCHAR(200),
    Subcategory VARCHAR(200),
    GravityScore DECIMAL(5,2), -- Warehouse Gravity Zone score
    IsPerishable BIT,
    IsFragile BIT,
    StandardWeight DECIMAL(10,2),
    StandardVolume DECIMAL(10,2),
    INDEX IX_DimProduct_SKU NONCLUSTERED (SKU),
    INDEX IX_DimProduct_GravityScore NONCLUSTERED (GravityScore DESC)
);

CREATE TABLE DimSupplier (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierID VARCHAR(100) NOT NULL UNIQUE,
    SupplierName VARCHAR(500),
    LeadTimeDaysAvg DECIMAL(5,1),
    DefectRatePercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2),
    INDEX IX_DimSupplier_ID NONCLUSTERED (SupplierID)
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled INT,
    DwellTimeMinutes DECIMAL(10,2),
    CycleTimeMinutes DECIMAL(10,2),
    ZoneID VARCHAR(50),
    OperatorID VARCHAR(100),
    CONSTRAINT FK_FactWH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactWH_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactWH_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

CREATE COLUMNSTORE INDEX IX_FactWH_CS ON FactWarehouseOperations
(TimeKey, GeographyKey, ProductKey, OperationType, QuantityHandled, DwellTimeMinutes);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleID VARCHAR(100) NOT NULL,
    OriginGeographyKey INT,
    DestinationGeographyKey INT,
    DistanceKM DECIMAL(10,2),
    DurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKG DECIMAL(10,2),
    DelayMinutes DECIMAL(10,2),
    DelayReason VARCHAR(200),
    CONSTRAINT FK_FactFleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactFleet_Dest FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE COLUMNSTORE INDEX IX_FactFleet_CS ON FactFleetTrips
(TimeKey, VehicleID, OriginGeographyKey, DestinationGeographyKey, DurationMinutes, IdleTimeMinutes);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundShipmentID VARCHAR(100),
    OutboundShipmentID VARCHAR(100),
    QuantityTransferred INT,
    TransferTimeMinutes DECIMAL(10,2),
    CONSTRAINT FK_FactCD_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactCD_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactCD_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

CREATE COLUMNSTORE INDEX IX_FactCD_CS ON FactCrossDock
(TimeKey, GeographyKey, ProductKey, QuantityTransferred);
```

### Step 2: Populate Time Dimension

```sql
-- Populate DimTime with 15-minute intervals for 3 years
DECLARE @StartDate DATETIME2 = '2024-01-01 00:00:00';
DECLARE @EndDate DATETIME2 = '2027-12-31 23:45:00';
DECLARE @CurrentDate DATETIME2 = @StartDate;

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO DimTime (TimeKey, FullDateTime, TimeSlot15Min, HourOfDay, DayOfWeek, FiscalPeriod, IsWeekend)
    VALUES (
        CAST(FORMAT(@CurrentDate, 'yyyyMMddHHmm') AS INT),
        @CurrentDate,
        FORMAT(@CurrentDate, 'HH:mm'),
        DATEPART(HOUR, @CurrentDate),
        DATENAME(WEEKDAY, @CurrentDate),
        FORMAT(@CurrentDate, 'yyyy-MM'),
        CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1, 7) THEN 1 ELSE 0 END
    );
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate);
END;
```

### Step 3: Create ETL Stored Procedures

```sql
-- ETL procedure for warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LoadDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from staging table connected to WMS
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        QuantityHandled, DwellTimeMinutes, CycleTimeMinutes, ZoneID, OperatorID
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        stg.OperationType,
        stg.Quantity,
        stg.DwellTime,
        stg.CycleTime,
        stg.ZoneID,
        stg.OperatorID
    FROM Staging.WarehouseOps stg
    INNER JOIN DimTime t ON CAST(stg.OperationDateTime AS DATETIME2(0)) = t.FullDateTime
    INNER JOIN DimGeography g ON stg.LocationID = g.LocationID
    INNER JOIN DimProduct p ON stg.SKU = p.SKU
    WHERE CAST(stg.OperationDateTime AS DATE) = @LoadDate;
END;
GO

-- ETL procedure for fleet trips
CREATE PROCEDURE sp_LoadFleetTrips
    @LoadDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        TimeKey, VehicleID, OriginGeographyKey, DestinationGeographyKey,
        DistanceKM, DurationMinutes, IdleTimeMinutes, FuelConsumedLiters,
        LoadWeightKG, DelayMinutes, DelayReason
    )
    SELECT 
        t.TimeKey,
        stg.VehicleID,
        go.GeographyKey,
        gd.GeographyKey,
        stg.Distance,
        stg.Duration,
        stg.IdleTime,
        stg.FuelConsumed,
        stg.LoadWeight,
        stg.DelayMinutes,
        stg.DelayReason
    FROM Staging.FleetTrips stg
    INNER JOIN DimTime t ON CAST(stg.TripStartDateTime AS DATETIME2(0)) = t.FullDateTime
    INNER JOIN DimGeography go ON stg.OriginLocationID = go.LocationID
    INNER JOIN DimGeography gd ON stg.DestinationLocationID = gd.LocationID
    WHERE CAST(stg.TripStartDateTime AS DATE) = @LoadDate;
END;
GO
```

### Step 4: Create Analytical Views

```sql
-- Cross-fact view: Warehouse dwell time vs Fleet efficiency
CREATE VIEW vw_DwellTimeVsFleetEfficiency
AS
SELECT 
    t.FiscalPeriod,
    g.Region,
    p.Category,
    AVG(wh.DwellTimeMinutes) AS AvgDwellTime,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleTime,
    AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKM, 0)) AS AvgFuelEfficiency,
    COUNT(DISTINCT wh.OperationKey) AS TotalWarehouseOps,
    COUNT(DISTINCT ft.TripKey) AS TotalFleetTrips
FROM FactWarehouseOperations wh
INNER JOIN DimTime t ON wh.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wh.GeographyKey = g.GeographyKey
INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.TimeKey AND g.GeographyKey = ft.OriginGeographyKey
GROUP BY t.FiscalPeriod, g.Region, p.Category;
GO

-- Warehouse Gravity Zone performance
CREATE VIEW vw_GravityZonePerformance
AS
SELECT 
    p.GravityScore,
    CASE 
        WHEN p.GravityScore >= 8.0 THEN 'High Gravity'
        WHEN p.GravityScore >= 5.0 THEN 'Medium Gravity'
        ELSE 'Low Gravity'
    END AS GravityZone,
    COUNT(*) AS OperationCount,
    AVG(wh.CycleTimeMinutes) AS AvgCycleTime,
    AVG(wh.DwellTimeMinutes) AS AvgDwellTime,
    SUM(wh.QuantityHandled) AS TotalQuantity
FROM FactWarehouseOperations wh
INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
WHERE p.GravityScore IS NOT NULL
GROUP BY p.GravityScore;
GO
```

### Step 5: Configure Power BI Connection

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. When prompted, enter connection details:
   - **Server**: Your SQL Server instance name
   - **Database**: LogiFleetPulse
   - **Authentication**: Windows or SQL Server authentication

```m
// Power Query M code for data source configuration
let
    Source = Sql.Database(
        #"SQL_SERVER_HOST", 
        "LogiFleetPulse",
        [
            Query = "SELECT * FROM vw_DwellTimeVsFleetEfficiency"
        ]
    )
in
    Source
```

3. Set up incremental refresh for large fact tables:

```m
// Incremental refresh parameters
let
    RangeStart = DateTime.From(#date(2024, 1, 1)),
    RangeEnd = DateTime.Now(),
    Source = Sql.Database("SQL_SERVER_HOST", "LogiFleetPulse"),
    FilteredRows = Table.SelectRows(
        Source, 
        each [FullDateTime] >= RangeStart and [FullDateTime] < RangeEnd
    )
in
    FilteredRows
```

## Configuration

### Environment Variables

Set these environment variables for data source connections:

```bash
# SQL Server connection
LOGIFLEET_SQL_SERVER=your-server.database.windows.net
LOGIFLEET_SQL_DATABASE=LogiFleetPulse
LOGIFLEET_SQL_USER=logifleet_user
LOGIFLEET_SQL_PASSWORD=your-secure-password

# WMS API connection
LOGIFLEET_WMS_API_URL=https://wms-api.example.com
LOGIFLEET_WMS_API_KEY=your-wms-api-key

# Fleet telemetry API
LOGIFLEET_FLEET_API_URL=https://fleet-telemetry.example.com
LOGIFLEET_FLEET_API_TOKEN=your-fleet-api-token
```

### Row-Level Security Setup

```sql
-- Create security table for role-based access
CREATE TABLE SecurityUserRole (
    UserEmail VARCHAR(200) PRIMARY KEY,
    RoleName VARCHAR(100),
    AllowedRegions VARCHAR(MAX), -- Comma-separated list
    AllowedWarehouses VARCHAR(MAX)
);

-- Sample data
INSERT INTO SecurityUserRole VALUES 
('manager@company.com', 'Regional Manager', 'North America,Europe', NULL),
('operator@company.com', 'Warehouse Operator', NULL, 'WH-001,WH-002');

-- Create RLS function
CREATE FUNCTION fn_SecurityPredicate(@Region VARCHAR(100))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS SecurityPredicateResult
WHERE 
    @Region IN (
        SELECT value 
        FROM STRING_SPLIT(
            (SELECT AllowedRegions FROM dbo.SecurityUserRole WHERE UserEmail = USER_NAME()),
            ','
        )
    )
    OR USER_NAME() = 'dbo'; -- Admin bypass
GO

-- Apply security policy
CREATE SECURITY POLICY RegionSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(Region)
ON dbo.DimGeography
WITH (STATE = ON);
```

### Automated Alert Configuration

```sql
-- Create alert threshold table
CREATE TABLE AlertThresholds (
    AlertID INT IDENTITY(1,1) PRIMARY KEY,
    AlertName VARCHAR(200),
    MetricName VARCHAR(100),
    ThresholdValue DECIMAL(10,2),
    ComparisonOperator VARCHAR(10), -- '>', '<', '='
    NotificationEmail VARCHAR(MAX), -- Comma-separated
    IsActive BIT DEFAULT 1
);

-- Sample alert: Fleet idle time exceeds 15%
INSERT INTO AlertThresholds VALUES
('Fleet High Idle Time', 'IdleTimePercent', 15.0, '>', 'fleet-manager@company.com', 1);

-- Stored procedure to check alerts
CREATE PROCEDURE sp_CheckAlerts
AS
BEGIN
    DECLARE @CurrentIdlePercent DECIMAL(5,2);
    
    SELECT @CurrentIdlePercent = 
        AVG(IdleTimeMinutes * 100.0 / NULLIF(DurationMinutes, 0))
    FROM FactFleetTrips
    WHERE TimeKey >= (SELECT MAX(TimeKey) FROM DimTime WHERE FullDateTime >= DATEADD(HOUR, -24, GETDATE()));
    
    IF @CurrentIdlePercent > (SELECT ThresholdValue FROM AlertThresholds WHERE MetricName = 'IdleTimePercent')
    BEGIN
        -- Send email alert (integrate with sp_send_dbmail)
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = (SELECT NotificationEmail FROM AlertThresholds WHERE MetricName = 'IdleTimePercent'),
            @subject = 'ALERT: Fleet Idle Time Exceeded Threshold',
            @body = 'Current fleet idle time percentage: ' + CAST(@CurrentIdlePercent AS VARCHAR(10));
    END;
END;
GO
```

## Key SQL Queries & Patterns

### Cross-Fact Analysis: Warehouse Efficiency Impact on Fleet Performance

```sql
-- Find correlation between warehouse dwell time and fleet delivery delays
WITH WarehouseDwell AS (
    SELECT 
        t.FiscalPeriod,
        g.Region,
        AVG(wh.DwellTimeMinutes) AS AvgDwellTime
    FROM FactWarehouseOperations wh
    INNER JOIN DimTime t ON wh.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON wh.GeographyKey = g.GeographyKey
    WHERE wh.OperationType = 'Shipping'
    GROUP BY t.FiscalPeriod, g.Region
),
FleetDelays AS (
    SELECT 
        t.FiscalPeriod,
        g.Region,
        AVG(ft.DelayMinutes) AS AvgDelay,
        SUM(CASE WHEN ft.DelayMinutes > 30 THEN 1 ELSE 0 END) AS MajorDelayCount
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON ft.OriginGeographyKey = g.GeographyKey
    GROUP BY t.FiscalPeriod, g.Region
)
SELECT 
    wd.FiscalPeriod,
    wd.Region,
    wd.AvgDwellTime,
    fd.AvgDelay,
    fd.MajorDelayCount,
    -- Calculate correlation strength
    CASE 
        WHEN wd.AvgDwellTime > 120 AND fd.AvgDelay > 45 THEN 'High Impact'
        WHEN wd.AvgDwellTime > 60 AND fd.AvgDelay > 20 THEN 'Medium Impact'
        ELSE 'Low Impact'
    END AS ImpactLevel
FROM WarehouseDwell wd
INNER JOIN FleetDelays fd ON wd.FiscalPeriod = fd.FiscalPeriod AND wd.Region = fd.Region
ORDER BY wd.FiscalPeriod, wd.AvgDwellTime DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products that should be moved to different gravity zones
WITH ProductPerformance AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        p.GravityScore AS CurrentGravityScore,
        COUNT(*) AS PickFrequency,
        AVG(wh.CycleTimeMinutes) AS AvgCycleTime,
        SUM(wh.QuantityHandled * p.StandardWeight) AS TotalValueHandled
    FROM FactWarehouseOperations wh
    INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
    WHERE wh.OperationType = 'Picking'
        AND wh.TimeKey >= (SELECT MAX(TimeKey) FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()))
    GROUP BY p.ProductKey, p.SKU, p.ProductName, p.GravityScore
),
OptimalGravity AS (
    SELECT 
        *,
        -- Calculate optimal gravity score based on frequency and value
        CASE 
            WHEN PickFrequency > 500 AND TotalValueHandled > 10000 THEN 9.5
            WHEN PickFrequency > 300 AND TotalValueHandled > 5000 THEN 8.0
            WHEN PickFrequency > 100 THEN 6.0
            ELSE 3.0
        END AS OptimalGravityScore
    FROM ProductPerformance
)
SELECT 
    SKU,
    ProductName,
    CurrentGravityScore,
    OptimalGravityScore,
    OptimalGravityScore - CurrentGravityScore AS GravityScoreDelta,
    PickFrequency,
    AvgCycleTime,
    TotalValueHandled,
    CASE 
        WHEN OptimalGravityScore - CurrentGravityScore > 2.0 THEN 'Move to Higher Gravity Zone'
        WHEN OptimalGravityScore - CurrentGravityScore < -2.0 THEN 'Move to Lower Gravity Zone'
        ELSE 'Current Zone OK'
    END AS Recommendation
FROM OptimalGravity
WHERE ABS(OptimalGravityScore - CurrentGravityScore) > 1.0
ORDER BY ABS(OptimalGravityScore - CurrentGravityScore) DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify upcoming bottlenecks using time-series trends
WITH HourlyMetrics AS (
    SELECT 
        t.HourOfDay,
        t.DayOfWeek,
        AVG(wh.CycleTimeMinutes) AS AvgCycleTime,
        COUNT(*) AS OperationVolume,
        STDEV(wh.CycleTimeMinutes) AS CycleTimeVariance
    FROM FactWarehouseOperations wh
    INNER JOIN DimTime t ON wh.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -14, GETDATE())
    GROUP BY t.HourOfDay, t.DayOfWeek
),
Thresholds AS (
    SELECT 
        AVG(AvgCycleTime) + (2 * STDEV(AvgCycleTime)) AS CycleTimeThreshold,
        AVG(OperationVolume) + (2 * STDEV(OperationVolume)) AS VolumeThreshold
    FROM HourlyMetrics
)
SELECT 
    hm.DayOfWeek,
    hm.HourOfDay,
    hm.AvgCycleTime,
    hm.OperationVolume,
    hm.CycleTimeVariance,
    t.CycleTimeThreshold,
    t.VolumeThreshold,
    CASE 
        WHEN hm.AvgCycleTime > t.CycleTimeThreshold AND hm.OperationVolume > t.VolumeThreshold 
        THEN 'Critical Bottleneck Risk'
        WHEN hm.AvgCycleTime > t.CycleTimeThreshold OR hm.OperationVolume > t.VolumeThreshold 
        THEN 'Moderate Bottleneck Risk'
        ELSE 'Low Risk'
    END AS BottleneckRisk
FROM HourlyMetrics hm
CROSS JOIN Thresholds t
WHERE hm.AvgCycleTime > t.CycleTimeThreshold OR hm.OperationVolume > t.VolumeThreshold
ORDER BY hm.AvgCycleTime DESC;
```

### Fleet Maintenance Priority Queue

```sql
-- Generate maintenance priority based on vehicle telemetry and cargo value
WITH VehiclePerformance AS (
    SELECT 
        ft.VehicleID,
        COUNT(*) AS TripCount,
        SUM(ft.DistanceKM) AS TotalDistanceKM,
        AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKM, 0)) AS AvgFuelEfficiency,
        AVG(ft.IdleTimeMinutes * 100.0 / NULLIF(ft.DurationMinutes, 0)) AS AvgIdlePercent,
        MAX(ft.LoadWeightKG) AS MaxLoadWeight
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY ft.VehicleID
),
MaintenancePriority AS (
    SELECT 
        *,
        -- Calculate priority score (higher = more urgent)
        (
            (TotalDistanceKM / 1000.0) * 0.3 +  -- Distance factor
            (AvgFuelEfficiency - 8.0) * 10 * 0.2 +  -- Efficiency degradation
            (AvgIdlePercent - 10.0) * 5 * 0.3 +  -- Idle time factor
            (MaxLoadWeight / 1000.0) * 0.2  -- Heavy load factor
        ) AS PriorityScore
    FROM VehiclePerformance
)
SELECT 
    VehicleID,
    TripCount,
    TotalDistanceKM,
    ROUND(AvgFuelEfficiency, 2) AS AvgFuelEfficiency,
    ROUND(AvgIdlePercent, 2) AS AvgIdlePercent,
    MaxLoadWeight,
    ROUND(PriorityScore, 2) AS PriorityScore,
    CASE 
        WHEN PriorityScore > 20 THEN 'Urgent - Schedule within 24 hours'
        WHEN PriorityScore > 10 THEN 'High - Schedule within 1 week'
        WHEN PriorityScore > 5 THEN 'Medium - Schedule within 2 weeks'
        ELSE 'Low - Monitor'
    END AS MaintenanceRecommendation
FROM MaintenancePriority
ORDER BY PriorityScore DESC;
```

## Power BI DAX Measures

### Key Performance Indicators

```dax
// Total Warehouse Operations
Total Operations = 
COUNTROWS(FactWarehouseOperations)

// Average Dwell Time with trend
Avg Dwell Time = 
AVERAGE(FactWarehouseOperations[DwellTimeMinutes])

Avg Dwell Time LM = 
CALCULATE(
    [Avg Dwell Time],
    DATEADD(DimTime[FullDateTime], -1, MONTH)
)

Dwell Time % Change = 
DIVIDE(
    [Avg Dwell Time] - [Avg Dwell Time LM],
    [Avg Dwell Time LM],
    0
)

// Fleet Efficiency Score (0-100)
Fleet Efficiency Score = 
VAR AvgIdlePercent = 
    AVERAGEX(
        FactFleetTrips,
        DIVIDE(
            FactFleetTrips[IdleTimeMinutes],
            FactFleetTrips[DurationMinutes],
            0
        ) * 100
    )
VAR AvgDelayPercent = 
    AVERAGEX(
        FactFleetTrips,
        DIVIDE(
            FactFleetTrips[DelayMinutes],
            FactFleetTrips[DurationMinutes],
            0
        ) * 100
    )
VAR FuelEfficiency = 
    DIVIDE(
        SUM(FactFleetTrips[DistanceKM]),
        SUM(FactFleetTrips[FuelConsumedLiters]),
        0
    )
RETURN
    100 - (AvgIdlePercent * 0.4) - (AvgDelayPercent * 0.4) + (FuelEfficiency * 2)

// Cross-Fact KPI: Warehouse Impact on Fleet
Warehouse-Fleet Impact Index = 
VAR AvgShippingDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        FactWarehouseOperations[OperationType] = "Shipping"
    )
VAR AvgFleetDelay = 
    AVERAGE(FactFleetTrips[DelayMinutes])
RETURN
    DIVIDE(AvgShippingDwell, 60, 0) * DIVIDE(AvgFleetDelay, 30, 0) * 10

// Gravity Zone Effectiveness
Gravity Zone Effectiveness = 
VAR HighGravityItems = 
    CALCULATE(
        COUNT(DimProduct[ProductKey]),
        DimProduct[GravityScore] >= 8
    )
VAR HighGravityAvgCycleTime = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[CycleTimeMinutes]),
        DimProduct[GravityScore] >= 8
    )
VAR LowGravityAvgCycle
