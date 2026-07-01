---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI logistics data warehouse with multi-fact star schema for warehouse, fleet, and supply chain intelligence
triggers:
  - set up logifleet pulse analytics warehouse
  - deploy supply chain data model with power bi
  - configure logistics intelligence star schema
  - implement warehouse and fleet kpi dashboards
  - create multi-fact supply chain data warehouse
  - build cross-modal logistics analytics platform
  - integrate warehouse operations with fleet telemetry
  - design real-time logistics intelligence system
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is a comprehensive data warehousing and analytics solution for logistics operations, combining MS SQL Server multi-fact star schema with Power BI dashboards. It harmonizes warehouse operations, fleet telemetry, inventory management, and external signals into unified supply chain intelligence with cross-fact KPI analysis.

## What It Does

- **Multi-Fact Star Schema**: Connects warehouse operations, fleet trips, and cross-dock activities through shared dimensions
- **Warehouse Gravity Zones**: Spatial optimization based on pick frequency, item value, and replenishment lead time
- **Fleet Triage Engine**: Proactive maintenance prioritization based on telemetry and revenue impact
- **Temporal Elasticity**: Time-phased simulation scenarios for capacity planning
- **Real-Time Dashboards**: 15-minute refresh intervals with predictive bottleneck detection
- **Cross-Fact Analytics**: Links inventory turnover with fuel consumption, dwell time with route efficiency

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS)
- Database permissions: CREATE TABLE, CREATE VIEW, CREATE PROCEDURE

### Deploy the SQL Schema

```sql
-- 1. Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- 2. Create dimension tables (time-aware)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2(0) NOT NULL,
    FifteenMinuteBucket SMALLINT NOT NULL,
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayOfMonth TINYINT NOT NULL,
    Month TINYINT NOT NULL,
    Quarter TINYINT NOT NULL,
    FiscalYear SMALLINT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT DEFAULT 0
)

CREATE CLUSTERED COLUMNSTORE INDEX CCI_DimTime ON DimTime
CREATE NONCLUSTERED INDEX IX_DimTime_DateTime ON DimTime(DateTime)
GO

-- 3. Geography dimension with hierarchy
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID NVARCHAR(50) NOT NULL UNIQUE,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50) NOT NULL, -- 'Warehouse', 'RouteNode', 'CrossDock'
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    IsActive BIT DEFAULT 1,
    ValidFrom DATETIME2(0) DEFAULT GETDATE(),
    ValidTo DATETIME2(0) NULL
)

CREATE NONCLUSTERED INDEX IX_DimGeography_LocationType ON DimGeography(LocationType) INCLUDE (LocationID)
GO

-- 4. Product dimension with gravity scoring
CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU NVARCHAR(100) NOT NULL UNIQUE,
    ProductName NVARCHAR(300) NOT NULL,
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    UnitValue DECIMAL(18,2),
    WeightKG DECIMAL(10,3),
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    -- Gravity scoring components
    PickFrequencyScore INT DEFAULT 0, -- 0-100
    ValueScore INT DEFAULT 0, -- 0-100
    FragilityScore INT DEFAULT 0, -- 0-100
    GravityIndex AS (PickFrequencyScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    GravityZone AS (
        CASE 
            WHEN (PickFrequencyScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) > 70 THEN 'High'
            WHEN (PickFrequencyScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) > 40 THEN 'Medium'
            ELSE 'Low'
        END
    ) PERSISTED
)

CREATE NONCLUSTERED INDEX IX_DimProduct_GravityZone ON DimProduct(GravityZone) INCLUDE (SKU, GravityIndex)
GO

-- 5. Supplier reliability dimension
CREATE TABLE DimSupplier (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierID NVARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200) NOT NULL,
    Country NVARCHAR(100),
    AvgLeadTimeDays INT,
    LeadTimeVariance DECIMAL(10,2),
    DefectRate DECIMAL(5,4),
    ComplianceScore INT, -- 0-100
    ReliabilityTier NVARCHAR(20) -- 'Platinum', 'Gold', 'Silver', 'Bronze'
)
GO

-- 6. Fact table: Warehouse operations
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProduct(ProductKey),
    OperationType NVARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OrderID NVARCHAR(100),
    UnitsHandled INT NOT NULL,
    DurationMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    StorageZone NVARCHAR(50),
    OperatorID NVARCHAR(50),
    EquipmentID NVARCHAR(50),
    ErrorFlag BIT DEFAULT 0,
    ErrorReason NVARCHAR(500)
)

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOps ON FactWarehouseOperations
CREATE NONCLUSTERED INDEX IX_FactWH_Time ON FactWarehouseOperations(TimeKey)
CREATE NONCLUSTERED INDEX IX_FactWH_Product ON FactWarehouseOperations(ProductKey) INCLUDE (UnitsHandled, DwellTimeHours)
GO

-- 7. Fact table: Fleet trips
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    StartTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    EndTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    RouteID NVARCHAR(100),
    DistanceKM DECIMAL(10,2),
    DurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,3),
    LoadWeightKG DECIMAL(10,2),
    TireAveragePressurePSI DECIMAL(5,2),
    EngineTemperatureAvg DECIMAL(5,2),
    DelayMinutes DECIMAL(10,2) DEFAULT 0,
    DelayReason NVARCHAR(500),
    OnTimeFlag BIT DEFAULT 1
)

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips ON FactFleetTrips
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID, StartTimeKey)
CREATE NONCLUSTERED INDEX IX_FactFleet_Route ON FactFleetTrips(RouteID) INCLUDE (DistanceKM, FuelConsumedLiters)
GO

-- 8. Bridge table: Trip to Product (many-to-many)
CREATE TABLE BridgeTripProduct (
    TripKey BIGINT NOT NULL FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProduct(ProductKey),
    UnitsShipped INT NOT NULL,
    PRIMARY KEY (TripKey, ProductKey)
)
GO

-- 9. Fact table: Cross-dock operations
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    InboundTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OutboundTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProduct(ProductKey),
    InboundVehicleID NVARCHAR(50),
    OutboundVehicleID NVARCHAR(50),
    UnitsTransferred INT NOT NULL,
    TransferDurationMinutes DECIMAL(10,2),
    DwellTimeMinutes DECIMAL(10,2)
)

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactCrossDock ON FactCrossDock
GO
```

