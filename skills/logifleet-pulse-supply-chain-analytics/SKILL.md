---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing engine for logistics, fleet management, and supply chain intelligence with multi-fact star schema
triggers:
  - set up logistics analytics warehouse
  - configure logifleet pulse dashboard
  - implement supply chain data model
  - create fleet management power bi report
  - build warehouse operations star schema
  - integrate logistics intelligence platform
  - deploy supply chain analytics engine
  - configure cross-modal logistics tracking
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced logistics intelligence platform built on MS SQL Server and Power BI. It provides:

- **Multi-fact star schema** for warehouse operations, fleet trips, and cross-dock activities
- **Time-phased dimensions** with 15-minute granularity for real-time analytics
- **Cross-fact KPI harmonization** linking inventory, fleet, and operational metrics
- **Predictive analytics** for bottleneck detection and fleet maintenance prioritization
- **Warehouse Gravity Zones™** for spatial optimization based on pick frequency and value
- **Power BI dashboards** with role-based access and natural language query support

The platform integrates WMS (Warehouse Management Systems), telematics, GPS feeds, supplier portals, and external APIs for weather/traffic.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- Access to source systems (WMS, TMS, ERP) via SQL, REST API, or file exports

### Step 1: Deploy SQL Schema

```sql
-- Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create time dimension (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay TINYINT NOT NULL,
    MinuteBucket TINYINT NOT NULL, -- 0, 15, 30, 45
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    IsWeekend BIT NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    MonthNumber TINYINT NOT NULL,
    MonthName VARCHAR(10) NOT NULL,
    Quarter TINYINT NOT NULL,
    Year SMALLINT NOT NULL,
    FiscalPeriod VARCHAR(10) NOT NULL
);

-- Create clustered columnstore index for performance
CREATE CLUSTERED COLUMNSTORE INDEX IX_DimTime_CCS ON DimTime;
GO

-- Create geography dimension (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) UNIQUE NOT NULL,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- Warehouse, Route Node, Distribution Center
    Address VARCHAR(500),
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50),
    IsActive BIT DEFAULT 1,
    ValidFrom DATETIME DEFAULT GETDATE(),
    ValidTo DATETIME NULL
);

CREATE NONCLUSTERED INDEX IX_DimGeography_LocationType ON DimGeography(LocationType, IsActive);
CREATE NONCLUSTERED INDEX IX_DimGeography_Hierarchy ON DimGeography(Continent, Country, Region);
GO

-- Create product dimension with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) UNIQUE NOT NULL,
    ProductName VARCHAR(500) NOT NULL,
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    ShelfLifeDays INT,
    UnitValue DECIMAL(12,2),
    -- Gravity scoring components
    VelocityScore DECIMAL(5,2), -- Pick frequency normalized 0-100
    ValueScore DECIMAL(5,2), -- Unit value normalized 0-100
    FragilityScore DECIMAL(5,2), -- Fragility/perishability 0-100
    GravityZone VARCHAR(20), -- High, Medium, Low
    CompositeGravity AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    LastRecalculated DATETIME DEFAULT GETDATE()
);

CREATE NONCLUSTERED INDEX IX_DimProduct_Gravity ON DimProductGravity(GravityZone, CompositeGravity DESC);
CREATE NONCLUSTERED INDEX IX_DimProduct_Category ON DimProductGravity(Category, SubCategory);
GO

-- Create supplier reliability dimension
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) UNIQUE NOT NULL,
    SupplierName VARCHAR(200) NOT NULL,
    Country VARCHAR(100),
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2), -- Standard deviation in days
    DefectRate DECIMAL(5,4), -- Percentage as decimal
    OnTimeDeliveryRate DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityTier VARCHAR(20), -- Platinum, Gold, Silver, Bronze
    LastEvaluated DATETIME DEFAULT GETDATE()
);

CREATE NONCLUSTERED INDEX IX_DimSupplier_Tier ON DimSupplierReliability(ReliabilityTier, ComplianceScore DESC);
GO

-- Create warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT NULL FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OrderID VARCHAR(50),
    BatchID VARCHAR(50),
    Quantity INT NOT NULL,
    StorageZone VARCHAR(50),
    OperationStartTime DATETIME NOT NULL,
    OperationEndTime DATETIME NOT NULL,
    CycleTimeMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime) PERSISTED,
    DwellTimeHours DECIMAL(10,2), -- Time in storage before next operation
    OperatorID VARCHAR(50),
    EquipmentID VARCHAR(50),
    ErrorCode VARCHAR(20) NULL,
    IsRework BIT DEFAULT 0
);

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey) INCLUDE (OperationType, CycleTimeMinutes);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey, GeographyKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Type ON FactWarehouseOperations(OperationType, TimeKey);
GO

-- Create fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    TripStartTime DATETIME NOT NULL,
    TripEndTime DATETIME NULL,
    PlannedDurationMinutes INT,
    ActualDurationMinutes AS DATEDIFF(MINUTE, TripStartTime, TripEndTime) PERSISTED,
    DistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT DEFAULT 0,
    LoadingTimeMinutes INT DEFAULT 0,
    UnloadingTimeMinutes INT DEFAULT 0,
    LoadWeightKg DECIMAL(10,2),
    NumberOfStops TINYINT DEFAULT 1,
    RouteID VARCHAR(50),
    WeatherCondition VARCHAR(50),
    TrafficDelayMinutes INT DEFAULT 0,
    MaintenanceAlerts INT DEFAULT 0,
    TirePressureAvg DECIMAL(5,2),
    EngineHealthScore DECIMAL(5,2), -- 0-100
    TripStatus VARCHAR(20) -- Completed, InProgress, Delayed, Cancelled
);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey) INCLUDE (TripStatus, ActualDurationMinutes);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID, TripStartTime);
CREATE NONCLUSTERED INDEX IX_FactFleet_Route ON FactFleetTrips(OriginGeographyKey, DestinationGeographyKey);
GO

-- Create cross-dock fact table
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT NULL FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT NULL FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    ArrivalTime DATETIME NOT NULL,
    DepartureTime DATETIME NOT NULL,
    DwellTimeMinutes AS DATEDIFF(MINUTE, ArrivalTime, DepartureTime) PERSISTED,
    Quantity INT NOT NULL,
    Priority VARCHAR(20), -- Critical, High, Normal, Low
    TemperatureZone VARCHAR(50) NULL
);

CREATE NONCLUSTERED INDEX IX_FactCrossDock_Time ON FactCrossDock(TimeKey) INCLUDE (DwellTimeMinutes);
CREATE NONCLUSTERED INDEX IX_FactCrossDock_Product ON FactCrossDock(ProductKey, Priority);
GO
```

