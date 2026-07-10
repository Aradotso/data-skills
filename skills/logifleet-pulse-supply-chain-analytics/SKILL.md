---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server data warehouse and Power BI analytics engine for logistics, fleet management, and supply chain intelligence
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure warehouse and fleet logistics dashboard"
  - "deploy LogiCore supply chain data warehouse"
  - "create Power BI logistics intelligence reports"
  - "implement multi-fact star schema for logistics"
  - "build real-time fleet optimization dashboard"
  - "set up warehouse gravity zone analytics"
  - "configure cross-modal supply chain KPI tracking"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to help developers deploy and use LogiFleet Pulse, a comprehensive logistics intelligence platform built on MS SQL Server and Power BI. The system provides real-time analytics for warehouse operations, fleet management, and cross-modal supply chain orchestration using a multi-fact star schema architecture.

## What LogiFleet Pulse Does

LogiFleet Pulse is a data warehousing and business intelligence solution that:

- Integrates warehouse operations, fleet telemetry, inventory management, and external data sources
- Uses a multi-fact star schema with time-phased dimensions for sub-second cross-fact queries
- Provides real-time Power BI dashboards refreshed every 15 minutes
- Implements predictive bottleneck detection and adaptive fleet triage
- Offers warehouse gravity zone optimization based on pick frequency, value, and fragility
- Supports temporal elasticity modeling for supply chain scenario planning

## Installation and Setup

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version)
- Access to data sources: WMS, GPS/telematics, ERP systems

### Database Schema Deployment

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**

```sql
-- Create the LogiFleet database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the main schema script
-- This creates all dimension and fact tables
:r schema/01_create_dimensions.sql
:r schema/02_create_facts.sql
:r schema/03_create_relationships.sql
:r schema/04_create_views.sql
:r schema/05_create_stored_procedures.sql
```

3. **Configure data source connections:**

Create a `config.json` file with your connection details:

```json
{
  "connections": {
    "wms_api": {
      "url": "${WMS_API_URL}",
      "auth_token": "${WMS_AUTH_TOKEN}"
    },
    "fleet_telemetry": {
      "url": "${FLEET_API_URL}",
      "api_key": "${FLEET_API_KEY}"
    },
    "erp_database": {
      "server": "${ERP_SQL_SERVER}",
      "database": "${ERP_DATABASE}",
      "username": "${ERP_SQL_USER}",
      "password": "${ERP_SQL_PASSWORD}"
    }
  },
  "refresh_interval_minutes": 15,
  "alert_email": "${ALERT_EMAIL_ADDRESS}"
}
```

## Core Data Model

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTimeStamp DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayName VARCHAR(20) NOT NULL,
    WeekOfYear INT NOT NULL,
    MonthOfYear INT NOT NULL,
    MonthName VARCHAR(20) NOT NULL,
    QuarterOfYear INT NOT NULL,
    FiscalYear INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL,
    CONSTRAINT UQ_DimTime_DateTimeStamp UNIQUE (DateTimeStamp)
);

CREATE CLUSTERED INDEX IX_DimTime_DateTimeStamp ON DimTime(DateTimeStamp);

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- 'Warehouse', 'Route Node', 'Distribution Center'
    Address VARCHAR(500),
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50),
    IsActive BIT DEFAULT 1,
    EffectiveDate DATE NOT NULL,
    ExpirationDate DATE NULL,
    CONSTRAINT UQ_DimGeography_LocationID UNIQUE (LocationID, EffectiveDate)
);

-- DimProductGravity: Product categorization with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL,
    ProductName VARCHAR(500) NOT NULL,
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    PickFrequencyScore DECIMAL(5,2), -- 0-100
    ValueScore DECIMAL(5,2), -- 0-100 based on unit price
    FragilityScore DECIMAL(5,2), -- 0-100
    GravityScore AS (PickFrequencyScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    GravityZone AS (
        CASE 
            WHEN (PickFrequencyScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) >= 75 THEN 'High'
            WHEN (PickFrequencyScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) >= 50 THEN 'Medium'
            ELSE 'Low'
        END
    ) PERSISTED,
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,2),
    TemperatureRequirement VARCHAR(50), -- 'Ambient', 'Refrigerated', 'Frozen'
    IsActive BIT DEFAULT 1
);

