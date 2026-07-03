---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server data warehouse and Power BI logistics intelligence platform for unified fleet, warehouse, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy logistics data warehouse with Power BI"
  - "configure multi-fact star schema for fleet operations"
  - "implement warehouse gravity zone optimization"
  - "create cross-modal supply chain dashboards"
  - "build fleet telemetry and inventory analytics"
  - "integrate WMS data with SQL Server warehouse"
  - "set up real-time logistics KPI monitoring"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced logistics intelligence platform combining MS SQL Server data warehousing with Power BI visualization. It unifies warehouse operations, fleet telemetry, inventory management, and external data feeds into a multi-fact star schema with time-phased dimensions. The system enables cross-fact KPI analysis, predictive bottleneck detection, and real-time operational awareness across the entire supply chain.

**Core capabilities:**
- Multi-fact star schema for warehouse, fleet, and cross-dock operations
- 15-minute granularity time-phased dimensions
- Warehouse Gravity Zones™ for spatial optimization
- Adaptive fleet triage engine with predictive maintenance
- Real-time Power BI dashboards with role-based access
- Cross-fact KPI harmonization (e.g., inventory turnover vs. fuel costs)

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance and create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the schema deployment script
-- This creates all dimension and fact tables with proper indexing
```

**Core dimension tables:**

```sql
-- DimTime: 15-minute granularity temporal dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    TimeStamp DATETIME NOT NULL,
    FifteenMinuteBucket INT,
    HourOfDay INT,
    DayOfWeek INT,
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT,
    IsHoliday BIT
);

-- Create clustered columnstore index for analytics
CREATE CLUSTERED COLUMNSTORE INDEX CCI_DimTime ON DimTime;

-- DimGeography: Hierarchical location data
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    Continent VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    WarehouseID VARCHAR(50),
    RouteNodeID VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);

-- DimProductGravity: Warehouse gravity zone assignments
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL,
    ProductName VARCHAR(255),
    Category VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Higher = faster moving, more valuable
    VelocityRank INT,
    FragilityIndex DECIMAL(3,2),
    OptimalZone VARCHAR(50)
);

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(100) NOT NULL,
    SupplierName VARCHAR(255),
    LeadTimeVarianceDays DECIMAL(5,2),
    DefectPercentage DECIMAL(5,4),
    ComplianceScore DECIMAL(3,2),
    LastAuditDate DATE
);
```

**Core fact tables:**

```sql
-- FactWarehouseOperations: Warehouse activity tracking
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    DwellTimeMinutes INT,
    PickRateUnitsPerHour DECIMAL(8,2),
    PackingTimeMinutes DECIMAL(8,2),
    StorageZone VARCHAR(50),
    OperatorID VARCHAR(50),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

-- FactFleetTrips: Vehicle telemetry and route data
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKeyOrigin INT NOT NULL,
    GeographyKeyDestination INT NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    RouteSegment VARCHAR(100),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DistanceKM DECIMAL(10,2),
    AverageSpeedKPH DECIMAL(5,2),
    WeatherCondition VARCHAR(50),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKeyOrigin) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (GeographyKeyDestination) REFERENCES DimGeography(GeographyKey)
);

-- FactCrossDock: Cross-dock transfer operations
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    TransferTimeMinutes INT,
    TemperatureCompliance BIT,
    QualityCheckPassed BIT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    FOREIGN KEY (SupplierKey) REFERENCES DimSupplierReliability(SupplierKey)
);

-- Create optimized indexes for common query patterns
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time_Product 
    ON FactWarehouseOperations(TimeKey, ProductKey) 
    INCLUDE (DwellTimeMinutes, StorageZone);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time_Vehicle 
    ON FactFleetTrips(TimeKey, VehicleID) 
    INCLUDE (FuelConsumedLiters, IdleTimeMinutes);
