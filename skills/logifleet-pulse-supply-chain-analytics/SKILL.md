---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI data warehousing solution for multi-modal logistics intelligence with cross-fact KPI harmonization and predictive fleet optimization
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure logicore warehouse and fleet tracking"
  - "deploy SQL schema for logistics data warehouse"
  - "create Power BI dashboard for supply chain metrics"
  - "implement cross-modal logistics intelligence"
  - "build multi-fact star schema for fleet operations"
  - "integrate warehouse gravity zones analytics"
  - "setup real-time fleet telemetry dashboards"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced MS SQL Server and Power BI data warehousing solution for logistics and supply chain analytics. It provides:

- **Multi-fact star schema** with time-phased dimensions for warehouse operations, fleet trips, and cross-dock activities
- **Cross-fact KPI harmonization** linking inventory metrics with fleet performance
- **Warehouse Gravity Zones™** spatial optimization based on pick frequency and item value
- **Predictive bottleneck detection** using temporal elasticity modeling
- **Real-time Power BI dashboards** with 15-minute refresh cycles
- **Role-based access control** with row-level security

Primary language: **SQL** (T-SQL for MS SQL Server) with Power BI integration.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- WMS/TMS data sources with SQL, REST API, or flat file export capability

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
    FullDateTime DATETIME2 NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    DayOfMonth INT NOT NULL,
    DayOfWeek INT NOT NULL,
    HourOfDay INT NOT NULL,
    MinuteBucket INT NOT NULL, -- 0, 15, 30, 45
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT DEFAULT 0,
    IsHoliday BIT DEFAULT 0
)
GO

CREATE CLUSTERED INDEX IX_DimTime_FullDateTime ON DimTime(FullDateTime)
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    Continent VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    WarehouseCode VARCHAR(50),
    WarehouseName VARCHAR(200),
    RouteNodeID VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50)
)
GO

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(500),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    UnitValue DECIMAL(10,2),
    VelocityScore DECIMAL(5,2), -- Calculated from historical pick frequency
    FragilityScore DECIMAL(5,2), -- 0-10 scale
    GravityScore AS (VelocityScore * 0.5 + UnitValue * 0.3 + FragilityScore * 0.2), -- Computed column
    RecommendedZone VARCHAR(50)
)
GO

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(100) NOT NULL UNIQUE,
    SupplierName VARCHAR(500),
    LeadTimeMean INT, -- Days
    LeadTimeStdDev DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    ComplianceScore DECIMAL(3,2), -- 0-1 scale
    LastUpdated DATETIME2 DEFAULT GETDATE()
)
GO

CREATE TABLE DimFleet (
    FleetKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID VARCHAR(100) NOT NULL UNIQUE,
    VehicleType VARCHAR(100),
    CapacityKg DECIMAL(10,2),
    CapacityCubicM DECIMAL(10,2),
    FuelType VARCHAR(50),
    LastMaintenanceDate DATE,
    NextMaintenanceDue DATE,
    TireConditionScore DECIMAL(3,2)
)
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    Quantity INT NOT NULL,
    DwellTimeMinutes INT,
    PickTimeSeconds INT,
    PackTimeSeconds INT,
    StorageZone VARCHAR(50),
    BatchNumber VARCHAR(100),
    OrderID VARCHAR(100)
)
GO

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey)
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey)
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Geography ON FactWarehouseOperations(GeographyKey)
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    FleetKey INT NOT NULL FOREIGN KEY REFERENCES DimFleet(FleetKey),
    TimeKeyStart INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    TimeKeyEnd INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DelayMinutes INT,
    DelayReason VARCHAR(200),
    DriverID VARCHAR(100)
)
GO