CREATE INDEX IX_DimProductGravity_GravityScore ON DimProductGravity(GravityScore DESC);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Warehouse activity tracking
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationStartTime DATETIME2 NOT NULL,
    OperationEndTime DATETIME2,
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime) PERSISTED,
    QuantityHandled INT NOT NULL,
    WorkerID VARCHAR(50),
    EquipmentID VARCHAR(50),
    StorageZone VARCHAR(50),
    DwellTimeHours DECIMAL(10,2), -- For Putaway operations
    PickPathMeters DECIMAL(10,2), -- For Picking operations
    ErrorCount INT DEFAULT 0,
    IsRework BIT DEFAULT 0
);

CREATE CLUSTERED INDEX IX_FactWarehouseOps_Time ON FactWarehouseOperations(TimeKey, OperationType);
CREATE INDEX IX_FactWarehouseOps_Product ON FactWarehouseOperations(ProductKey);

-- FactFleetTrips: Fleet telemetry and trip data
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TripStartTime DATETIME2 NOT NULL,
    TripEndTime DATETIME2,
    PlannedDurationMinutes INT,
    ActualDurationMinutes AS DATEDIFF(MINUTE, TripStartTime, TripEndTime) PERSISTED,
    DistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    AverageSpeedKph DECIMAL(5,2),
    MaxSpeedKph DECIMAL(5,2),
    HarshBrakingEvents INT DEFAULT 0,
    HarshAccelerationEvents INT DEFAULT 0,
    WeatherCondition VARCHAR(50),
    TrafficDelayMinutes DECIMAL(10,2),
    OnTimeDelivery BIT,
    CargoValue DECIMAL(15,2)
);

CREATE CLUSTERED INDEX IX_FactFleetTrips_Time ON FactFleetTrips(TimeKey);
CREATE INDEX IX_FactFleetTrips_Vehicle ON FactFleetTrips(VehicleID, TripStartTime);

