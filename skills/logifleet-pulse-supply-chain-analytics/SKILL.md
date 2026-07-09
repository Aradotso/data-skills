---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing platform for logistics, fleet, and supply chain intelligence with multi-fact star schema
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics data warehouse with power bi
  - configure fleet and warehouse analytics platform
  - implement multi-fact star schema for supply chain
  - create logistics intelligence dashboard
  - build cross-modal supply chain data model
  - set up warehouse gravity zone analytics
  - configure real-time fleet telemetry tracking
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

LogiFleet Pulse is an advanced MS SQL Server and Power BI data warehousing platform for logistics intelligence. It implements a multi-fact star schema that unifies warehouse operations, fleet telemetry, inventory management, and external data sources (weather, traffic, supplier data) into a single semantic layer for cross-modal supply chain analytics.

**Key Capabilities:**
- Multi-fact star schema with time-phased dimensions (15-minute granularity)
- Cross-fact KPI harmonization (warehouse + fleet + inventory)
- Real-time Power BI dashboards with role-based access
- Predictive bottleneck detection and fleet triage
- Warehouse Gravity Zones™ for spatial optimization
- Temporal elasticity modeling for scenario planning

**Primary Language:** SQL (MS SQL Server 2019+)  
**Visualization:** Power BI Desktop/Service  
**Data Integration:** Polybase, External Tables, REST APIs

## Installation

### Prerequisites

```bash
# Required software
- MS SQL Server 2019+ (Developer, Standard, or Enterprise)
- SQL Server Management Studio (SSMS) 18+
- Power BI Desktop (latest version)
- .NET Framework 4.8+ (for data integration)
```

### Database Setup

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance via SSMS
-- Run the schema creation script
:r schema/01_CreateDatabase.sql
:r schema/02_CreateDimensions.sql
:r schema/03_CreateFacts.sql
:r schema/04_CreateViews.sql
:r schema/05_CreateProcedures.sql
:r schema/06_CreateIndexes.sql
```

3. **Configure data sources:**
```json
// Update config_sample.json with your connection strings
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}"
  },
  "wms_api": {
    "endpoint": "${WMS_API_ENDPOINT}",
    "api_key": "${WMS_API_KEY}"
  },
  "telemetry_api": {
    "endpoint": "${FLEET_TELEMETRY_ENDPOINT}",
    "api_key": "${TELEMETRY_API_KEY}"
  }
}
```

4. **Import Power BI template:**
```powershell
# Open the .pbit template file
Start-Process "LogiFleet_Pulse_Master.pbit"
# Enter SQL Server connection details when prompted
```

## Core Data Model

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE dbo.DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2(0) NOT NULL,
    TimeInterval VARCHAR(5),           -- '00:00', '00:15', etc.
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    DayOfMonth TINYINT,
    FiscalWeek INT,
    FiscalMonth INT,
    FiscalQuarter TINYINT,
    FiscalYear SMALLINT,
    IsWeekend BIT,
    IsHoliday BIT
);

-- DimGeography: Hierarchical location dimension
CREATE TABLE dbo.DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationID VARCHAR(20) NOT NULL,
    LocationName VARCHAR(100),
    LocationType VARCHAR(20),          -- 'Warehouse', 'Hub', 'RouteNode'
    ParentLocationID VARCHAR(20),
    Region VARCHAR(50),
    Country VARCHAR(50),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50)
);

-- DimProductGravity: Product dimension with gravity scoring
CREATE TABLE dbo.DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    VelocityScore DECIMAL(5,2),       -- Pick frequency metric
    ValueScore DECIMAL(5,2),          -- Revenue impact metric
    FragilityScore DECIMAL(5,2),      -- Handling risk metric
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    OptimalStorageZone VARCHAR(20),
    RequiresClimateControl BIT
);

-- DimSupplierReliability: Supplier performance tracking
CREATE TABLE dbo.DimSupplierReliability (
    SupplierKey INT PRIMARY KEY,
    SupplierID VARCHAR(20) NOT NULL,
    SupplierName VARCHAR(200),
    LeadTimeMean INT,                 -- Days
    LeadTimeStdDev DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    OnTimeDeliveryRate DECIMAL(5,4),
    ComplianceScore DECIMAL(3,2),
    ReliabilityTier VARCHAR(10)       -- 'A', 'B', 'C', 'D'
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity tracking
CREATE TABLE dbo.FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20),         -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationStartTime DATETIME2(0),
    OperationEndTime DATETIME2(0),
    OperationDurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime) PERSISTED,
    QuantityHandled INT,
    DwellTimeHours DECIMAL(8,2),       -- Time in storage
    StorageZone VARCHAR(20),
    OperatorID VARCHAR(20),
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES dbo.DimTime(TimeKey),
    CONSTRAINT FK_WH_Geography FOREIGN KEY (GeographyKey) REFERENCES dbo.DimGeography(GeographyKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES dbo.DimProductGravity(ProductKey)
);

-- FactFleetTrips: Fleet telemetry and trip data
CREATE TABLE dbo.FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleID VARCHAR(20) NOT NULL,
    DriverID VARCHAR(20),
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripStartTime DATETIME2(0),
    TripEndTime DATETIME2(0),
    TripDurationMinutes AS DATEDIFF(MINUTE, TripStartTime, TripEndTime) PERSISTED,
    DistanceKm DECIMAL(8,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    AverageSpeedKph DECIMAL(5,2),
    MaxSpeedKph DECIMAL(5,2),
    LoadWeightKg DECIMAL(10,2),
    DelayMinutes INT,
    DelayReason VARCHAR(100),
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES dbo.DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES dbo.DimGeography(GeographyKey),
    CONSTRAINT FK_Fleet_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES dbo.DimGeography(GeographyKey)
);

-- FactCrossDock: Cross-docking operations
CREATE TABLE dbo.FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    DockDurationMinutes INT,
    QuantityTransferred INT,
    TransferEfficiency AS CAST(QuantityTransferred AS DECIMAL(10,2)) / NULLIF(DockDurationMinutes, 0) PERSISTED,
    CONSTRAINT FK_CD_Time FOREIGN KEY (TimeKey) REFERENCES dbo.DimTime(TimeKey),
    CONSTRAINT FK_CD_Geography FOREIGN KEY (GeographyKey) REFERENCES dbo.DimGeography(GeographyKey),
    CONSTRAINT FK_CD_Product FOREIGN KEY (ProductKey) REFERENCES dbo.DimProductGravity(ProductKey)
);
```

