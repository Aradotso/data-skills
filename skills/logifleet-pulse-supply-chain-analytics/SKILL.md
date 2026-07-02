---
name: logifleet-pulse-supply-chain-analytics
description: Advanced MS SQL Server & Power BI data warehouse for fleet logistics, warehouse operations, and cross-modal supply chain intelligence with multi-fact star schema
triggers:
  - set up logifleet pulse logistics analytics
  - configure supply chain data warehouse sql server
  - build power bi dashboard for fleet and warehouse operations
  - implement multi-fact star schema for logistics
  - create warehouse gravity zone analytics
  - optimize fleet telemetry data model
  - design cross-dock operations dashboard
  - query logistics kpi harmonization layer
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that combines:

- **Multi-fact star schema** data warehouse on MS SQL Server
- **Power BI dashboards** for real-time fleet, warehouse, and supply chain visibility
- **Cross-modal analytics** linking warehouse operations, fleet telemetry, inventory aging, and external signals
- **Predictive bottleneck detection** and adaptive fleet triage
- **Warehouse Gravity Zones™** for optimal spatial inventory placement
- **Temporal elasticity modeling** for scenario simulation

The platform harmonizes disparate data sources (WMS, telematics, GPS, supplier portals, weather APIs) into a unified semantic layer with time-aware dimensions and bridge tables for many-to-many relationships.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Network access to data sources (WMS, TMS, telematics APIs)

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance and create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Deploy dimension tables first
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME NOT NULL,
    TimeSlot TIME,
    HourOfDay INT,
    DayOfWeek INT,
    DayName NVARCHAR(20),
    WeekOfYear INT,
    MonthNumber INT,
    MonthName NVARCHAR(20),
    Quarter INT,
    Year INT,
    FiscalPeriod INT,
    IsWeekend BIT,
    IsHoliday BIT
)

CREATE CLUSTERED INDEX IX_DimTime_DateTime ON DimTime(DateTime)
GO

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID NVARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200),
    LocationType NVARCHAR(50), -- Warehouse, Route Node, Distribution Center
    StreetAddress NVARCHAR(500),
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    IsActive BIT DEFAULT 1,
    EffectiveDate DATE,
    ExpirationDate DATE
)

CREATE INDEX IX_DimGeography_LocationID ON DimGeography(LocationID)
CREATE INDEX IX_DimGeography_LocationType ON DimGeography(LocationType)
GO

-- DimProductGravity: Products with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU NVARCHAR(100) NOT NULL,
    ProductName NVARCHAR(500),
    Category NVARCHAR(200),
    Subcategory NVARCHAR(200),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    UnitValue DECIMAL(12,2),
    FragilityScore DECIMAL(5,2), -- 0-100
    VelocityScore DECIMAL(5,2), -- 0-100, based on pick frequency
    GravityScore AS (VelocityScore * 0.5 + (UnitValue / 1000.0) * 0.3 + FragilityScore * 0.2) PERSISTED,
    RecommendedZone NVARCHAR(50), -- High-Gravity, Medium-Gravity, Low-Gravity
    IsPerishable BIT DEFAULT 0,
    ShelfLifeDays INT,
    IsActive BIT DEFAULT 1
)

CREATE UNIQUE INDEX IX_DimProductGravity_SKU ON DimProductGravity(SKU)
CREATE INDEX IX_DimProductGravity_GravityScore ON DimProductGravity(GravityScore DESC)
GO

-- DimSupplierReliability: Supplier performance tracking
CREATE TABLE DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierID NVARCHAR(100) NOT NULL,
    SupplierName NVARCHAR(500),
    Country NVARCHAR(100),
    LeadTimeDaysAvg DECIMAL(6,2),
    LeadTimeVariance DECIMAL(6,2),
    DefectPercentage DECIMAL(5,2),
    OnTimeDeliveryPercentage DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    IsActive BIT DEFAULT 1
)