```

### Step 2: Configure Data Sources

Create a configuration file for data ingestion:

```sql
-- External table for WMS data ingestion (requires PolyBase)
CREATE EXTERNAL DATA SOURCE WMS_DataFeed
WITH (
    TYPE = RDBMS,
    LOCATION = 'sql_server_instance',
    DATABASE_NAME = 'WMS_Production',
    CREDENTIAL = WMS_Credential
);

-- Stored procedure for incremental warehouse data load
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        DwellTimeMinutes, PickRateUnitsPerHour, PackingTimeMinutes,
        StorageZone, OperatorID
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        wms.OperationType,
        DATEDIFF(MINUTE, wms.StartTime, wms.EndTime) AS DwellTimeMinutes,
        wms.UnitsProcessed / NULLIF(DATEDIFF(HOUR, wms.StartTime, wms.EndTime), 0) AS PickRateUnitsPerHour,
        wms.PackingDuration,
        wms.Zone,
        wms.OperatorID
    FROM ExternalWMSData wms
    INNER JOIN DimTime t ON CAST(wms.StartTime AS DATETIME) = t.TimeStamp
    INNER JOIN DimGeography g ON wms.WarehouseID = g.WarehouseID
    INNER JOIN DimProductGravity p ON wms.SKU = p.SKU
    WHERE wms.StartTime > @LastLoadTime;
    
    -- Update last load timestamp
    UPDATE ETLControl 
    SET LastLoadTime = GETDATE() 
    WHERE TableName = 'FactWarehouseOperations';
END;
GO

-- Stored procedure for fleet telemetry ingestion
CREATE PROCEDURE usp_LoadFleetTrips
    @LastLoadTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        TimeKey, GeographyKeyOrigin, GeographyKeyDestination,
        VehicleID, DriverID, RouteSegment,
        FuelConsumedLiters, IdleTimeMinutes, LoadingTimeMinutes,
        UnloadingTimeMinutes, DistanceKM, AverageSpeedKPH, WeatherCondition
    )
    SELECT 
        t.TimeKey,
        g_origin.GeographyKey,
        g_dest.GeographyKey,
        tel.VehicleID,
        tel.DriverID,
        tel.RouteSegment,
        tel.FuelUsed,
        tel.IdleMinutes,
        tel.LoadMinutes,
        tel.UnloadMinutes,
        tel.Distance,
        tel.AvgSpeed,
        weather.Condition
    FROM TelemetryFeed tel
    INNER JOIN DimTime t ON tel.TripStartTime = t.TimeStamp
    INNER JOIN DimGeography g_origin ON tel.OriginWarehouse = g_origin.WarehouseID
    INNER JOIN DimGeography g_dest ON tel.DestinationWarehouse = g_dest.WarehouseID
    LEFT JOIN WeatherAPI weather ON tel.TripStartTime = weather.ObservationTime 
        AND g_origin.GeographyKey = weather.LocationKey
    WHERE tel.TripStartTime > @LastLoadTime;
END;
GO
```

### Step 3: Import Power BI Template

1. Download `LogiFleet_Pulse_Master.pbit` from the repository
2. Open in Power BI Desktop
3. Enter SQL Server connection details when prompted
4. Configure refresh schedule (minimum 15 minutes for real-time dashboards)

**Power BI connection string format:**
```
Server=YOUR_SQL_SERVER;Database=LogiFleetPulse;Trusted_Connection=True;
```

For Azure SQL:
```
Server=YOUR_SERVER.database.windows.net;Database=LogiFleetPulse;Authentication=Active Directory Interactive;
```

## Key Analytical Queries

### Cross-Fact KPI: Dwell Time vs. Fleet Idle Cost

```sql
-- Correlate high-dwell products with downstream fleet idle time
WITH HighDwellProducts AS (
    SELECT 
        p.SKU,
        p.ProductName,
        AVG(w.DwellTimeMinutes) AS AvgDwellMinutes,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.TimeStamp >= DATEADD(DAY, -30, GETDATE())
        AND w.DwellTimeMinutes > 180 -- More than 3 hours
    GROUP BY p.SKU, p.ProductName
    HAVING COUNT(*) > 10
),
FleetImpact AS (
    SELECT 
        cd.ProductKey,
        SUM(f.IdleTimeMinutes) AS TotalFleetIdleMinutes,
        SUM(f.FuelConsumedLiters) AS TotalFuelWasted
    FROM FactCrossDock cd
    INNER JOIN FactFleetTrips f ON cd.OutboundTripKey = f.TripKey
    WHERE cd.TimeKey IN (
        SELECT TimeKey FROM DimTime 
        WHERE TimeStamp >= DATEADD(DAY, -30, GETDATE())
    )
    GROUP BY cd.ProductKey
)
SELECT 
    hdp.SKU,
    hdp.ProductName,
    hdp.AvgDwellMinutes,
    hdp.OperationCount,
    fi.TotalFleetIdleMinutes,
    fi.TotalFuelWasted,
    -- Cost calculation (example: $2.50/liter fuel, $50/hour idle labor)
    (fi.TotalFuelWasted * 2.50) + ((fi.TotalFleetIdleMinutes / 60.0) * 50) AS EstimatedCostImpact
