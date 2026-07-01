---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for warehouse, fleet, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy logistics data warehouse with Power BI"
  - "configure multi-fact star schema for fleet management"
  - "implement warehouse gravity zones optimization"
  - "create cross-modal supply chain dashboard"
  - "build logistics KPI harmonization layer"
  - "integrate fleet telemetry with warehouse operations"
  - "design adaptive supply chain data model"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform built on MS SQL Server and Power BI that unifies warehouse operations, fleet management, and supply chain data into a single semantic layer. It provides:

- **Multi-fact star schema** linking warehouse operations, fleet trips, and cross-dock activities
- **Time-phased dimensions** with 15-minute granularity for real-time analytics
- **Warehouse Gravity Zones** for spatial optimization based on pick frequency and item value
- **Cross-fact KPI harmonization** enabling queries across warehouse and fleet metrics
- **Predictive bottleneck detection** using temporal elasticity modeling
- **Power BI dashboards** with role-based access and natural language query support

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Access to data sources: WMS, fleet telemetry API, ERP systems
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance and create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create time dimension with 15-minute granularity
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    Hour INT NOT NULL,
    MinuteInterval INT NOT NULL, -- 0, 15, 30, 45
    DayOfWeek VARCHAR(20),
    IsWeekend BIT,
    FiscalQuarter VARCHAR(10),
    FiscalYear INT
);
CREATE CLUSTERED INDEX IX_DimTime_DateTime ON DimTime(FullDateTime);

-- Create geography dimension with hierarchy
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    NodeID VARCHAR(50) UNIQUE NOT NULL,
    NodeName VARCHAR(200) NOT NULL,
    NodeType VARCHAR(50), -- Warehouse, Distribution Center, Route Node
    Continent VARCHAR(100),
    Country VARCHAR(100),
    Region VARCHAR(100),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    IsActive BIT DEFAULT 1
);
CREATE NONCLUSTERED INDEX IX_DimGeography_Type ON DimGeography(NodeType, IsActive);

-- Create product gravity dimension
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * fragility factor
    VelocityClass VARCHAR(20), -- Fast, Medium, Slow
    ValueTier VARCHAR(20), -- High, Medium, Low
    FragilityIndex DECIMAL(3,2),
    OptimalZone VARCHAR(50),
    LastGravityUpdate DATETIME2
);
CREATE NONCLUSTERED INDEX IX_DimProduct_Gravity ON DimProductGravity(GravityScore DESC);

-- Create supplier reliability dimension
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) UNIQUE NOT NULL,
    SupplierName VARCHAR(200),
    LeadTimeVariance DECIMAL(5,2), -- Standard deviation in days
    DefectPercentage DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2), -- 0-100 scale
    ReliabilityRating VARCHAR(20), -- Excellent, Good, Fair, Poor
    LastUpdated DATETIME2
);

-- Create warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    WarehouseKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    Quantity INT,
    DwellTimeMinutes INT,
    CycleTimeSeconds INT,
    ZoneLocation VARCHAR(50),
    OperatorID VARCHAR(50),
    BatchNumber VARCHAR(50)
);
CREATE NONCLUSTERED INDEX IX_FactWH_Time ON FactWarehouseOperations(TimeKey, WarehouseKey);
CREATE NONCLUSTERED INDEX IX_FactWH_Product ON FactWarehouseOperations(ProductKey, OperationType);

-- Create fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    TripDurationMinutes INT,
    LoadWeightKG DECIMAL(10,2),
    DelayMinutes INT,
    DelayReason VARCHAR(200)
);
CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey, OriginKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID, TimeKey);

