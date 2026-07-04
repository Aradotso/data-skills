---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for real-time logistics intelligence, fleet optimization, and supply chain analytics
triggers:
  - "set up logifleet pulse warehouse analytics"
  - "configure supply chain logistics dashboard"
  - "build power bi fleet management reports"
  - "implement cross-modal logistics intelligence"
  - "create warehouse gravity zone analytics"
  - "deploy sql server logistics data model"
  - "integrate fleet telemetry with warehouse operations"
  - "optimize supply chain data warehouse schema"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a multi-modal logistics intelligence engine built on MS SQL Server and Power BI. It provides:

- **Unified semantic layer** connecting warehouse operations, fleet telemetry, inventory, and external data sources
- **Multi-fact star schema** with time-phased dimensions for cross-functional KPI analysis
- **Real-time dashboards** (15-minute refresh) for operational awareness
- **Predictive analytics** including bottleneck detection and fleet maintenance triage
- **Warehouse Gravity Zones™** spatial optimization based on pick frequency and item value
- **Temporal elasticity modeling** for scenario planning

Primary use cases: 3PL operations, retail supply chains, food distribution, multi-warehouse networks.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Database credentials with CREATE TABLE, CREATE VIEW, CREATE PROCEDURE permissions
- Optional: Azure Synapse Analytics for big data enrichment

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Run the schema deployment script (provided in repository)

USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTimeStamp DATETIME2 NOT NULL,
    FifteenMinuteBucket INT,
    HourOfDay INT,
    DayOfWeek INT,
    FiscalPeriod VARCHAR(20),
    INDEX IX_DimTime_DateTimeStamp (DateTimeStamp)
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY,
    LocationID VARCHAR(50) NOT NULL,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'RouteNode', 'Dock'
    ParentGeographyKey INT,
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6),
    INDEX IX_DimGeography_LocationID (LocationID)
);

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY,
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(500),
    Category VARCHAR(200),
    GravityScore DECIMAL(5,2), -- Higher = higher priority placement
    VelocityClass VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueClass VARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityScore DECIMAL(3,2),
    OptimalZone VARCHAR(50),
    INDEX IX_DimProductGravity_SKU (SKU),
    INDEX IX_DimProductGravity_GravityScore (GravityScore)
);

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY,
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(500),
    LeadTimeVariance DECIMAL(8,2),
    DefectPercentage DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2),
    LastUpdated DATETIME2,
    INDEX IX_DimSupplierReliability_SupplierID (SupplierID)
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityProcessed INT,
    CycleTimeMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    OperatorID VARCHAR(50),
    ZoneID VARCHAR(50),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    INDEX IX_FactWO_Time (TimeKey),
    INDEX IX_FactWO_Product (ProductKey)
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    DistanceKM DECIMAL(10,2),
    FuelLiters DECIMAL(8,2),
    IdleTimeMinutes DECIMAL(8,2),
    LoadingTimeMinutes DECIMAL(8,2),
    UnloadingTimeMinutes DECIMAL(8,2),
    DelayMinutes DECIMAL(8,2),
    DelayReason VARCHAR(200),
    FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (EndTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey),
    INDEX IX_FactFleet_StartTime (StartTimeKey),
    INDEX IX_FactFleet_Vehicle (VehicleID)
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    QuantityTransferred INT,
    TransferTimeMinutes DECIMAL(8,2),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    FOREIGN KEY (SupplierKey) REFERENCES DimSupplierReliability(SupplierKey),
    INDEX IX_FactCD_Time (TimeKey)
);
```

### Step 2: Configure Data Sources

Create a configuration file to manage connection strings and data source endpoints:

```sql
-- Create configuration table
CREATE TABLE SystemConfiguration (
    ConfigKey VARCHAR(100) PRIMARY KEY,
    ConfigValue VARCHAR(MAX),
    ConfigType VARCHAR(50),
    LastUpdated DATETIME2 DEFAULT GETDATE()
);