FROM HighDwellProducts hdp
INNER JOIN DimProductGravity p ON hdp.SKU = p.SKU
LEFT JOIN FleetImpact fi ON p.ProductKey = fi.ProductKey
ORDER BY EstimatedCostImpact DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products in wrong gravity zones based on velocity
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZone AS CurrentOptimalZone,
    w.StorageZone AS ActualZone,
    COUNT(*) AS PickOperations,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    AVG(w.PickRateUnitsPerHour) AS AvgPickRate,
    CASE 
        WHEN p.GravityScore > 80 AND w.StorageZone NOT LIKE 'Zone-A%' THEN 'RELOCATE_TO_HIGH_GRAVITY'
        WHEN p.GravityScore < 30 AND w.StorageZone LIKE 'Zone-A%' THEN 'RELOCATE_TO_LOW_GRAVITY'
        ELSE 'OPTIMAL'
    END AS RecommendedAction
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE t.TimeStamp >= DATEADD(DAY, -90, GETDATE())
    AND w.OperationType IN ('Picking', 'Putaway')
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.OptimalZone, w.StorageZone
HAVING COUNT(*) > 20
ORDER BY 
    CASE 
        WHEN p.GravityScore > 80 AND w.StorageZone NOT LIKE 'Zone-A%' THEN 1
        WHEN p.GravityScore < 30 AND w.StorageZone LIKE 'Zone-A%' THEN 2
        ELSE 3
    END,
    p.GravityScore DESC;
```

### Adaptive Fleet Triage - Predictive Maintenance Queue

```sql
-- Rank vehicles by maintenance urgency based on telemetry and cargo value
WITH VehicleHealth AS (
    SELECT 
        f.VehicleID,
        COUNT(*) AS TripCount,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        SUM(f.DistanceKM) AS TotalDistance,
        AVG(f.AverageSpeedKPH) AS AvgSpeed,
        -- Telemetry risk score (example: high idle + low speed = concern)
        CASE 
            WHEN AVG(f.IdleTimeMinutes) > 30 THEN 2
            WHEN AVG(f.IdleTimeMinutes) > 15 THEN 1
            ELSE 0
        END AS IdleRiskScore,
        CASE 
            WHEN AVG(f.AverageSpeedKPH) < 40 THEN 2
            WHEN AVG(f.AverageSpeedKPH) < 60 THEN 1
            ELSE 0
        END AS SpeedRiskScore
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.TimeStamp >= DATEADD(DAY, -14, GETDATE())
    GROUP BY f.VehicleID
),
CargoValue AS (
    SELECT 
        f.VehicleID,
        AVG(p.GravityScore) AS AvgCargoGravity,
        COUNT(DISTINCT p.ProductKey) AS UniqueProductsHauled
    FROM FactFleetTrips f
    INNER JOIN FactCrossDock cd ON f.TripKey = cd.OutboundTripKey
    INNER JOIN DimProductGravity p ON cd.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.TimeStamp >= DATEADD(DAY, -14, GETDATE())
    GROUP BY f.VehicleID
)
SELECT 
    vh.VehicleID,
    vh.TripCount,
    vh.TotalDistance,
    vh.AvgIdleTime,
    vh.AvgSpeed,
    cv.AvgCargoGravity,
    cv.UniqueProductsHauled,
    -- Composite urgency score
    (vh.IdleRiskScore * 2 + vh.SpeedRiskScore * 3 + 
     (CASE WHEN cv.AvgCargoGravity > 70 THEN 5 ELSE 0 END)) AS MaintenanceUrgencyScore,
    CASE 
        WHEN (vh.IdleRiskScore * 2 + vh.SpeedRiskScore * 3 + 
              (CASE WHEN cv.AvgCargoGravity > 70 THEN 5 ELSE 0 END)) >= 10 THEN 'CRITICAL - Schedule Immediately'
        WHEN (vh.IdleRiskScore * 2 + vh.SpeedRiskScore * 3 + 
              (CASE WHEN cv.AvgCargoGravity > 70 THEN 5 ELSE 0 END)) >= 5 THEN 'MEDIUM - Schedule Within 48h'
        ELSE 'LOW - Routine Check'
    END AS MaintenancePriority