## Key Stored Procedures

### Incremental Data Loading

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE dbo.usp_LoadWarehouseOperations
    @StartTime DATETIME2(0),
    @EndTime DATETIME2(0)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Staging table for external data
    INSERT INTO dbo.FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OperationStartTime, OperationEndTime, QuantityHandled,
        DwellTimeHours, StorageZone, OperatorID
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        s.OperationType,
        s.OperationStartTime,
        s.OperationEndTime,
        s.QuantityHandled,
        s.DwellTimeHours,
        s.StorageZone,
        s.OperatorID
    FROM dbo.StagingWarehouseOps s
    INNER JOIN dbo.DimTime dt 
        ON CAST(s.OperationStartTime AS DATE) = CAST(dt.DateTime AS DATE)
        AND DATEPART(HOUR, s.OperationStartTime) = dt.HourOfDay
        AND (DATEPART(MINUTE, s.OperationStartTime) / 15) * 15 = CAST(SUBSTRING(dt.TimeInterval, 4, 2) AS INT)
    INNER JOIN dbo.DimGeography dg 
        ON s.WarehouseID = dg.LocationID
    INNER JOIN dbo.DimProductGravity dp 
        ON s.SKU = dp.SKU
    WHERE s.OperationStartTime BETWEEN @StartTime AND @EndTime
        AND s.LoadedFlag = 0;
    
    -- Mark as loaded
    UPDATE dbo.StagingWarehouseOps
    SET LoadedFlag = 1
    WHERE OperationStartTime BETWEEN @StartTime AND @EndTime;
END;
GO