### Step 2: Create Data Loading Procedures

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE usp_LoadWarehouseOperations
    @StartTime DATETIME,
    @EndTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Assuming source data in staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, OrderID, BatchID, Quantity,
        StorageZone, OperationStartTime, OperationEndTime,
        DwellTimeHours, OperatorID, EquipmentID, ErrorCode, IsRework
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        ds.SupplierKey,
        s.OperationType,
        s.OrderID,
        s.BatchID,
        s.Quantity,
        s.StorageZone,
        s.OperationStartTime,
        s.OperationEndTime,
        s.DwellTimeHours,
        s.OperatorID,
        s.EquipmentID,
        s.ErrorCode,
        s.IsRework
    FROM StagingWarehouseOps s
    INNER JOIN DimTime dt ON 
        dt.FullDateTime = DATEADD(MINUTE, 
            (DATEPART(MINUTE, s.OperationStartTime) / 15) * 15, 
            DATEADD(HOUR, DATEDIFF(HOUR, 0, s.OperationStartTime), 0))
    INNER JOIN DimGeography dg ON dg.LocationID = s.WarehouseID
    INNER JOIN DimProductGravity dp ON dp.SKU = s.SKU
    LEFT JOIN DimSupplierReliability ds ON ds.SupplierID = s.SupplierID
    WHERE s.OperationStartTime BETWEEN @StartTime AND @EndTime;
    
    -- Update product gravity scores based on new operations
    EXEC usp_RecalculateProductGravity;
