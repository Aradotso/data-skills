---
name: logifleet-pulse-supply-chain-analytics
description: Advanced MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for fleet and warehouse intelligence
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics data warehouse with power bi
  - configure fleet and warehouse analytics database
  - implement multi-fact star schema for logistics
  - create supply chain kpi dashboard
  - build warehouse gravity zone analytics
  - set up real-time fleet tracking database
  - configure cross-dock operations reporting
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform built on MS SQL Server and Power BI. It provides a multi-fact star schema data warehouse that unifies warehouse operations, fleet telemetry, inventory management, and supply chain KPIs into a single semantic layer. The system uses time-phased dimensions, bridge tables for many-to-many relationships, and cross-fact aggregation to enable deep analytics across logistics operations.

**Primary Language**: SQL (T-SQL for MS SQL Server)  
**Visualization**: Power BI Desktop (.pbit templates)  
**Data Model**: Multi-fact star schema with shared dimensions

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Appropriate database permissions (CREATE TABLE, CREATE VIEW, CREATE PROCEDURE)

### Database Schema Deployment

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Execute the core schema script** (deploy in this order):

```sql
-- Connect to your SQL Server instance
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Deploy dimension tables first
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    MonthOfYear TINYINT NOT NULL,
    MonthName VARCHAR(10) NOT NULL,
    Quarter TINYINT NOT NULL,
    FiscalYear SMALLINT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL DEFAULT 0
);

-- Create clustered columnstore index for analytics
CREATE CLUSTERED COLUMNSTORE INDEX CCI_DimTime ON DimTime;

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationCode VARCHAR(20) NOT NULL UNIQUE,
    LocationName VARCHAR(100) NOT NULL,
    LocationType VARCHAR(20) NOT NULL, -- 'Warehouse', 'Hub', 'RouteNode'
    AddressLine1 VARCHAR(200),
    AddressLine2 VARCHAR(200),
    City VARCHAR(100),
    StateProvince VARCHAR(50),
    PostalCode VARCHAR(20),
    Country VARCHAR(50) NOT NULL,
    Region VARCHAR(50),
    Continent VARCHAR(30),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50),
    IsActive BIT NOT NULL DEFAULT 1,
    ValidFrom DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    ValidTo DATETIME2 NULL
);

CREATE INDEX IX_DimGeography_LocationCode ON DimGeography(LocationCode);
CREATE INDEX IX_DimGeography_LocationType ON DimGeography(LocationType);

-- DimProductGravity: Products with velocity-based gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200) NOT NULL,
    ProductCategory VARCHAR(50),
    ProductSubcategory VARCHAR(50),
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,2),
    IsFragile BIT NOT NULL DEFAULT 0,
    IsPerishable BIT NOT NULL DEFAULT 0,
    RequiresColdStorage BIT NOT NULL DEFAULT 0,
    GravityScore DECIMAL(5,2) NOT NULL DEFAULT 50.0, -- 0-100, higher = faster moving
    VelocityClass VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ReplenishmentLeadTimeDays INT,
    UnitCost DECIMAL(10,2),
    UnitPrice DECIMAL(10,2),
    ValidFrom DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    ValidTo DATETIME2 NULL
);

CREATE INDEX IX_DimProductGravity_SKU ON DimProductGravity(SKU);
CREATE INDEX IX_DimProductGravity_GravityScore ON DimProductGravity(GravityScore DESC);

-- DimSupplierReliability: Supplier performance tracking
CREATE TABLE DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierCode VARCHAR(20) NOT NULL UNIQUE,
    SupplierName VARCHAR(200) NOT NULL,
    ContactEmail VARCHAR(100),
    ContactPhone VARCHAR(30),
    LeadTimeVarianceDays DECIMAL(5,2),
    DefectRate DECIMAL(5,4), -- 0-1
    OnTimeDeliveryRate DECIMAL(5,4), -- 0-1
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityRating VARCHAR(10), -- 'A', 'B', 'C', 'D'
    IsActive BIT NOT NULL DEFAULT 1,
    ValidFrom DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    ValidTo DATETIME2 NULL
);

CREATE INDEX IX_DimSupplierReliability_SupplierCode ON DimSupplierReliability(SupplierCode);

-- FactWarehouseOperations: Core warehouse activity fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationID VARCHAR(50) NOT NULL,
    OrderID VARCHAR(50),
    QuantityHandled INT NOT NULL,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes DECIMAL(10,2),
    StorageZone VARCHAR(20),
    ZoneGravity DECIMAL(5,2),
    EmployeeID VARCHAR(20),
    EquipmentID VARCHAR(20),
    OperationTimestamp DATETIME2 NOT NULL,
    CONSTRAINT FK_FactWarehouse_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactWarehouse_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactWarehouse_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOperations ON FactWarehouseOperations;
CREATE NONCLUSTERED INDEX IX_FactWarehouse_TimeKey ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_OperationType ON FactWarehouseOperations(OperationType);

-- FactFleetTrips: Fleet telemetry and trip data
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TripID VARCHAR(50) NOT NULL,
    VehicleID VARCHAR(20) NOT NULL,
    DriverID VARCHAR(20),
    StartTimeKey INT NOT NULL,
    EndTimeKey INT,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT,
    RouteSegment VARCHAR(100),
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    LoadWeightLbs DECIMAL(10,2),
    OnTimeStatus VARCHAR(20), -- 'OnTime', 'Delayed', 'Early'
    DelayReasonCode VARCHAR(20),
    WeatherCondition VARCHAR(50),
    TripCost DECIMAL(10,2),
    TripTimestamp DATETIME2 NOT NULL,
    CONSTRAINT FK_FactFleet_StartTime FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_OriginGeography FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips ON FactFleetTrips;
CREATE NONCLUSTERED INDEX IX_FactFleet_VehicleID ON FactFleetTrips(VehicleID);
CREATE NONCLUSTERED INDEX IX_FactFleet_StartTimeKey ON FactFleetTrips(StartTimeKey);
```