CREATE UNIQUE INDEX IX_DimSupplierReliability_SupplierID ON DimSupplierReliability(SupplierID)
GO

-- FactWarehouseOperations: Warehouse activity fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType NVARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    OperationID NVARCHAR(100),
    OrderID NVARCHAR(100),
    QuantityHandled INT,
    DwellTimeMinutes INT,
    CycleTimeMinutes INT,
    StorageZone NVARCHAR(100),
    EmployeeID NVARCHAR(100),
    EquipmentID NVARCHAR(100),
    OperationCost DECIMAL(12,2),
    CONSTRAINT FK_FactWarehouse_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactWarehouse_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactWarehouse_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
)

CREATE INDEX IX_FactWarehouse_TimeKey ON FactWarehouseOperations(TimeKey)
CREATE INDEX IX_FactWarehouse_ProductKey ON FactWarehouseOperations(ProductKey)
CREATE INDEX IX_FactWarehouse_OperationType ON FactWarehouseOperations(OperationType)
GO

-- FactFleetTrips: Fleet telemetry and trip fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKeyStart INT NOT NULL,
    TimeKeyEnd INT,
    GeographyKeyOrigin INT NOT NULL,
    GeographyKeyDestination INT,
    VehicleID NVARCHAR(100),
    DriverID NVARCHAR(100),
    TripID NVARCHAR(100),
    RouteSegment NVARCHAR(200),
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,3),
    AvgSpeedKmh DECIMAL(6,2),
    LoadWeightKg DECIMAL(10,2),
    LoadVolumeM3 DECIMAL(10,2),
    LoadRevenue DECIMAL(12,2),
    TripCost DECIMAL(12,2),
    DelayMinutes INT,
    DelayReason NVARCHAR(500),
    WeatherCondition NVARCHAR(100),
    TrafficCondition NVARCHAR(100),
    CONSTRAINT FK_FactFleet_TimeStart FOREIGN KEY (TimeKeyStart) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_GeographyOrigin FOREIGN KEY (GeographyKeyOrigin) REFERENCES DimGeography(GeographyKey)
)

CREATE INDEX IX_FactFleet_TimeKeyStart ON FactFleetTrips(TimeKeyStart)
CREATE INDEX IX_FactFleet_VehicleID ON FactFleetTrips(VehicleID)
CREATE INDEX IX_FactFleet_RouteSegment ON FactFleetTrips(RouteSegment)
GO

-- FactCrossDock: Cross-dock operations without storage
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKeyInbound INT NOT NULL,
    TimeKeyOutbound INT,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    Quantity INT,
    DwellTimeMinutes INT,
    HandlingCost DECIMAL(12,2),
    CONSTRAINT FK_FactCrossDock_Time FOREIGN KEY (TimeKeyInbound) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactCrossDock_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactCrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
)