END;
GO

-- Procedure to recalculate product gravity scores
CREATE PROCEDURE usp_RecalculateProductGravity
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH ProductMetrics AS (
        SELECT 
            ProductKey,
            -- Velocity: picks per day normalized
            CAST(COUNT(*) * 100.0 / NULLIF(MAX(PicksPerDay), 0) AS DECIMAL(5,2)) AS VelocityScore,
            -- Value score from product dimension (already normalized)
            AVG(CAST(p.UnitValue AS DECIMAL(5,2))) AS AvgValue
        FROM FactWarehouseOperations f
        INNER JOIN DimProductGravity p ON f.ProductKey = p.ProductKey
        WHERE f.OperationType = 'Picking'
            AND f.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
        CROSS APPLY (
            SELECT COUNT(DISTINCT CAST(OperationStartTime AS DATE)) AS PicksPerDay
            FROM FactWarehouseOperations
            WHERE OperationType = 'Picking'
                AND OperationStartTime >= DATEADD(DAY, -30, GETDATE())
        ) MaxPicks
        GROUP BY ProductKey
    )
    UPDATE p
    SET 
        VelocityScore = ISNULL(pm.VelocityScore, 0),
        ValueScore = CASE 
            WHEN p.UnitValue < 10 THEN 20
            WHEN p.UnitValue < 50 THEN 40
            WHEN p.UnitValue < 100 THEN 60
            WHEN p.UnitValue < 500 THEN 80
            ELSE 100
        END,
        FragilityScore = CASE 
            WHEN p.IsFragile = 1 OR p.IsPerishable = 1 THEN 80
            ELSE 20
        END,
        GravityZone = CASE
            WHEN (ISNULL(pm.VelocityScore, 0) * 0.5 + 
                  (CASE WHEN p.UnitValue < 10 THEN 20 ELSE 60 END) * 0.3 +
                  (CASE WHEN p.IsFragile = 1 THEN 80 ELSE 20 END) * 0.2) >= 70 THEN 'High'
            WHEN (ISNULL(pm.VelocityScore, 0) * 0.5 + 
                  (CASE WHEN p.UnitValue < 10 THEN 20 ELSE 60 END) * 0.3 +
                  (CASE WHEN p.IsFragile = 1 THEN 80 ELSE 20 END) * 0.2) >= 40 THEN 'Medium'
            ELSE 'Low'
        END,
        LastRecalculated = GETDATE()
    FROM DimProductGravity p
    LEFT JOIN ProductMetrics pm ON p.ProductKey = pm.ProductKey;
END;
GO

-- Procedure for fleet maintenance prioritization
CREATE PROCEDURE usp_GenerateFleetMaintenanceQueue
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        f.VehicleID,
        COUNT(*) AS TripCount,
        AVG(f.EngineHealthScore) AS AvgEngineHealth,
        AVG(f.TirePressureAvg) AS AvgTirePressure,
        SUM(f.MaintenanceAlerts) AS TotalAlerts,
        SUM(CASE WHEN p.GravityZone = 'High' THEN f.LoadWeightKg ELSE 0 END) AS HighValueLoadKg,
        -- Priority score: lower health + more alerts + high-value cargo = higher priority
        (100 - AVG(f.EngineHealthScore)) * 0.4 +
        (SUM(f.MaintenanceAlerts) * 10) * 0.3 +
        (SUM(CASE WHEN p.GravityZone = 'High' THEN 1 ELSE 0 END) * 20) * 0.3 AS PriorityScore
    FROM FactFleetTrips f
    LEFT JOIN FactWarehouseOperations w ON w.OrderID = CAST(f.TripKey AS VARCHAR(50))
    LEFT JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE f.TripStartTime >= DATEADD(DAY, -7, GETDATE())
        AND (f.EngineHealthScore < 80 OR f.MaintenanceAlerts > 0)
    GROUP BY f.VehicleID
    ORDER BY PriorityScore DESC;
