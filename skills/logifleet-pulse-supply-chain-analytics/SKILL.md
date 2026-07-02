---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for fleet and warehouse intelligence
triggers:
  - "set up logistics analytics warehouse"
  - "configure supply chain power bi dashboard"
  - "implement fleet and warehouse data model"
  - "create multi-fact star schema for logistics"
  - "deploy logifleet pulse analytics"
  - "build warehouse gravity zone optimization"
  - "integrate fleet telemetry with warehouse operations"
  - "configure cross-modal supply chain dashboards"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

LogiFleet Pulse is an advanced MS SQL Server data warehouse and Power BI visualization platform for logistics intelligence. It implements a multi-fact star schema that harmonizes warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer for cross-modal supply chain analytics.

## What It Does

- **Multi-Fact Data Warehouse**: Combines warehouse operations, fleet trips, cross-dock events, and supplier data in a time-phased star schema
- **Warehouse Gravity Zones**: Spatial optimization based on pick frequency, item value, and replenishment lead time
- **Fleet Triage Engine**: Proactive maintenance prioritization weighted by revenue impact
- **Cross-Fact KPI Harmonization**: Links inventory turnover with fleet metrics through shared dimensions
- **Temporal Elasticity Modeling**: Time-phased simulation scenarios for capacity planning
- **Power BI Dashboards**: Real-time visualization with role-based access and natural language queries

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- Network access to your WMS, TMS, or telemetry data sources

### Deploy Database Schema

1. Clone the repository:

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. Execute the schema creation script in SSMS:

```sql
-- Run as database administrator
-- Creates LogiFleetPulse database, tables, relationships, and stored procedures
:r schema\01_CreateDatabase.sql
:r schema\02_CreateDimensions.sql
:r schema\03_CreateFacts.sql
:r schema\04_CreateViews.sql
:r schema\05_CreateStoredProcedures.sql
```

3. Configure data source connections in the configuration file:

```json
{
  "dataSources": {
    "wms": {
      "connectionString": "Server=${WMS_SERVER};Database=${WMS_DB};Integrated Security=true;",
      "refreshInterval": "15min"
    },
    "fleetTelemetry": {
      "apiEndpoint": "${FLEET_API_ENDPOINT}",
      "apiKey": "${FLEET_API_KEY}",
      "refreshInterval": "5min"
    },
    "weatherAPI": {
      "endpoint": "${WEATHER_API_ENDPOINT}",
      "apiKey": "${WEATHER_API_KEY}"
    }
  }
}
```

### Power BI Template Setup

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter your SQL Server connection details when prompted
3. Configure data refresh schedule (recommended: every 15 minutes)
4. Publish to Power BI Service for team access

## Core Database Schema

### Dimension Tables

**DimTime** - Time hierarchy with 15-minute granularity:

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME NOT NULL,
    FifteenMinuteBucket TINYINT,
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    DayName VARCHAR(20),
    IsWeekend BIT,
    FiscalPeriod CHAR(7),
    FiscalYear SMALLINT,
    INDEX IX_DimTime_DateTime (DateTime)
);
```

**DimProductGravity** - Product classification with velocity scoring:

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value / fragility
    VelocityClass VARCHAR(20), -- Fast/Medium/Slow mover
    ValueTier VARCHAR(20), -- High/Medium/Low value
    FragilityIndex DECIMAL(3,2),
    OptimalZoneDistance INT, -- Meters from shipping dock
    LastRecalculated DATETIME,
    INDEX IX_ProductGravity_Score (GravityScore DESC)
);
```

**DimGeography** - Hierarchical location dimension:

```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationCode VARCHAR(20) UNIQUE,
    LocationName NVARCHAR(100),
    LocationType VARCHAR(30), -- Warehouse, Distribution Center, Route Node
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6),
    RegionKey INT,
    RegionName NVARCHAR(100),
    CountryKey INT,
    CountryName NVARCHAR(100),
    INDEX IX_Geography_Region (RegionKey),
    INDEX IX_Geography_Type (LocationType)
);
```

### Fact Tables

