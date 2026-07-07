---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform for warehouse, fleet, and supply chain KPI harmonization
triggers:
  - set up LogiFleet Pulse supply chain analytics
  - deploy warehouse and fleet tracking dashboard
  - configure Power BI logistics intelligence
  - create multi-fact star schema for supply chain
  - build real-time logistics KPI dashboard
  - implement warehouse gravity zone analytics
  - set up cross-modal supply chain reporting
  - configure fleet telemetry and warehouse integration
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is a multi-modal logistics intelligence platform that combines warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer. Built on MS SQL Server with Power BI visualization, it provides real-time cross-fact KPI analysis through a custom star schema architecture.

## What It Does

- **Unified Data Model**: Multi-fact star schema linking warehouse operations, fleet trips, cross-dock transfers, and supplier reliability
- **Cross-Fact KPIs**: Harmonizes metrics across warehouse and fleet (e.g., dwell time vs. fleet idling costs)
- **Warehouse Gravity Zones**: Spatial optimization based on pick frequency, item value, and fragility
- **Predictive Analytics**: Temporal elasticity modeling and bottleneck detection
- **Real-Time Dashboards**: Power BI dashboards with 15-minute refresh cycles
- **Role-Based Access**: Row-level security for different user roles

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (recommended for temporal tables and polybase support)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, TMS, telemetry APIs, ERP systems

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance and create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create time dimension (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    FullDateTime DATETIME2(0) NOT NULL,
    TimeInterval15Min SMALLINT NOT NULL, -- 0-95 (24 hours * 4)
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(20) NOT NULL,
    IsWeekend BIT NOT NULL,
    FiscalPeriod VARCHAR(10),
    FiscalYear SMALLINT,
    CONSTRAINT UQ_DimTime_DateTime UNIQUE (FullDateTime)
);

-- Create geography dimension (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationCode VARCHAR(50) NOT NULL,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- Warehouse, Route Node, Distribution Center
    Address VARCHAR(500),
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    IsActive BIT DEFAULT 1,
    CONSTRAINT UQ_DimGeography_Code UNIQUE (LocationCode)
);

-- Create product dimension with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL,
    ProductName VARCHAR(500) NOT NULL,
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    PickFrequencyScore DECIMAL(5,2), -- 0-100 based on velocity
    ValueScore DECIMAL(5,2), -- 0-100 based on unit value
    FragilityScore DECIMAL(5,2), -- 0-100
    GravityScore AS (PickFrequencyScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    OptimalZone VARCHAR(50), -- Calculated zone assignment
    LastRecalculatedDate DATETIME2(0),
    CONSTRAINT UQ_DimProduct_SKU UNIQUE (SKU)
);

-- Create supplier reliability dimension
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode VARCHAR(50) NOT NULL,
    SupplierName VARCHAR(200) NOT NULL,
    LeadTimeMean DECIMAL(8,2), -- Average days
    LeadTimeVariance DECIMAL(8,2),
    DefectPercentage DECIMAL(5,2),
    OnTimeDeliveryRate DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityTier VARCHAR(20), -- Platinum, Gold, Silver, Bronze, Critical
    CONSTRAINT UQ_DimSupplier_Code UNIQUE (SupplierCode)
);

-- Create warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OperationStartTime DATETIME2(0),
    OperationEndTime DATETIME2(0),
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime) PERSISTED,
    DwellTimeHours DECIMAL(10,2), -- Time in storage before next operation
    QuantityHandled INT,
    StorageZone VARCHAR(50),
    OperatorID VARCHAR(50),
    BatchNumber VARCHAR(100),
    QualityIssueFlag BIT DEFAULT 0
);

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey) INCLUDE (OperationType, DurationMinutes);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey) INCLUDE (DwellTimeHours, QuantityHandled);