-- Create cross-dock fact table
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    FacilityKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT NOT NULL FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    Quantity INT,
    TransferTimeMinutes INT,
    TemperatureCompliance BIT,
    QualityCheckPassed BIT
);
CREATE NONCLUSTERED INDEX IX_FactCross_Time ON FactCrossDock(TimeKey, FacilityKey);
```

### Step 2: Create Data Loading Procedures

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations from source system
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType,
        Quantity, DwellTimeMinutes, CycleTimeSeconds,
        ZoneLocation, OperatorID, BatchNumber
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        src.OperationType,
        src.Quantity,
        src.DwellTimeMinutes,
        src.CycleTimeSeconds,
        src.ZoneLocation,
        src.OperatorID,
        src.BatchNumber
    FROM WarehouseSourceSystem.dbo.Operations src
    INNER JOIN DimTime t ON CAST(src.OperationDateTime AS DATETIME2) = t.FullDateTime
    INNER JOIN DimGeography g ON src.WarehouseID = g.NodeID
    INNER JOIN DimProductGravity p ON src.SKU = p.SKU
    WHERE src.OperationDateTime > @LastLoadDateTime
        AND src.IsProcessed = 0;
    
    -- Update gravity scores based on new operations
    EXEC usp_UpdateGravityScores;
END;
GO

-- Stored procedure to recalculate gravity scores
CREATE PROCEDURE usp_UpdateGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH ProductMetrics AS (
        SELECT 
            ProductKey,
            COUNT(*) as PickFrequency,
            AVG(CycleTimeSeconds) as AvgCycleTime,
            SUM(Quantity) as TotalVolume
        FROM FactWarehouseOperations
        WHERE TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM FactWarehouseOperations) -- Last 30 days
            AND OperationType = 'Picking'
        GROUP BY ProductKey
    )
    UPDATE p
    SET 
        GravityScore = (pm.PickFrequency * 0.4) + 
                      ((100 - pm.AvgCycleTime/10) * 0.3) + 
                      (p.FragilityIndex * 20 * 0.3),
        VelocityClass = CASE 
            WHEN pm.PickFrequency > 100 THEN 'Fast'
            WHEN pm.PickFrequency > 30 THEN 'Medium'
            ELSE 'Slow'
        END,
        OptimalZone = CASE
            WHEN (pm.PickFrequency * 0.4) + ((100 - pm.AvgCycleTime/10) * 0.3) + (p.FragilityIndex * 20 * 0.3) > 70 THEN 'High-Gravity-A'
            WHEN (pm.PickFrequency * 0.4) + ((100 - pm.AvgCycleTime/10) * 0.3) + (p.FragilityIndex * 20 * 0.3) > 40 THEN 'Medium-Gravity-B'
            ELSE 'Low-Gravity-C'
        END,
        LastGravityUpdate = GETDATE()
    FROM DimProductGravity p
    INNER JOIN ProductMetrics pm ON p.ProductKey = pm.ProductKey;
END;
GO

-- Stored procedure for cross-fact KPI query
CREATE PROCEDURE usp_GetCrossModalEfficiency
    @StartDate DATE,
    @EndDate DATE,
    @WarehouseID VARCHAR(50) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        g.NodeName as Warehouse,
        p.Category as ProductCategory,
        p.OptimalZone as CurrentZone,
        -- Warehouse metrics
        AVG(wh.DwellTimeMinutes) as AvgDwellTime,
        SUM(wh.Quantity) as TotalUnitsProcessed,
        -- Fleet metrics for shipments from this warehouse
        AVG(ft.IdleTimeMinutes) as AvgFleetIdleTime,
        AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKM, 0)) as AvgFuelEfficiency,
        -- Cross-modal KPI
        AVG(wh.DwellTimeMinutes) * AVG(ft.IdleTimeMinutes) / 100.0 as EfficiencyFrictionIndex,
        -- Recommendations
        CASE 
            WHEN AVG(wh.DwellTimeMinutes) > 72 AND p.GravityScore > 50 THEN 'HIGH PRIORITY: Move to high-gravity zone'
            WHEN AVG(ft.IdleTimeMinutes) > 30 AND AVG(wh.DwellTimeMinutes) > 48 THEN 'Optimize cross-dock timing'
            ELSE 'Within acceptable range'
        END as Recommendation
    FROM FactWarehouseOperations wh
    INNER JOIN DimTime t ON wh.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON wh.WarehouseKey = g.GeographyKey
    INNER JOIN DimProductGravity p ON wh.ProductKey = p.ProductKey
    LEFT JOIN FactFleetTrips ft ON ft.OriginKey = g.GeographyKey 
        AND ft.TimeKey BETWEEN wh.TimeKey AND wh.TimeKey + 96 -- Within 24 hours
    WHERE t.Date BETWEEN @StartDate AND @EndDate
        AND (@WarehouseID IS NULL OR g.NodeID = @WarehouseID)
    GROUP BY g.NodeName, p.Category, p.OptimalZone, p.GravityScore
    ORDER BY EfficiencyFrictionIndex DESC;
END;
GO
```