### Data Loading Stored Procedures

Create ETL procedures for incremental loading:

```sql
-- Stored procedure for time dimension population
CREATE PROCEDURE dbo.PopulateTimeDimension
    @StartDate DATETIME,
    @EndDate DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME = @StartDate;
    DECLARE @TimeKey INT;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        SET @TimeKey = 
            YEAR(@CurrentDateTime) * 100000000 +
            MONTH(@CurrentDateTime) * 1000000 +
            DAY(@CurrentDateTime) * 10000 +
            DATEPART(HOUR, @CurrentDateTime) * 100 +
            (DATEPART(MINUTE, @CurrentDateTime) / 15) * 15;
        
        IF NOT EXISTS (SELECT 1 FROM DimTime WHERE TimeKey = @TimeKey)
        BEGIN
            INSERT INTO DimTime (
                TimeKey, FullDateTime, DateKey, TimeOfDay, 
                HourOfDay, DayOfWeek, DayName, WeekOfYear, 
                MonthOfYear, MonthName, Quarter, FiscalYear, IsWeekend
            )
            VALUES (
                @TimeKey,
                @CurrentDateTime,
                YEAR(@CurrentDateTime) * 10000 + MONTH(@CurrentDateTime) * 100 + DAY(@CurrentDateTime),
                CAST(@CurrentDateTime AS TIME),
                DATEPART(HOUR, @CurrentDateTime),
                DATEPART(WEEKDAY, @CurrentDateTime),
                DATENAME(WEEKDAY, @CurrentDateTime),
                DATEPART(WEEK, @CurrentDateTime),
                MONTH(@CurrentDateTime),
                DATENAME(MONTH, @CurrentDateTime),
                DATEPART(QUARTER, @CurrentDateTime),
                YEAR(@CurrentDateTime),
                CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1,7) THEN 1 ELSE 0 END
            );
        END
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

-- Execute to populate 2 years of time data
EXEC dbo.PopulateTimeDimension 
    @StartDate = '2025-01-01', 
    @EndDate = '2026-12-31';
GO

-- Cross-fact KPI view: Warehouse dwell time vs Fleet idle time correlation
CREATE VIEW vw_DwellTimeFleetIdleCorrelation
AS
SELECT 
    t.DateKey,
    t.MonthName,
    t.FiscalYear,
    g.LocationName AS WarehouseLocation,
    p.ProductCategory,
    AVG(w.DwellTimeMinutes) AS AvgDwellTimeMinutes,
    COUNT(DISTINCT w.OperationID) AS WarehouseOperations,
    AVG(f.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    COUNT(DISTINCT f.TripID) AS FleetTrips,
    -- Correlation ratio: high dwell = high idle time = inefficiency
    CASE 
        WHEN AVG(w.DwellTimeMinutes) > 120 AND AVG(f.IdleTimeMinutes) > 30 THEN 'Critical'
        WHEN AVG(w.DwellTimeMinutes) > 60 AND AVG(f.IdleTimeMinutes) > 15 THEN 'Warning'
        ELSE 'Normal'
    END AS EfficiencyStatus
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips f ON 
    f.OriginGeographyKey = w.GeographyKey AND
    f.StartTimeKey = w.TimeKey
WHERE w.OperationType IN ('Picking', 'Packing')
GROUP BY 
    t.DateKey, t.MonthName, t.FiscalYear,
    g.LocationName, p.ProductCategory;
GO
```

