---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing and analytics platform for logistics, fleet management, and supply chain intelligence
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "deploy MS SQL supply chain data warehouse"
  - "create fleet management analytics schema"
  - "build warehouse operations KPI dashboard"
  - "implement cross-modal logistics intelligence"
  - "query fleet telemetry and warehouse data"
  - "configure supply chain anomaly detection"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to assist with **LogiFleet Pulse**, an advanced MS SQL Server and Power BI data warehousing platform for supply chain analytics. It combines warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer for real-time logistics intelligence.

## What LogiFleet Pulse Does

LogiFleet Pulse is a multi-modal logistics intelligence engine that:

- **Unifies data sources**: Warehouse Management Systems (WMS), fleet telematics/GPS, supplier portals, weather/traffic APIs, and customer order history
- **Multi-fact star schema**: Custom data model with time-phased dimensions for cross-fact KPI harmonization
- **Real-time dashboards**: Power BI visualizations refreshed every 15 minutes
- **Predictive analytics**: Bottleneck detection, adaptive fleet triage, warehouse gravity zone optimization
- **Temporal modeling**: Time-phased simulation scenarios for capacity planning
- **Role-based access**: Row-level security for different user types

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (recommended for polybase support)
- Power BI Desktop (latest version)
- Access to source systems: WMS, telematics APIs, ERP feeds

### Step 1: Deploy SQL Schema

```sql
-- Deploy the complete schema (tables, views, stored procedures)
-- Run this on your MS SQL Server instance

-- Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes INT,
    QuantityHandled DECIMAL(18,2),
    OperationStartTime DATETIME2,
    OperationEndTime DATETIME2,
    EmployeeID INT,
    ZoneGravityScore DECIMAL(5,2),
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_WH_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
);

CREATE TABLE FactFleetTrips (
    TripID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginWarehouseKey INT,
    DestinationKey INT,
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightKG DECIMAL(12,2),
    AverageSpeedKMH DECIMAL(6,2),
    DelayMinutes INT,
    DelayReason VARCHAR(100),
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_Fleet_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE TABLE FactCrossDock (
    CrossDockID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripID BIGINT,
    OutboundTripID BIGINT,
    DockTransferMinutes INT,
    QuantityTransferred DECIMAL(18,2),
    CrossDockZone VARCHAR(20),
    CONSTRAINT FK_CD_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CD_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    DayOfMonth INT,
    DayOfWeek INT,
    HourOfDay INT,
    MinuteBucket INT, -- 15-minute granularity: 0, 15, 30, 45
    IsWeekend BIT,
    IsHoliday BIT,
    FiscalPeriod VARCHAR(10)
);

CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    ProductSKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    IsFragile BIT,
    RequiresRefrigeration BIT,
    UnitWeightKG DECIMAL(10,4),
    UnitVolumeCM3 DECIMAL(12,2),
    GravityScore DECIMAL(5,2), -- Warehouse Gravity Zones™ metric
    VelocityClassification VARCHAR(20) -- 'Fast-Mover', 'Medium-Mover', 'Slow-Mover'
);

CREATE TABLE DimWarehouse (
    WarehouseKey INT IDENTITY(1,1) PRIMARY KEY,
    WarehouseCode VARCHAR(20) NOT NULL UNIQUE,
    WarehouseName VARCHAR(200),
    GeographyKey INT,
    CapacityPallets INT,
    HasRefrigeration BIT,
    HasCrossDock BIT,
    CONSTRAINT FK_WH_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE TABLE DimVehicle (
    VehicleKey INT IDENTITY(1,1) PRIMARY KEY,
    VehicleRegistration VARCHAR(20) NOT NULL UNIQUE,
    VehicleType VARCHAR(50), -- 'Van', 'Truck', 'Semi', 'Refrigerated'
    FuelType VARCHAR(20),
    MaxLoadCapacityKG DECIMAL(10,2),
    LastMaintenanceDate DATE,
    TriagePriorityScore DECIMAL(5,2) -- Adaptive Fleet Triage Engine metric
);

CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    Continent VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    City VARCHAR(100),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7)
);

CREATE TABLE DimRoute (
    RouteKey INT IDENTITY(1,1) PRIMARY KEY,
    RouteCode VARCHAR(50) NOT NULL UNIQUE,
    RouteName VARCHAR(200),
    PlannedDistanceKM DECIMAL(10,2),
    AverageTrafficDelay DECIMAL(6,2),
    WeatherRiskFactor DECIMAL(3,2)
);

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWH_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWH_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleKey);
CREATE COLUMNSTORE INDEX CCI_FactWH ON FactWarehouseOperations(TimeKey, ProductKey, WarehouseKey, DwellTimeMinutes, QuantityHandled);
CREATE COLUMNSTORE INDEX CCI_FactFleet ON FactFleetTrips(TimeKey, VehicleKey, RouteKey, FuelConsumedLiters, IdleTimeMinutes);
```