-- Fleet data incremental load
CREATE PROCEDURE dbo.usp_LoadFleetTrips
    @StartTime DATETIME2(0),
    @EndTime DATETIME2(0)
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO dbo.FactFleetTrips (
        TimeKey, VehicleID, DriverID, OriginGeographyKey, DestinationGeographyKey,
        TripStartTime, TripEndTime, DistanceKm, FuelConsumedLiters,
        IdleTimeMinutes, AverageSpeedKph, MaxSpeedKph, LoadWeightKg,
        DelayMinutes, DelayReason
    )
    SELECT 
        dt.TimeKey,
        s.VehicleID,
        s.DriverID,
        origin.GeographyKey AS OriginGeographyKey,
        dest.GeographyKey AS DestinationGeographyKey,
        s.TripStartTime,
        s.TripEndTime,
        s.DistanceKm,
        s.FuelConsumedLiters,
        s.IdleTimeMinutes,
        s.AverageSpeedKph,
        s.MaxSpeedKph,
        s.LoadWeightKg,
        s.DelayMinutes,
        s.DelayReason
    FROM dbo.StagingFleetTrips s
    INNER JOIN dbo.DimTime dt 
        ON CAST(s.TripStartTime AS DATE) = CAST(dt.DateTime AS DATE)
        AND DATEPART(HOUR, s.TripStartTime) = dt.HourOfDay
    INNER JOIN dbo.DimGeography origin 
        ON s.OriginLocationID = origin.LocationID
    INNER JOIN dbo.DimGeography dest 
        ON s.DestinationLocationID = dest.LocationID
    WHERE s.TripStartTime BETWEEN @StartTime AND @EndTime
        AND s.LoadedFlag = 0;
    
    UPDATE dbo.StagingFleetTrips
    SET LoadedFlag = 1
    WHERE TripStartTime BETWEEN @StartTime AND @EndTime;
END;
GO
```

### Cross-Fact Analytics

```sql
-- Cross-fact KPI: Warehouse dwell time impact on fleet efficiency
CREATE PROCEDURE dbo.usp_GetDwellTimeFleetImpact
    @StartDate DATE,
    @EndDate DATE,
    @MinDwellHours INT = 72
AS
BEGIN
    WITH HighDwellProducts AS (
        SELECT 
            ProductKey,
            AVG(DwellTimeHours) AS AvgDwellTime,
            COUNT(*) AS OperationCount
        FROM dbo.FactWarehouseOperations
        WHERE OperationStartTime BETWEEN @StartDate AND @EndDate
            AND DwellTimeHours >= @MinDwellHours
        GROUP BY ProductKey
    ),
    FleetPerformance AS (
        SELECT 
            ft.VehicleID,
            AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
            AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0)) AS AvgFuelEfficiency,
            COUNT(*) AS TripCount
        FROM dbo.FactFleetTrips ft
        WHERE ft.TripStartTime BETWEEN @StartDate AND @EndDate
        GROUP BY ft.VehicleID
    )
    SELECT 
        dp.SKU,
        dp.ProductName,
        dp.GravityScore,
        hdp.AvgDwellTime,
        hdp.OperationCount,
        AVG(fp.AvgIdleTime) AS CorrelatedFleetIdleTime,
        AVG(fp.AvgFuelEfficiency) AS CorrelatedFuelEfficiency
    FROM HighDwellProducts hdp
    INNER JOIN dbo.DimProductGravity dp ON hdp.ProductKey = dp.ProductKey
    CROSS APPLY (
        -- Find fleet trips that might have been affected
        SELECT TOP 10 *
        FROM FleetPerformance
        ORDER BY AvgIdleTime DESC
    ) fp
    ORDER BY hdp.AvgDwellTime DESC;
END;
GO

-- Warehouse gravity zone optimization recommendation
CREATE PROCEDURE dbo.usp_GetGravityZoneRecommendations
AS
BEGIN
    WITH CurrentZonePerformance AS (
        SELECT 
            wo.StorageZone,
            dp.GravityScore,
            AVG(wo.OperationDurationMinutes) AS AvgPickTime,
            COUNT(*) AS PickCount
        FROM dbo.FactWarehouseOperations wo
        INNER JOIN dbo.DimProductGravity dp ON wo.ProductKey = dp.ProductKey
        WHERE wo.OperationType = 'Picking'
            AND wo.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
        GROUP BY wo.StorageZone, dp.GravityScore
    )
    SELECT 
        czp.StorageZone,
        czp.GravityScore,
        czp.AvgPickTime,
        czp.PickCount,
        CASE 
            WHEN czp.GravityScore > 7.5 AND czp.StorageZone NOT LIKE 'HIGH-GRAVITY%' 
                THEN 'Move to HIGH-GRAVITY zone'
            WHEN czp.GravityScore < 3.0 AND czp.StorageZone LIKE 'HIGH-GRAVITY%' 
                THEN 'Move to LOW-GRAVITY zone'
            ELSE 'Current zone optimal'
        END AS Recommendation,
        CASE 
            WHEN czp.GravityScore > 7.5 AND czp.StorageZone NOT LIKE 'HIGH-GRAVITY%' 
                THEN czp.AvgPickTime * 0.25  -- Estimated 25% time savings
            WHEN czp.GravityScore < 3.0 AND czp.StorageZone LIKE 'HIGH-GRAVITY%' 
                THEN czp.AvgPickTime * 0.15  -- Estimated 15% efficiency gain
            ELSE 0
        END AS EstimatedTimeSavingsMinutes
    FROM CurrentZonePerformance czp
    WHERE czp.PickCount > 10  -- Minimum statistical significance
    ORDER BY EstimatedTimeSavingsMinutes DESC;