-- Create fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    TripStartTime DATETIME2(0),
    TripEndTime DATETIME2(0),
    TripDurationMinutes AS DATEDIFF(MINUTE, TripStartTime, TripEndTime) PERSISTED,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    LoadWeightKG DECIMAL(10,2),
    WeatherDelayFlag BIT DEFAULT 0,
    TrafficDelayFlag BIT DEFAULT 0,
    MaintenanceAlertFlag BIT DEFAULT 0
);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey) INCLUDE (TripDurationMinutes, FuelConsumedLiters);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID) INCLUDE (IdleTimeMinutes, MaintenanceAlertFlag);

-- Create cross-dock fact table
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    ArrivalTime DATETIME2(0),
    DepartureTime DATETIME2(0),
    CrossDockDurationMinutes AS DATEDIFF(MINUTE, ArrivalTime, DepartureTime) PERSISTED,
    QuantityTransferred INT,
    DirectTransferFlag BIT -- 1 if no storage, 0 if temporary hold
);
```

### Step 2: Create Stored Procedures

```sql
-- Stored procedure for incremental data loading
CREATE PROCEDURE usp_LoadWarehouseOperations
    @StartDate DATETIME2,
    @EndDate DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from external WMS table
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, OperationStartTime, OperationEndTime,
        DwellTimeHours, QuantityHandled, StorageZone, OperatorID
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        wms.OperationType,
        wms.StartDateTime,
        wms.EndDateTime,
        wms.DwellHours,
        wms.Quantity,
        wms.Zone,
        wms.OperatorCode
    FROM ExternalWMS.dbo.Operations wms
    INNER JOIN DimTime t ON CAST(wms.StartDateTime AS DATETIME2(0)) = t.FullDateTime
    INNER JOIN DimGeography g ON wms.WarehouseCode = g.LocationCode
    INNER JOIN DimProductGravity p ON wms.SKU = p.SKU
    LEFT JOIN DimSupplierReliability s ON wms.SupplierCode = s.SupplierCode
    WHERE wms.StartDateTime >= @StartDate 
      AND wms.StartDateTime < @EndDate
      AND NOT EXISTS (
          SELECT 1 FROM FactWarehouseOperations f
          WHERE f.OperationStartTime = wms.StartDateTime
            AND f.ProductKey = p.ProductKey
      );
END;
GO

-- Stored procedure for gravity score recalculation
CREATE PROCEDURE usp_RecalculateProductGravity
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update pick frequency scores based on last 90 days
    WITH PickFrequency AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickCount,
            NTILE(100) OVER (ORDER BY COUNT(*)) AS FrequencyPercentile
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
          AND OperationStartTime >= DATEADD(DAY, -90, GETDATE())
        GROUP BY ProductKey
    )
    UPDATE p
    SET 
        PickFrequencyScore = pf.FrequencyPercentile,
        LastRecalculatedDate = GETDATE(),
        OptimalZone = CASE 
            WHEN (pf.FrequencyPercentile * 0.5 + p.ValueScore * 0.3 + p.FragilityScore * 0.2) >= 80 THEN 'A-High'
            WHEN (pf.FrequencyPercentile * 0.5 + p.ValueScore * 0.3 + p.FragilityScore * 0.2) >= 60 THEN 'B-Medium'
            ELSE 'C-Low'
        END
    FROM DimProductGravity p
    INNER JOIN PickFrequency pf ON p.ProductKey = pf.ProductKey;
END;
GO

-- Stored procedure for automated alerting
CREATE PROCEDURE usp_CheckFleetIdleThreshold
    @IdleThresholdPercent DECIMAL(5,2) = 15.0
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Find trips exceeding idle threshold
    SELECT 
        TripKey,
        VehicleID,
        DriverID,
        TripStartTime,
        TripDurationMinutes,
        IdleTimeMinutes,
        (IdleTimeMinutes * 100.0 / NULLIF(TripDurationMinutes, 0)) AS IdlePercentage,
        og.LocationName AS Origin,
        dg.LocationName AS Destination
    FROM FactFleetTrips ft
    INNER JOIN DimGeography og ON ft.OriginGeographyKey = og.GeographyKey
    INNER JOIN DimGeography dg ON ft.DestinationGeographyKey = dg.GeographyKey
    WHERE (IdleTimeMinutes * 100.0 / NULLIF(TripDurationMinutes, 0)) > @IdleThresholdPercent
      AND TripStartTime >= DATEADD(HOUR, -24, GETDATE())
    ORDER BY IdlePercentage DESC;