### Step 2: Create Data Loading Stored Procedures

```sql
-- Incremental loading procedure for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LoadDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from external staging table or API feed
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType, 
        DwellTimeMinutes, QuantityHandled, OperationStartTime, 
        OperationEndTime, EmployeeID, ZoneGravityScore
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        w.WarehouseKey,
        s.OperationType,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
        s.Quantity,
        s.StartTime,
        s.EndTime,
        s.EmployeeID,
        p.GravityScore
    FROM StagingWarehouseOps s
    INNER JOIN DimTime t ON CAST(s.StartTime AS DATE) = CAST(t.FullDateTime AS DATE)
        AND DATEPART(HOUR, s.StartTime) = t.HourOfDay
        AND (DATEPART(MINUTE, s.StartTime) / 15) * 15 = t.MinuteBucket
    INNER JOIN DimProduct p ON s.SKU = p.ProductSKU
    INNER JOIN DimWarehouse w ON s.WarehouseCode = w.WarehouseCode
    WHERE CAST(s.StartTime AS DATE) = @LoadDate;
    
    PRINT 'Loaded ' + CAST(@@ROWCOUNT AS VARCHAR(10)) + ' warehouse operations';
END;
GO

-- Fleet telemetry loading procedure
CREATE PROCEDURE usp_LoadFleetTrips
    @LoadDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        TimeKey, VehicleKey, RouteKey, OriginWarehouseKey,
        DestinationKey, TripStartTime, TripEndTime, DistanceKM,
        FuelConsumedLiters, IdleTimeMinutes, LoadWeightKG,
        AverageSpeedKMH, DelayMinutes, DelayReason
    )
    SELECT 
        t.TimeKey,
        v.VehicleKey,
        r.RouteKey,
        ow.WarehouseKey AS OriginWarehouseKey,
        dg.GeographyKey AS DestinationKey,
        s.TripStart,
        s.TripEnd,
        s.Distance,
        s.FuelUsed,
        s.IdleTime,
        s.LoadWeight,
        s.AvgSpeed,
        s.DelayMinutes,
        s.DelayReason
    FROM StagingFleetTelemetry s
    INNER JOIN DimTime t ON CAST(s.TripStart AS DATE) = CAST(t.FullDateTime AS DATE)
        AND DATEPART(HOUR, s.TripStart) = t.HourOfDay
        AND (DATEPART(MINUTE, s.TripStart) / 15) * 15 = t.MinuteBucket
    INNER JOIN DimVehicle v ON s.VehicleReg = v.VehicleRegistration
    INNER JOIN DimRoute r ON s.RouteCode = r.RouteCode
    LEFT JOIN DimWarehouse ow ON s.OriginWarehouseCode = ow.WarehouseCode
    LEFT JOIN DimGeography dg ON s.DestinationCity = dg.City
    WHERE CAST(s.TripStart AS DATE) = @LoadDate;
    
    PRINT 'Loaded ' + CAST(@@ROWCOUNT AS VARCHAR(10)) + ' fleet trips';
END;
GO
```

### Step 3: Create Cross-Fact KPI Views