**FactWarehouseOperations** - Granular warehouse events:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OperationType VARCHAR(20), -- Receiving, Putaway, Picking, Packing, Shipping
    Quantity INT,
    DurationMinutes DECIMAL(8,2),
    DwellHours DECIMAL(10,2), -- Time since previous operation
    StorageZone VARCHAR(20),
    StorageZoneDistance INT, -- Meters from dock
    OperatorID INT,
    EquipmentID VARCHAR(20),
    BatchNumber VARCHAR(50),
    INDEX IX_FactWH_Time (TimeKey),
    INDEX IX_FactWH_Product (ProductKey),
    INDEX IX_FactWH_Type (OperationType),
    INDEX IX_FactWH_DwellTime (DwellHours DESC)
);
```

**FactFleetTrips** - Fleet telemetry and route data:

```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DriverKey INT FOREIGN KEY REFERENCES DimDriver(DriverKey),
    TripStartTime DATETIME,
    TripEndTime DATETIME,
    DistanceKM DECIMAL(10,2),
    FuelLiters DECIMAL(8,2),
    IdleMinutes DECIMAL(8,2),
    LoadingMinutes DECIMAL(8,2),
    UnloadingMinutes DECIMAL(8,2),
    LoadWeightKG DECIMAL(10,2),
    RevenueValue DECIMAL(12,2),
    DelayMinutes INT,
    DelayReason VARCHAR(100),
    WeatherCondition VARCHAR(50),
    INDEX IX_FactFleet_Time (TimeKey),
    INDEX IX_FactFleet_Vehicle (VehicleKey),
    INDEX IX_FactFleet_Route (OriginGeographyKey, DestinationGeographyKey),
    INDEX IX_FactFleet_IdleTime (IdleMinutes DESC)
);
```

**FactCrossDock** - Transfer operations without long-term storage:

```sql
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    Quantity INT,
    DockToDockMinutes DECIMAL(8,2),
    QualityCheckPassed BIT,
    INDEX IX_FactCrossDock_Time (TimeKey),
    INDEX IX_FactCrossDock_Duration (DockToDockMinutes)
);
```

## Key Stored Procedures

### Data Loading and ETL

**Load Warehouse Operations Incrementally**:

```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last 24 hours if not specified
    IF @LastLoadTime IS NULL
        SET @LastLoadTime = DATEADD(HOUR, -24, GETDATE());
    
    -- Insert new operations from WMS source
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, GeographyKey, OperationType,
        Quantity, DurationMinutes, DwellHours, StorageZone,
        StorageZoneDistance, OperatorID, EquipmentID, BatchNumber
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        wms.OperationType,
        wms.Quantity,
        wms.DurationMinutes,
        DATEDIFF(MINUTE, prev.OperationTime, wms.OperationTime) / 60.0 AS DwellHours,
        wms.StorageZone,
        z.DistanceFromDock,
        wms.OperatorID,
        wms.EquipmentID,
        wms.BatchNumber
    FROM WMS_ExternalSource.dbo.Operations wms
    JOIN DimTime t ON CAST(wms.OperationTime AS DATETIME) = t.DateTime
    JOIN DimProductGravity p ON wms.SKU = p.SKU
    JOIN DimGeography g ON wms.WarehouseCode = g.LocationCode
    LEFT JOIN StorageZones z ON wms.StorageZone = z.ZoneCode
    OUTER APPLY (
        SELECT TOP 1 OperationTime
        FROM WMS_ExternalSource.dbo.Operations prev
        WHERE prev.SKU = wms.SKU
          AND prev.OperationTime < wms.OperationTime
        ORDER BY prev.OperationTime DESC
    ) prev
    WHERE wms.OperationTime > @LastLoadTime
      AND NOT EXISTS (
          SELECT 1 FROM FactWarehouseOperations f
          WHERE f.BatchNumber = wms.BatchNumber
      );
    
    -- Update last load timestamp
    UPDATE ETL_Config
    SET LastLoadTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