END;
GO
```

### Step 3: Configure Connection String

Create a configuration file for data source connections:

```json
{
  "connections": {
    "sql_server": {
      "server": "${SQL_SERVER_HOST}",
      "database": "LogiFleetPulse",
      "authentication": "ActiveDirectoryIntegrated",
      "connection_string": "Server=${SQL_SERVER_HOST};Database=LogiFleetPulse;Trusted_Connection=True;Encrypt=True;"
    },
    "wms_api": {
      "base_url": "${WMS_API_URL}",
      "api_key": "${WMS_API_KEY}",
      "timeout_seconds": 30
    },
    "telematics_feed": {
      "endpoint": "${TELEMATICS_ENDPOINT}",
      "auth_token": "${TELEMATICS_TOKEN}"
    }
  },
  "refresh_schedule": {
    "warehouse_ops": "*/15 * * * *",
    "fleet_trips": "*/15 * * * *",
    "dimension_updates": "0 2 * * *"
  }
}
```

### Step 4: Set Up Power BI Template

Connect Power BI to the SQL Server database:

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Enter server and database name
4. Import tables: `Fact*` and `Dim*` tables
5. Verify relationships in Model view (should auto-detect based on foreign keys)

## Key DAX Measures for Power BI

```dax
// Total Warehouse Cycle Time
TotalCycleTime = SUM(FactWarehouseOperations[CycleTimeMinutes])

// Average Dwell Time by Gravity Zone
AvgDwellByGravity = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeHours]),
    USERELATIONSHIP(FactWarehouseOperations[ProductKey], DimProductGravity[ProductKey])
)

// Fleet Fuel Efficiency (km per liter)
FuelEfficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKm]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)

// Cross-Fact KPI: Dwell Time vs Fleet Idle Cost
DwellImpactOnFleet = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR AvgIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
RETURN
    AvgDwell * AvgIdle * 0.5 // Simplified correlation factor

// On-Time Delivery Rate
OnTimeRate = 
DIVIDE(
    COUNTROWS(FILTER(FactFleetTrips, FactFleetTrips[ActualDurationMinutes] <= FactFleetTrips[PlannedDurationMinutes] * 1.1)),
    COUNTROWS(FactFleetTrips),
    0
)

// High-Gravity Products Delayed
HighGravityDelayed = 
CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    DimProductGravity[GravityZone] = "High",
    FactWarehouseOperations[DwellTimeHours] > 72
)

// Predictive Bottleneck Index (simplified)
BottleneckIndex = 
VAR CurrentDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR HistoricalAvg = CALCULATE(AVERAGE(FactWarehouseOperations[DwellTimeHours]), ALL(DimTime))
VAR FleetUtilization = DIVIDE(SUM(FactFleetTrips[ActualDurationMinutes]), SUM(FactFleetTrips[PlannedDurationMinutes]))
RETURN
    (CurrentDwell / HistoricalAvg) * FleetUtilization * 100
```

## Common Usage Patterns

### Pattern 1: Analyzing Warehouse Gravity Zone Efficiency

```sql
-- Compare cycle times across gravity zones
SELECT 
    p.GravityZone,
    p.Category,
    COUNT(*) AS OperationCount,
    AVG(f.CycleTimeMinutes) AS AvgCycleTime,
    AVG(f.DwellTimeHours) AS AvgDwellTime,
    STDEV(f.CycleTimeMinutes) AS CycleTimeVariance
FROM FactWarehouseOperations f
INNER JOIN DimProductGravity p ON f.ProductKey = p.ProductKey
WHERE f.OperationType = 'Picking'
    AND f.OperationStartTime >= DATEADD(MONTH, -1, GETDATE())
GROUP BY p.GravityZone, p.Category
ORDER BY p.GravityZone, AvgCycleTime DESC;