END;
GO
```

### Step 3: Configure Power BI Connection

```powerquery
// Power BI M Query for dynamic connection
let
    Source = Sql.Database(
        // Use environment variable or parameter
        #"SQL_SERVER_NAME",
        #"LOGIFLEET_DATABASE"
    ),
    
    // Load fact tables
    FactWarehouse = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
    FactFleet = Source{[Schema="dbo",Item="FactFleetTrips"]}[Data],
    FactCrossDock = Source{[Schema="dbo",Item="FactCrossDock"]}[Data],
    
    // Load dimension tables
    DimTime = Source{[Schema="dbo",Item="DimTime"]}[Data],
    DimGeography = Source{[Schema="dbo",Item="DimGeography"]}[Data],
    DimProduct = Source{[Schema="dbo",Item="DimProductGravity"]}[Data],
    DimSupplier = Source{[Schema="dbo",Item="DimSupplierReliability"]}[Data]
in
    Source
```

## Key Queries & Analysis Patterns

### Cross-Fact KPI: Dwell Time vs Fleet Idle Costs

```sql
-- Correlate warehouse dwell time with fleet idle costs
WITH WarehouseDwell AS (
    SELECT 
        p.SKU,
        p.ProductName,
        p.GravityScore,
        AVG(wo.DwellTimeHours) AS AvgDwellHours,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.SKU, p.ProductName, p.GravityScore
),
FleetIdle AS (
    SELECT 
        ft.VehicleID,
        AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
        SUM(ft.FuelConsumedLiters) AS TotalFuel,
        COUNT(*) AS TripCount
    FROM FactFleetTrips ft
    WHERE ft.TripStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY ft.VehicleID
)
SELECT 
    wd.SKU,
    wd.ProductName,
    wd.AvgDwellHours,
    wd.GravityScore,
    fi.AvgIdleMinutes,
    fi.TotalFuel,
    -- Derived metric: efficiency ratio
    (wd.AvgDwellHours * fi.AvgIdleMinutes) AS CompoundInefficiencyScore
FROM WarehouseDwell wd
CROSS JOIN FleetIdle fi
WHERE wd.GravityScore > 70 -- Focus on high-value items
ORDER BY CompoundInefficiencyScore DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products misplaced relative to optimal zone
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZone AS RecommendedZone,
    wo.StorageZone AS CurrentZone,
    COUNT(*) AS MisplacedOperations,
    AVG(wo.DurationMinutes) AS AvgOperationTime
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE wo.StorageZone <> p.OptimalZone
  AND wo.OperationStartTime >= DATEADD(DAY, -7, GETDATE())
  AND wo.OperationType IN ('Picking', 'Putaway')
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.OptimalZone, wo.StorageZone
HAVING COUNT(*) > 10 -- Threshold for actionable volume
ORDER BY p.GravityScore DESC, COUNT(*) DESC;
```

### Predictive Bottleneck Detection

```sql
-- Detect time periods with operation congestion
WITH HourlyOperations AS (
    SELECT 
        t.HourOfDay,
        t.DayOfWeek,
        wo.OperationType,
        COUNT(*) AS OperationVolume,
        AVG(wo.DurationMinutes) AS AvgDuration,
        STDEV(wo.DurationMinutes) AS StdDevDuration
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE wo.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY t.HourOfDay, t.DayOfWeek, wo.OperationType
)
SELECT 
    HourOfDay,
    DayOfWeek,
    OperationType,
    OperationVolume,
    AvgDuration,
    StdDevDuration,
    -- Bottleneck index: high volume + high duration + high variance
    (OperationVolume * AvgDuration * ISNULL(StdDevDuration, 1)) AS BottleneckIndex