-- FactCrossDock: Cross-docking operations
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    CrossDockStartTime DATETIME2 NOT NULL,
    CrossDockEndTime DATETIME2,
    DwellTimeMinutes AS DATEDIFF(MINUTE, CrossDockStartTime, CrossDockEndTime) PERSISTED,
    QuantityTransferred INT NOT NULL,
    TemperatureCompliant BIT,
    QualityCheckPassed BIT
);
```

## Key Stored Procedures

### Incremental Data Loading

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRANSACTION;
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OperationStartTime, OperationEndTime, QuantityHandled,
        WorkerID, EquipmentID, StorageZone, DwellTimeHours,
        PickPathMeters, ErrorCount, IsRework
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        wms.OperationType,
        wms.StartTime,
        wms.EndTime,
        wms.Quantity,
        wms.WorkerID,
        wms.EquipmentID,
        wms.Zone,
        CASE WHEN wms.OperationType = 'Putaway' THEN wms.DwellHours ELSE NULL END,
        CASE WHEN wms.OperationType = 'Picking' THEN wms.PathMeters ELSE NULL END,
        wms.Errors,
        wms.IsRework
    FROM ExternalWMS.dbo.Operations wms
    INNER JOIN DimTime t ON CAST(wms.StartTime AS DATETIME2) = t.DateTimeStamp
    INNER JOIN DimGeography g ON wms.LocationID = g.LocationID AND wms.StartTime BETWEEN g.EffectiveDate AND ISNULL(g.ExpirationDate, '9999-12-31')
    INNER JOIN DimProductGravity p ON wms.SKU = p.SKU AND p.IsActive = 1
    WHERE wms.LastModified > @LastLoadTime;
    
    COMMIT TRANSACTION;
END;
GO

-- Incremental load for fleet trips
CREATE PROCEDURE sp_LoadFleetTrips
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRANSACTION;
    
    INSERT INTO FactFleetTrips (
        TimeKey, VehicleID, DriverID, OriginGeographyKey, DestinationGeographyKey,
        TripStartTime, TripEndTime, PlannedDurationMinutes, DistanceKm,
        FuelConsumedLiters, IdleTimeMinutes, LoadingTimeMinutes, UnloadingTimeMinutes,
        AverageSpeedKph, MaxSpeedKph, HarshBrakingEvents, HarshAccelerationEvents,
        WeatherCondition, TrafficDelayMinutes, OnTimeDelivery, CargoValue
    )
    SELECT 
        t.TimeKey,
        tel.VehicleID,
        tel.DriverID,
        go.GeographyKey AS OriginKey,
        gd.GeographyKey AS DestinationKey,
        tel.StartTime,
        tel.EndTime,
        tel.PlannedMinutes,
        tel.DistanceKm,
        tel.FuelLiters,
        tel.IdleMinutes,
        tel.LoadMinutes,
        tel.UnloadMinutes,
        tel.AvgSpeed,
        tel.MaxSpeed,
        tel.HardBrakes,
        tel.HardAccel,
        tel.Weather,
        tel.TrafficDelay,
        CASE WHEN tel.ActualArrival <= tel.ScheduledArrival THEN 1 ELSE 0 END,
        tel.CargoValue
    FROM ExternalFleet.dbo.TripTelemetry tel
    INNER JOIN DimTime t ON CAST(tel.StartTime AS DATETIME2) = t.DateTimeStamp
    INNER JOIN DimGeography go ON tel.OriginID = go.LocationID AND tel.StartTime BETWEEN go.EffectiveDate AND ISNULL(go.ExpirationDate, '9999-12-31')
    INNER JOIN DimGeography gd ON tel.DestinationID = gd.LocationID AND tel.StartTime BETWEEN gd.EffectiveDate AND ISNULL(gd.ExpirationDate, '9999-12-31')
    WHERE tel.LastModified > @LastLoadTime;
    
    COMMIT TRANSACTION;
END;
GO
```

### Analytical Views