```

**Recalculate Product Gravity Scores**:

```sql
CREATE PROCEDURE usp_RecalculateGravityScores
    @LookbackDays INT = 90
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate velocity, value, and fragility metrics
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            COUNT(*) AS PickCount,
            SUM(wo.Quantity) AS TotalQuantity,
            AVG(wo.DurationMinutes) AS AvgPickTime,
            MAX(p.UnitValue) AS UnitValue,
            MAX(p.FragilityIndex) AS FragilityIndex
        FROM DimProductGravity p
        JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
        WHERE wo.OperationType = 'Picking'
          AND wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTime = DATEADD(DAY, -@LookbackDays, GETDATE()))
        GROUP BY p.ProductKey
    ),
    VelocityClassification AS (
        SELECT 
            ProductKey,
            PickCount,
            UnitValue,
            FragilityIndex,
            NTILE(3) OVER (ORDER BY PickCount DESC) AS VelocityTier
        FROM ProductMetrics
    )
    UPDATE p
    SET 
        p.GravityScore = (v.PickCount * v.UnitValue) / NULLIF(v.FragilityIndex, 0),
        p.VelocityClass = CASE v.VelocityTier
            WHEN 1 THEN 'Fast'
            WHEN 2 THEN 'Medium'
            ELSE 'Slow'
        END,
        p.OptimalZoneDistance = CASE v.VelocityTier
            WHEN 1 THEN 15  -- Fast movers within 15m
            WHEN 2 THEN 50  -- Medium movers within 50m
            ELSE 100        -- Slow movers can be further
        END,
        p.LastRecalculated = GETDATE()
    FROM DimProductGravity p
    JOIN VelocityClassification v ON p.ProductKey = v.ProductKey;
END;
```

### Analytics and Reporting

**Cross-Fact KPI Query - Dwell Time vs Fleet Idle Time**:

```sql
CREATE PROCEDURE usp_CrossFactAnalysis_DwellVsIdle
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        t.FiscalPeriod,
        p.Category AS ProductCategory,
        g.RegionName,
        -- Warehouse metrics
        AVG(wo.DwellHours) AS AvgDwellHours,
        SUM(wo.Quantity) AS TotalUnitsProcessed,
        -- Fleet metrics
        AVG(ft.IdleMinutes) AS AvgFleetIdleMinutes,
        SUM(ft.FuelLiters) AS TotalFuelConsumed,
        -- Cross-fact correlation
        CASE 
            WHEN AVG(wo.DwellHours) > 72 AND AVG(ft.IdleMinutes) > 30 
            THEN 'High Risk - Both Inefficient'
            WHEN AVG(wo.DwellHours) > 72 
            THEN 'Warehouse Bottleneck'
            WHEN AVG(ft.IdleMinutes) > 30 
            THEN 'Fleet Inefficiency'
            ELSE 'Optimal'
        END AS PerformanceStatus
    FROM FactWarehouseOperations wo
    JOIN DimTime t ON wo.TimeKey = t.TimeKey
    JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    -- Link to fleet through geography and overlapping time windows
    LEFT JOIN FactFleetTrips ft ON 
        ft.OriginGeographyKey = g.GeographyKey
        AND ABS(DATEDIFF(HOUR, t.DateTime, (SELECT DateTime FROM DimTime WHERE TimeKey = ft.TimeKey))) <= 2
    WHERE t.DateTime BETWEEN @StartDate AND @EndDate
    GROUP BY t.FiscalPeriod, p.Category, g.RegionName
    ORDER BY AvgDwellHours DESC, AvgFleetIdleMinutes DESC;