```sql
-- View: Unified warehouse and fleet performance
CREATE VIEW vw_UnifiedLogisticsKPI AS
SELECT 
    t.Year,
    t.Month,
    t.Week,
    p.Category AS ProductCategory,
    w.WarehouseName,
    
    -- Warehouse metrics
    COUNT(DISTINCT wo.OperationID) AS TotalOperations,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
    SUM(wo.QuantityHandled) AS TotalUnitsHandled,
    AVG(wo.ZoneGravityScore) AS AvgGravityScore,
    
    -- Fleet metrics (trips originating from this warehouse)
    COUNT(DISTINCT ft.TripID) AS TotalTripsFromWarehouse,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    AVG(ft.FuelConsumedLiters) AS AvgFuelPerTrip,
    SUM(ft.DelayMinutes) AS TotalDelayMinutes,
    
    -- Cross-fact KPI: Dwell time impact on fleet efficiency
    CASE 
        WHEN AVG(wo.DwellTimeMinutes) > 60 THEN 'High-Dwell-Impact'
        WHEN AVG(wo.DwellTimeMinutes) BETWEEN 30 AND 60 THEN 'Medium-Dwell-Impact'
        ELSE 'Low-Dwell-Impact'
    END AS DwellImpactCategory,
    
    -- Composite efficiency score
    (100.0 - (AVG(wo.DwellTimeMinutes) / 240.0 * 100)) * 
    (100.0 - (AVG(ft.IdleTimeMinutes) / AVG(DATEDIFF(MINUTE, ft.TripStartTime, ft.TripEndTime)) * 100)) / 100.0 
    AS CompositeEfficiencyScore

FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
INNER JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
LEFT JOIN FactFleetTrips ft ON ft.OriginWarehouseKey = w.WarehouseKey 
    AND ft.TimeKey = t.TimeKey

GROUP BY 
    t.Year, t.Month, t.Week,
    p.Category, w.WarehouseName;
GO

-- View: Predictive bottleneck index
CREATE VIEW vw_PredictiveBottleneckIndex AS
SELECT 
    t.FullDateTime,
    w.WarehouseName,
    p.Category,
    
    -- Warehouse congestion indicators
    COUNT(wo.OperationID) AS OperationsPerHour,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    
    -- Fleet pressure indicators
    COUNT(ft.TripID) AS ScheduledTrips,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    
    -- Bottleneck risk score (0-100, higher = more risk)
    (
        (COUNT(wo.OperationID) / NULLIF(w.CapacityPallets, 0) * 100.0) * 0.4 +
        (AVG(wo.DwellTimeMinutes) / 240.0 * 100.0) * 0.3 +
        (AVG(ft.IdleTimeMinutes) / 60.0 * 100.0) * 0.3
    ) AS BottleneckRiskScore,
    
    CASE 
        WHEN (COUNT(wo.OperationID) / NULLIF(w.CapacityPallets, 0) * 100.0) > 85 THEN 'CRITICAL'
        WHEN (COUNT(wo.OperationID) / NULLIF(w.CapacityPallets, 0) * 100.0) > 70 THEN 'HIGH'
        WHEN (COUNT(wo.OperationID) / NULLIF(w.CapacityPallets, 0) * 100.0) > 50 THEN 'MEDIUM'
        ELSE 'LOW'
    END AS AlertLevel

FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips ft ON ft.OriginWarehouseKey = w.WarehouseKey 
    AND ft.TimeKey = t.TimeKey

GROUP BY 
    t.FullDateTime, w.WarehouseName, w.CapacityPallets, p.Category

HAVING COUNT(wo.OperationID) > 0;
GO
```

### Step 4: Configure Automated Alerts

```sql
-- Stored procedure for automated alerting
CREATE PROCEDURE usp_CheckLogisticsAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check for high bottleneck risk
    IF EXISTS (
        SELECT 1 FROM vw_PredictiveBottleneckIndex 
        WHERE AlertLevel IN ('CRITICAL', 'HIGH')
        AND FullDateTime >= DATEADD(HOUR, -1, GETDATE())
    )
    BEGIN
        SELECT @AlertMessage = STRING_AGG(
            CONCAT(
                'Warehouse: ', WarehouseName, 
                ' | Category: ', Category,
                ' | Risk Score: ', CAST(BottleneckRiskScore AS VARCHAR(10)),
                ' | Alert Level: ', AlertLevel
            ), CHAR(13) + CHAR(10)
        )
        FROM vw_PredictiveBottleneckIndex
        WHERE AlertLevel IN ('CRITICAL', 'HIGH')
        AND FullDateTime >= DATEADD(HOUR, -1, GETDATE());
        
        -- Send alert (integrate with email/Teams/SMS via sp_send_dbmail or external API)
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'logistics-ops@company.com',
            @subject = 'LOGIFLEET ALERT: Bottleneck Risk Detected',
            @body = @AlertMessage;
    END;
    
    -- Check for excessive fleet idle time
    IF EXISTS (
        SELECT 1 FROM FactFleetTrips
        WHERE TimeKey IN (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(HOUR, -2, GETDATE()))
        AND (IdleTimeMinutes * 1.0 / DATEDIFF(MINUTE, TripStartTime, TripEndTime)) > 0.15
    )
    BEGIN
        SELECT @AlertMessage = STRING_AGG(
            CONCAT(
                'Vehicle: ', v.VehicleRegistration,
                ' | Route: ', r.RouteName,
                ' | Idle %: ', CAST((ft.IdleTimeMinutes * 100.0 / DATEDIFF(MINUTE, ft.TripStartTime, ft.TripEndTime)) AS VARCHAR(10))
            ), CHAR(13) + CHAR(10)
        )
        FROM FactFleetTrips ft
        INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
        INNER JOIN DimRoute r ON ft.RouteKey = r.RouteKey
        WHERE TimeKey IN (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(HOUR, -2, GETDATE()))
        AND (ft.IdleTimeMinutes * 1.0 / DATEDIFF(MINUTE, ft.TripStartTime, ft.TripEndTime)) > 0.15;
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'fleet-managers@company.com',
            @subject = 'LOGIFLEET ALERT: Excessive Fleet Idle Time',
            @body = @AlertMessage;
    END;
END;
GO

-- Schedule the alert procedure (run every 15 minutes via SQL Agent Job)
-- This is an example job creation script
USE msdb;
GO

EXEC dbo.sp_add_job
    @job_name = N'LogiFleet_AlertMonitoring';

EXEC dbo.sp_add_jobstep
    @job_name = N'LogiFleet_AlertMonitoring',
    @step_name = N'Check Alerts',
    @subsystem = N'TSQL',
    @command = N'EXEC LogiFleetPulse.dbo.usp_CheckLogisticsAlerts;',
    @retry_attempts = 3,
    @retry_interval = 5;

EXEC dbo.sp_add_schedule
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC dbo.sp_attach_schedule
    @job_name = N'LogiFleet_AlertMonitoring',
    @schedule_name = N'Every15Minutes';

EXEC dbo.sp_add_jobserver
    @job_name = N'LogiFleet_AlertMonitoring';
GO
```