## Configuration

### Connection String Setup

Store connection details in environment variables:

```bash
# Windows
setx LOGIFLEET_SQL_SERVER "your-server-name.database.windows.net"
setx LOGIFLEET_SQL_DATABASE "LogiFleetPulse"
setx LOGIFLEET_SQL_USERNAME "your-username"
setx LOGIFLEET_SQL_PASSWORD "your-password"

# Linux/Mac
export LOGIFLEET_SQL_SERVER="your-server-name"
export LOGIFLEET_SQL_DATABASE="LogiFleetPulse"
export LOGIFLEET_SQL_USERNAME="your-username"
export LOGIFLEET_SQL_PASSWORD="your-password"
```

### Power BI Configuration

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. When prompted, enter SQL Server connection:
   - **Server**: Value from `LOGIFLEET_SQL_SERVER`
   - **Database**: Value from `LOGIFLEET_SQL_DATABASE`
3. Select **DirectQuery** mode for real-time dashboards or **Import** for scheduled refresh
4. Configure row-level security in Power BI:

```dax
-- RLS filter for geography-based access
[GeographyKey] IN (
    LOOKUPVALUE(
        UserGeographyAccess[GeographyKey],
        UserGeographyAccess[UserEmail], USERPRINCIPALNAME()
    )
)
```

### Scheduled Refresh Setup (Import Mode)

```sql
-- Create stored procedure for data refresh orchestration
CREATE PROCEDURE dbo.RefreshAnalyticsData
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update gravity scores based on last 30 days velocity
    UPDATE p
    SET 
        GravityScore = CASE 
            WHEN velocity.PicksPerDay > 50 THEN 90
            WHEN velocity.PicksPerDay > 20 THEN 70
            WHEN velocity.PicksPerDay > 5 THEN 50
            ELSE 30
        END,
        VelocityClass = CASE 
            WHEN velocity.PicksPerDay > 50 THEN 'Fast'
            WHEN velocity.PicksPerDay > 5 THEN 'Medium'
            ELSE 'Slow'
        END
    FROM DimProductGravity p
    INNER JOIN (
        SELECT 
            ProductKey,
            COUNT(*) * 1.0 / 30 AS PicksPerDay
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
            AND OperationTimestamp >= DATEADD(DAY, -30, GETUTCDATE())
        GROUP BY ProductKey
    ) velocity ON p.ProductKey = velocity.ProductKey;
    
    -- Update supplier reliability scores
    UPDATE s
    SET 
        ComplianceScore = CAST(metrics.OnTimeDeliveries * 100.0 / NULLIF(metrics.TotalDeliveries, 0) AS DECIMAL(5,2)),
        ReliabilityRating = CASE 
            WHEN metrics.OnTimeDeliveries * 100.0 / NULLIF(metrics.TotalDeliveries, 0) >= 95 THEN 'A'
            WHEN metrics.OnTimeDeliveries * 100.0 / NULLIF(metrics.TotalDeliveries, 0) >= 85 THEN 'B'
            WHEN metrics.OnTimeDeliveries * 100.0 / NULLIF(metrics.TotalDeliveries, 0) >= 70 THEN 'C'
            ELSE 'D'
        END
    FROM DimSupplierReliability s
    INNER JOIN (
        SELECT 
            SupplierKey,
            COUNT(*) AS TotalDeliveries,
            SUM(CASE WHEN OnTimeStatus = 'OnTime' THEN 1 ELSE 0 END) AS OnTimeDeliveries
        FROM FactFleetTrips f
        WHERE TripTimestamp >= DATEADD(DAY, -90, GETUTCDATE())
        GROUP BY SupplierKey
    ) metrics ON s.SupplierKey = metrics.SupplierKey;
END;
GO

-- Schedule via SQL Server Agent (run daily at 2 AM)
-- Or call from external scheduler
```