FROM HourlyOperations
WHERE OperationVolume > (SELECT AVG(OperationVolume) FROM HourlyOperations)
ORDER BY BottleneckIndex DESC;
```

### Supplier Performance Impact Analysis

```sql
-- Link supplier reliability to warehouse delays
SELECT 
    s.SupplierName,
    s.LeadTimeMean,
    s.LeadTimeVariance,
    s.OnTimeDeliveryRate,
    s.ReliabilityTier,
    COUNT(wo.OperationKey) AS ReceivingOperations,
    AVG(wo.DurationMinutes) AS AvgReceivingTime,
    SUM(CASE WHEN wo.QualityIssueFlag = 1 THEN 1 ELSE 0 END) AS QualityIssues
FROM FactWarehouseOperations wo
INNER JOIN DimSupplierReliability s ON wo.SupplierKey = s.SupplierKey
WHERE wo.OperationType = 'Receiving'
  AND wo.OperationStartTime >= DATEADD(DAY, -90, GETDATE())
GROUP BY s.SupplierName, s.LeadTimeMean, s.LeadTimeVariance, 
         s.OnTimeDeliveryRate, s.ReliabilityTier
HAVING COUNT(wo.OperationKey) > 5
ORDER BY s.OnTimeDeliveryRate ASC, QualityIssues DESC;
```

## Power BI DAX Measures

### Cross-Modal Efficiency Score

```dax
// Composite measure across warehouse and fleet
CrossModalEfficiency = 
VAR WarehouseUtil = 
    DIVIDE(
        COUNTROWS(FactWarehouseOperations),
        CALCULATE(
            COUNTROWS(FactWarehouseOperations),
            ALL(DimTime[HourOfDay])
        )
    )