### Step 3: Configure Data Sources

Create a configuration file for external data connections:

```sql
-- Configure external data sources (adjust for your environment)
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '${SQL_MASTER_KEY_PASSWORD}';

CREATE DATABASE SCOPED CREDENTIAL WMSCredential
WITH IDENTITY = '${WMS_USERNAME}',
SECRET = '${WMS_PASSWORD}';

CREATE EXTERNAL DATA SOURCE WMSSource
WITH (
    TYPE = RDBMS,
    LOCATION = '${WMS_SERVER}',
    DATABASE_NAME = '${WMS_DATABASE}',
    CREDENTIAL = WMSCredential
);

-- Create external table for streaming telemetry data
CREATE EXTERNAL TABLE ext_FleetTelemetry (
    VehicleID VARCHAR(50),
    EventDateTime DATETIME2,
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2),
    EngineTemp DECIMAL(5,2)
)
WITH (
    DATA_SOURCE = FleetTelemetryAPI,
    LOCATION = '${TELEMETRY_ENDPOINT}'
);
```

### Step 4: Import Power BI Template

```bash
# Download the Power BI template file
# Open Power BI Desktop and load LogiFleet_Pulse_Master.pbit
# When prompted, enter your SQL Server connection details:
# - Server: your-sql-server.database.windows.net
# - Database: LogiFleetPulse
# - Data Connectivity mode: DirectQuery (recommended for real-time) or Import
```

## Key DAX Measures for Power BI

Create these measures in Power BI for advanced analytics:

```dax
-- Warehouse Efficiency Index
Warehouse Efficiency Index = 
VAR AvgCycleTime = AVERAGE(FactWarehouseOperations[CycleTimeSeconds])
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR IdealCycleTime = 45  -- seconds
VAR IdealDwellTime = 24  -- hours * 60
RETURN
    (IdealCycleTime / AvgCycleTime) * 0.5 + 
    (IdealDwellTime / AvgDwellTime) * 0.5

-- Fleet Utilization Rate
Fleet Utilization Rate = 
DIVIDE(
    SUM(FactFleetTrips[TripDurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes]),
    0
)

-- Cross-Modal Friction Index (lower is better)
Cross-Modal Friction Index = 
VAR WarehouseDwell = CALCULATE(AVERAGE(FactWarehouseOperations[DwellTimeMinutes]))
VAR FleetIdle = CALCULATE(AVERAGE(FactFleetTrips[IdleTimeMinutes]))
VAR FleetDelay = CALCULATE(AVERAGE(FactFleetTrips[DelayMinutes]))
RETURN
    (WarehouseDwell * 0.4) + (FleetIdle * 0.3) + (FleetDelay * 0.3)

-- Gravity Zone Compliance
Gravity Zone Compliance % = 
VAR TotalOps = COUNTROWS(FactWarehouseOperations)
VAR CompliantOps = CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FactWarehouseOperations[ZoneLocation] = RELATED(DimProductGravity[OptimalZone])
)
RETURN
    DIVIDE(CompliantOps, TotalOps, 0) * 100

-- Predictive Bottleneck Score
Predictive Bottleneck Score = 
VAR RecentDwellIncrease = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATESINPERIOD(DimTime[Date], MAX(DimTime[Date]), -7, DAY)
    ) / 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATESINPERIOD(DimTime[Date], MAX(DimTime[Date]), -30, DAY)
    )
VAR FleetDelayTrend = 
    CALCULATE(
        AVERAGE(FactFleetTrips[DelayMinutes]),
        DATESINPERIOD(DimTime[Date], MAX(DimTime[Date]), -7, DAY)
    ) / 
    CALCULATE(
        AVERAGE(FactFleetTrips[DelayMinutes]),
        DATESINPERIOD(DimTime[Date], MAX(DimTime[Date]), -30, DAY)
    )
RETURN
    (RecentDwellIncrease * 0.6) + (FleetDelayTrend * 0.4)

-- Supplier Reliability Impact
Supplier Impact Score = 
CALCULATE(
    AVERAGE(DimSupplierReliability[ComplianceScore]) * 
    (1 - AVERAGE(DimSupplierReliability[DefectPercentage])) *
    (1 / (1 + AVERAGE(DimSupplierReliability[LeadTimeVariance])))
)
```