## Key Queries & Analytics Patterns

### Pattern 1: Warehouse Gravity Zone Analysis

Identify products in suboptimal storage zones:

```sql
-- Find high-gravity products in low-velocity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.VelocityClass,
    w.StorageZone,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(*) AS PickOperations,
    -- Recommendation logic
    CASE 
        WHEN p.GravityScore > 70 AND w.StorageZone LIKE '%BACK%' THEN 'Move to front zone'
        WHEN p.GravityScore < 40 AND w.StorageZone LIKE '%FRONT%' THEN 'Move to back zone'
        ELSE 'Zone optimal'
    END AS Recommendation
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE w.OperationType = 'Picking'
    AND t.FullDateTime >= DATEADD(DAY, -30, GETUTCDATE())
GROUP BY 
    p.SKU, p.ProductName, p.GravityScore, 
    p.VelocityClass, w.StorageZone
HAVING AVG(w.DwellTimeMinutes) > 60
ORDER BY p.GravityScore DESC, AvgDwellTime DESC;
```

### Pattern 2: Fleet Optimization by Route Segment

```sql
-- Identify routes with high idle time and fuel inefficiency
WITH RouteMetrics AS (
    SELECT 
        f.RouteSegment,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        AVG(f.FuelConsumedGallons) AS AvgFuelConsumed,
        AVG(f.DistanceMiles) AS AvgDistance,
        AVG(f.FuelConsumedGallons / NULLIF(f.DistanceMiles, 0)) AS FuelEfficiency,
        COUNT(*) AS TripCount,
        SUM(CASE WHEN f.OnTimeStatus = 'Delayed' THEN 1 ELSE 0 END) AS DelayedTrips
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.StartTimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -30, GETUTCDATE())
        AND f.RouteSegment IS NOT NULL
    GROUP BY f.RouteSegment
)
SELECT 
    RouteSegment,
    AvgIdleTime,
    AvgFuelConsumed,
    AvgDistance,
    CAST(FuelEfficiency AS DECIMAL(10,3)) AS MPG_Inverse,
    TripCount,
    DelayedTrips,
    CAST(DelayedTrips * 100.0 / TripCount AS DECIMAL(5,2)) AS DelayRate,
    -- Priority score for optimization
    (AvgIdleTime * 0.4 + (DelayedTrips * 100.0 / TripCount) * 0.6) AS OptimizationPriority
FROM RouteMetrics
WHERE TripCount >= 5
ORDER BY OptimizationPriority DESC;
```

### Pattern 3: Cross-Fact Bottleneck Detection