-- Identify products that should be reclassified
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityZone AS CurrentZone,
    p.CompositeGravity AS CurrentScore,
    COUNT(f.OperationKey) AS RecentPicks,
    AVG(p.CompositeGravity) OVER (PARTITION BY p.Category) AS CategoryAvgGravity,
    CASE 
        WHEN COUNT(f.OperationKey) > 50 AND p.GravityZone = 'Low' THEN 'Consider upgrading to Medium'
        WHEN COUNT(f.OperationKey) < 10 AND p.GravityZone = 'High' THEN 'Consider downgrading to Medium'
        ELSE 'Appropriately classified'
    END AS Recommendation
FROM DimProductGravity p
LEFT JOIN FactWarehouseOperations f ON p.ProductKey = f.ProductKey 
    AND f.OperationType = 'Picking'
    AND f.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY p.SKU, p.ProductName, p.GravityZone, p.CompositeGravity, p.Category;
```

### Pattern 2: Cross-Fact Fleet and Warehouse Correlation

```sql
-- Analyze relationship between warehouse dwell and fleet delays
WITH WarehouseDwell AS (
    SELECT 
        w.OrderID,
        w.ProductKey,
        g.LocationID AS WarehouseID,
        AVG(w.DwellTimeHours) AS AvgDwell
    FROM FactWarehouseOperations w
    INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    WHERE w.OperationType IN ('Picking', 'Packing')
        AND w.OperationStartTime >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY w.OrderID, w.ProductKey, g.LocationID
),
FleetPerformance AS (
    SELECT 
        f.TripKey,
        og.LocationID AS OriginWarehouse,
        f.ActualDurationMinutes - f.PlannedDurationMinutes AS DelayMinutes,
        f.LoadWeightKg
    FROM FactFleetTrips f
    INNER JOIN DimGeography og ON f.OriginGeographyKey = og.GeographyKey
    WHERE f.TripStartTime >= DATEADD(MONTH, -3, GETDATE())
        AND f.TripStatus = 'Completed'
)
SELECT 
    fp.OriginWarehouse,
    AVG(wd.AvgDwell) AS AvgWarehouseDwell,
    AVG(fp.DelayMinutes) AS AvgFleetDelay,
    COUNT(DISTINCT fp.TripKey) AS TripCount,
    -- Correlation indicator (simplified)
    CASE 
        WHEN AVG(wd.AvgDwell) > 48 AND AVG(fp.DelayMinutes) > 30 THEN 'High correlation - investigate'
        WHEN AVG(wd.AvgDwell) > 48 OR AVG(fp.DelayMinutes) > 30 THEN 'Moderate correlation'
        ELSE 'Low correlation'
    END AS CorrelationStatus
FROM FleetPerformance fp
LEFT JOIN WarehouseDwell wd ON fp.OriginWarehouse = wd.WarehouseID
GROUP BY fp.OriginWarehouse
ORDER BY AvgFleetDelay DESC;
```

### Pattern 3: Real-Time Alert Configuration

```sql
-- Create alert table
CREATE TABLE AlertConfiguration (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName VARCHAR(200) NOT NULL,
    MetricType VARCHAR(50) NOT NULL, -- Dwell, FleetIdle, MaintenanceScore, etc.
    ThresholdValue DECIMAL(10,2) NOT NULL,
    ComparisonOperator VARCHAR(10) NOT NULL, -- '>', '<', '>=', '<=', '='
    AlertSeverity VARCHAR(20) NOT NULL, -- Critical, High, Medium, Low
    NotificationChannel VARCHAR(50), -- Email, SMS, Teams
    IsActive BIT DEFAULT 1
);

-- Insert sample alerts
INSERT INTO AlertConfiguration (AlertName, MetricType, ThresholdValue, ComparisonOperator, AlertSeverity, NotificationChannel)
VALUES 
    ('High-Value Product Excessive Dwell', 'DwellHours', 72, '>', 'Critical', 'Email,Teams'),
    ('Fleet Idling Above Threshold', 'IdleTimePercent', 15, '>', 'High', 'SMS,Email'),
    ('Engine Health Degradation', 'EngineHealthScore', 70, '<', 'High', 'Email'),
    ('Cross-Dock Bottleneck', 'CrossDockDwellMinutes', 120, '>', 'Medium', 'Teams');