## Configuration Examples

### Setting Up Automated Alerts

```sql
-- Create alert configuration table
CREATE TABLE AlertConfiguration (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName VARCHAR(200) NOT NULL,
    MetricType VARCHAR(100), -- DwellTime, IdleTime, DelayMinutes, etc.
    ThresholdValue DECIMAL(10,2),
    ComparisonOperator VARCHAR(10), -- >, <, =, >=, <=
    AlertSeverity VARCHAR(20), -- Critical, Warning, Info
    NotificationEmail VARCHAR(500),
    IsActive BIT DEFAULT 1
);

-- Insert sample alert configurations
INSERT INTO AlertConfiguration (AlertName, MetricType, ThresholdValue, ComparisonOperator, AlertSeverity, NotificationEmail, IsActive)
VALUES 
    ('Critical Dwell Time', 'DwellTimeMinutes', 72, '>', 'Critical', '${LOGISTICS_MANAGER_EMAIL}', 1),
    ('High Fleet Idle Time', 'IdleTimeMinutes', 30, '>', 'Warning', '${FLEET_MANAGER_EMAIL}', 1),
    ('Fuel Efficiency Drop', 'FuelEfficiency', 0.08, '<', 'Warning', '${OPERATIONS_EMAIL}', 1),
    ('Gravity Zone Mismatch', 'GravityCompliance', 70, '<', 'Warning', '${WAREHOUSE_SUPERVISOR_EMAIL}', 1);

-- Stored procedure to check alerts and send notifications
CREATE PROCEDURE usp_CheckAndTriggerAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check dwell time alerts
    INSERT INTO AlertLog (AlertID, TriggeredDateTime, MetricValue, AffectedEntity)
    SELECT 
        ac.AlertID,
        GETDATE(),
        AVG(wh.DwellTimeMinutes),
        g.NodeName
    FROM AlertConfiguration ac
    CROSS APPLY (
        SELECT TOP 100
            wh.WarehouseKey,
            wh.DwellTimeMinutes
        FROM FactWarehouseOperations wh
        INNER JOIN DimTime t ON wh.TimeKey = t.TimeKey
        WHERE t.Date >= DATEADD(hour, -4, GETDATE())
    ) wh
    INNER JOIN DimGeography g ON wh.WarehouseKey = g.GeographyKey
    WHERE ac.MetricType = 'DwellTimeMinutes'
        AND ac.IsActive = 1
    GROUP BY ac.AlertID, g.NodeName, ac.ThresholdValue, ac.ComparisonOperator
    HAVING (ac.ComparisonOperator = '>' AND AVG(wh.DwellTimeMinutes) > ac.ThresholdValue);
    
    -- Similar logic for other alert types...
    -- In production, integrate with SQL Server Database Mail or external notification service
END;
GO

-- Schedule this procedure to run every 15 minutes
```

### Row-Level Security Setup