### Populate Time Dimension

```sql
-- Generate 15-minute time buckets for 5 years
DECLARE @StartDate DATETIME2 = '2024-01-01'
DECLARE @EndDate DATETIME2 = '2029-12-31'
DECLARE @CurrentDate DATETIME2 = @StartDate

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO DimTime (TimeKey, DateTime, FifteenMinuteBucket, HourOfDay, DayOfWeek, 
                         DayOfMonth, Month, Quarter, FiscalYear, IsWeekend)
    VALUES (
        CONVERT(INT, FORMAT(@CurrentDate, 'yyyyMMddHHmm')),
        @CurrentDate,
        DATEPART(MINUTE, @CurrentDate) / 15,
        DATEPART(HOUR, @CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        DATEPART(DAY, @CurrentDate),
        DATEPART(MONTH, @CurrentDate),
        DATEPART(QUARTER, @CurrentDate),
        CASE WHEN DATEPART(MONTH, @CurrentDate) >= 7 THEN YEAR(@CurrentDate) + 1 ELSE YEAR(@CurrentDate) END,
        CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1, 7) THEN 1 ELSE 0 END
    )
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate)
END
GO
```

### Create Stored Procedures for ETL

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LoadDateTime DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON
    
    IF @LoadDateTime IS NULL
        SET @LoadDateTime = GETDATE()
    
    -- Get last loaded timestamp from metadata table (create if needed)
    DECLARE @LastLoadTime DATETIME2
    SELECT @LastLoadTime = ISNULL(MAX(LoadDateTime), '2024-01-01') 
    FROM ETLMetadata WHERE TableName = 'FactWarehouseOperations'
    
    -- Insert from staging table (assumes staging.WarehouseOps exists)
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType, OrderID, 
        UnitsHandled, DurationMinutes, DwellTimeHours, StorageZone, 
        OperatorID, EquipmentID, ErrorFlag, ErrorReason
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        s.OrderID,
        s.UnitsHandled,
        s.DurationMinutes,
        s.DwellTimeHours,
        s.StorageZone,
        s.OperatorID,
        s.EquipmentID,
        CASE WHEN s.ErrorCode IS NOT NULL THEN 1 ELSE 0 END,
        s.ErrorReason
    FROM staging.WarehouseOps s
    INNER JOIN DimTime t ON CONVERT(INT, FORMAT(s.OperationDateTime, 'yyyyMMddHHmm')) = t.TimeKey
    INNER JOIN DimGeography g ON s.LocationID = g.LocationID AND g.IsActive = 1
    INNER JOIN DimProduct p ON s.SKU = p.SKU
    WHERE s.OperationDateTime > @LastLoadTime
        AND s.OperationDateTime <= @LoadDateTime
    
    -- Update metadata
    INSERT INTO ETLMetadata (TableName, LoadDateTime, RowsLoaded)
    VALUES ('FactWarehouseOperations', @LoadDateTime, @@ROWCOUNT)