CREATE NONCLUSTERED INDEX IX_FactFleet_TimeStart ON FactFleetTrips(TimeKeyStart)
CREATE NONCLUSTERED INDEX IX_FactFleet_Fleet ON FactFleetTrips(FleetKey)
CREATE NONCLUSTERED INDEX IX_FactFleet_Origin ON FactFleetTrips(OriginGeographyKey)
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKeyReceived INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    TimeKeyShipped INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    Quantity INT NOT NULL,
    CrossDockDurationMinutes INT
)
GO
```

### Step 2: Populate Time Dimension

```sql
-- Stored procedure to populate DimTime at 15-minute granularity
CREATE PROCEDURE PopulateDimTime
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    DECLARE @CurrentDateTime DATETIME2 = @StartDate
    DECLARE @TimeKey INT
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        SET @TimeKey = CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT)
        
        INSERT INTO DimTime (TimeKey, FullDateTime, Year, Quarter, Month, DayOfMonth, DayOfWeek, HourOfDay, MinuteBucket, IsWeekend)
        VALUES (
            @TimeKey,
            @CurrentDateTime,
            YEAR(@CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            MONTH(@CurrentDateTime),
            DAY(@CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATEPART(HOUR, @CurrentDateTime),
            (DATEPART(MINUTE, @CurrentDateTime) / 15) * 15,
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END
        )
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime)
    END
END
GO

-- Populate for current year + 2 years
EXEC PopulateDimTime '2025-01-01', '2027-12-31'
GO
```

### Step 3: Configure Data Ingestion

```sql
-- Example stored procedure for incremental warehouse operations load
CREATE PROCEDURE LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    -- Assuming source system table: staging.WarehouseOps
    INSERT INTO FactWarehouseOperations (
        TimeKey,
        GeographyKey,
        ProductKey,
        SupplierKey,
        OperationType,
        Quantity,
        DwellTimeMinutes,
        PickTimeSeconds,
        PackTimeSeconds,
        StorageZone,
        BatchNumber,
        OrderID
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        ds.SupplierKey,
        so.OperationType,
        so.Quantity,
        so.DwellTimeMinutes,
        so.PickTimeSeconds,
        so.PackTimeSeconds,
        so.StorageZone,
        so.BatchNumber,
        so.OrderID
    FROM staging.WarehouseOps so
    INNER JOIN DimTime dt ON dt.FullDateTime = DATEADD(MINUTE, (DATEPART(MINUTE, so.OperationDateTime) / 15) * 15, DATEADD(HOUR, DATEDIFF(HOUR, 0, so.OperationDateTime), 0))
    INNER JOIN DimGeography dg ON dg.WarehouseCode = so.WarehouseCode
    INNER JOIN DimProductGravity dp ON dp.SKU = so.SKU
    LEFT JOIN DimSupplierReliability ds ON ds.SupplierID = so.SupplierID
    WHERE so.OperationDateTime > @LastLoadDateTime
END
GO
```

### Step 4: Power BI Connection

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Enter server name and database: `LogiFleetPulse`
4. Authentication: Windows or SQL Server (use `${SQL_USERNAME}` and `${SQL_PASSWORD}` env vars)
5. Select tables: `FactWarehouseOperations`, `FactFleetTrips`, `FactCrossDock`, and all `Dim*` tables
6. Power BI auto-detects relationships; verify in Model view

## Key SQL Views & Queries

### Cross-Fact KPI: Dwell Time vs Fleet Idle Time

```sql
CREATE VIEW vw_DwellVsIdleCorrelation AS
SELECT 
    dt.Year,
    dt.Month,
    dg.WarehouseName,
    dp.Category,
    AVG(CAST(fwo.DwellTimeMinutes AS FLOAT)) AS AvgDwellTimeMin,
    AVG(CAST(fft.IdleTimeMinutes AS FLOAT)) AS AvgIdleTimeMin,
    COUNT(DISTINCT fwo.ProductKey) AS DistinctProducts,
    COUNT(DISTINCT fft.TripKey) AS TotalTrips
FROM FactWarehouseOperations fwo
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
INNER JOIN DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
LEFT JOIN FactCrossDock fcd ON fcd.ProductKey = fwo.ProductKey AND fcd.GeographyKey = fwo.GeographyKey
LEFT JOIN FactFleetTrips fft ON fft.TripKey = fcd.OutboundTripKey
WHERE fwo.OperationType = 'Shipping'
GROUP BY dt.Year, dt.Month, dg.WarehouseName, dp.Category
GO
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products in suboptimal zones
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    dp.RecommendedZone,
    fwo.StorageZone AS CurrentZone,
    COUNT(*) AS PickCount,
    AVG(fwo.PickTimeSeconds) AS AvgPickTime
FROM FactWarehouseOperations fwo
INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
WHERE fwo.OperationType = 'Picking'
    AND fwo.StorageZone != dp.RecommendedZone
    AND fwo.TimeKey >= CAST(FORMAT(DATEADD(MONTH, -3, GETDATE()), 'yyyyMMddHHmm') AS INT)
GROUP BY dp.SKU, dp.ProductName, dp.GravityScore, dp.RecommendedZone, fwo.StorageZone
HAVING COUNT(*) > 10
ORDER BY dp.GravityScore DESC, AVG(fwo.PickTimeSeconds) DESC
GO
```

### Fleet Maintenance Priority Queue

```sql
CREATE VIEW vw_FleetMaintenancePriority AS
SELECT 
    df.VehicleID,
    df.VehicleType,
    df.TireConditionScore,
    DATEDIFF(DAY, df.LastMaintenanceDate, GETDATE()) AS DaysSinceMaintenance,
    DATEDIFF(DAY, GETDATE(), df.NextMaintenanceDue) AS DaysUntilDue,
    SUM(fft.DistanceKm) AS TotalDistanceLast30Days,
    AVG(fft.IdleTimeMinutes) AS AvgIdleTime,
    COUNT(DISTINCT CASE WHEN fft.DelayMinutes > 30 THEN fft.TripKey END) AS DelayedTrips,
    -- Weighted priority score
    (
        (1.0 - df.TireConditionScore) * 30 +
        CASE WHEN DATEDIFF(DAY, GETDATE(), df.NextMaintenanceDue) < 0 THEN 50 ELSE 0 END +
        (AVG(fft.IdleTimeMinutes) / 60.0) * 20
    ) AS PriorityScore
FROM DimFleet df
LEFT JOIN FactFleetTrips fft ON df.FleetKey = fft.FleetKey
WHERE fft.TimeKeyStart >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
GROUP BY df.VehicleID, df.VehicleType, df.TireConditionScore, df.LastMaintenanceDate, df.NextMaintenanceDue
GO
```

## Power BI DAX Measures

### Cross-Fact Composite Metric

```dax
// Measure: Inventory Turn-Fleet Efficiency Index
InventoryTurnFleetIndex = 
VAR AvgDwellHours = AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) / 60
VAR AvgFleetUtilization = 
    DIVIDE(
        AVERAGE(FactFleetTrips[DurationMinutes]) - AVERAGE(FactFleetTrips[IdleTimeMinutes]),
        AVERAGE(FactFleetTrips[DurationMinutes])
    )
VAR TurnIndex = DIVIDE(1, AvgDwellHours, 0) * 100
VAR FleetIndex = AvgFleetUtilization * 100
RETURN
    (TurnIndex * 0.6) + (FleetIndex * 0.4)
```

### Predictive Bottleneck Score

```dax
// Measure: Bottleneck Probability Index
BottleneckIndex = 
VAR CurrentDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR HistoricalDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATESBETWEEN(DimTime[FullDateTime], DATE(2025,1,1), DATE(2025,12,31))
    )