```sql
-- Create role-based security
CREATE TABLE UserRoles (
    UserID VARCHAR(100) PRIMARY KEY,
    RoleName VARCHAR(50), -- Executive, Manager, Supervisor, Analyst
    AccessLevel VARCHAR(50), -- Global, Regional, Facility
    GeographyFilter VARCHAR(100) -- NULL for global, Region/Warehouse name for restricted
);

-- Create security predicate function
CREATE FUNCTION dbo.fn_SecurityPredicate(@GeographyKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS fn_SecurityPredicate_Result
    FROM dbo.UserRoles ur
    WHERE ur.UserID = USER_NAME()
        AND (
            ur.AccessLevel = 'Global'
            OR EXISTS (
                SELECT 1 
                FROM dbo.DimGeography g
                WHERE g.GeographyKey = @GeographyKey
                    AND (ur.GeographyFilter IS NULL OR g.Region = ur.GeographyFilter OR g.NodeName = ur.GeographyFilter)
            )
        );
GO

-- Apply security policy to warehouse operations
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(WarehouseKey)
ON dbo.FactWarehouseOperations
WITH (STATE = ON);
GO
```

## Common Patterns & Queries

### Pattern 1: Cross-Fact Analysis

```sql
-- Find products with high warehouse dwell time AND high fleet delays
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    AVG(wh.DwellTimeMinutes) as AvgWarehouseDwell,
    AVG(ft.DelayMinutes) as AvgFleetDelay,
    AVG(wh.DwellTimeMinutes) + AVG(ft.DelayMinutes) as TotalLatencyMinutes
FROM FactWarehouseOperations wh
INNER JOIN DimProductGravity p ON wh.ProductKey = p.ProductKey
INNER JOIN FactCrossDock cd ON cd.ProductKey = p.ProductKey
INNER JOIN FactFleetTrips ft ON cd.OutboundTripKey = ft.TripKey
WHERE wh.TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime) -- Last 7 days
GROUP BY p.SKU, p.ProductName, p.GravityScore
HAVING AVG(wh.DwellTimeMinutes) > 48 AND AVG(ft.DelayMinutes) > 15
ORDER BY TotalLatencyMinutes DESC;
```

### Pattern 2: Temporal Elasticity Analysis

```sql
-- Simulate warehouse capacity increase impact on fleet utilization
WITH BaselineMetrics AS (
    SELECT 
        g.NodeName,
        COUNT(DISTINCT wh.OperationKey) as CurrentOperations,
        AVG(ft.TripDurationMinutes) as AvgTripDuration,
        AVG(ft.IdleTimeMinutes) as AvgIdleTime
    FROM FactWarehouseOperations wh
    INNER JOIN DimGeography g ON wh.WarehouseKey = g.GeographyKey
    LEFT JOIN FactFleetTrips ft ON ft.OriginKey = g.GeographyKey
        AND ft.TimeKey BETWEEN wh.TimeKey AND wh.TimeKey + 48
    WHERE wh.TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime)
    GROUP BY g.NodeName
),
SimulatedMetrics AS (
    SELECT 
        NodeName,
        CurrentOperations * 1.15 as SimulatedOperations, -- 15% increase
        AvgTripDuration * 0.92 as ProjectedTripDuration, -- Assumed 8% improvement
        AvgIdleTime * 1.08 as ProjectedIdleTime -- Assumed 8% increase
    FROM BaselineMetrics
)
SELECT 
    b.NodeName,
    b.CurrentOperations,
    s.SimulatedOperations,
    b.AvgTripDuration,
    s.ProjectedTripDuration,
    b.AvgIdleTime,
    s.ProjectedIdleTime,
    (s.ProjectedTripDuration - b.AvgTripDuration) / b.AvgTripDuration * 100 as TripDurationChangePercent,
    CASE 
        WHEN (s.ProjectedIdleTime - b.AvgIdleTime) / b.AvgIdleTime > 0.10 THEN 'Capacity bottleneck likely'
        ELSE 'Acceptable elasticity'
    END as RiskAssessment
FROM BaselineMetrics b
INNER JOIN SimulatedMetrics s ON b.NodeName = s.NodeName;
```

### Pattern 3: Gravity Zone Optimization