FROM VehicleHealth vh
LEFT JOIN CargoValue cv ON vh.VehicleID = cv.VehicleID
ORDER BY MaintenanceUrgencyScore DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify time windows and locations with emerging congestion patterns
WITH HourlyOperations AS (
    SELECT 
        t.HourOfDay,
        t.DayOfWeek,
        g.WarehouseID,
        w.StorageZone,
        COUNT(*) AS OperationCount,
        AVG(w.DwellTimeMinutes) AS AvgDwell,
        STDEV(w.DwellTimeMinutes) AS DwellVariance
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    WHERE t.TimeStamp >= DATEADD(DAY, -60, GETDATE())
    GROUP BY t.HourOfDay, t.DayOfWeek, g.WarehouseID, w.StorageZone
),
HistoricalBaseline AS (
    SELECT 
        HourOfDay,
        DayOfWeek,
        WarehouseID,
        AVG(AvgDwell) AS BaselineDwell,
        AVG(OperationCount) AS BaselineOperations
    FROM HourlyOperations
    GROUP BY HourOfDay, DayOfWeek, WarehouseID
)
SELECT 
    ho.HourOfDay,
    ho.DayOfWeek,
    ho.WarehouseID,
    ho.StorageZone,
    ho.OperationCount,
    ho.AvgDwell,
    hb.BaselineDwell,
    -- Anomaly detection: current dwell > 1.5x historical baseline
    CASE 
        WHEN ho.AvgDwell > (hb.BaselineDwell * 1.5) THEN 'BOTTLENECK_DETECTED'
        WHEN ho.AvgDwell > (hb.BaselineDwell * 1.2) THEN 'WATCH'
        ELSE 'NORMAL'
    END AS BottleneckStatus,
    -- Predicted impact in next shift
    CASE 
        WHEN ho.AvgDwell > (hb.BaselineDwell * 1.5) 
        THEN (ho.OperationCount * (ho.AvgDwell - hb.BaselineDwell)) / 60.0
        ELSE 0
    END AS PredictedDelayHours
FROM HourlyOperations ho
INNER JOIN HistoricalBaseline hb 
    ON ho.HourOfDay = hb.HourOfDay 
    AND ho.DayOfWeek = hb.DayOfWeek 
    AND ho.WarehouseID = hb.WarehouseID
WHERE ho.AvgDwell > (hb.BaselineDwell * 1.2) -- Only show potential issues
ORDER BY PredictedDelayHours DESC;
```

## Automated Alerting

### Set Up Threshold Alerts

```sql
-- Create alert configuration table
CREATE TABLE AlertThresholds (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName VARCHAR(100) NOT NULL,
    MetricType VARCHAR(50), -- 'DWELL_TIME', 'FLEET_IDLE', 'FUEL_EFFICIENCY', etc.
    ThresholdValue DECIMAL(10,2),
    ComparisonOperator VARCHAR(10), -- '>', '<', '>=', '<=', '='
    AlertLevel VARCHAR(20), -- 'INFO', 'WARNING', 'CRITICAL'
    NotificationEmail VARCHAR(255),
    IsActive BIT DEFAULT 1
);