END;
```

**Fleet Triage Priority Queue**:

```sql
CREATE PROCEDURE usp_GenerateFleetTriagePriority
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH VehicleHealth AS (
        SELECT 
            v.VehicleKey,
            v.VehicleID,
            v.TirePressurePSI,
            v.EngineHoursTotal,
            v.LastMaintenanceDate,
            DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) AS DaysSinceMaintenance,
            -- Recent trip metrics
            AVG(ft.LoadWeightKG) AS AvgLoadWeight,
            SUM(ft.RevenueValue) AS TotalRevenueAtRisk,
            MAX(CASE WHEN ft.TripStartTime >= DATEADD(DAY, -7, GETDATE()) THEN 1 ELSE 0 END) AS ActiveLast7Days
        FROM DimVehicle v
        LEFT JOIN FactFleetTrips ft ON v.VehicleKey = ft.VehicleKey
        WHERE ft.TripStartTime >= DATEADD(DAY, -30, GETDATE())
        GROUP BY v.VehicleKey, v.VehicleID, v.TirePressurePSI, 
                 v.EngineHoursTotal, v.LastMaintenanceDate
    ),
    RiskScoring AS (
        SELECT 
            VehicleKey,
            VehicleID,
            -- Risk factors (0-100 scale)
            CASE 
                WHEN TirePressurePSI < 28 THEN 80
                WHEN TirePressurePSI < 32 THEN 40
                ELSE 0
            END AS TirePressureRisk,
            CASE 
                WHEN DaysSinceMaintenance > 90 THEN 100
                WHEN DaysSinceMaintenance > 60 THEN 60
                WHEN DaysSinceMaintenance > 30 THEN 20
                ELSE 0
            END AS MaintenanceOverdueRisk,
            CASE 
                WHEN AvgLoadWeight > 8000 THEN 50
                WHEN AvgLoadWeight > 6000 THEN 25
                ELSE 0
            END AS OverloadRisk,
            TotalRevenueAtRisk,
            ActiveLast7Days
        FROM VehicleHealth
    )
    SELECT TOP 50
        VehicleID,
        -- Weighted composite priority score
        (TirePressureRisk * 0.3 + 
         MaintenanceOverdueRisk * 0.4 + 
         OverloadRisk * 0.3) * 
        (TotalRevenueAtRisk / 10000.0) AS PriorityScore,
        TirePressureRisk,
        MaintenanceOverdueRisk,
        OverloadRisk,
        TotalRevenueAtRisk,
        CASE 
            WHEN ActiveLast7Days = 1 THEN 'Active - Immediate Action'
            ELSE 'Monitor'
        END AS ActionStatus
    FROM RiskScoring
    WHERE (TirePressureRisk > 0 OR MaintenanceOverdueRisk > 0 OR OverloadRisk > 0)
    ORDER BY PriorityScore DESC;
END;
```

## Power BI Integration

### DAX Measures for Cross-Fact Analysis

**Warehouse Efficiency Composite Score**:

```dax
Warehouse Efficiency Score = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellHours])
VAR TargetDwell = 48
VAR DwellScore = (TargetDwell / AvgDwell) * 100

VAR AvgPickTime = AVERAGE(FactWarehouseOperations[DurationMinutes])
VAR TargetPickTime = 3.5
VAR PickScore = (TargetPickTime / AvgPickTime) * 100

VAR ZoneOptimization = 
    DIVIDE(
        COUNTROWS(
            FILTER(
                FactWarehouseOperations,
                FactWarehouseOperations[StorageZoneDistance] <= 
                RELATED(DimProductGravity[OptimalZoneDistance])
            )
        ),
        COUNTROWS(FactWarehouseOperations)
    ) * 100

RETURN (DwellScore * 0.4) + (PickScore * 0.3) + (ZoneOptimization * 0.3)
```

**Fleet Revenue Per Kilometer**:

```dax
Fleet Revenue Per KM = 
DIVIDE(
    SUM(FactFleetTrips[RevenueValue]),
    SUM(FactFleetTrips[DistanceKM]),
    0
)
```

**Cross-Fact: Impact of Dwell Time on Delivery Delays**:

```dax
Dwell Impact on Delays = 
VAR HighDwellTrips = 
    CALCULATETABLE(
        FactFleetTrips,
        FILTER(
            FactWarehouseOperations,
            FactWarehouseOperations[DwellHours] > 72
        )
    )
RETURN
    DIVIDE(
        CALCULATE(AVERAGE(FactFleetTrips[DelayMinutes]), HighDwellTrips),
        AVERAGE(FactFleetTrips[DelayMinutes])
    ) - 1
```

### Row-Level Security Configuration

```dax
// RLS filter for regional managers
[RegionAccess] = 
VAR UserRegions = 
    SELECTCOLUMNS(
        FILTER(
            SecurityUserRegions,
            SecurityUserRegions[UserEmail] = USERPRINCIPALNAME()
        ),
        "AllowedRegion", SecurityUserRegions[RegionName]
    )
RETURN
    DimGeography[RegionName] IN UserRegions