-- Scheduled job to check alerts (run every 15 minutes)
CREATE PROCEDURE usp_CheckAlerts
AS
BEGIN
    -- Check dwell time alerts
    INSERT INTO AlertLog (AlertID, EntityID, MetricValue, AlertTime)
    SELECT 
        ac.AlertID,
        CAST(f.ProductKey AS VARCHAR(50)),
        f.DwellTimeHours,
        GETDATE()
    FROM AlertConfiguration ac
    CROSS APPLY (
        SELECT TOP 100
            ProductKey,
            DwellTimeHours
        FROM FactWarehouseOperations
        WHERE OperationStartTime >= DATEADD(MINUTE, -15, GETDATE())
            AND DwellTimeHours IS NOT NULL
    ) f
    INNER JOIN DimProductGravity p ON f.ProductKey = p.ProductKey
    WHERE ac.MetricType = 'DwellHours'
        AND ac.IsActive = 1
        AND p.GravityZone = 'High'
        AND (
            (ac.ComparisonOperator = '>' AND f.DwellTimeHours > ac.ThresholdValue) OR
            (ac.ComparisonOperator = '<' AND f.DwellTimeHours < ac.ThresholdValue)
        );
    
    -- Check fleet idling alerts
    INSERT INTO AlertLog (AlertID, EntityID, MetricValue, AlertTime)
    SELECT 
        ac.AlertID,
        f.VehicleID,
        CAST((f.IdleTimeMinutes * 100.0 / NULLIF(f.ActualDurationMinutes, 0)) AS DECIMAL(10,2)),
        GETDATE()
    FROM AlertConfiguration ac
    CROSS APPLY (
        SELECT 
            VehicleID,
            IdleTimeMinutes,
            ActualDurationMinutes
        FROM FactFleetTrips
        WHERE TripStartTime >= DATEADD(MINUTE, -15, GETDATE())
            AND ActualDurationMinutes > 0
    ) f
    WHERE ac.MetricType = 'IdleTimePercent'
        AND ac.IsActive = 1
        AND (f.IdleTimeMinutes * 100.0 / NULLIF(f.ActualDurationMinutes, 0)) > ac.ThresholdValue;
END;
GO
```

## Configuration Options

### Time Dimension Granularity

Adjust the time bucket size by modifying the DimTime population script:

```sql
-- Generate time dimension with custom granularity
DECLARE @StartDate DATETIME = '2026-01-01 00:00:00';
DECLARE @EndDate DATETIME = '2027-12-31 23:59:59';
DECLARE @BucketMinutes INT = 15; -- Change to 5, 10, 30, or 60

WITH TimeSeries AS (
    SELECT @StartDate AS CurrentTime
    UNION ALL
    SELECT DATEADD(MINUTE, @BucketMinutes, CurrentTime)
    FROM TimeSeries
    WHERE CurrentTime < @EndDate
)
INSERT INTO DimTime (
    TimeKey, FullDateTime, DateKey, TimeOfDay, HourOfDay, 
    MinuteBucket, DayOfWeek, DayName, IsWeekend, WeekOfYear,
    MonthNumber, MonthName, Quarter, Year, FiscalPeriod
)
SELECT 
    CAST(FORMAT(CurrentTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
    CurrentTime,
    CAST(FORMAT(CurrentTime, 'yyyyMMdd') AS INT) AS DateKey,
    CAST(CurrentTime AS TIME) AS TimeOfDay,
    DATEPART(HOUR, CurrentTime) AS HourOfDay,
    DATEPART(MINUTE, CurrentTime) AS MinuteBucket,
    DATEPART(WEEKDAY, CurrentTime) AS DayOfWeek,
    DATENAME(WEEKDAY, CurrentTime) AS DayName,
    CASE WHEN DATEPART(WEEKDAY, CurrentTime) IN (1, 7) THEN 1 ELSE 0