END
GO
```

### Analytical Views

```sql
-- Cross-fact KPI: Dwell time vs. fleet idle correlation
CREATE VIEW vw_DwellTimeFleetIdleCorrelation AS
SELECT 
    t.FiscalYear,
    t.Quarter,
    t.Month,
    g.Region,
    p.GravityZone,
    -- Warehouse metrics
    AVG(wh.DwellTimeHours) AS AvgDwellTimeHours,
    SUM(wh.UnitsHandled) AS TotalUnitsHandled,
    -- Fleet metrics (joined via product bridge)
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    AVG(ft.FuelConsumedLiters) AS AvgFuelPerTrip,
    -- Cross-fact KPI
    (AVG(wh.DwellTimeHours) * AVG(ft.IdleTimeMinutes)) AS DwellIdleImpactScore
FROM FactWarehouseOperations wh
INNER JOIN DimTime t ON wh.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wh.GeographyKey = g.GeographyKey
INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
INNER JOIN BridgeTripProduct btp ON p.ProductKey = btp.ProductKey
INNER JOIN FactFleetTrips ft ON btp.TripKey = ft.TripKey
GROUP BY t.FiscalYear, t.Quarter, t.Month, g.Region, p.GravityZone
GO

-- Warehouse gravity zone performance
CREATE VIEW vw_WarehouseGravityPerformance AS
SELECT 
    g.LocationName AS Warehouse,
    p.GravityZone,
    COUNT(*) AS OperationCount,
    SUM(wh.UnitsHandled) AS TotalUnits,
    AVG(wh.DurationMinutes) AS AvgDurationMinutes,
    AVG(wh.DwellTimeHours) AS AvgDwellTimeHours,
    SUM(CASE WHEN wh.ErrorFlag = 1 THEN 1 ELSE 0 END) AS ErrorCount,
    SUM(CASE WHEN wh.ErrorFlag = 1 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS ErrorRate
FROM FactWarehouseOperations wh
INNER JOIN DimGeography g ON wh.GeographyKey = g.GeographyKey
INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
GROUP BY g.LocationName, p.GravityZone
GO

-- Fleet maintenance priority scoring
CREATE VIEW vw_FleetMaintenancePriority AS
SELECT 
    ft.VehicleID,
    COUNT(*) AS TripCount,
    SUM(ft.DistanceKM) AS TotalDistanceKM,
    AVG(ft.TireAveragePressurePSI) AS AvgTirePressure,
    AVG(ft.EngineTemperatureAvg) AS AvgEngineTemp,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    -- Revenue impact score (simplified)
    SUM(p.UnitValue * btp.UnitsShipped) AS TotalCargoValue,
    -- Priority score: lower tire pressure + higher cargo value = higher priority
    (35 - ISNULL(AVG(ft.TireAveragePressurePSI), 35)) * 10 + 
    LOG(SUM(p.UnitValue * btp.UnitsShipped) + 1) AS MaintenancePriorityScore
FROM FactFleetTrips ft
INNER JOIN BridgeTripProduct btp ON ft.TripKey = btp.TripKey
INNER JOIN DimProduct p ON btp.ProductKey = p.ProductKey
WHERE ft.StartTimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd0000'))
GROUP BY ft.VehicleID
GO
```

## Power BI Configuration

### Connect to SQL Server

1. Open Power BI Desktop
2. **Get Data** → **SQL Server**
3. Enter server name and database: `LogiFleetPulse`
4. Select **DirectQuery** for real-time dashboards or **Import** for scheduled refresh
5. Choose tables:
   - All `Dim*` tables
   - All `Fact*` tables
   - `BridgeTripProduct`
   - All analytical views (`vw_*`)

### Define Relationships (Auto-detect should work)

```
DimTime (TimeKey) → FactWarehouseOperations (TimeKey) [1:*]
DimTime (TimeKey) → FactFleetTrips (StartTimeKey) [1:*]
DimGeography (GeographyKey) → FactWarehouseOperations (GeographyKey) [1:*]
DimProduct (ProductKey) → FactWarehouseOperations (ProductKey) [1:*]
FactFleetTrips (TripKey) → BridgeTripProduct (TripKey) [1:*]
DimProduct (ProductKey) → BridgeTripProduct (ProductKey) [1:*]
```

### Create DAX Measures

```dax
// Total Units Handled (Warehouse)
TotalUnitsHandled = SUM(FactWarehouseOperations[UnitsHandled])

// Average Dwell Time
AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeHours])