```

## Configuration Patterns

### Automated Alert Configuration

```sql
-- Create alert threshold table
CREATE TABLE AlertThresholds (
    AlertID INT PRIMARY KEY IDENTITY,
    AlertName VARCHAR(100),
    MetricType VARCHAR(50), -- DwellTime, FleetIdle, MaintenanceOverdue
    ThresholdValue DECIMAL(10,2),
    ComparisonOperator VARCHAR(10), -- GreaterThan, LessThan, Equals
    EvaluationWindow VARCHAR(20), -- 1Hour, 1Day, 1Week
    NotificationChannels VARCHAR(200), -- Email,Teams,SMS
    IsActive BIT DEFAULT 1
);

-- Insert sample thresholds
INSERT INTO AlertThresholds (AlertName, MetricType, ThresholdValue, ComparisonOperator, EvaluationWindow, NotificationChannels)
VALUES 
    ('Excessive Dwell Time', 'DwellTime', 72.0, 'GreaterThan', '1Day', 'Email,Teams'),
    ('High Fleet Idle Time', 'FleetIdle', 15.0, 'GreaterThan', '1Hour', 'Email,SMS'),
    ('Maintenance Overdue', 'MaintenanceOverdue', 90.0, 'GreaterThan', '1Day', 'Email,Teams');

-- Automated alert evaluation procedure
CREATE PROCEDURE usp_EvaluateAlerts
AS
BEGIN
    DECLARE @AlertName VARCHAR(100);
    DECLARE @CurrentValue DECIMAL(10,2);
    DECLARE @ThresholdValue DECIMAL(10,2);
    
    -- Check dwell time alerts
    SELECT @CurrentValue = AVG(DwellHours)
    FROM FactWarehouseOperations
    WHERE TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTime >= DATEADD(DAY, -1, GETDATE()));
    
    SELECT @ThresholdValue = ThresholdValue
    FROM AlertThresholds
    WHERE MetricType = 'DwellTime' AND IsActive = 1;
    
    IF @CurrentValue > @ThresholdValue
    BEGIN
        -- Send notification (integrate with email/Teams API)
        EXEC usp_SendNotification 
            @AlertType = 'DwellTime',
            @Message = 'Average dwell time exceeded threshold',
            @CurrentValue = @CurrentValue,
            @Threshold = @ThresholdValue;
    END;
    
    -- Similar logic for other alert types
END;
```

### Data Refresh Scheduling

```sql
-- ETL orchestration master procedure
CREATE PROCEDURE usp_MasterETLRefresh
    @RefreshType VARCHAR(20) = 'Incremental' -- Incremental or Full
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRANSACTION;
    BEGIN TRY
        -- Step 1: Load dimension updates
        EXEC usp_LoadDimensionUpdates;
        
        -- Step 2: Load warehouse operations
        IF @RefreshType = 'Incremental'
            EXEC usp_LoadWarehouseOperations @LastLoadTime = NULL;
        ELSE
            EXEC usp_LoadWarehouseOperations_Full;
        
        -- Step 3: Load fleet trips
        IF @RefreshType = 'Incremental'
            EXEC usp_LoadFleetTrips @LastLoadTime = NULL;
        ELSE
            EXEC usp_LoadFleetTrips_Full;
        
        -- Step 4: Recalculate gravity scores (daily)
        IF DATEPART(HOUR, GETDATE()) = 2 -- Run at 2 AM
            EXEC usp_RecalculateGravityScores @LookbackDays = 90;
        
        -- Step 5: Refresh materialized views
        EXEC sp_refreshview 'vw_WarehouseFleetSummary';
        
        -- Step 6: Evaluate alerts
        EXEC usp_EvaluateAlerts;
        
        COMMIT TRANSACTION;
        
        -- Log success
        INSERT INTO ETL_Log (RefreshTime, Status, Message)
        VALUES (GETDATE(), 'Success', 'Master ETL completed successfully');
        
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        
        -- Log error
        INSERT INTO ETL_Log (RefreshTime, Status, Message)
        VALUES (GETDATE(), 'Error', ERROR_MESSAGE());
        
        THROW;
    END CATCH;
END;
```

### SQL Server Agent Job for Scheduled Refresh

```sql
-- Schedule ETL refresh every 15 minutes during business hours
USE msdb;
EXEC sp_add_job @job_name = 'LogiFleetPulse_ETL_Refresh';