```sql
-- Cross-fact KPI view: Warehouse efficiency vs Fleet utilization
CREATE VIEW vw_WarehouseFleetKPI AS
SELECT 
    t.DateKey,
    t.WeekOfYear,
    t.MonthName,
    g.LocationName AS Warehouse,
    -- Warehouse metrics
    SUM(CASE WHEN wo.OperationType = 'Picking' THEN wo.QuantityHandled ELSE 0 END) AS TotalPicks,
    AVG(CASE WHEN wo.OperationType = 'Picking' THEN wo.DurationMinutes ELSE NULL END) AS AvgPickTimeMinutes,
    AVG(CASE WHEN wo.OperationType = 'Putaway' THEN wo.DwellTimeHours ELSE NULL END) AS AvgDwellTimeHours,
    SUM(wo.ErrorCount) AS TotalErrors,
    -- Fleet metrics
    COUNT(DISTINCT ft.TripKey) AS TotalTrips,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTimeMinutes,
    SUM(ft.FuelConsumedLiters) AS TotalFuelLiters,
    SUM(CASE WHEN ft.OnTimeDelivery = 1 THEN 1 ELSE 0 END) * 100.0 / NULLIF(COUNT(ft.TripKey), 0) AS OnTimePercentage,
    -- Cross-metric KPI
    SUM(CASE WHEN wo.OperationType = 'Picking' THEN wo.QuantityHandled ELSE 0 END) * 1.0 / NULLIF(COUNT(DISTINCT ft.TripKey), 0) AS PicksPerTrip
FROM DimTime t
LEFT JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.TimeKey
LEFT JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
WHERE g.LocationType = 'Warehouse'
GROUP BY t.DateKey, t.WeekOfYear, t.MonthName, g.LocationName;
GO

-- Predictive bottleneck view
CREATE VIEW vw_PredictiveBottlenecks AS
WITH OperationTrends AS (
    SELECT 
        GeographyKey,
        ProductKey,
        OperationType,
        AVG(DurationMinutes) AS AvgDuration,
        STDEV(DurationMinutes) AS StdDevDuration,
        COUNT(*) AS SampleSize
    FROM FactWarehouseOperations
    WHERE OperationEndTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY GeographyKey, ProductKey, OperationType
)
SELECT 
    wo.OperationKey,
    wo.OperationStartTime,
    g.LocationName,
    p.SKU,
    p.ProductName,
    p.GravityZone,
    wo.OperationType,
    wo.DurationMinutes AS ActualDuration,
    ot.AvgDuration,
    ot.StdDevDuration,
    (wo.DurationMinutes - ot.AvgDuration) / NULLIF(ot.StdDevDuration, 0) AS ZScore,
    CASE 
        WHEN (wo.DurationMinutes - ot.AvgDuration) / NULLIF(ot.StdDevDuration, 0) > 2 THEN 'Critical'
        WHEN (wo.DurationMinutes - ot.AvgDuration) / NULLIF(ot.StdDevDuration, 0) > 1 THEN 'Warning'
        ELSE 'Normal'
    END AS AlertLevel
FROM FactWarehouseOperations wo
INNER JOIN OperationTrends ot ON wo.GeographyKey = ot.GeographyKey 
    AND wo.ProductKey = ot.ProductKey 
    AND wo.OperationType = ot.OperationType
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationEndTime >= DATEADD(HOUR, -24, GETDATE())
    AND ot.SampleSize >= 10;
GO
```

### Automated Alert System

```sql
-- Alert stored procedure
CREATE PROCEDURE sp_ProcessAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertBody NVARCHAR(MAX) = '';
    
    -- Check for critical warehouse bottlenecks
    IF EXISTS (SELECT 1 FROM vw_PredictiveBottlenecks WHERE AlertLevel = 'Critical')
    BEGIN
        SET @AlertBody = @AlertBody + '<h2>Critical Warehouse Bottlenecks Detected</h2>';
        SET @AlertBody = @AlertBody + '<table border="1"><tr><th>Location</th><th>SKU</th><th>Operation</th><th>Duration</th><th>Expected</th></tr>';
        
        SELECT @AlertBody = @AlertBody + 
            '<tr><td>' + LocationName + '</td><td>' + SKU + '</td><td>' + OperationType + 
            '</td><td>' + CAST(ActualDuration AS VARCHAR) + ' min</td><td>' + 
            CAST(AvgDuration AS VARCHAR) + ' min</td></tr>'
        FROM vw_PredictiveBottlenecks
        WHERE AlertLevel = 'Critical';
        
        SET @AlertBody = @AlertBody + '</table><br/>';
    END
    
    -- Check for fleet idle time threshold breaches (>15%)
    IF EXISTS (
        SELECT 1 FROM FactFleetTrips 
        WHERE TripEndTime >= DATEADD(HOUR, -2, GETDATE())
            AND (IdleTimeMinutes * 100.0 / NULLIF(ActualDurationMinutes, 0)) > 15
    )
    BEGIN
        SET @AlertBody = @AlertBody + '<h2>Excessive Fleet Idle Time</h2>';
        SET @AlertBody = @AlertBody + '<table border="1"><tr><th>Vehicle</th><th>Trip Start</th><th>Idle Time</th><th>% of Trip</th></tr>';
        
        SELECT @AlertBody = @AlertBody + 
            '<tr><td>' + VehicleID + '</td><td>' + CONVERT(VARCHAR, TripStartTime, 120) + 
            '</td><td>' + CAST(IdleTimeMinutes AS VARCHAR) + ' min</td><td>' + 
            CAST(CAST((IdleTimeMinutes * 100.0 / NULLIF(ActualDurationMinutes, 0)) AS DECIMAL(5,2)) AS VARCHAR) + '%</td></tr>'
        FROM FactFleetTrips
        WHERE TripEndTime >= DATEADD(HOUR, -2, GETDATE())
            AND (IdleTimeMinutes * 100.0 / NULLIF(ActualDurationMinutes, 0)) > 15;
        
        SET @AlertBody = @AlertBody + '</table><br/>';
    END
    
    -- Send email if any alerts were generated
    IF LEN(@AlertBody) > 0
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_ADDRESS}',
            @subject = 'LogiFleet Pulse - Critical Alerts',
            @body = @AlertBody,
            @body_format = 'HTML';
    END
END;
GO

-- Schedule the alert procedure (run every 15 minutes)
EXEC msdb.dbo.sp_add_job
    @job_name = 'LogiFleet_AlertMonitoring';

EXEC msdb.dbo.sp_add_jobstep
    @job_name = 'LogiFleet_AlertMonitoring',
    @step_name = 'ProcessAlerts',
    @subsystem = 'TSQL',
    @database_name = 'LogiFleetPulse',
    @command = 'EXEC sp_ProcessAlerts';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = 'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = 'LogiFleet_AlertMonitoring',
    @schedule_name = 'Every15Minutes';

EXEC msdb.dbo.sp_add_jobserver
    @job_name = 'LogiFleet_AlertMonitoring';
```