-- Insert example thresholds
INSERT INTO AlertThresholds (AlertName, MetricType, ThresholdValue, ComparisonOperator, AlertLevel, NotificationEmail)
VALUES 
    ('High Dwell Time Alert', 'DWELL_TIME', 240, '>', 'WARNING', 'warehouse-ops@company.com'),
    ('Critical Fleet Idle', 'FLEET_IDLE', 20, '>', 'CRITICAL', 'fleet-manager@company.com'),
    ('Low Fuel Efficiency', 'FUEL_EFFICIENCY', 6.5, '<', 'WARNING', 'logistics@company.com');

-- Automated alert check procedure (run every 15 minutes via SQL Agent)
CREATE PROCEDURE usp_CheckAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Check dwell time alerts
    INSERT INTO AlertLog (AlertID, TimeTriggered, MetricValue, AffectedEntity)
    SELECT 
        a.AlertID,
        GETDATE(),
        AVG(w.DwellTimeMinutes),
        p.SKU
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    CROSS JOIN AlertThresholds a
    WHERE a.MetricType = 'DWELL_TIME'
        AND a.IsActive = 1
        AND t.TimeStamp >= DATEADD(MINUTE, -15, GETDATE())
    GROUP BY a.AlertID, p.SKU
    HAVING AVG(w.DwellTimeMinutes) > a.ThresholdValue;
    
    -- Check fleet idle alerts
    INSERT INTO AlertLog (AlertID, TimeTriggered, MetricValue, AffectedEntity)
    SELECT 
        a.AlertID,
        GETDATE(),
        (SUM(f.IdleTimeMinutes) * 100.0) / NULLIF(SUM(f.IdleTimeMinutes + DATEDIFF(MINUTE, 0, DATEADD(MINUTE, f.LoadingTimeMinutes + f.UnloadingTimeMinutes, 0))), 0),
        f.VehicleID
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    CROSS JOIN AlertThresholds a
    WHERE a.MetricType = 'FLEET_IDLE'
        AND a.IsActive = 1
        AND t.TimeStamp >= DATEADD(MINUTE, -15, GETDATE())
    GROUP BY a.AlertID, f.VehicleID, a.ThresholdValue
    HAVING (SUM(f.IdleTimeMinutes) * 100.0) / NULLIF(SUM(f.IdleTimeMinutes + DATEDIFF(MINUTE, 0, DATEADD(MINUTE, f.LoadingTimeMinutes + f.UnloadingTimeMinutes, 0))), 0) > a.ThresholdValue;
    
    -- Send notifications for new alerts
    -- (Integrate with Database Mail or external notification service)
END;
GO
```

## Power BI DAX Measures

### Cross-Fact KPI: Inventory-to-Fleet Cost Ratio

```dax
// Total Warehouse Dwell Cost (assuming $25/hour holding cost)
TotalDwellCost = 
SUMX(
    FactWarehouseOperations,
    (FactWarehouseOperations[DwellTimeMinutes] / 60) * 25
)

// Total Fleet Idle Cost (assuming $50/hour idle cost)
TotalFleetIdleCost = 
SUMX(
    FactFleetTrips,
    (FactFleetTrips[IdleTimeMinutes] / 60) * 50
)

// Combined Logistics Inefficiency Cost
TotalLogisticsFrictionCost = 
[TotalDwellCost] + [TotalFleetIdleCost]