// Fleet Utilization Rate
FleetUtilization = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes])
)

// On-Time Delivery Rate
OnTimeRate = 
DIVIDE(
    COUNTROWS(FILTER(FactFleetTrips, FactFleetTrips[OnTimeFlag] = TRUE())),
    COUNTROWS(FactFleetTrips)
)

// Cross-Fact KPI: Cost per Unit Shipped
CostPerUnit = 
DIVIDE(
    SUM(FactFleetTrips[FuelConsumedLiters]) * 1.5, -- Assume $1.50 per liter
    SUMX(
        BridgeTripProduct,
        BridgeTripProduct[UnitsShipped]
    )
)

// High Gravity Zone Efficiency
HighGravityEfficiency = 
CALCULATE(
    [TotalUnitsHandled] / [AvgDwellTime],
    DimProduct[GravityZone] = "High"
)

// Predictive Bottleneck Index (simplified)
BottleneckIndex = 
VAR CurrentDwell = [AvgDwellTime]
VAR HistoricalDwell = 
    CALCULATE(
        [AvgDwellTime],
        DATESINPERIOD(DimTime[DateTime], MAX(DimTime[DateTime]), -30, DAY)
    )
RETURN
IF(CurrentDwell > HistoricalDwell * 1.2, CurrentDwell / HistoricalDwell, 0)
```

### Dashboard Components

**Page 1: Warehouse Operations Overview**
- KPI Cards: Total Units Handled, Avg Dwell Time, Error Rate
- Line Chart: Units Handled by Hour (15-min buckets)
- Bar Chart: Operations by Gravity Zone
- Heatmap: Dwell Time by Storage Zone and Time of Day

**Page 2: Fleet Intelligence**
- KPI Cards: Fleet Utilization, Fuel Efficiency, On-Time Rate
- Map Visual: Active Routes with Color by Delay Status
- Table: Fleet Maintenance Priority (from `vw_FleetMaintenancePriority`)
- Scatter Chart: Distance vs. Idle Time (bubble size = cargo value)

**Page 3: Cross-Fact Analytics**
- Matrix: Dwell Time vs. Fleet Idle Correlation by Region
- Line Chart: Cost Per Unit Trend over Time
- Decomposition Tree: High Bottleneck Index drill-down
- What-If Parameter: Simulate capacity changes (80% to 95%)

### Row-Level Security

```dax
// Create role: RegionalManager
[Region] = USERNAME()

// Apply to DimGeography table
// Users see only data from their assigned region
```

## Common Patterns and Use Cases

### Pattern 1: Identify Slow-Moving SKUs in Wrong Zones

```sql
-- Find products in high-gravity zones with low pick frequency
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityZone,
    p.PickFrequencyScore,
    AVG(wh.DwellTimeHours) AS AvgDwellTime,
    COUNT(*) AS PickCount