```sql
-- Predictive bottleneck detection using time-series patterns
WITH HourlyMetrics AS (
    SELECT 
        t.DateKey,
        t.HourOfDay,
        t.DayName,
        COUNT(DISTINCT w.OperationID) AS WarehouseOps,
        COUNT(DISTINCT f.TripID) AS FleetTrips,
        AVG(w.ProcessingTimeMinutes) AS AvgProcessingTime,
        AVG(f.LoadingTimeMinutes + f.UnloadingTimeMinutes) AS AvgLoadUnloadTime
    FROM DimTime t
    LEFT JOIN FactWarehouseOperations w ON t.TimeKey = w.TimeKey
    LEFT JOIN FactFleetTrips f ON t.TimeKey = f.StartTimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -14, GETUTCDATE())
    GROUP BY t.DateKey, t.HourOfDay, t.DayName
),
ThresholdCalc AS (
    SELECT 
        AVG(WarehouseOps) AS AvgWarehouseOps,
        STDEV(WarehouseOps) AS StdDevWarehouseOps,
        AVG(FleetTrips) AS AvgFleetTrips,
        STDEV(FleetTrips) AS StdDevFleetTrips
    FROM HourlyMetrics
)
SELECT 
    h.DateKey,
    h.HourOfDay,
    h.DayName,
    h.WarehouseOps,
    h.FleetTrips,
    h.AvgProcessingTime,
    h.AvgLoadUnloadTime,
    -- Anomaly detection
    CASE 
        WHEN h.WarehouseOps > (t.AvgWarehouseOps + 2 * t.StdDevWarehouseOps) THEN 'Warehouse bottleneck'
        WHEN h.FleetTrips > (t.AvgFleetTrips + 2 * t.StdDevFleetTrips) THEN 'Fleet congestion'
        WHEN h.AvgProcessingTime > 30 THEN 'Processing delay'
        ELSE 'Normal'
    END AS BottleneckType
FROM HourlyMetrics h
CROSS JOIN ThresholdCalc t
WHERE h.DateKey >= CAST(CONVERT(VARCHAR(8), GETUTCDATE(), 112) AS INT)
ORDER BY h.DateKey DESC, h.HourOfDay;
```

### Pattern 4: Supplier Performance Dashboard Query

```sql
-- Comprehensive supplier scorecard
SELECT 
    s.SupplierCode,
    s.SupplierName,
    s.ReliabilityRating,
    s.ComplianceScore,
    s.OnTimeDeliveryRate,
    COUNT(DISTINCT f.TripID) AS TotalDeliveries,
    SUM(CASE WHEN f.OnTimeStatus = 'Delayed' THEN 1 ELSE 0 END) AS DelayedDeliveries,
    AVG(f.IdleTimeMinutes) AS AvgUnloadingIdleTime,
    SUM(f.TripCost) AS TotalLogisticsCost,
    -- Financial impact of delays
    SUM(CASE WHEN f.OnTimeStatus = 'Delayed' 
        THEN f.TripCost * 0.15  -- 15% penalty for delays
        ELSE 0 
    END) AS EstimatedDelayCost
FROM DimSupplierReliability s
LEFT JOIN FactFleetTrips f ON s.SupplierKey = f.SupplierKey
WHERE f.TripTimestamp >= DATEADD(MONTH, -3, GETUTCDATE())
GROUP BY 
    s.SupplierCode, s.SupplierName, 
    s.ReliabilityRating, s.ComplianceScore, s.OnTimeDeliveryRate
HAVING COUNT(DISTINCT f.TripID) > 0
ORDER BY s.ComplianceScore DESC, TotalLogisticsCost DESC;
```

## Alerting & Automation

### Automated Alert Stored Procedure