-- Insert connection configuration (use environment variables)
INSERT INTO SystemConfiguration (ConfigKey, ConfigValue, ConfigType)
VALUES 
    ('WMS_API_ENDPOINT', '${WMS_API_URL}', 'DataSource'),
    ('TELEMETRY_API_ENDPOINT', '${TELEMETRY_API_URL}', 'DataSource'),
    ('WEATHER_API_KEY', '${WEATHER_API_KEY}', 'APIKey'),
    ('REFRESH_INTERVAL_MINUTES', '15', 'SystemSetting'),
    ('ALERT_EMAIL_LIST', '${ALERT_RECIPIENTS}', 'Notification');
```

### Step 3: Create ETL Stored Procedures

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations from staging/source
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        QuantityProcessed, CycleTimeMinutes, DwellTimeHours,
        OperatorID, ZoneID
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        src.OperationType,
        src.Quantity,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime),
        DATEDIFF(HOUR, src.ArrivalTime, src.ProcessTime),
        src.OperatorID,
        src.ZoneID
    FROM StagingWarehouseOps src
    INNER JOIN DimTime t ON DATEADD(MINUTE, 
        (DATEDIFF(MINUTE, '2000-01-01', src.OperationTime) / 15) * 15, 
        '2000-01-01') = t.DateTimeStamp
    INNER JOIN DimGeography g ON src.LocationID = g.LocationID
    INNER JOIN DimProductGravity p ON src.SKU = p.SKU
    WHERE src.LastModified > @LastLoadDateTime;
    
    -- Update gravity scores based on recent velocity
    UPDATE DimProductGravity
    SET GravityScore = (
        SELECT 
            (AVG(QuantityProcessed) * 0.4) +
            (1.0 / AVG(NULLIF(DwellTimeHours, 0)) * 0.6) * 100
        FROM FactWarehouseOperations
        WHERE ProductKey = DimProductGravity.ProductKey
            AND TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime) -- Last week
    );
END;
GO

-- Fleet trip processing with anomaly detection
CREATE PROCEDURE usp_LoadFleetTrips
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        StartTimeKey, EndTimeKey, OriginGeographyKey, DestinationGeographyKey,
        VehicleID, DriverID, DistanceKM, FuelLiters, IdleTimeMinutes,
        LoadingTimeMinutes, UnloadingTimeMinutes, DelayMinutes, DelayReason
    )
    SELECT 
        tStart.TimeKey,
        tEnd.TimeKey,
        gOrigin.GeographyKey,
        gDest.GeographyKey,
        src.VehicleID,
        src.DriverID,
        src.DistanceKM,
        src.FuelLiters,
        src.IdleMinutes,
        src.LoadMinutes,
        src.UnloadMinutes,
        DATEDIFF(MINUTE, src.ScheduledArrival, src.ActualArrival),
        src.DelayReason
    FROM StagingFleetTrips src
    INNER JOIN DimTime tStart ON DATEADD(MINUTE, 
        (DATEDIFF(MINUTE, '2000-01-01', src.DepartureTime) / 15) * 15, 
        '2000-01-01') = tStart.DateTimeStamp
    INNER JOIN DimTime tEnd ON DATEADD(MINUTE, 
        (DATEDIFF(MINUTE, '2000-01-01', src.ArrivalTime) / 15) * 15, 
        '2000-01-01') = tEnd.DateTimeStamp
    INNER JOIN DimGeography gOrigin ON src.OriginLocationID = gOrigin.LocationID
    INNER JOIN DimGeography gDest ON src.DestinationLocationID = gDest.LocationID
    WHERE src.LastModified > @LastLoadDateTime;
    
    -- Flag anomalies (excessive idle time)
    INSERT INTO AlertQueue (AlertType, Severity, Message, VehicleID, TripKey)
    SELECT 
        'ExcessiveIdleTime',
        'High',
        'Vehicle ' + VehicleID + ' had ' + CAST(IdleTimeMinutes AS VARCHAR) + ' minutes idle time',
        VehicleID,
        TripKey
    FROM FactFleetTrips
    WHERE IdleTimeMinutes > 60 -- More than 1 hour idle
        AND StartTimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime); -- Last 24 hours
END;
GO
```

### Step 4: Load Power BI Template