END;
GO
```

## Power BI Integration

### DAX Measures

```dax
// Total warehouse operations
Total Operations = COUNTROWS('FactWarehouseOperations')

// Average dwell time
Avg Dwell Time (Hours) = 
AVERAGE('FactWarehouseOperations'[DwellTimeHours])

// Fleet efficiency score
Fleet Efficiency Score = 
DIVIDE(
    SUM('FactFleetTrips'[DistanceKm]),
    SUM('FactFleetTrips'[FuelConsumedLiters]) + SUM('FactFleetTrips'[IdleTimeMinutes]) / 60,
    0
)

// Cross-fact KPI: Operations per fleet trip
Operations Per Fleet Trip = 
DIVIDE(
    [Total Operations],
    COUNTROWS('FactFleetTrips'),
    0
)

// Predictive bottleneck index
Bottleneck Index = 
VAR AvgDwell = [Avg Dwell Time (Hours)]
VAR AvgIdle = AVERAGE('FactFleetTrips'[IdleTimeMinutes])
VAR Threshold_Dwell = 48
VAR Threshold_Idle = 30
RETURN
    SWITCH(
        TRUE(),
        AvgDwell > Threshold_Dwell && AvgIdle > Threshold_Idle, "Critical",
        AvgDwell > Threshold_Dwell || AvgIdle > Threshold_Idle, "Warning",
        "Normal"
    )

// Warehouse gravity score average
Avg Gravity Score = 
AVERAGE('DimProductGravity'[GravityScore])

// Time-phased demand elasticity
Demand Elasticity = 
VAR CurrentPeriod = [Total Operations]
VAR PreviousPeriod = 
    CALCULATE(
        [Total Operations],
        DATEADD('DimTime'[DateTime], -1, MONTH)
    )
RETURN
    DIVIDE(
        CurrentPeriod - PreviousPeriod,
        PreviousPeriod,
        0
    )
```

### Power BI Template Configuration

```powerquery
// Power Query M script for data refresh
let
    Source = Sql.Database(
        ServerName, 
        "LogiFleetPulse"
    ),
    
    // Load fact tables
    FactWarehouse = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
    FactFleet = Source{[Schema="dbo",Item="FactFleetTrips"]}[Data],
    FactCrossDock = Source{[Schema="dbo",Item="FactCrossDock"]}[Data],
    
    // Load dimensions
    DimTime = Source{[Schema="dbo",Item="DimTime"]}[Data],
    DimGeography = Source{[Schema="dbo",Item="DimGeography"]}[Data],
    DimProduct = Source{[Schema="dbo",Item="DimProductGravity"]}[Data],
    DimSupplier = Source{[Schema="dbo",Item="DimSupplierReliability"]}[Data],
    
    // Filter to last 90 days for performance
    FilteredWarehouse = Table.SelectRows(
        FactWarehouse, 
        each [OperationStartTime] >= Date.AddDays(DateTime.LocalNow(), -90)
    ),
    
    FilteredFleet = Table.SelectRows(
        FactFleet,
        each [TripStartTime] >= Date.AddDays(DateTime.LocalNow(), -90)
    )
in
    FilteredWarehouse