CREATE INDEX IX_FactCrossDock_TimeKeyInbound ON FactCrossDock(TimeKeyInbound)
CREATE INDEX IX_FactCrossDock_ProductKey ON FactCrossDock(ProductKey)
GO
```

### Step 2: Populate Time Dimension

```sql
-- Stored procedure to populate DimTime with 15-minute granularity
CREATE PROCEDURE PopulateDimTime
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME = @StartDate;
    DECLARE @TimeKey INT;
    
    WHILE @CurrentDateTime < @EndDate
    BEGIN
        SET @TimeKey = CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT);
        
        INSERT INTO DimTime (
            TimeKey, DateTime, TimeSlot, HourOfDay, DayOfWeek, DayName,
            WeekOfYear, MonthNumber, MonthName, Quarter, Year, FiscalPeriod,
            IsWeekend, IsHoliday
        )
        VALUES (
            @TimeKey,
            @CurrentDateTime,
            CAST(@CurrentDateTime AS TIME),
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            MONTH(@CurrentDateTime),
            DATENAME(MONTH, @CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            YEAR(@CurrentDateTime),
            CASE 
                WHEN MONTH(@CurrentDateTime) <= 6 THEN YEAR(@CurrentDateTime) * 10 + 1
                ELSE YEAR(@CurrentDateTime) * 10 + 2
            END,
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END,
            0 -- Update separately for holidays
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END
GO

-- Execute to populate 2 years of time data
EXEC PopulateDimTime @StartDate = '2025-01-01', @EndDate = '2027-01-01';
GO
```

### Step 3: Create Key Views and Aggregations

```sql
-- View: Warehouse operations with gravity scoring
CREATE VIEW vw_WarehouseOperationsEnriched AS
SELECT 
    wo.OperationKey,
    t.DateTime,
    t.DayName,
    t.HourOfDay,
    g.LocationName AS WarehouseName,
    g.City,
    p.SKU,
    p.ProductName,
    p.Category,
    p.GravityScore,
    p.RecommendedZone,
    wo.OperationType,
    wo.StorageZone,
    wo.QuantityHandled,
    wo.DwellTimeMinutes,
    wo.CycleTimeMinutes,
    wo.OperationCost,
    CASE 
        WHEN p.GravityScore >= 70 AND wo.StorageZone NOT LIKE '%High%' THEN 'Misplaced-High'
        WHEN p.GravityScore BETWEEN 40 AND 69 AND wo.StorageZone NOT LIKE '%Medium%' THEN 'Misplaced-Medium'
        WHEN p.GravityScore < 40 AND wo.StorageZone NOT LIKE '%Low%' THEN 'Misplaced-Low'
        ELSE 'Correct'
    END AS PlacementStatus
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
GO

-- View: Fleet performance with efficiency metrics
CREATE VIEW vw_FleetPerformance AS
SELECT 
    ft.TripKey,
    t1.DateTime AS TripStartTime,
    t2.DateTime AS TripEndTime,
    g1.LocationName AS OriginLocation,
    g2.LocationName AS DestinationLocation,
    ft.VehicleID,
    ft.DriverID,
    ft.DistanceKm,
    ft.DurationMinutes,
    ft.IdleTimeMinutes,
    ft.FuelConsumedLiters,
    ft.LoadWeightKg,
    ft.LoadRevenue,
    ft.TripCost,
    CASE WHEN ft.DurationMinutes > 0 THEN (ft.IdleTimeMinutes * 100.0 / ft.DurationMinutes) ELSE 0 END AS IdlePercentage,
    CASE WHEN ft.DistanceKm > 0 THEN (ft.FuelConsumedLiters / ft.DistanceKm) ELSE 0 END AS FuelEfficiency,
    CASE WHEN ft.TripCost > 0 THEN (ft.LoadRevenue / ft.TripCost) ELSE 0 END AS RevenuePerCost,
    ft.DelayMinutes,
    ft.DelayReason,
    CASE 
        WHEN ft.IdleTimeMinutes * 100.0 / NULLIF(ft.DurationMinutes, 0) > 15 THEN 'High Idle'
        WHEN ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0) > 0.4 THEN 'Fuel Inefficient'
        WHEN ft.DelayMinutes > 30 THEN 'Delayed'
        ELSE 'Normal'
    END AS PerformanceFlag
FROM FactFleetTrips ft
INNER JOIN DimTime t1 ON ft.TimeKeyStart = t1.TimeKey
LEFT JOIN DimTime t2 ON ft.TimeKeyEnd = t2.TimeKey
INNER JOIN DimGeography g1 ON ft.GeographyKeyOrigin = g1.GeographyKey
LEFT JOIN DimGeography g2 ON ft.GeographyKeyDestination = g2.GeographyKey
GO

-- View: Cross-fact KPI harmonization
CREATE VIEW vw_CrossFactKPIs AS
SELECT 
    t.Year,
    t.MonthName,
    g.LocationName,
    -- Warehouse metrics
    SUM(CASE WHEN wo.OperationType = 'Picking' THEN wo.QuantityHandled ELSE 0 END) AS TotalPicks,
    AVG(CASE WHEN wo.OperationType = 'Picking' THEN wo.CycleTimeMinutes ELSE NULL END) AS AvgPickCycleTime,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    -- Fleet metrics
    COUNT(DISTINCT ft.TripKey) AS TotalTrips,
    SUM(ft.DistanceKm) AS TotalDistanceKm,
    AVG(ft.IdleTimeMinutes * 100.0 / NULLIF(ft.DurationMinutes, 0)) AS AvgIdlePercentage,
    SUM(ft.LoadRevenue) AS TotalRevenue,
    SUM(ft.TripCost) AS TotalTripCost
FROM DimTime t
INNER JOIN DimGeography g ON 1=1
LEFT JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey AND g.GeographyKey = wo.GeographyKey
LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.TimeKeyStart AND g.GeographyKey = ft.GeographyKeyOrigin
GROUP BY t.Year, t.MonthName, g.LocationName
GO
```

### Step 4: Configure Data Ingestion

```sql
-- Stored procedure for incremental warehouse data load
CREATE PROCEDURE usp_LoadWarehouseOperations
    @SourceConnectionString NVARCHAR(1000),
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Use OPENQUERY or BULK INSERT to pull from source WMS
    -- Example assumes external staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType, OperationID,
        OrderID, QuantityHandled, DwellTimeMinutes, CycleTimeMinutes,
        StorageZone, EmployeeID, EquipmentID, OperationCost
    )
    SELECT 
        CAST(FORMAT(s.OperationDateTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        s.OperationID,
        s.OrderID,
        s.QuantityHandled,
        DATEDIFF(MINUTE, s.OperationStartTime, s.OperationEndTime) AS DwellTimeMinutes,
        s.CycleTimeMinutes,
        s.StorageZone,
        s.EmployeeID,
        s.EquipmentID,
        s.OperationCost
    FROM StagingWarehouseOperations s
    INNER JOIN DimGeography g ON s.LocationID = g.LocationID
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    WHERE s.OperationDateTime > @LastLoadDateTime;
    
    -- Log load metadata
    INSERT INTO LoadLog (TableName, LoadDateTime, RowsLoaded)
    VALUES ('FactWarehouseOperations', GETDATE(), @@ROWCOUNT);
END
GO

-- Scheduled job placeholder (configure via SQL Agent)
-- EXEC msdb.dbo.sp_add_job @job_name = 'LoadWarehouseOperations_Every15Min'
```

### Step 5: Power BI Configuration

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Connect to your SQL Server instance and LogiFleetPulse database
4. Import tables:
   - `DimTime`
   - `DimGeography`
   - `DimProductGravity`
   - `DimSupplierReliability`
   - `FactWarehouseOperations`
   - `FactFleetTrips`
   - `FactCrossDock`
   - Views: `vw_WarehouseOperationsEnriched`, `vw_FleetPerformance`

5. Configure relationships in Power BI Model view (should auto-detect based on foreign keys)

6. Create DAX measures:

```dax
// Total Operations
TotalOperations = COUNTROWS(FactWarehouseOperations)

// Avg Dwell Time
AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])