## Power BI Configuration

### Connecting to Data Source

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. When prompted, enter your SQL Server connection details:
   - **Server**: Your MS SQL Server instance
   - **Database**: LogiFleetPulse
   - **Data Connectivity mode**: DirectQuery (for real-time) or Import (for performance)

### Key DAX Measures

```dax
// Total Warehouse Operations
Total Operations = COUNTROWS(FactWarehouseOperations)

// Average Pick Duration
Avg Pick Duration = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DurationMinutes]),
    FactWarehouseOperations[OperationType] = "Picking"
)

// Warehouse Efficiency Score (0-100)
Warehouse Efficiency = 
VAR AvgDuration = AVERAGE(FactWarehouseOperations[DurationMinutes])
VAR ErrorRate = DIVIDE(SUM(FactWarehouseOperations[ErrorCount]), [Total Operations], 0)
VAR EfficiencyBase = 100 - (ErrorRate * 100)
VAR DurationPenalty = MIN(AvgDuration / 5, 20) // Cap at 20 point penalty
RETURN MAX(EfficiencyBase - DurationPenalty, 0)

// Fleet Fuel Efficiency (km per liter)
Fuel Efficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKm]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)

// On-Time Delivery Rate
OTD Rate = 
DIVIDE(
    CALCULATE(COUNTROWS(FactFleetTrips), FactFleetTrips[OnTimeDelivery] = TRUE),
    COUNTROWS(FactFleetTrips),
    0
)

// Cross-Fact: Cost per Pick
Cost Per Pick = 
VAR TotalPicks = CALCULATE([Total Operations], FactWarehouseOperations[OperationType] = "Picking")
VAR TotalFuelCost = SUM(FactFleetTrips[FuelConsumedLiters]) * 1.5 // $1.50 per liter assumption
VAR LaborCost = [Total Operations] * 0.5 // $0.50 per operation assumption
RETURN DIVIDE(TotalFuelCost + LaborCost, TotalPicks, 0)

// Gravity Zone Utilization
High Gravity Utilization = 
CALCULATE(
    [Total Operations],
    DimProductGravity[GravityZone] = "High"
) / [Total Operations]

// Predictive Bottleneck Index (0-100, higher = more risk)
Bottleneck Risk Index = 
VAR CriticalCount = CALCULATE(COUNTROWS(vw_PredictiveBottlenecks), vw_PredictiveBottlenecks[AlertLevel] = "Critical")
VAR WarningCount = CALCULATE(COUNTROWS(vw_PredictiveBottlenecks), vw_PredictiveBottlenecks[AlertLevel] = "Warning")
VAR TotalOps = COUNTROWS(vw_PredictiveBottlenecks)
RETURN MIN((CriticalCount * 10 + WarningCount * 5) / NULLIF(TotalOps, 0) * 100, 100)
```