EXEC sp_add_jobstep 
    @job_name = 'LogiFleetPulse_ETL_Refresh',
    @step_name = 'Run Incremental Refresh',
    @command = 'EXEC LogiFleetPulse.dbo.usp_MasterETLRefresh @RefreshType = ''Incremental'';',
    @database_name = 'LogiFleetPulse';

EXEC sp_add_schedule 
    @schedule_name = 'Every15Minutes_BusinessHours',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15,
    @active_start_time = 060000, -- 6 AM
    @active_end_time = 200000; -- 8 PM

EXEC sp_attach_schedule 
    @job_name = 'LogiFleetPulse_ETL_Refresh',
    @schedule_name = 'Every15Minutes_BusinessHours';

EXEC sp_add_jobserver 
    @job_name = 'LogiFleetPulse_ETL_Refresh';
```

## Common Usage Patterns

### Pattern 1: Identify Misaligned Warehouse Zones

```sql
-- Find products stored far from optimal location
SELECT 
    p.SKU,
    p.ProductName,
    p.VelocityClass,
    p.OptimalZoneDistance AS RecommendedDistance,
    AVG(wo.StorageZoneDistance) AS CurrentAvgDistance,
    AVG(wo.StorageZoneDistance) - p.OptimalZoneDistance AS DistanceDeviation,
    COUNT(*) AS OperationCount,
    SUM(wo.DurationMinutes) AS TotalPickTimeMinutes,
    -- Estimated time savings if optimized
    (AVG(wo.StorageZoneDistance) - p.OptimalZoneDistance) * 0.5 AS EstimatedMinutesSavingsPerPick
FROM FactWarehouseOperations wo
JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationType = 'Picking'
  AND wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTime >= DATEADD(DAY, -30, GETDATE()))
GROUP BY p.SKU, p.ProductName, p.VelocityClass, p.OptimalZoneDistance
HAVING AVG(wo.StorageZoneDistance) > p.OptimalZoneDistance + 10
ORDER BY EstimatedMinutesSavingsPerPick DESC;
```

### Pattern 2: Fleet Route Optimization Analysis

```sql
-- Identify routes with high idle time and low revenue efficiency
WITH RouteMetrics AS (
    SELECT 
        CONCAT(og.LocationName, ' → ', dg.LocationName) AS Route,
        COUNT(*) AS TripCount,
        AVG(ft.IdleMinutes) AS AvgIdleMinutes,
        AVG(ft.DistanceKM) AS AvgDistanceKM,
        AVG(ft.FuelLiters) AS AvgFuelLiters,
        SUM(ft.RevenueValue) AS TotalRevenue,
        SUM(ft.RevenueValue) / NULLIF(SUM(ft.FuelLiters), 0) AS RevenuePerLiter
    FROM FactFleetTrips ft
    JOIN DimGeography og ON ft.OriginGeographyKey = og.GeographyKey
    JOIN DimGeography dg ON ft.DestinationGeographyKey = dg.GeographyKey
    WHERE ft.TripStartTime >= DATEADD(DAY, -90, GETDATE())
    GROUP BY og.LocationName, dg.LocationName
)
SELECT 
    Route,
    TripCount,
    AvgIdleMinutes,
    AvgDistanceKM,
    AvgFuelLiters,
    TotalRevenue,
    RevenuePerLiter,
    CASE 
        WHEN AvgIdleMinutes > 30 AND RevenuePerLiter < 50 THEN 'Critical - Reroute Recommended'
        WHEN AvgIdleMinutes > 30 THEN 'High Idle - Optimize Timing'
        WHEN RevenuePerLiter < 50 THEN 'Low Revenue - Consider Consolidation'
        ELSE 'Acceptable'
    END AS RecommendedAction
FROM RouteMetrics
ORDER BY AvgIdleMinutes DESC, RevenuePerLiter ASC;
```

### Pattern 3: Temporal Elasticity Simulation

```sql
-- Simulate impact of increasing warehouse capacity from 80% to 95%
WITH CurrentCapacity AS (
    SELECT 
        t.FiscalPeriod,
        COUNT(*) AS OperationCount,
        AVG(wo.DwellHours) AS AvgDwellHours,
        SUM(wo.Quantity) AS TotalUnits,
        -- Assuming 10000 units = 80% capacity
        (SUM(wo.Quantity) / 10