```sql
CREATE PROCEDURE dbo.CheckKPIThresholdsAndAlert
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertLog TABLE (
        AlertType VARCHAR(50),
        AlertMessage NVARCHAR(500),
        Severity VARCHAR(20),
        MetricValue DECIMAL(10,2)
    );
    
    -- Alert 1: High fleet idle time
    INSERT INTO @AlertLog
    SELECT 
        'Fleet Idle Time',
        'Vehicle ' + VehicleID + ' idle time exceeded 15% on route ' + RouteSegment,
        'Warning',
        CAST(IdleTimeMinutes * 100.0 / NULLIF(DATEDIFF(MINUTE, StartTime, EndTime), 0) AS DECIMAL(10,2))
    FROM (
        SELECT 
            f.VehicleID,
            f.RouteSegment,
            f.IdleTimeMinutes,
            t1.FullDateTime AS StartTime,
            t2.FullDateTime AS EndTime
        FROM FactFleetTrips f
        INNER JOIN DimTime t1 ON f.StartTimeKey = t1.TimeKey
        LEFT JOIN DimTime t2 ON f.EndTimeKey = t2.TimeKey
        WHERE t1.FullDateTime >= DATEADD(HOUR, -4, GETUTCDATE())
    ) trips
    WHERE IdleTimeMinutes * 100.0 / NULLIF(DATEDIFF(MINUTE, StartTime, EndTime), 0) > 15;
    
    -- Alert 2: Warehouse dwell time spike
    INSERT INTO @AlertLog
    SELECT 
        'Warehouse Dwell Time',
        'SKU ' + p.SKU + ' dwell time exceeded 72 hours at ' + g.LocationName,
        'Critical',
        AVG(w.DwellTimeMinutes)
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(HOUR, -4, GETUTCDATE())
        AND w.OperationType = 'Putaway'
    GROUP BY p.SKU, g.LocationName
    HAVING AVG(w.DwellTimeMinutes) > 4320; -- 72 hours
    
    -- Alert 3: Supplier delivery compliance drop
    INSERT INTO @AlertLog
    SELECT 
        'Supplier Compliance',
        'Supplier ' + s.SupplierName + ' compliance dropped below 70%',
        'Warning',
        s.ComplianceScore
    FROM DimSupplierReliability s
    WHERE s.ComplianceScore < 70 AND s.IsActive = 1;
    
    -- Output alerts (could be logged to table or sent via email)
    SELECT * FROM @AlertLog;
    
    -- Optional: Insert into permanent log table
    -- INSERT INTO AlertHistory SELECT *, GETUTCDATE() FROM @AlertLog;
END;
GO

-- Schedule this to run every 15 minutes via SQL Server Agent
```

### Integration with External Systems

```sql
-- External table for streaming data ingestion (Polybase example)
CREATE EXTERNAL DATA SOURCE TelematicsAPI
WITH (
    TYPE = HADOOP,
    LOCATION = 'https://telematics-api.example.com',
    CREDENTIAL = TelematicsAPICredential
);

CREATE EXTERNAL FILE FORMAT JSONFormat
WITH (FORMAT_TYPE = JSON);

CREATE EXTERNAL TABLE ext_FleetTelemetryStream (
    VehicleID VARCHAR(20),
    Timestamp DATETIME2,
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2),
    EngineTemp DECIMAL(5,2)
)
WITH (
    LOCATION = '/api/v1/telemetry/stream',
    DATA_SOURCE = TelematicsAPI,
    FILE_FORMAT = JSONFormat
);

-- Continuous ingestion pattern (called by scheduled job)
INSERT INTO FactFleetTrips (
    TripID, VehicleID, StartTimeKey, OriginGeographyKey, 
    DistanceMiles, FuelConsumedGallons, TripTimestamp
)
SELECT 
    NEWID() AS TripID,
    t.VehicleID,
    -- Map timestamp to TimeKey (simplified)
    YEAR(t.Timestamp) * 100000000 + MONTH(t.Timestamp) * 1000000 + DAY(t.Timestamp) * 10000 + DATEPART(HOUR, t.Timestamp) * 100,
    g.GeographyKey,
    -- Calculate distance from GPS coordinates (simplified)
    geography::Point(t.Latitude, t.Longitude, 4326).STDistance(geography::Point(prev.Latitude, prev.Longitude, 4326)) / 1609.34,
    -- Estimate fuel from speed and distance
    (t.FuelLevel - prev.FuelLevel) * -1,
    t.Timestamp
FROM ext_FleetTelemetryStream t
CROSS APPLY (
    SELECT TOP 1 Latitude, Longitude, FuelLevel
    FROM ext_FleetTelemetryStream prev
    WHERE prev.VehicleID = t.VehicleID AND prev.Timestamp < t.Timestamp
    ORDER BY prev.Timestamp DESC
) prev
INNER JOIN DimGeography g ON g.Latitude BETWEEN t.Latitude - 0.1 AND t.Latitude + 0.1
WHERE NOT EXISTS (
    SELECT 1 FROM FactFleetTrips existing 
    WHERE existing.VehicleID = t.VehicleID AND existing.TripTimestamp = t.Timestamp
);
```

## Power BI DAX Measures

Key measures to implement in Power BI:

```dax
// Total Warehouse Operations
TotalWarehouse