FROM DimProduct p
INNER JOIN FactWarehouseOperations wh ON p.ProductKey = wh.ProductKey
WHERE wh.OperationType = 'Picking'
    AND wh.TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMdd0000'))
GROUP BY p.SKU, p.ProductName, p.GravityZone, p.PickFrequencyScore
HAVING p.GravityZone = 'High' AND COUNT(*) < 10
ORDER BY AvgDwellTime DESC
```

### Pattern 2: Fleet Triage for Proactive Maintenance

```sql
-- Vehicles needing immediate attention
SELECT TOP 10
    VehicleID,
    MaintenancePriorityScore,
    TotalCargoValue,
    AvgTirePressure,
    AvgEngineTemp,
    CASE 
        WHEN AvgTirePressure < 30 THEN 'Critical - Tire Pressure'
        WHEN AvgEngineTemp > 105 THEN 'Critical - Overheating'
        WHEN AvgIdleTime > 20 THEN 'Warning - High Idle'
        ELSE 'Monitor'
    END AS PriorityReason
FROM vw_FleetMaintenancePriority
ORDER BY MaintenancePriorityScore DESC
```

### Pattern 3: Cross-Dock Efficiency Analysis

```sql
-- Cross-dock operations exceeding target dwell time
SELECT 
    g.LocationName,
    p.Category,
    COUNT(*) AS TransferCount,
    AVG(cd.TransferDurationMinutes) AS AvgTransferMinutes,
    AVG(cd.DwellTimeMinutes) AS AvgDwellMinutes,
    SUM(CASE WHEN cd.DwellTimeMinutes > 120 THEN 1 ELSE 0 END) AS OverThresholdCount
FROM FactCrossDock cd
INNER JOIN DimGeography g ON cd.GeographyKey = g.GeographyKey
INNER JOIN DimProduct p ON cd.ProductKey = p.ProductKey
WHERE cd.InboundTimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMdd0000'))
GROUP BY g.LocationName, p.Category
HAVING AVG(cd.DwellTimeMinutes) > 60
ORDER BY AvgDwellMinutes DESC
```

### Pattern 4: Temporal Elasticity Simulation (What-If)

```sql
-- Simulate 95% warehouse capacity impact
WITH CurrentCapacity AS (
    SELECT 
        GeographyKey,
        SUM(UnitsHandled) AS CurrentUnits,
        AVG(DurationMinutes) AS AvgDuration
    FROM FactWarehouseOperations
    WHERE TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd0000'))
    GROUP BY GeographyKey
),
SimulatedCapacity AS (
    SELECT 
        GeographyKey,
        CurrentUnits * 1.1875 AS SimulatedUnits, -- 80% to 95% = 18.75% increase
        AvgDuration * 1.15 AS SimulatedDuration -- Assume 15% longer ops
    FROM CurrentCapacity
)
SELECT 
    g.LocationName,
    cc.CurrentUnits,
    sc.SimulatedUnits,
    cc.AvgDuration,
    sc.SimulatedDuration,
    (sc.SimulatedDuration - cc.AvgDuration) AS ExtraMinutesNeeded,
    (sc.SimulatedUnits - cc.CurrentUnits) AS AdditionalCapacity
FROM CurrentCapacity cc
INNER JOIN SimulatedCapacity sc ON cc.GeographyKey = sc.GeographyKey
INNER JOIN DimGeography g ON cc.GeographyKey = g.GeographyKey
ORDER BY ExtraMinutesNeeded DESC
```

## Integration with External Systems

### Ingest Telemetry via REST API

```sql
-- Create external table for JSON telemetry (requires PolyBase)
CREATE EXTERNAL DATA SOURCE TelemetryAPI
WITH (
    TYPE = REST,
    LOCATION = N'https://telemetry-api.example.com/v1/trips',
    CREDENTIAL = TelemetryAPIKey -- Stored credential
)

-- Scheduled job to pull and insert
INSERT INTO FactFleetTrips (...)
SELECT 
    -- JSON parsing logic