```powershell
# Download and open the Power BI template
# File: LogiFleet_Pulse_Master.pbit

# Configure data source connection
# Power BI Desktop > Transform Data > Data Source Settings
# Update Server: your-sql-server.database.windows.net
# Database: LogiFleetPulse
# Authentication: SQL Server / Windows / Azure AD
```

## Key Configuration

### Cross-Fact KPI Measures (DAX)

Create calculated measures in Power BI for cross-functional analytics:

```dax
-- Combined warehouse-fleet efficiency
Fleet-Warehouse Efficiency = 
VAR TotalDwellHours = SUM(FactWarehouseOperations[DwellTimeHours])
VAR TotalIdleMinutes = SUM(FactFleetTrips[IdleTimeMinutes])
VAR TotalOperationTime = SUM(FactWarehouseOperations[CycleTimeMinutes])
VAR TotalTripTime = SUM(FactFleetTrips[DistanceKM]) / 60 -- Assuming 60 km/h average

RETURN
DIVIDE(
    TotalOperationTime + TotalTripTime,
    TotalOperationTime + TotalTripTime + TotalDwellHours * 60 + TotalIdleMinutes,
    0
) * 100

-- Gravity-weighted inventory cost
Inventory Carrying Cost by Gravity = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeHours] * 
    RELATED(DimProductGravity[GravityScore]) * 
    0.15 -- Daily carrying cost percentage
)

-- Predictive bottleneck index
Bottleneck Risk Score = 
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR StdDevDwellTime = STDEV.P(FactWarehouseOperations[DwellTimeHours])
VAR CurrentDwellTime = MAX(FactWarehouseOperations[DwellTimeHours])

VAR AvgDelayMinutes = AVERAGE(FactFleetTrips[DelayMinutes])
VAR StdDevDelayMinutes = STDEV.P(FactFleetTrips[DelayMinutes])
VAR CurrentDelayMinutes = MAX(FactFleetTrips[DelayMinutes])

RETURN
(
    (CurrentDwellTime - AvgDwellTime) / NULLIF(StdDevDwellTime, 0) * 0.6 +
    (CurrentDelayMinutes - AvgDelayMinutes) / NULLIF(StdDevDelayMinutes, 0) * 0.4
) * 10
```

### Row-Level Security

```dax
-- Create role: Regional Manager
-- Table: DimGeography
-- Filter: [Region] = USERNAME()

-- Create role: Warehouse Supervisor
-- Table: DimGeography
-- Filter: [LocationName] IN (
--     SELECT LocationName FROM UserWarehouseAccess 
--     WHERE UserEmail = USERPRINCIPALNAME()
-- )
```

### Automated Alerting

```sql
-- Create alert monitoring procedure
CREATE PROCEDURE usp_MonitorKPIThresholds
AS
BEGIN
    DECLARE @AlertThreshold_IdleTimePercent DECIMAL(5,2) = 15.0;
    DECLARE @AlertThreshold_DwellTimeHours DECIMAL(8,2) = 72.0;
    
    -- Fleet idle time alerts
    INSERT INTO AlertQueue (AlertType, Severity, Message, VehicleID, GeneratedAt)
    SELECT 
        'HighIdleTime',
        CASE 
            WHEN (IdleTimeMinutes / NULLIF(DATEDIFF(MINUTE, tStart.DateTimeStamp, tEnd.DateTimeStamp), 0) * 100) > 25 
            THEN 'Critical'
            ELSE 'Warning'
        END,
        'Vehicle ' + VehicleID + ' had ' + 
        CAST(IdleTimeMinutes / NULLIF(DATEDIFF(MINUTE, tStart.DateTimeStamp, tEnd.DateTimeStamp), 0) * 100 AS VARCHAR(10)) + 
        '% idle time on trip',
        VehicleID,
        GETDATE()
    FROM FactFleetTrips ft
    INNER JOIN DimTime tStart ON ft.StartTimeKey = tStart.TimeKey
    INNER JOIN DimTime tEnd ON ft.EndTimeKey = tEnd.TimeKey
    WHERE (IdleTimeMinutes / NULLIF(DATEDIFF(MINUTE, tStart.DateTimeStamp, tEnd.DateTimeStamp), 0) * 100) > @AlertThreshold_IdleTimePercent
        AND ft.StartTimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime); -- Last 24 hours
    
    -- Warehouse dwell time alerts
    INSERT INTO AlertQueue (AlertType, Severity, Message, ProductSKU, GeneratedAt)
    SELECT 
        'ExcessiveDwellTime',
        'Warning',
        'SKU ' + p.SKU + ' has been in zone ' + wo.ZoneID + ' for ' + 
        CAST(wo.DwellTimeHours AS VARCHAR(10)) + ' hours',
        p.SKU,
        GETDATE()
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.DwellTimeHours > @AlertThreshold_DwellTimeHours
        AND wo.TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime);
END;
GO

-- Schedule the procedure (SQL Server Agent Job)
-- Run every 15 minutes
```