### Time Intelligence Calculations

```dax
// Year-over-Year Growth
YoY Operations Growth = 
VAR CurrentPeriod = [Total Operations]
VAR PreviousYear = CALCULATE([Total Operations], DATEADD(DimTime[DateTimeStamp], -1, YEAR))
RETURN DIVIDE(CurrentPeriod - PreviousYear, PreviousYear, 0)

// Moving Average (30 days)
MA 30 Days = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DurationMinutes]),
    DATESINPERIOD(DimTime[DateTimeStamp], LASTDATE(DimTime[DateTimeStamp]), -30, DAY)
)

// Week-over-Week Comparison
WoW Change = 
VAR CurrentWeek = [Total Operations]
VAR PreviousWeek = CALCULATE([Total Operations], DATEADD(DimTime[DateTimeStamp], -7, DAY))
RETURN CurrentWeek - PreviousWeek
```

## Common Usage Patterns

### Pattern 1: Identifying Underutilized Gravity Zones

```sql
-- Find products in wrong gravity zones
WITH ProductPerformance AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        p.GravityZone AS CurrentZone,
        COUNT(*) AS PickCount,
        AVG(wo.DurationMinutes) AS AvgPickTime,
        SUM(wo.QuantityHandled) AS TotalVolume
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.OperationType = 'Picking'
        AND wo.OperationStartTime >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY p.ProductKey, p.SKU, p.ProductName, p.GravityZone
),
CalculatedZone AS (
    SELECT 
        *,
        CASE 
            WHEN PickCount >= 100 AND TotalVolume >= 1000 THEN 'High'
            WHEN PickCount >= 30 AND TotalVolume >= 300 THEN 'Medium'
            ELSE 'Low'
        END AS RecommendedZone
    FROM ProductPerformance
)
SELECT 
    SKU,
    ProductName,
    CurrentZone,
    RecommendedZone,
    PickCount,
    AvgPickTime,
    TotalVolume,
    CASE 
        WHEN CurrentZone <> RecommendedZone THEN 'REZONE RECOMMENDED'
        ELSE 'Optimal'
    END AS Action
FROM CalculatedZone
WHERE CurrentZone <> RecommendedZone
ORDER BY PickCount DESC;
```

### Pattern 2: Fleet Route Optimization Analysis

```sql
-- Identify routes with high idle time and suggest consolidation
WITH RouteAnalysis AS (
    SELECT 
        go.LocationName AS Origin,
        gd.LocationName AS Destination,
        COUNT(*) AS TripCount,
        AVG(ft.ActualDurationMinutes) AS AvgDuration,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        AVG(ft.FuelConsumedLiters) AS AvgFuel,
        SUM(CASE WHEN ft.OnTimeDelivery = 1 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS OTDRate,
        AVG(ft.DistanceKm) AS AvgDistance
    FROM FactFleetTrips ft
    INNER JOIN DimGeography go ON ft.OriginGeographyKey = go.GeographyKey
    INNER JOIN DimGeography gd ON ft.DestinationGeographyKey = gd.GeographyKey
    WHERE ft.TripStartTime >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY go.LocationName, gd.LocationName
)
SELECT 
    Origin,
    Destination,
    TripCount,
    AvgDuration,
    AvgIdleTime,
    (AvgIdleTime * 100.0 / NULLIF(AvgDuration, 0))