FROM OPENROWSET(
    BULK '/trips/latest',
    DATA_SOURCE = 'TelemetryAPI',
    FORMAT = 'JSON'
) AS telemetry
```

### Export to Power Automate

```sql
-- Stored procedure called by Power Automate webhook
CREATE PROCEDURE usp_TriggerHighDwellAlert
AS
BEGIN
    SELECT 
        p.SKU,
        g.LocationName,
        wh.DwellTimeHours,
        wh.OrderID
    FROM FactWarehouseOperations wh
    INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON wh.GeographyKey = g.GeographyKey
    WHERE wh.DwellTimeHours > 72
        AND wh.TimeKey >= CONVERT(INT, FORMAT(DATEADD(HOUR, -1, GETDATE()), 'yyyyMMddHHmm'))
END
GO
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Solution**: Ensure columnstore indexes are rebuilt and statistics are updated

```sql
-- Rebuild columnstore indexes
ALTER INDEX CCI_FactWarehouseOps ON FactWarehouseOperations REBUILD
ALTER INDEX CCI_FactFleetTrips ON FactFleetTrips REBUILD

-- Update statistics with full scan
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN
UPDATE STATISTICS BridgeTripProduct WITH FULLSCAN
```

### Issue: Power BI Relationship Ambiguity

**Symptom**: Measures returning incorrect totals when filtering by time

**Solution**: Mark the primary time relationship as active, others as inactive

```
DimTime → FactFleetTrips[StartTimeKey] = ACTIVE
DimTime → FactFleetTrips[EndTimeKey] = INACTIVE
```

Use `USERELATIONSHIP()` in DAX for alternative time context:

```dax
TripsByEndTime = 
CALCULATE(
    COUNTROWS(FactFleetTrips),
    USERELATIONSHIP(DimTime[TimeKey], FactFleetTrips[EndTimeKey])
)
```

### Issue: Missing Data After ETL Load

**Check**: Verify foreign key matches and staging data quality

```sql
-- Find orphaned records
SELECT *
FROM staging.WarehouseOps s
WHERE NOT EXISTS (
    SELECT 1 FROM DimProduct p WHERE p.SKU = s.SKU
)

-- Add missing products to dimension
INSERT INTO DimProduct (SKU, ProductName, Category)
SELECT DISTINCT SKU, SKU AS ProductName, 'Unknown' AS Category
FROM staging.WarehouseOps
WHERE SKU NOT IN (SELECT SKU FROM DimProduct)
```

### Issue: Incorrect Gravity Zone Assignments

**Solution**: Recalculate gravity scores based on recent activity

```sql
-- Update pick frequency scores from last 90 days
WITH PickFrequency AS (
    SELECT 
        ProductKey,
        COUNT(*) AS PickCount,
        NTILE(100) OVER (ORDER BY COUNT(*)) AS FrequencyPercentile
    FROM FactWarehouseOperations
    WHERE OperationType = 'Picking'
        AND TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMdd0000'))
    GROUP BY ProductKey
)
UPDATE p
SET PickFrequencyScore = pf.FrequencyPercentile
FROM DimProduct p
INNER JOIN PickFrequency pf ON p.ProductKey = pf.ProductKey
```

## Environment Variables

Reference configuration via environment variables (never hardcode):

```sql
-- Use SQL Server configuration table
CREATE TABLE SystemConfig (
    ConfigKey NVARCHAR(100) PRIMARY KEY,
    ConfigValue NVARCHAR(500),
    Description NVARCHAR(500)
)

INSERT INTO SystemConfig VALUES
('TELEMETRY_API_ENDPOINT', '$(ENV:TELEMETRY_API_URL)', 'Fleet telemetry API endpoint'),
('ALERT_EMAIL', '$(ENV:ALERT_EMAIL_ADDRESS)', 'Email for critical alerts'),
('REFRESH_INTERVAL_MINUTES', '15', 'Data refresh cadence')
```

Power BI connection string:

```
Server=$(ENV:SQL_SERVER);Database=LogiFleetPulse;Trusted_Connection=True
```