```

## Automated Alerting

```sql
-- Configure threshold-based alerts
CREATE PROCEDURE dbo.usp_CheckAlertThresholds
AS
BEGIN
    DECLARE @AlertLog TABLE (
        AlertType VARCHAR(50),
        AlertMessage VARCHAR(500),
        Severity VARCHAR(20),
        DetectedAt DATETIME2(0)
    );
    
    -- Alert 1: Fleet idling exceeds threshold
    INSERT INTO @AlertLog
    SELECT 
        'Fleet Idling' AS AlertType,
        'Vehicle ' + VehicleID + ' has ' + CAST(AvgIdlePercent AS VARCHAR(10)) + '% idle time' AS AlertMessage,
        CASE 
            WHEN AvgIdlePercent > 20 THEN 'Critical'
            WHEN AvgIdlePercent > 15 THEN 'Warning'
            ELSE 'Info'
        END AS Severity,
        GETDATE() AS DetectedAt
    FROM (
        SELECT 
            VehicleID,
            AVG(CAST(IdleTimeMinutes AS DECIMAL(10,2)) / NULLIF(TripDurationMinutes, 0) * 100) AS AvgIdlePercent
        FROM dbo.FactFleetTrips
        WHERE TripStartTime >= DATEADD(HOUR, -24, GETDATE())
        GROUP BY VehicleID
    ) sub
    WHERE AvgIdlePercent > 15;
    
    -- Alert 2: High dwell time for high-gravity products
    INSERT INTO @AlertLog
    SELECT 
        'Warehouse Dwell' AS AlertType,
        'SKU ' + dp.SKU + ' (Gravity: ' + CAST(dp.GravityScore AS VARCHAR(10)) + ') has ' + 
        CAST(AVG(wo.DwellTimeHours) AS VARCHAR(10)) + ' hours dwell time' AS AlertMessage,
        CASE 
            WHEN AVG(wo.DwellTimeHours) > 72 THEN 'Critical'
            WHEN AVG(wo.DwellTimeHours) > 48 THEN 'Warning'
            ELSE 'Info'
        END AS Severity,
        GETDATE() AS DetectedAt
    FROM dbo.FactWarehouseOperations wo
    INNER JOIN dbo.DimProductGravity dp ON wo.ProductKey = dp.ProductKey
    WHERE wo.OperationStartTime >= DATEADD(HOUR, -24, GETDATE())
        AND dp.GravityScore > 7.0
    GROUP BY dp.SKU, dp.GravityScore
    HAVING AVG(wo.DwellTimeHours) > 48;
    
    -- Alert 3: Cross-dock efficiency drop
    INSERT INTO @AlertLog
    SELECT 
        'Cross-Dock Efficiency' AS AlertType,
        'Location ' + dg.LocationName + ' has efficiency drop to ' + 
        CAST(AVG(cd.TransferEfficiency) AS VARCHAR(10)) AS AlertMessage,
        CASE 
            WHEN AVG(cd.TransferEfficiency) < 5 THEN 'Critical'
            WHEN AVG(cd.TransferEfficiency) < 8 THEN 'Warning'
            ELSE 'Info'
        END AS Severity,
        GETDATE() AS DetectedAt
    FROM dbo.FactCrossDock cd
    INNER JOIN dbo.DimGeography dg ON cd.GeographyKey = dg.GeographyKey
    WHERE cd.TimeKey >= (SELECT TimeKey FROM dbo.DimTime WHERE DateTime >= DATEADD(HOUR, -24, GETDATE()))
    GROUP BY dg.LocationName
    HAVING AVG(cd.TransferEfficiency) < 8;
    
    -- Output all alerts
    SELECT * FROM @AlertLog
    WHERE Severity IN ('Critical', 'Warning')
    ORDER BY 
        CASE Severity 
            WHEN 'Critical' THEN 1 
            WHEN 'Warning' THEN 2 
            ELSE 3 
        END,
        DetectedAt DESC;
END;
GO

-- Schedule execution via SQL Agent
-- Job: LogiFleetPulse_AlertCheck
-- Schedule: Every 15 minutes
```

## Common Patterns

### Pattern 1: Multi-Fact Cross-Analysis

```sql
-- Correlate warehouse operations with fleet performance
SELECT 
    dt.FiscalWeek,
    dg.Region,
    -- Warehouse metrics
    COUNT(DISTINCT wo.OperationKey) AS TotalWarehouseOps,
    AVG(wo.OperationDurationMinutes) AS AvgOpDuration,
    AVG(wo.DwellTimeHours) AS AvgDwellTime,
    -- Fleet metrics
    COUNT(DISTINCT ft.TripKey) AS TotalFleetTrips,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0)) AS AvgFuelPerKm,
    -- Cross-fact KPI
    CAST(COUNT(DISTINCT wo.OperationKey) AS DECIMAL(10,2)) / NULLIF(COUNT(DISTINCT ft.TripKey), 0) AS OpsPerTrip