## Common Usage Patterns

### Pattern 1: Warehouse Gravity Zone Optimization

```sql
-- Identify products that should be relocated
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZone AS RecommendedZone,
    wo.ZoneID AS CurrentZone,
    AVG(wo.CycleTimeMinutes) AS AvgCycleTime,
    AVG(wo.DwellTimeHours) AS AvgDwellTime,
    COUNT(*) AS PickFrequency
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.DateTimeStamp >= DATEADD(DAY, -30, GETDATE())
    AND wo.OperationType = 'Picking'
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.OptimalZone, wo.ZoneID
HAVING p.OptimalZone <> wo.ZoneID
    AND COUNT(*) > 10
ORDER BY p.GravityScore DESC, PickFrequency DESC;
```

### Pattern 2: Cross-Fact Fleet-Warehouse Correlation

```sql
-- Find correlations between warehouse delays and fleet delays
WITH WarehouseDelays AS (
    SELECT 
        wo.GeographyKey,
        t.DateTimeStamp,
        AVG(wo.DwellTimeHours) AS AvgDwellTime
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.DateTimeStamp >= DATEADD(DAY, -7, GETDATE())
    GROUP BY wo.GeographyKey, t.DateTimeStamp
),
FleetDelays AS (
    SELECT 
        ft.OriginGeographyKey AS GeographyKey,
        t.DateTimeStamp,
        AVG(ft.DelayMinutes) AS AvgDelayMinutes
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
    WHERE t.DateTimeStamp >= DATEADD(DAY, -7, GETDATE())
    GROUP BY ft.OriginGeographyKey, t.DateTimeStamp
)
SELECT 
    g.LocationName,
    AVG(wd.AvgDwellTime) AS WarehouseAvgDwellHours,
    AVG(fd.AvgDelayMinutes) AS FleetAvgDelayMinutes,
    CORR(wd.AvgDwellTime, fd.AvgDelayMinutes) OVER (PARTITION BY g.GeographyKey) AS Correlation
FROM WarehouseDelays wd
INNER JOIN FleetDelays fd ON wd.GeographyKey = fd.GeographyKey 
    AND CAST(wd.DateTimeStamp AS DATE) = CAST(fd.DateTimeStamp AS DATE)
INNER JOIN DimGeography g ON wd.GeographyKey = g.GeographyKey
GROUP BY g.GeographyKey, g.LocationName;
```

### Pattern 3: Temporal Elasticity Scenario Modeling

```sql
-- Simulate impact of capacity increase on fleet utilization
WITH CurrentState AS (
    SELECT 
        AVG(wo.QuantityProcessed) AS AvgQuantity,
        AVG(ft.DistanceKM) AS AvgDistance,
        AVG(ft.FuelLiters) AS AvgFuel
    FROM FactWarehouseOperations wo
    CROSS JOIN FactFleetTrips ft
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.DateTimeStamp >= DATEADD(DAY, -30, GETDATE())
),
SimulatedState AS (
    SELECT 
        AvgQuantity * 1.25 AS SimulatedQuantity, -- 25% increase
        AvgDistance * 1.15 AS SimulatedDistance, -- Estimated 15% increase in trips
        AvgFuel * 1.18 AS SimulatedFuel -- Estimated 18% increase in fuel
    FROM CurrentState
)
SELECT 
    'Current' AS Scenario,
    AvgQuantity AS Quantity,
    AvgDistance AS Distance,
    AvgFuel AS Fuel
FROM CurrentState
UNION ALL
SELECT 
    'Simulated +25% Capacity',
    SimulatedQuantity,
    SimulatedDistance,
    SimulatedFuel
FROM SimulatedState;
```