// High Gravity Misplacement Rate
MisplacementRate = 
DIVIDE(
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        vw_WarehouseOperationsEnriched[PlacementStatus] <> "Correct"
    ),
    COUNTROWS(FactWarehouseOperations),
    0
) * 100

// Fleet Idle Percentage
FleetIdlePercentage = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

// Revenue Per Trip
RevenuePerTrip = 
DIVIDE(
    SUM(FactFleetTrips[LoadRevenue]),
    COUNTROWS(FactFleetTrips),
    0
)

// Predictive Bottleneck Index (simplified)
BottleneckIndex = 
VAR HighDwell = IF([AvgDwellTime] > 120, 1, 0)
VAR HighIdle = IF([FleetIdlePercentage] > 15, 1, 0)
VAR HighMisplacement = IF([MisplacementRate] > 10, 1, 0)
RETURN (HighDwell + HighIdle + HighMisplacement) * 33.33
```

## Key API Patterns

### Querying Warehouse Gravity Zone Violations

```sql
-- Find all SKUs in wrong gravity zones in last 7 days
DECLARE @StartDate DATE = DATEADD(DAY, -7, GETDATE());

SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.RecommendedZone,
    wo.StorageZone AS CurrentZone,
    COUNT(*) AS ViolationCount,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    SUM(wo.OperationCost) AS TotalCost
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.DateTime >= @StartDate
    AND (
        (p.GravityScore >= 70 AND wo.StorageZone NOT LIKE '%High%')
        OR (p.GravityScore BETWEEN 40 AND 69 AND wo.StorageZone NOT LIKE '%Medium%')
        OR (p.GravityScore < 40 AND wo.StorageZone NOT LIKE '%Low%')
    )
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.RecommendedZone, wo.StorageZone
ORDER BY ViolationCount DESC;
```

### Fleet Triage Query

```sql
-- Generate proactive maintenance queue based on weighted scoring
SELECT 
    ft.VehicleID,
    COUNT(DISTINCT ft.TripKey) AS RecentTrips,
    AVG(ft.IdleTimeMinutes * 100.0 / NULLIF(ft.DurationMinutes, 0)) AS AvgIdlePercentage,
    AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0)) AS AvgFuelConsumption,
    SUM(ft.LoadRevenue) AS TotalRevenueAtRisk,
    CASE 
        WHEN AVG(ft.IdleTimeMinutes * 100.0 / NULLIF(ft.DurationMinutes, 0)) > 20 THEN 30
        WHEN AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0)) > 0.45 THEN 25
        ELSE 0
    END + 
    CASE WHEN SUM(ft.LoadRevenue) > 50000 THEN 45 ELSE 0 END AS TriageScore