FROM dbo.DimTime dt
LEFT JOIN dbo.FactWarehouseOperations wo ON dt.TimeKey = wo.TimeKey
LEFT JOIN dbo.FactFleetTrips ft ON dt.TimeKey = ft.TimeKey
LEFT JOIN dbo.DimGeography dg ON wo.GeographyKey = dg.GeographyKey
WHERE dt.FiscalYear = 2026
    AND dt.FiscalQuarter = 2
GROUP BY dt.FiscalWeek, dg.Region
ORDER BY dt.FiscalWeek, dg.Region;
```

### Pattern 2: Temporal Elasticity Scenario

```sql
-- Simulate warehouse capacity increase impact
DECLARE @CapacityIncrease DECIMAL(3,2) = 0.15;  -- 15% increase

WITH BaselineMetrics AS (
    SELECT 
        AVG(OperationDurationMinutes) AS AvgDuration,
        AVG(DwellTimeHours) AS AvgDwell,
        COUNT(*) AS TotalOps
    FROM dbo.FactWarehouseOperations
    WHERE OperationStartTime >= DATEADD(MONTH, -3, GETDATE())
),
SimulatedImpact AS (
    SELECT 
        AvgDuration * (1 - (@CapacityIncrease * 0.3)) AS ProjectedDuration,  -- 30% of capacity gain affects duration
        AvgDwell * (1 - (@CapacityIncrease * 0.5)) AS ProjectedDwell,        -- 50% of capacity gain affects dwell
        TotalOps * (1 + @CapacityIncrease) AS ProjectedOps
    FROM BaselineMetrics
)
SELECT 
    b.AvgDuration AS Current_AvgDuration,
    s.ProjectedDuration,
    b.AvgDuration - s.ProjectedDuration AS Duration_Improvement,
    b.AvgDwell AS Current_AvgDwell,
    s.ProjectedDwell,
    b.AvgDwell - s.ProjectedDwell AS Dwell_Improvement,
    b.TotalOps AS Current_TotalOps,
    s.ProjectedOps,
    s.ProjectedOps - b.TotalOps AS Ops_Increase
FROM BaselineMetrics b
CROSS JOIN SimulatedImpact s;
```

### Pattern 3: Gravity Zone Rebalancing

```sql
-- Identify products for zone rebalancing
WITH ProductZoneMetrics AS (
    SELECT 
        wo.ProductKey,
        wo.StorageZone,
        COUNT(*) AS PickCount,
        AVG(wo.OperationDurationMinutes) AS AvgPickTime,
        dp.GravityScore,
        dp.OptimalStorageZone
    FROM dbo.FactWarehouseOperations wo
    INNER JOIN dbo.DimProductGravity dp ON wo.ProductKey = dp.ProductKey
    WHERE wo.OperationType = 'Picking'
        AND wo.OperationStartTime >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY wo.ProductKey, wo.StorageZone, dp.GravityScore, dp.OptimalStorageZone
),
ZoneMismatch AS (
    SELECT 
        *,
        CASE 
            WHEN StorageZone <> OptimalStorageZone THEN 1 
            ELSE 0 
        END AS IsMisaligned,
        AvgPickTime * PickCount AS TotalPickTimeMinutes
    FROM ProductZoneMetrics
)
SELECT 
    dp.SKU,
    dp.ProductName,
    zm.StorageZone AS CurrentZone,
    zm.OptimalStorageZone,
    zm.GravityScore,
    zm.PickCount,
    zm.AvgPickTime,
    zm.TotalPickTimeMinutes,
    CASE 
        WHEN zm.IsMisaligned = 1 THEN zm.TotalPickTimeMinutes * 0.20  -- Estimated 20% time savings
        ELSE 0
    END AS EstimatedTimeSavings
FROM ZoneMismatch zm
INNER JOIN dbo.DimProductGravity dp ON zm.ProductKey = dp.ProductKey
WHERE zm.IsMisaligned = 1
    AND zm.PickCount > 20  -- Minimum volume threshold
ORDER BY EstimatedTimeSavings DESC;
```

## Configuration

### Environment Variables

```bash
# SQL Server connection
export SQL_SERVER_HOST="your-sql