### Step 5: Configure Power BI Template

Create a `config_sample.json` for connection strings:

```json
{
  "sql_server": {
    "server": "YOUR_SQL_SERVER_NAME",
    "database": "LogiFleetPulse",
    "authentication": "Windows",
    "connection_timeout": 30
  },
  "refresh_schedule": {
    "frequency_minutes": 15,
    "incremental_refresh": true
  },
  "external_apis": {
    "weather_api": {
      "endpoint": "https://api.weather.com/v3",
      "api_key_env": "WEATHER_API_KEY"
    },
    "traffic_api": {
      "endpoint": "https://api.traffic.com/live",
      "api_key_env": "TRAFFIC_API_KEY"
    }
  },
  "row_level_security": {
    "enabled": true,
    "user_table": "DimUsers",
    "role_column": "UserRole"
  }
}
```

**Power BI DAX Measures** (add these to your Power BI model):

```dax
// Cross-Fact KPI: Dwell Time vs Fleet Idle Correlation
DwellVsIdleCorrelation = 
VAR DwellAvg = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR IdleAvg = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR DwellStdDev = STDEV.P(FactWarehouseOperations[DwellTimeMinutes])
VAR IdleStdDev = STDEV.P(FactFleetTrips[IdleTimeMinutes])
RETURN
    DIVIDE(
        SUMX(
            FILTER(
                CROSSJOIN(FactWarehouseOperations, FactFleetTrips),
                FactWarehouseOperations[TimeKey] = FactFleetTrips[TimeKey]
            ),
            (FactWarehouseOperations[DwellTimeMinutes] - DwellAvg) * 
            (FactFleetTrips[IdleTimeMinutes] - IdleAvg)
        ),
        COUNTROWS(FactWarehouseOperations) * DwellStdDev * IdleStdDev,
        0
    )

// Warehouse Gravity Zone Efficiency
GravityZoneEfficiency = 
DIVIDE(
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[QuantityHandled] * 
        RELATED(DimProduct[GravityScore])
    ),
    SUM(FactWarehouseOperations[DwellTimeMinutes]),
    0
)

// Fleet Triage Priority Score
FleetTriagePriority = 
SUMX(
    FactFleetTrips,
    RELATED(DimVehicle[TriagePriorityScore]) * 
    (FactFleetTrips[DelayMinutes] + FactFleetTrips[IdleTimeMinutes])
)

// Predictive Bottleneck Index (Real-Time)
BottleneckIndex = 
VAR CapacityUtilization = 
    DIVIDE(
        COUNTROWS(FactWarehouseOperations),
        MAX(DimWarehouse[CapacityPallets]),
        0
    ) * 100
VAR DwellPressure = 
    DIVIDE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        240,
        0
    ) * 100
VAR FleetPressure = 
    DIVIDE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes]),
        60,
        0
    ) * 100
RETURN
    (CapacityUtilization * 0.4) + (DwellPressure * 0.3) + (FleetPressure * 0.3)

// Temporal Elasticity Simulation
TemporalElasticityImpact = 
VAR BaselineCapacity = 0.80
VAR SimulatedCapacity = 0.95
VAR HistoricalUtilization = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
RETURN
    HistoricalUtilization * (SimulatedCapacity / BaselineCapacity)
```