### Pattern 4: Real-Time Dashboard Refresh

```sql
-- Create view for Power BI real-time streaming
CREATE VIEW vw_RealTimeFleetStatus AS
SELECT 
    ft.VehicleID,
    ft.DriverID,
    g.LocationName AS CurrentLocation,
    t.DateTimeStamp AS LastUpdate,
    ft.IdleTimeMinutes,
    ft.DelayMinutes,
    CASE 
        WHEN ft.IdleTimeMinutes > 30 THEN 'Alert'
        WHEN ft.DelayMinutes > 15 THEN 'Warning'
        ELSE 'Normal'
    END AS Status
FROM FactFleetTrips ft
INNER JOIN DimTime t ON ft.EndTimeKey = t.TimeKey
INNER JOIN DimGeography g ON ft.DestinationGeographyKey = g.GeographyKey
WHERE t.TimeKey >= (SELECT MAX(TimeKey) - 4 FROM DimTime) -- Last hour
ORDER BY t.DateTimeStamp DESC;
GO
```

## Advanced Features

### Predictive Maintenance Queue

```sql
-- Fleet triage based on maintenance priority
CREATE VIEW vw_FleetMaintenancePriority AS
WITH FleetMetrics AS (
    SELECT 
        VehicleID,
        AVG(FuelLiters / NULLIF(DistanceKM, 0)) AS AvgFuelEfficiency,
        AVG(IdleTimeMinutes) AS AvgIdleTime,
        COUNT(*) AS TripCount
    FROM FactFleetTrips
    WHERE StartTimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime) -- Last week
    GROUP BY VehicleID
),
LoadMetrics AS (
    SELECT 
        ft.VehicleID,
        AVG(p.GravityScore * wo.QuantityProcessed) AS AvgHighValueLoad
    FROM FactFleetTrips ft
    INNER JOIN FactWarehouseOperations wo ON ft.StartTimeKey = wo.TimeKey
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    GROUP BY ft.VehicleID
)
SELECT 
    fm.VehicleID,
    fm.AvgFuelEfficiency,
    fm.AvgIdleTime,
    lm.AvgHighValueLoad,
    (
        (1.0 / NULLIF(fm.AvgFuelEfficiency, 0)) * 0.3 +
        (fm.AvgIdleTime / 60.0) * 0.3 +
        (lm.AvgHighValueLoad / 1000.0) * 0.4
    ) AS MaintenancePriorityScore
FROM FleetMetrics fm
LEFT JOIN LoadMetrics lm ON fm.VehicleID = lm.VehicleID
ORDER BY MaintenancePriorityScore DESC;
GO
```

### Multi-Language Dashboard Setup

```dax
-- Create language selection parameter
-- Power BI > Modeling > New Parameter
-- Name: SelectedLanguage
-- Values: EN, ES, FR, DE, ZH

-- Create translated measure
Dashboard Title = 
SWITCH(
    SELECTEDVALUE(SelectedLanguage[Language]),
    "ES", "Panel de Control Logístico",
    "FR", "Tableau de Bord Logistique",
    "DE", "Logistik-Dashboard",
    "ZH", "物流仪表板",
    "Logistics Dashboard" -- Default EN
)
```

## Troubleshooting

### Issue: Slow dashboard refresh times

**Solution**: Verify indexing and partitioning

```sql
-- Check missing indexes
SELECT 
    t.name AS TableName,
    dm_mid.equality_columns,
    dm_mid.inequality_columns,
    dm_mid.included_columns
FROM sys.dm_db_missing_index_details dm_mid
INNER JOIN sys.tables t ON dm_mid.object_id = t.object_id
WHERE dm_mid.database_id = DB_ID('LogiFleetPulse');

-- Partition fact tables by time
ALTER TABLE FactFleetTrips
ADD CONSTRAINT PK_FactFleetTrips_Partitioned 
PRIMARY KEY (TripKey, StartTimeKey);

CREATE PARTITION SCHEME PS_ByMonth
AS PARTITION PF_ByMonth
TO ([PRIMARY], [FG_Q1_2026], [FG_Q2_2026], [FG_Q3_2026], [FG_Q4_2026]);
```