VAR DwellDeviation = DIVIDE(CurrentDwell - HistoricalDwell, HistoricalDwell, 0)

VAR CurrentIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR HistoricalIdle = 
    CALCULATE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes]),
        DATESBETWEEN(DimTime[FullDateTime], DATE(2025,1,1), DATE(2025,12,31))
    )
VAR IdleDeviation = DIVIDE(CurrentIdle - HistoricalIdle, HistoricalIdle, 0)

RETURN
    (DwellDeviation * 0.5 + IdleDeviation * 0.5) * 100
```

## Configuration

### Environment Variables

Store connection strings and API keys in environment variables:

```sql
-- SQL Server connection (use in application config, not hardcoded)
-- Server=${SQL_SERVER_HOST}
-- Database=LogiFleetPulse
-- User=${SQL_USERNAME}
-- Password=${SQL_PASSWORD}
```

### Row-Level Security Setup

```sql
-- Create security table
CREATE TABLE SecurityRoles (
    UserEmail VARCHAR(255),
    RoleName VARCHAR(100),
    WarehouseAccessList VARCHAR(MAX), -- Comma-separated warehouse codes
    ProductCategoryAccess VARCHAR(MAX)
)
GO

-- Create RLS function
CREATE FUNCTION fn_SecurityPredicate(@GeographyKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN 
    SELECT 1 AS accessResult
    WHERE 
        @GeographyKey IN (
            SELECT dg.GeographyKey
            FROM dbo.DimGeography dg
            INNER JOIN dbo.SecurityRoles sr ON 
                CHARINDEX(dg.WarehouseCode, sr.WarehouseAccessList) > 0
            WHERE sr.UserEmail = USER_NAME()
        )
GO

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(GeographyKey) ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(OriginGeographyKey) ON dbo.FactFleetTrips
WITH (STATE = ON)
GO
```

### Automated Alerts Configuration

```sql
CREATE PROCEDURE ConfigureAlert
    @AlertName VARCHAR(100),
    @MetricType VARCHAR(50), -- 'DwellTime', 'IdleTime', 'DelayRate'
    @ThresholdValue DECIMAL(10,2),
    @ComparisonOperator VARCHAR(10), -- '>', '<', '='
    @RecipientEmails VARCHAR(MAX)
AS
BEGIN
    -- Store alert config in AlertConfig table
    INSERT INTO AlertConfig (AlertName, MetricType, ThresholdValue, ComparisonOperator, RecipientEmails, IsActive)
    VALUES (@AlertName, @MetricType, @ThresholdValue, @ComparisonOperator, @RecipientEmails, 1)
    
    -- Schedule SQL Server Agent job to run every 15 minutes
    -- Implementation depends on environment
END
GO
```

## Common Patterns

### Pattern 1: Time-Phased Query with 15-Min Buckets

```sql
-- Get hourly aggregates from 15-minute buckets
SELECT 
    dt.Year,
    dt.Month,
    dt.DayOfMonth,
    dt.HourOfDay,
    COUNT(*) AS OperationCount,
    AVG(fwo.DwellTimeMinutes) AS AvgDwell,
    MAX(fwo.DwellTimeMinutes) AS MaxDwell
FROM FactWarehouseOperations fwo
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
WHERE dt.FullDateTime >= DATEADD(DAY, -7, GETDATE())
GROUP BY dt.Year, dt.Month, dt.DayOfMonth, dt.HourOfDay
ORDER BY dt.Year, dt.Month, dt.DayOfMonth, dt.HourOfDay
```

### Pattern 2: Bridge Table for Many-to-Many

```sql
-- If products can be on multiple routes simultaneously
CREATE TABLE BridgeProductRoute (
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    TripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    QuantityOnTrip INT
)
GO

-- Query using bridge
SELECT 
    dp.SKU,
    df.VehicleID,
    SUM(bpr.QuantityOnTrip) AS TotalQuantity
FROM BridgeProductRoute bpr
INNER JOIN DimProductGravity dp ON bpr.ProductKey = dp.ProductKey
INNER JOIN FactFleetTrips fft ON bpr.TripKey = fft.TripKey
INNER JOIN DimFleet df ON fft.FleetKey = df.FleetKey
GROUP BY dp.SKU, df.VehicleID
```

### Pattern 3: Incremental Load with Change Detection

```sql
CREATE PROCEDURE IncrementalLoadFleetTrips
AS
BEGIN
    DECLARE @LastLoadTime DATETIME2
    SELECT @LastLoadTime = MAX(dt.FullDateTime) 
    FROM FactFleetTrips fft
    INNER JOIN DimTime dt ON fft.TimeKeyStart = dt.TimeKey
    
    INSERT INTO FactFleetTrips (FleetKey, TimeKeyStart, TimeKeyEnd, OriginGeographyKey, DestinationGeographyKey, DistanceKm, DurationMinutes, IdleTimeMinutes, FuelConsumedLiters, LoadWeightKg)
    SELECT 
        df.FleetKey,
        dtStart.TimeKey,
        dtEnd.TimeKey,
        dgOrigin.GeographyKey,
        dgDest.GeographyKey,
        st.DistanceKm,
        st.DurationMinutes,
        st.IdleTimeMinutes,
        st.FuelConsumedLiters,
        st.LoadWeightKg
    FROM staging.FleetTelemetry st
    INNER JOIN DimFleet df ON df.VehicleID = st.VehicleID
    INNER JOIN DimTime dtStart ON dtStart.FullDateTime = DATEADD(MINUTE, (DATEPART(MINUTE, st.StartDateTime) / 15) * 15, DATEADD(HOUR, DATEDIFF(HOUR, 0, st.StartDateTime), 0))
    INNER JOIN DimTime dtEnd ON dtEnd.FullDateTime = DATEADD(MINUTE, (DATEPART(MINUTE, st.EndDateTime) / 15) * 15, DATEADD(HOUR, DATEDIFF(HOUR, 0, st.EndDateTime), 0))
    INNER JOIN DimGeography dgOrigin ON dgOrigin.RouteNodeID = st.OriginNodeID
    INNER JOIN DimGeography dgDest ON dgDest.RouteNodeID = st.DestinationNodeID
    WHERE st.StartDateTime > @LastLoadTime
END
GO
```

## Troubleshooting

### Issue: Time dimension missing keys

**Symptom**: Fact table inserts fail with foreign key constraint errors on `TimeKey`.

**Solution**: Ensure time dimension is populated for the full date range:

```sql
-- Check time dimension coverage
SELECT MIN(FullDateTime) AS MinDate, MAX(FullDateTime) AS MaxDate FROM DimTime

-- Extend if needed
EXEC PopulateDimTime '2027-01-01', '2028-12-31'
```

### Issue: Power BI refresh fails with timeout

**Symptom**: Scheduled refresh in Power BI Service times out after 2 hours.

**Solution**: Implement incremental refresh in Power BI:

1. Add `RangeStart` and `RangeEnd` parameters (DateTime type)
2. Filter fact tables: `FullDateTime >= RangeStart and FullDateTime < RangeEnd`
3. In Power BI Service, configure Incremental Refresh policy (e.g., refresh last 30 days, keep 2 years)

### Issue: Cross-fact queries slow

**Symptom**: Views joining multiple fact tables take > 30 seconds.

**Solution**: Create indexed views or aggregate tables:

```sql
CREATE VIEW vw_DailyFleetSummary
WITH SCHEMABINDING
AS
SELECT 
    fft.FleetKey,
    dt.Year,
    dt.Month,
    dt.DayOfMonth,
    COUNT_BIG(*) AS TripCount,
    SUM(fft.DistanceKm) AS TotalDistance,
    SUM(fft.FuelConsumedLiters) AS TotalFuel,
    AVG(fft.IdleTimeMinutes) AS AvgIdle
FROM dbo.FactFleetTrips fft
INNER JOIN dbo.DimTime dt ON fft.TimeKeyStart = dt.TimeKey
GROUP BY fft.FleetKey, dt.Year, dt.Month, dt.DayOfMonth
GO

CREATE UNIQUE CLUSTERED INDEX IX_FleetSummary ON vw_DailyFleetSummary(FleetKey, Year, Month, DayOfMonth)
GO
```

### Issue: Gravity score not updating

**Symptom**: `DimProductGravity.GravityScore` shows outdated values.

**Solution**: GravityScore is a computed column; update base columns:

```sql
UPDATE DimProductGravity
SET VelocityScore = (
    SELECT COUNT(*) / 90.0 -- Picks per day over last 90 days
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    WHERE fwo.ProductKey = DimProductGravity.ProductKey
        AND fwo.OperationType = 'Picking'
        AND dt.FullDateTime >= DATEADD(DAY, -90, GETDATE())
)
WHERE ProductKey IN (SELECT DISTINCT ProductKey FROM FactWarehouseOperations)
```

### Issue: Row-level security not working

**Symptom**: Users see all warehouses despite RLS policy.

**Solution**: Verify user is mapped correctly:

```sql
-- Check current user mapping
SELECT USER_NAME()

-- Verify security table entries
SELECT * FROM SecurityRoles WHERE UserEmail = 'user@company.com'

-- Test security function directly
SELECT * FROM dbo.fn_SecurityPredicate(5) -- Replace 5 with GeographyKey
```

## Advanced Use Cases

### Scenario: Correlate Weather Delays with Route Performance

```sql
-- Assuming external weather data in staging.WeatherEvents
CREATE TABLE staging.WeatherEvents (
    EventDateTime DATETIME2,
    GeographyKey INT,
    WeatherType VARCHAR(50), -- 'Snow', 'Rain', 'Fog'
    SeverityLevel INT -- 1-5
)
GO

-- Query delayed trips during weather events
SELECT 
    dt.Year,
    dt.Month,
    we.WeatherType,
    COUNT(fft.TripKey) AS AffectedTrips,
    AVG(fft.DelayMinutes) AS AvgDelay,
    SUM(fft.FuelConsumedLiters) AS TotalFuelWasted
FROM FactFleetTrips fft
INNER JOIN DimTime dt ON fft.TimeKeyStart = dt.TimeKey
INNER JOIN staging.WeatherEvents we ON 
    fft.OriginGeographyKey = we.GeographyKey
    AND ABS(DATEDIFF(MINUTE, dt.FullDateTime, we.EventDateTime)) < 120
WHERE fft.DelayMinutes > 30
GROUP BY dt.Year, dt.Month, we.WeatherType
ORDER BY AVG(fft.DelayMinutes) DESC
```

### Scenario: Simulate Capacity Changes

```sql
-- Stored procedure for temporal elasticity modeling
CREATE PROCEDURE SimulateCapacityChange
    @CapacityIncreasePct DECIMAL(5,2),
    @SimulationStartDate DATE,
    @SimulationEndDate DATE
AS
BEGIN
    -- Create temp table with simulated capacity
    SELECT 
        fwo.OperationKey,
        fwo.ProductKey,
        fwo.Quantity,
        fwo.DwellTimeMinutes,
        -- Simulate reduced dwell time with more capacity
        fwo.DwellTimeMinutes / (1 + (@CapacityIncreasePct / 100.0)) AS SimulatedDwellTime
    INTO #SimulatedWarehouse
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    WHERE dt.FullDateTime BETWEEN @SimulationStartDate AND @SimulationEndDate
    
    -- Compare actual vs simulated metrics
    SELECT 
        'Actual' AS Scenario,
        AVG(DwellTimeMinutes) AS AvgDwell,
        COUNT(*) AS OperationCount
    FROM #SimulatedWarehouse
    UNION ALL
    SELECT 
        'Simulated +' + CAST(@CapacityIncreasePct AS VARCHAR) + '%' AS Scenario,
        AVG(SimulatedDwellTime) AS AvgDwell,
        COUNT(*) AS OperationCount
    FROM #SimulatedWarehouse
    
    DROP TABLE #SimulatedWarehouse
END
GO

-- Run simulation
EXEC SimulateCapacityChange 15, '2026-01-01', '2026-03-31'
```

## Integration Examples

### Export to Azure Synapse Analytics

```sql
-- Create external data source (one-time setup)
CREATE EXTERNAL DATA SOURCE SynapseDataLake
WITH (
    TYPE = HADOOP,
    LOCATION = 'wasbs://container@${AZURE_STORAGE_ACCOUNT}.blob.core.windows.net',
    CREDENTIAL = AzureStorageCredential
)
GO

-- Export aggregated data
CREATE EXTERNAL TABLE ext_FleetSummary
WITH (
    LOCATION = '/logifleet/fleet_summary/',
    DATA_SOURCE = SynapseDataLake,
    FILE_FORMAT = ParquetFormat