// Efficiency Ratio (lower is better)
LogisticsEfficiencyRatio = 
DIVIDE(
    [TotalLogisticsFrictionCost],
    SUMX(FactFleetTrips, FactFleetTrips[DistanceKM]),
    0
)
```

### Gravity Zone Performance Score

```dax
// Weighted performance by gravity score
GravityZonePerformance = 
AVERAGEX(
    SUMMARIZE(
        FactWarehouseOperations,
        DimProductGravity[GravityScore],
        DimProductGravity[OptimalZone],
        FactWarehouseOperations[StorageZone]
    ),
    IF(
        DimProductGravity[OptimalZone] = FactWarehouseOperations[StorageZone],
        DimProductGravity[GravityScore], // Bonus for correct placement
        DimProductGravity[GravityScore] * 0.5 // Penalty for misplacement
    )
)
```

### Predictive Bottleneck Index

```dax
// Time-series anomaly detection using historical averages
BottleneckIndex = 
VAR CurrentDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR HistoricalDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATESINPERIOD(DimTime[TimeStamp], LASTDATE(DimTime[TimeStamp]), -30, DAY)
    )
VAR DwellDeviation = DIVIDE(CurrentDwell - HistoricalDwell, HistoricalDwell, 0)
RETURN
    IF(
        DwellDeviation > 0.5, "CRITICAL",
        IF(DwellDeviation > 0.2, "WARNING", "NORMAL")
    )
```

## Role-Based Security

### Implement Row-Level Security (RLS)

```sql
-- Create user roles table
CREATE TABLE UserRoles (
    UserID VARCHAR(100) PRIMARY KEY,
    RoleName VARCHAR(50), -- 'Warehouse_Supervisor', 'Fleet_Manager', 'Executive'
    GeographyFilter VARCHAR(100), -- Comma-separated warehouse IDs or 'ALL'
    CanViewCosts BIT DEFAULT 0
);

-- Insert example users
INSERT INTO UserRoles (UserID, RoleName, GeographyFilter, CanViewCosts)
VALUES 
    ('warehouse.ops@company.com', 'Warehouse_Supervisor', 'WH-001,WH-002', 0),
    ('fleet.manager@company.com', 'Fleet_Manager', 'ALL', 1),
    ('executive@company.com', 'Executive', 'ALL', 1);
```

**Power BI RLS configuration:**

1. In Power BI Desktop, go to Modeling > Manage Roles
2. Create roles matching `RoleName` values
3. Add DAX filter to DimGeography table:

```dax
[WarehouseID] IN 
{
    LOOKUPVALUE(
        UserRoles[GeographyFilter],
        UserRoles[UserID],
        USERPRINCIPALNAME()
    )
}
```

## Common Patterns & Best Practices

### Incremental Data Loading

```sql
-- Create ETL control table for tracking last load times
CREATE TABLE ETLControl (
    TableName VARCHAR(100) PRIMARY KEY,
    LastLoadTime DATETIME,
    LastLoadStatus VARCHAR(50),
    RowsLoaded INT
);

-- Incremental load pattern for all fact tables
DECLARE @LastLoad DATETIME;

SELECT @LastLoad = LastLoadTime 
FROM ETLControl 
WHERE TableName = 'FactWarehouseOperations';

-- Execute load
EXEC usp_LoadWarehouseOperations @LastLoadTime = @LastLoad;

-- Update control table
UPDATE ETLControl 
SET 
    LastLoadTime = GETDATE(),
    LastLoadStatus = 'SUCCESS',
    RowsLoaded = @@ROWCOUNT
WHERE TableName = 'FactWarehouseOperations';
```

### Temporal Elasticity Simulation

```sql
-- Scenario analysis: What-if warehouse capacity increases by 15%?
WITH CurrentCapacity AS (
    SELECT 
        g.WarehouseID,
        COUNT(DISTINCT w.StorageZone) AS CurrentZones,
        AVG(w.DwellTimeMinutes) AS AvgDwell,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations w
    INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    GROUP BY g.WarehouseID
),
SimulatedCapacity AS (
    SELECT 
        WarehouseID,
        CurrentZones * 1.15 AS SimulatedZones,
        AvgDwell * 0.88 AS