### Issue: ETL procedures timing out

**Solution**: Implement incremental loading with checkpoints

```sql
-- Create checkpoint table
CREATE TABLE ETLCheckpoints (
    ProcessName VARCHAR(100) PRIMARY KEY,
    LastSuccessfulRun DATETIME2,
    LastProcessedID BIGINT
);

-- Modified incremental load
CREATE PROCEDURE usp_LoadWarehouseOperations_Incremental
AS
BEGIN
    DECLARE @LastRun DATETIME2;
    DECLARE @LastID BIGINT;
    
    SELECT @LastRun = LastSuccessfulRun, @LastID = LastProcessedID
    FROM ETLCheckpoints
    WHERE ProcessName = 'LoadWarehouseOperations';
    
    -- Process in batches
    DECLARE @BatchSize INT = 10000;
    DECLARE @ProcessedCount INT = 0;
    
    WHILE @ProcessedCount < (SELECT COUNT(*) FROM StagingWarehouseOps WHERE LastModified > @LastRun)
    BEGIN
        INSERT INTO FactWarehouseOperations (...)
        SELECT TOP (@BatchSize) ...
        FROM StagingWarehouseOps
        WHERE LastModified > @LastRun
            AND OperationID > @LastID
        ORDER BY OperationID;
        
        SET @ProcessedCount += @@ROWCOUNT;
        SET @LastID = (SELECT MAX(OperationID) FROM FactWarehouseOperations);
        
        -- Update checkpoint
        UPDATE ETLCheckpoints
        SET LastProcessedID = @LastID
        WHERE ProcessName = 'LoadWarehouseOperations';
    END;
    
    -- Final checkpoint update
    UPDATE ETLCheckpoints
    SET LastSuccessfulRun = GETDATE()
    WHERE ProcessName = 'LoadWarehouseOperations';
END;
```

### Issue: Power BI connection authentication failures

**Solution**: Use Azure AD authentication with service principal

```powershell
# Configure service principal in Azure AD
az ad sp create-for-rbac --name "LogiFleetPulsePowerBI" \
    --role Contributor \
    --scopes /subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RESOURCE_GROUP}

# Store credentials in environment variables
# AZURE_CLIENT_ID
# AZURE_CLIENT_SECRET
# AZURE_TENANT_ID

# Update Power BI connection string
# Data Source=your-server.database.windows.net;
# Initial Catalog=LogiFleetPulse;
# Authentication=Active Directory Service Principal;
# User ID=${AZURE_CLIENT_ID};
# Password=${AZURE_CLIENT_SECRET}
```

### Issue: Gravity score calculations producing unrealistic values

**Solution**: Add boundary constraints and normalization

```sql
-- Update gravity score calculation with normalization
UPDATE DimProductGravity
SET GravityScore = 
    CASE 
        WHEN RawScore > 100 THEN 100
        WHEN RawScore < 0 THEN 0
        ELSE RawScore
    END
FROM (
    SELECT 
        ProductKey,
        (
            (VelocityScore / NULLIF(MaxVelocity, 0) * 40) +
            (ValueScore / NULLIF(MaxValue, 0) * 35) +
            ((1 - FragilityScore) * 25)
        ) AS RawScore
    FROM (
        SELECT 
            p.ProductKey,
            COALESCE(SUM(wo.QuantityProcessed) / 30.0, 0) AS VelocityScore,
            p.ValueClass AS ValueScore,
            p.FragilityScore,
            MAX(SUM(wo.QuantityProcessed) / 30.0) OVER () AS MaxVelocity,
            MAX(p.ValueClass) OVER () AS MaxValue
        FROM DimProductGravity p
        LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
        GROUP BY p.ProductKey, p.ValueClass, p.FragilityScore
    ) Scores
) Normalized
WHERE DimProductGravity.ProductKey = Normalized.ProductKey;
```

## Best Practices

1. **Run ETL during off-peak hours**: Schedule usp_LoadWarehouseOperations and usp_LoadFleetTrips during low-