FROM FactFleetTrips ft
INNER JOIN DimTime t ON ft.TimeKeyStart = t.TimeKey
WHERE t.DateTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY ft.VehicleID
HAVING AVG(ft.IdleTimeMinutes * 100.0 / NULLIF(ft.DurationMinutes, 0)) > 15
    OR AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0)) > 0.4
ORDER BY TriageScore DESC;
```

### Cross-Fact Correlation Query

```sql
-- Correlate warehouse dwell time with fleet delays for same products
SELECT 
    p.SKU,
    p.ProductName,
    AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwell,
    AVG(ft.DelayMinutes) AS AvgFleetDelay,
    COUNT(DISTINCT wo.OperationKey) AS WarehouseOperations,
    COUNT(DISTINCT ft.TripKey) AS FleetTrips,
    CASE 
        WHEN AVG(wo.DwellTimeMinutes) > 72 * 60 AND AVG(ft.DelayMinutes) > 30 THEN 'High Risk'
        WHEN AVG(wo.DwellTimeMinutes) > 48 * 60 OR AVG(ft.DelayMinutes) > 20 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS RiskCategory
FROM DimProductGravity p
LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
LEFT JOIN FactFleetTrips ft ON 1=1 -- Would need trip-product bridge table in production
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.DateTime >= DATEADD(DAY, -90, GETDATE())
GROUP BY p.SKU, p.ProductName
HAVING COUNT(DISTINCT wo.OperationKey) > 0
ORDER BY AvgWarehouseDwell DESC;
```

## Configuration

### Environment Variables

Set these in your deployment environment:

```bash
# SQL Server connection
export LOGIFLEET_SQL_SERVER="your-server.database.windows.net"
export LOGIFLEET_SQL_DATABASE="LogiFleetPulse"
export LOGIFLEET_SQL_USER="logiadmin"
export LOGIFLEET_SQL_PASSWORD="${SQL_PASSWORD}"