## Key Commands & Queries

### Query Cross-Fact KPIs

```sql
-- Show shipments delayed due to weather from cold storage with >72hr dwell
SELECT 
    ft.TripID,
    w.WarehouseName,
    p.ProductName,
    wo.DwellTimeMinutes,
    ft.DelayMinutes,
    ft.DelayReason,
    t.FullDateTime
FROM FactFleetTrips ft
INNER JOIN FactWarehouseOperations wo 
    ON ft.OriginWarehouseKey = wo.WarehouseKey
    AND ft.TimeKey = wo.TimeKey
INNER JOIN DimWarehouse w ON ft.OriginWarehouseKey = w.WarehouseKey
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
WHERE 
    ft.DelayReason LIKE '%weather%'
    AND w.HasRefrigeration = 1
    AND wo.DwellTimeMinutes > 4320 -- 72 hours
    AND t.FullDateTime >= DATEADD(QUARTER, -1, GETDATE());
```

### Calculate Fleet Utilization by Route

```sql
-- Fleet utilization efficiency by route
SELECT 
    r.RouteName,
    COUNT(ft.TripID) AS TotalTrips,
    AVG(ft.DistanceKM) AS AvgDistanceKM,
    AVG(ft.FuelConsumedLiters) AS AvgFuelLiters,
    AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
    AVG(ft.LoadWeightKG) AS AvgLoadKG,
    
    -- Utilization score
    100.0 - (AVG(ft.IdleTimeMinutes) * 100.0 / 
        AVG(DATEDIFF(MINUTE, ft.TripStartTime, ft.TripEndTime))) AS UtilizationScore,
    
    -- Fuel efficiency
    AVG(ft.DistanceKM) / NULLIF(AVG(ft.FuelConsumedLiters), 0) AS KMPerLiter

FROM FactFleetTrips ft
INNER JOIN DimRoute r ON ft.RouteKey = r.RouteKey
INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
WHERE t.FullDateTime >= DATEADD(MONTH, -1, GETDATE())
GROUP BY r.RouteName
ORDER BY UtilizationScore DESC;
```

### Warehouse Gravity Zone Analysis

```sql
-- Identify products that should be moved to different gravity zones
SELECT 
    p.ProductSKU,
    p.ProductName,
    p.VelocityClassification AS CurrentVelocity,
    p.GravityScore AS CurrentGravity,
    
    -- Actual performance
    COUNT(wo.OperationID) AS PickFrequencyLast30Days,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    
    -- Recommended gravity score based on actual velocity
    CASE 
        WHEN COUNT(wo.OperationID) > 100 THEN 8.5
        WHEN COUNT(wo.OperationID) > 50 THEN 6.5
        WHEN COUNT(wo.OperationID) > 20 THEN 4.5
        ELSE 2.0
    END AS RecommendedGravity,
    
    -- Gap analysis
    CASE 
        WHEN COUNT(wo.OperationID) > 100 THEN 'Fast-Mover'
        WHEN COUNT(wo.OperationID) > 50 THEN 'Medium-Mover'
        ELSE 'Slow-Mover'
    END AS RecommendedVelocity,
    
    CASE 
        WHEN ABS(p.GravityScore - 
            CASE 
                WHEN COUNT(wo.OperationID) > 100 THEN 8.5
                WHEN COUNT(wo.OperationID) > 50 THEN 6.5
                WHEN COUNT(wo.OperationID) > 20 THEN 4.5
                ELSE 2.0
            END) > 2.0 THEN 'RELOCATION_NEEDED'
        ELSE 'OPTIMAL'
    END AS RecommendedAction

FROM DimProduct p
INNER JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY 
    p.ProductSKU, p.ProductName, p.VelocityClassification, p.GravityScore
HAVING COUNT(wo.OperationID) > 0
ORDER BY 
    CASE 
        WHEN ABS(p.GravityScore - 
            CASE 
                WHEN COUNT(wo.OperationID) > 100 THEN 8.5
                WHEN COUNT(wo.OperationID) > 50 THEN 6.5