VAR FleetUtil = 
    DIVIDE(
        SUM(FactFleetTrips[TripDurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
        SUM(FactFleetTrips[TripDurationMinutes])
    )
RETURN
    (WarehouseUtil * 0.6) + (FleetUtil * 0.4)
```

### Dynamic Gravity Zone Adherence %

```dax
// Percentage of operations in optimal zone
GravityZoneAdherence = 
DIVIDE(
    COUNTROWS(
        FILTER(
            FactWarehouseOperations,
            RELATED(DimProductGravity[OptimalZone]) = FactWarehouseOperations[StorageZone]
        )
    ),
    COUNTROWS(FactWarehouseOperations)
) * 100
```

### Temporal Elasticity Simulation

```dax
// What-if parameter for warehouse capacity
SimulatedThroughput = 
VAR CurrentCapacity = 0.80  // 80% utilization
VAR TargetCapacity = [CapacitySlider]  // User-defined slider
VAR CapacityRatio = TargetCapacity / CurrentCapacity
RETURN
    SUM(FactWarehouseOperations[QuantityHandled]) * CapacityRatio
```

## Configuration & Data Source Setup

### External Data Source Configuration

```sql
-- Create external data source for WMS integration
CREATE EXTERNAL DATA SOURCE WMS_ExternalSource
WITH (
    TYPE = RDBMS,
    LOCATION = 'wms-server.company.com',
    DATABASE_NAME = 'WMS_Production',
    CREDENTIAL = WMS_Credential  -- Pre-created database scoped credential
);

-- Create external table for incremental load
CREATE EXTERNAL TABLE ExternalWMS.Operations (
    OperationType VARCHAR(50),
    StartDateTime DATETIME2,
    EndDateTime DATETIME2,
    SKU VARCHAR(100),
    Quantity INT,
    WarehouseCode VARCHAR(50),
    SupplierCode VARCHAR(50),
    Zone VARCHAR(50),
    OperatorCode VARCHAR(50),
    DwellHours DECIMAL(10,2)
)
WITH (
    DATA_SOURCE = WMS_ExternalSource,
    SCHEMA_NAME = 'dbo',
    OBJECT_NAME = 'vw_Operations_Incremental'
);
```

### Environment Variables Setup

```bash
# .env file structure (not committed to repo)
SQL_SERVER_NAME=your-sql-server.database.windows.net
LOGIFLEET_DATABASE=LogiFleetPulse
SQL_USERNAME=admin_user
SQL_PASSWORD=${SQL_DB_PASSWORD}  # From secure vault

# WMS API configuration
WMS_API_ENDPOINT=https://wms-api.company.com/v2
WMS_API_KEY=${WMS_SECRET_KEY}

# Telemetry API
FLEET_TELEMETRY_URL=https://fleet-gps.provider.com/api
FLEET_API_TOKEN=${FLEET_AUTH_TOKEN}

# Alert configuration
ALERT_EMAIL_SMTP=smtp.company.com
ALERT_EMAIL_FROM=logifleet-alerts@company.com
```

## Automated Maintenance Jobs

```sql
-- SQL Agent job for daily gravity recalculation
USE msdb;
GO

EXEC dbo.sp_add_job
    @job_name = N'LogiFleet_Daily_Gravity_Recalc',
    @enabled = 1,
    @description = N'Recalculate product gravity scores daily at 2 AM';

EXEC dbo.sp_add_jobstep
    @job_name = N'LogiFleet_Daily_Gravity_Recalc',
    @step_name = N'Execute Recalculation',
    @subsystem = N'TSQL',
    @database_name = N'LogiFleetPulse',
    @command = N'EXEC usp_RecalculateProductGravity;';

EXEC dbo.sp_add_schedule
    @schedule_name = N'Daily_2AM',
    @freq_type = 4,  -- Daily
    @freq_interval = 1,
    @active_start_time = 020000;  -- 2:00 AM

EXEC dbo.sp_attach_schedule
    @job_name = N'LogiFleet_Daily_Gravity_Recalc',
    @schedule_name = N'Daily_2AM';

EXEC dbo.sp_add_jobserver
    @job_name = N'LogiFleet_Daily_Gravity_Recalc',
    @server_name = N'(local)';
GO
```

## Troubleshooting

### Issue: Power BI Dashboard Not Refreshing

**Symptoms**: Stale data, refresh errors in Power BI service

**Solutions**:
1. Check SQL Server firewall rules for Power BI gateway IP ranges
2. Verify connection credentials haven't expired
3. Check incremental load stored procedures for date range logic

```sql
-- Diagnostic query: Check last loaded data timestamp
SELECT 
    'Warehouse' AS FactTable,
    MAX(OperationStartTime) AS LastLoadedTime,
    COUNT(*) AS RecordCount
FROM FactWarehouseOperations
UNION ALL
SELECT 
    'Fleet',
    MAX(TripStartTime),
    COUNT(*)
FROM FactFleetTrips;
```

### Issue: Slow Query Performance on Cross-Fact KPIs

**Symptoms**: Dashboard timeout, high CPU on SQL Server

**Solutions**:
1. Ensure columnstore indexes on fact tables for large datasets

```sql
-- Add columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouse_Columnstore
ON FactWarehouseOperations (TimeKey, ProductKey, GeographyKey, DurationMinutes, DwellTimeHours);

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleet_Columnstore
ON FactFleetTrips (TimeKey, VehicleID, TripDurationMinutes, IdleTimeMinutes, FuelConsumedLiters);
```

2. Use query hints for complex aggregations

```sql
-- Force parallel execution for heavy aggregations
SELECT /*+ OPTION(MAXDOP 4) */
    ...
FROM FactWarehouseOperations wo
INNER JOIN FactFleetTrips ft ...
```

### Issue: Gravity Zone Recommendations Not Updating

**Symptoms**: OptimalZone column shows NULL or outdated values

**Solutions**:
1. Verify pick frequency data exists for last 90 days
2. Check if recalculation job is running

```sql
-- Manual recalculation with diagnostics
EXEC usp_RecalculateProductGravity;

-- Check results
SELECT 
    SKU,
    PickFrequencyScore,
    ValueScore,
    FragilityScore,
    GravityScore,
    OptimalZone,
    LastRecalculatedDate
FROM DimProductGravity
WHERE LastRecalculatedDate IS NOT NULL
ORDER BY LastRecalculatedDate DESC;
```

### Issue: External Data Source Connection Failures

**Symptoms**: Error messages about WMS_ExternalSource not accessible

**Solutions**:
1. Test network connectivity from SQL Server to WMS

```sql
-- Test external table access
SELECT TOP 10 * FROM ExternalWMS.Operations;

-- Check credential status
SELECT * FROM sys.database_scoped_credentials;
```

2. Refresh credentials if expired

```sql
-- Drop and recreate credential with updated password
DROP DATABASE SCOPED CREDENTIAL WMS_Credential;

CREATE DATABASE SCOPED CREDENTIAL WMS_Credential
WITH IDENTITY = 'wms_integration_user',
SECRET = '${WMS_DB_PASSWORD}';  -- From environment variable
```

## Common Patterns

### Pattern 1: Time-Phased Dimension Joins

Always join fact tables to DimTime using both TimeKey and calculated columns for flexibility:

```sql
SELECT 
    t.DayOfWeek,
    t.HourOfDay,
    t.IsWeekend,
    COUNT(*) AS OperationCount
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.FiscalYear = 2026
  AND t.FiscalPeriod = 'Q2'
GROUP BY t.DayOfWeek, t.HourOfDay, t.IsWeekend;
```

### Pattern 2: Role-Playing Dimensions

FactFleetTrips uses DimGeography twice (origin/destination):

```sql
SELECT 
    og.LocationName AS Origin,
    dg.LocationName AS Destination,
    COUNT(*) AS TripCount,
    AVG(ft.TripDurationMinutes) AS AvgDuration
FROM FactFleetTrips ft
INNER JOIN DimGeography og ON ft.OriginGeographyKey = og.GeographyKey
INNER JOIN DimGeography dg ON ft.DestinationGeographyKey = dg.GeographyKey
GROUP BY og.LocationName, dg.LocationName;
```

### Pattern 3: Bridge Table Many-to-Many

For complex routes with multiple stops, use a bridge table:

```sql
CREATE TABLE BridgeRouteSKU (
    TripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    QuantityLoaded INT,
    SequenceOrder TINYINT
);

-- Query all SKUs per route
SELECT 
    ft.TripKey,
    ft.VehicleID,
    p.SKU,
    p.ProductName,
    br.QuantityLoaded,
    br.SequenceOrder
FROM FactFleetTrips ft
INNER JOIN BridgeRouteSKU br ON ft.TripKey = br.TripKey
INNER JOIN DimProductGravity p ON br.ProductKey = p.ProductKey
ORDER BY ft.TripKey, br.SequenceOrder;
```

## Advanced Use Cases

### Scenario-Based Simulation

```sql
-- What-if analysis: Impact of 20% faster picking on throughput
DECLARE @PickingSpeedIncrease DECIMAL(5,2) = 0.20;

WITH CurrentPerformance AS (
    SELECT 
        AVG(DurationMinutes) AS BaselinePickTime,
        COUNT(*) AS DailyPickVolume
    FROM FactWarehouseOperations
    WHERE OperationType = 'Picking'
      AND OperationStartTime >= DATEADD(DAY, -30, GETDATE())
),
SimulatedPerformance AS (
    SELECT 
        BaselinePickTime * (1 - @PickingSpeedIncrease) AS ProjectedPickTime,
        DailyPickVolume / 30.0 AS DailyAvg
    FROM CurrentPerformance
)
SELECT 
    BaselinePickTime,
    ProjectedPickTime,
    DailyAvg,
    (DailyAvg * 60 * 24) / BaselinePickTime AS CurrentDailyCapacity,
    (Dai