# External data sources
export LOGIFLEET_WMS_API_URL="https://wms.example.com/api/v1"
export LOGIFLEET_WMS_API_KEY="${WMS_API_KEY}"
export LOGIFLEET_TELEMATICS_API_URL="https://fleet.example.com/api/telemetry"
export LOGIFLEET_TELEMATICS_API_KEY="${TELEMATICS_API_KEY}"
export LOGIFLEET_WEATHER_API_KEY="${WEATHER_API_KEY}"

# Alert configuration
export LOGIFLEET_ALERT_EMAIL="alerts@yourcompany.com"
export LOGIFLEET_ALERT_SMS_NUMBER="+1234567890"
export LOGIFLEET_ALERT_TEAMS_WEBHOOK="${TEAMS_WEBHOOK_URL}"
```

### Alert Configuration SQL

```sql
-- Create alerts table
CREATE TABLE AlertConfig (
    AlertID INT IDENTITY(1,1) PRIMARY KEY,
    AlertName NVARCHAR(200),
    MetricType NVARCHAR(100), -- DwellTime, IdlePercentage, FuelConsumption, etc.
    ThresholdValue DECIMAL(10,2),
    ThresholdOperator NVARCHAR(10), -- >, <, =, >=, <=
    NotificationMethod NVARCHAR(50), -- Email, SMS, Teams
    NotificationTarget NVARCHAR(500),
    IsActive BIT DEFAULT 1
)

-- Insert sample alerts
INSERT INTO AlertConfig (AlertName, MetricType, ThresholdValue, ThresholdOperator, NotificationMethod, NotificationTarget, IsActive)
VALUES 
('High Dwell Time Alert', 'DwellTime', 120, '>', 'Email', 'warehouse@company.com', 1),
('Fleet Idle Warning', 'IdlePercentage', 15, '>', 'Teams', '${TEAMS_WEBHOOK_URL}', 1),
('Fuel Efficiency Alert', 'FuelConsumption', 0.45, '>', 'SMS', '+1234567890', 1);

-- Stored procedure to check alerts
CREATE PROCEDURE usp_CheckAlerts
AS
BEGIN
    -- Check warehouse dwell time alerts
    DECLARE @HighDwellCount INT = (
        SELECT COUNT(*)
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.DateTime >= DATEADD(HOUR, -1, GETDATE())
            AND wo.DwellTimeMinutes > (SELECT ThresholdValue FROM AlertConfig WHERE AlertName = 'High Dwell Time Alert')
    )
    
    IF @HighDwellCount > 0
    BEGIN
        -- Trigger notification (integrate with external API or database mail)
        PRINT 'Alert: ' + CAST(@HighDwellCount AS NVARCHAR) + ' operations exceeded dwell time threshold';
    END
    
    -- Add similar checks for other alerts
END
GO
```

## Common Patterns

### Pattern 1: Daily Warehouse Zone Optimization

```sql
-- Identify products that should be moved to different gravity zones
WITH ProductVelocity AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        COUNT(*) AS PicksLast30Days,
        AVG(wo.CycleTimeMinutes) AS AvgCycleTime,
        p.GravityScore AS CurrentGravityScore,
        -- Recalculate velocity score based on recent activity
        (COUNT(*) / 30.0) * 10 AS NewVelocityScore
    FROM DimProductGravity p
    LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(DAY, -30, GETDATE())
        AND wo.OperationType = 'Picking'
    GROUP BY p.ProductKey, p.SKU, p.ProductName, p.GravityScore
)
SELECT 
    SKU,
    ProductName,
    PicksLast30Days,
    CurrentGravityScore,
    (NewVelocityScore * 0.5 + (CurrentGravityScore - (CurrentGravityScore * 0.5))) AS RecommendedGravityScore,
    CASE 
        WHEN (NewVelocityScore * 0.5 + (CurrentGravityScore - (CurrentGravityScore * 0.5))) >= 70 THEN 'High-Gravity'
        WHEN (NewVelocityScore * 0.5 + (CurrentGravityScore - (CurrentGravity