```sql
-- Identify products that should be relocated to different gravity zones
WITH ProductPerformance AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.OptimalZone as CurrentZone,
        AVG(CASE WHEN wh.ZoneLocation = p.OptimalZone THEN wh.CycleTimeSeconds ELSE NULL END) as AvgCycleTimeInOptimalZone,
        AVG(CASE WHEN wh.ZoneLocation <> p.OptimalZone THEN wh.CycleTimeSeconds ELSE NULL END) as AvgCycleTimeInOtherZones,
        SUM(CASE WHEN wh.ZoneLocation = p.OptimalZone THEN 1 ELSE 0 END) as PicksInOptimalZone,
        SUM(CASE WHEN wh.ZoneLocation <> p.OptimalZone THEN 1 ELSE 0 END) as PicksInOtherZones,
        p.GravityScore
    FROM DimProductGravity p
    INNER JOIN FactWarehouseOperations wh ON p.ProductKey = wh.ProductKey
    WHERE wh.OperationType = 'Picking'
        AND wh.TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime)
    GROUP BY p.ProductKey, p.SKU, p.OptimalZone, p.GravityScore
)
SELECT 
    SKU,
    CurrentZone,
    GravityScore,
    PicksInOptimalZone,
    PicksInOtherZones,
    AvgCycleTimeInOptimalZone,
    AvgCycleTimeInOtherZones,
    CASE 
        WHEN PicksInOtherZones > PicksInOptimalZone * 2 THEN 'RELOCATE: Frequently picked from wrong zone'
        WHEN AvgCycleTimeInOtherZones < AvgCycleTimeInOptimalZone * 0.8 THEN 'REASSESS: Better performance in non-optimal zone'
        WHEN GravityScore > 70 AND PicksInOptimalZone < 50 THEN 'VERIFY: High gravity but low optimal picks'
        ELSE 'OK: Current zoning appropriate'
    END as RecommendedAction
FROM ProductPerformance
WHERE PicksInOptimalZone + PicksInOtherZones > 10 -- Minimum statistical significance
ORDER BY GravityScore DESC;
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

```sql
-- Add covering indexes for common join patterns
CREATE NONCLUSTERED INDEX IX_FactWH_CrossFact 
ON FactWarehouseOperations (ProductKey, TimeKey, WarehouseKey)
INCLUDE (DwellTimeMinutes, CycleTimeSeconds);

CREATE NONCLUSTERED INDEX IX_FactFleet_CrossFact
ON FactFleetTrips (OriginKey, TimeKey, DestinationKey)
INCLUDE (IdleTimeMinutes, DelayMinutes, FuelConsumedLiters);

-- Update statistics regularly
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

### Issue: Power BI Dashboard Refresh Failures

```powershell
# Use Power BI REST API to check refresh status
$headers = @{
    "Authorization" = "Bearer ${POWERBI_ACCESS_TOKEN}"
}

$refreshHistory = Invoke-RestMethod -Uri "https://api.powerbi.com/v1.0/myorg/groups/${WORKSPACE_ID}/datasets/${DATASET_ID}/refreshes" -Headers $headers -Method Get

# Check for errors
$refreshHistory.value | Where-Object { $_.status -eq "Failed" } | Select-Object startTime, endTime, serviceExceptionJson
```

### Issue: Gravity Scores Not Updating

```sql
-- Verify the stored procedure is running on schedule
SELECT 
    job.name AS JobName,
    job.enabled,
    schedule.next_run_date,
    schedule.next_run_time,
    history.run_status,
    history.message
FROM msdb.dbo.sysjobs job
INNER JOIN msdb.dbo.sysjobschedules schedule ON job.job_id = schedule.job_id
LEFT JOIN msdb.dbo.sysjobhistory history ON job.job_id = history.job_id
WHERE job.name LIKE '%GravityScore%'
ORDER BY history.run_date DESC, history.run_time DESC;

-- Manually trigger update
EXEC usp_UpdateGravityScores;

-- Check last update times
SELECT 
    SKU,
    ProductName,
    GravityScore,
    LastGravityUpdate,
    DATEDIFF(hour, LastGravityUpdate, GETDATE()) as HoursSinceUpdate
FROM DimProductGravity
WHERE LastGravityUpdate < DATEADD(day, -1, GETDATE())
ORDER BY HoursSinceUpdate DESC;
```

### Issue: External Data Source Connection Failures

```sql
-- Test external data source connectivity
SELECT TOP
