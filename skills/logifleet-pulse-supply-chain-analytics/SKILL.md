---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehousing with multi-fact star schema for warehouse, fleet, and cross-dock operations
triggers:
  - set up logifleet pulse supply chain analytics
  - implement logistics data warehouse with power bi
  - configure warehouse and fleet tracking database
  - create multi-fact star schema for logistics
  - build supply chain analytics dashboard
  - deploy logifleet pulse sql schema
  - integrate warehouse and fleet data sources
  - analyze logistics bottlenecks with logifleet
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced logistics intelligence platform built on MS SQL Server and Power BI. It provides a unified semantic layer combining warehouse operations, fleet telemetry, inventory management, and external data sources through a multi-fact star schema architecture. The system enables cross-fact KPI analysis, predictive bottleneck detection, and real-time operational dashboards.

**Core Components:**
- Multi-fact star schema (warehouse operations, fleet trips, cross-dock transfers)
- Time-phased dimensions with 15-minute granularity
- Power BI dashboards with role-based access
- Predictive analytics and anomaly detection
- Bridge tables for many-to-many relationships

## Installation

### Prerequisites

- MS SQL Server 2019+ (recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources (WMS, telemetry APIs, ERP systems)

### Initial Setup

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**

```sql
-- Connect to your SQL Server instance via SSMS
-- Run the schema creation script
:r "./sql/schema/01_create_database.sql"
:r "./sql/schema/02_create_dimensions.sql"
:r "./sql/schema/03_create_facts.sql"
:r "./sql/schema/04_create_relationships.sql"
:r "./sql/schema/05_create_views.sql"
:r "./sql/schema/06_create_procedures.sql"
```

3. **Configure data sources:**

Edit connection strings in `config.json`:

```json
{
  "sql_server": {
    "server": "YOUR_SQL_SERVER",
    "database": "LogiFleetPulse",
    "auth": "integrated"
  },
  "data_sources": {
    "wms_api": "${WMS_API_URL}",
    "fleet_telemetry": "${FLEET_API_URL}",
    "weather_api_key": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

## Core Database Schema

### Dimension Tables

**DimTime** - Time-phased dimension with 15-minute buckets:

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    FullDateTime DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10),
    WeekOfYear TINYINT,
    MonthNumber TINYINT,
    MonthName VARCHAR(20),
    Quarter TINYINT,
    FiscalPeriod TINYINT,
    Year SMALLINT,
    IsWeekend BIT,
    IsHoliday BIT
);

-- Create clustered index for time-series queries
CREATE CLUSTERED INDEX IX_DimTime_FullDateTime ON DimTime(FullDateTime);
```

**DimProductGravity** - Products with warehouse velocity scoring:

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    ProductSKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200),
    ProductCategory VARCHAR(100),
    ProductSubcategory VARCHAR(100),
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,2),
    IsFragile BIT,
    RequiresColdStorage BIT,
    GravityScore DECIMAL(5,2), -- Higher = faster moving
    OptimalZone VARCHAR(20),
    ReorderPoint INT,
    LeadTimeDays INT,
    UnitValue DECIMAL(10,2)
);

CREATE INDEX IX_Product_GravityScore ON DimProductGravity(GravityScore DESC);
```

**DimGeography** - Hierarchical location dimension:

```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL UNIQUE,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- Warehouse, Route Node, Distribution Center
    AddressLine1 VARCHAR(200),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    Country VARCHAR(100),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6),
    ParentLocationKey INT,
    RegionName VARCHAR(100),
    TimeZone VARCHAR(50)
);
```

**DimSupplierReliability** - Supplier performance metrics:

```sql
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200),
    SupplierCategory VARCHAR(100),
    AverageLeadTimeDays DECIMAL(5,2),
    LeadTimeVariance DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,2),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2),
    LastAuditDate DATE
);
```

### Fact Tables

**FactWarehouseOperations** - Core warehouse activity tracking:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    SupplierKey INT,
    OperationType VARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    OperationStartTime DATETIME NOT NULL,
    OperationEndTime DATETIME,
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime),
    Quantity INT NOT NULL,
    ZoneAssigned VARCHAR(20),
    DwellTimeHours DECIMAL(10,2),
    OperatorID VARCHAR(50),
    EquipmentID VARCHAR(50),
    BatchNumber VARCHAR(50),
    IsDamaged BIT DEFAULT 0,
    IsExpedited BIT DEFAULT 0,
    
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_WH_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WH_Supplier FOREIGN KEY (SupplierKey) REFERENCES DimSupplierReliability(SupplierKey)
);

-- Partition by date for performance
CREATE PARTITION FUNCTION PF_WarehouseOps_Date (DATETIME)
AS RANGE RIGHT FOR VALUES ('2026-01-01', '2026-04-01', '2026-07-01', '2026-10-01');

CREATE PARTITION SCHEME PS_WarehouseOps_Date
AS PARTITION PF_WarehouseOps_Date ALL TO ([PRIMARY]);
```

**FactFleetTrips** - Fleet and route tracking:

```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripStartTime DATETIME NOT NULL,
    TripEndTime DATETIME,
    TripDurationMinutes AS DATEDIFF(MINUTE, TripStartTime, TripEndTime),
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    AverageSpeedKPH DECIMAL(10,2),
    MaxSpeedKPH DECIMAL(10,2),
    HarshBrakingEvents INT,
    WeatherCondition VARCHAR(50),
    TrafficDelayMinutes DECIMAL(10,2),
    TollCost DECIMAL(10,2),
    IsDelayed BIT DEFAULT 0,
    IsEmergency BIT DEFAULT 0,
    
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_Fleet_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE INDEX IX_Fleet_StartTime ON FactFleetTrips(TripStartTime);
CREATE INDEX IX_Fleet_Vehicle ON FactFleetTrips(VehicleKey);
```

**FactCrossDock** - Direct transfer operations:

```sql
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    GeographyKey INT NOT NULL,
    ArrivalTime DATETIME NOT NULL,
    DepartureTime DATETIME,
    DockDwellMinutes AS DATEDIFF(MINUTE, ArrivalTime, DepartureTime),
    Quantity INT NOT NULL,
    TransferZone VARCHAR(20),
    IsPriority BIT DEFAULT 0,
    
    CONSTRAINT FK_CrossDock_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_CrossDock_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey)
);
```

## Key Stored Procedures

### Data Loading and ETL

**Incremental warehouse operations load:**

```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last hour if no parameter provided
    IF @LastLoadTime IS NULL
        SET @LastLoadTime = DATEADD(HOUR, -1, GETDATE());
    
    -- Load from staging or external source
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, GeographyKey, SupplierKey,
        OperationType, OperationStartTime, OperationEndTime,
        Quantity, ZoneAssigned, DwellTimeHours, OperatorID
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        s.SupplierKey,
        stg.OperationType,
        stg.StartTime,
        stg.EndTime,
        stg.Quantity,
        stg.Zone,
        DATEDIFF(HOUR, stg.StartTime, COALESCE(stg.EndTime, GETDATE())),
        stg.OperatorID
    FROM StagingWarehouseOps stg
    INNER JOIN DimTime t ON CAST(stg.StartTime AS DATE) = CAST(t.FullDateTime AS DATE)
        AND DATEPART(HOUR, stg.StartTime) = t.HourOfDay
        AND DATEPART(MINUTE, stg.StartTime) / 15 = DATEPART(MINUTE, t.FullDateTime) / 15
    INNER JOIN DimProductGravity p ON stg.SKU = p.ProductSKU
    INNER JOIN DimGeography g ON stg.WarehouseCode = g.LocationID
    LEFT JOIN DimSupplierReliability s ON stg.SupplierCode = s.SupplierID
    WHERE stg.LoadedToFact = 0
        AND stg.StartTime >= @LastLoadTime;
    
    -- Mark staging records as loaded
    UPDATE StagingWarehouseOps
    SET LoadedToFact = 1
    WHERE LoadedToFact = 0 AND StartTime >= @LastLoadTime;
    
    PRINT 'Loaded ' + CAST(@@ROWCOUNT AS VARCHAR) + ' warehouse operations.';
END;
```

**Fleet telemetry ingestion:**

```sql
CREATE PROCEDURE sp_LoadFleetTrips
    @LastLoadTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    IF @LastLoadTime IS NULL
        SET @LastLoadTime = DATEADD(HOUR, -1, GETDATE());
    
    INSERT INTO FactFleetTrips (
        TimeKey, VehicleKey, DriverKey, OriginGeographyKey, DestinationGeographyKey,
        TripStartTime, TripEndTime, DistanceKM, FuelConsumedLiters,
        IdleTimeMinutes, LoadingTimeMinutes, UnloadingTimeMinutes,
        AverageSpeedKPH, HarshBrakingEvents, WeatherCondition, IsDelayed
    )
    SELECT 
        t.TimeKey,
        v.VehicleKey,
        d.DriverKey,
        orig.GeographyKey,
        dest.GeographyKey,
        stg.StartTime,
        stg.EndTime,
        stg.Distance,
        stg.FuelUsed,
        stg.IdleTime,
        stg.LoadTime,
        stg.UnloadTime,
        stg.AvgSpeed,
        stg.BrakingEvents,
        stg.Weather,
        CASE WHEN stg.ActualArrival > stg.PlannedArrival THEN 1 ELSE 0 END
    FROM StagingFleetTrips stg
    INNER JOIN DimTime t ON CAST(stg.StartTime AS DATE) = CAST(t.FullDateTime AS DATE)
        AND DATEPART(HOUR, stg.StartTime) = t.HourOfDay
    INNER JOIN DimVehicle v ON stg.VehicleID = v.VehicleID
    INNER JOIN DimDriver d ON stg.DriverID = d.DriverID
    INNER JOIN DimGeography orig ON stg.OriginCode = orig.LocationID
    INNER JOIN DimGeography dest ON stg.DestinationCode = dest.LocationID
    WHERE stg.LoadedToFact = 0
        AND stg.StartTime >= @LastLoadTime;
    
    UPDATE StagingFleetTrips
    SET LoadedToFact = 1
    WHERE LoadedToFact = 0 AND StartTime >= @LastLoadTime;
    
    PRINT 'Loaded ' + CAST(@@ROWCOUNT AS VARCHAR) + ' fleet trips.';
END;
```

### Analytics Views

**Cross-fact KPI view - Warehouse efficiency vs fleet utilization:**

```sql
CREATE VIEW vw_CrossFactLogisticsKPI
AS
WITH WarehouseMetrics AS (
    SELECT
        dt.DateKey,
        dg.Country,
        dg.RegionName,
        dp.ProductCategory,
        COUNT(*) AS TotalOperations,
        AVG(DurationMinutes) AS AvgOperationMinutes,
        AVG(DwellTimeHours) AS AvgDwellHours,
        SUM(CASE WHEN OperationType = 'Shipping' THEN Quantity ELSE 0 END) AS UnitsShipped
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
    INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    GROUP BY dt.DateKey, dg.Country, dg.RegionName, dp.ProductCategory
),
FleetMetrics AS (
    SELECT
        dt.DateKey,
        orig.Country,
        orig.RegionName,
        COUNT(*) AS TotalTrips,
        AVG(TripDurationMinutes) AS AvgTripMinutes,
        AVG(IdleTimeMinutes) AS AvgIdleMinutes,
        SUM(FuelConsumedLiters) AS TotalFuelUsed,
        SUM(CASE WHEN IsDelayed = 1 THEN 1 ELSE 0 END) AS DelayedTrips
    FROM FactFleetTrips fft
    INNER JOIN DimTime dt ON fft.TimeKey = dt.TimeKey
    INNER JOIN DimGeography orig ON fft.OriginGeographyKey = orig.GeographyKey
    GROUP BY dt.DateKey, orig.Country, orig.RegionName
)
SELECT
    wm.DateKey,
    wm.Country,
    wm.RegionName,
    wm.ProductCategory,
    wm.TotalOperations,
    wm.AvgOperationMinutes,
    wm.AvgDwellHours,
    wm.UnitsShipped,
    fm.TotalTrips,
    fm.AvgTripMinutes,
    fm.AvgIdleMinutes,
    fm.TotalFuelUsed,
    fm.DelayedTrips,
    -- Composite KPIs
    CASE 
        WHEN wm.UnitsShipped > 0 
        THEN fm.TotalFuelUsed / CAST(wm.UnitsShipped AS DECIMAL(10,2))
        ELSE 0 
    END AS FuelPerUnitShipped,
    CASE
        WHEN fm.AvgTripMinutes > 0
        THEN (fm.AvgIdleMinutes / fm.AvgTripMinutes) * 100
        ELSE 0
    END AS IdleTimePercent,
    wm.AvgDwellHours * 60 + fm.AvgIdleMinutes AS TotalWasteMinutes
FROM WarehouseMetrics wm
FULL OUTER JOIN FleetMetrics fm 
    ON wm.DateKey = fm.DateKey 
    AND wm.Country = fm.Country 
    AND wm.RegionName = fm.RegionName;
```

**Predictive bottleneck detection:**

```sql
CREATE VIEW vw_BottleneckPrediction
AS
WITH RecentPerformance AS (
    SELECT
        GeographyKey,
        OperationType,
        AVG(DurationMinutes) AS AvgDuration,
        STDEV(DurationMinutes) AS StdDevDuration,
        AVG(DwellTimeHours) AS AvgDwell
    FROM FactWarehouseOperations
    WHERE OperationStartTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY GeographyKey, OperationType
),
CurrentLoad AS (
    SELECT
        GeographyKey,
        OperationType,
        COUNT(*) AS ActiveOperations,
        AVG(DurationMinutes) AS CurrentAvgDuration,
        MAX(DwellTimeHours) AS MaxDwell
    FROM FactWarehouseOperations
    WHERE OperationEndTime IS NULL
        OR OperationEndTime >= DATEADD(HOUR, -4, GETDATE())
    GROUP BY GeographyKey, OperationType
)
SELECT
    dg.LocationName,
    cl.OperationType,
    cl.ActiveOperations,
    cl.CurrentAvgDuration,
    rp.AvgDuration AS HistoricalAvg,
    CASE 
        WHEN cl.CurrentAvgDuration > rp.AvgDuration + (2 * rp.StdDevDuration)
        THEN 'Critical'
        WHEN cl.CurrentAvgDuration > rp.AvgDuration + rp.StdDevDuration
        THEN 'Warning'
        ELSE 'Normal'
    END AS BottleneckSeverity,
    cl.MaxDwell,
    rp.AvgDwell AS HistoricalDwellAvg,
    CASE
        WHEN cl.MaxDwell > rp.AvgDwell * 1.5 THEN 'High Risk'
        WHEN cl.MaxDwell > rp.AvgDwell * 1.2 THEN 'Moderate Risk'
        ELSE 'Low Risk'
    END AS DwellRisk
FROM CurrentLoad cl
INNER JOIN RecentPerformance rp 
    ON cl.GeographyKey = rp.GeographyKey 
    AND cl.OperationType = rp.OperationType
INNER JOIN DimGeography dg ON cl.GeographyKey = dg.GeographyKey
WHERE cl.CurrentAvgDuration > rp.AvgDuration * 0.8; -- Only show potential issues
```

## Power BI Integration

### Connecting to SQL Server

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. When prompted, enter connection details:

```
Server: YOUR_SQL_SERVER_NAME
Database: LogiFleetPulse
Data Connectivity mode: DirectQuery (recommended for real-time) or Import
```

3. Authenticate using Windows or SQL Server credentials

### DAX Measures for Cross-Fact Analysis

**Total throughput efficiency:**

```dax
Throughput Efficiency = 
VAR TotalShipped = 
    SUM(FactWarehouseOperations[Quantity])
VAR TotalTrips = 
    COUNTROWS(FactFleetTrips)
VAR AvgUnitsPerTrip = 
    DIVIDE(TotalShipped, TotalTrips, 0)
RETURN
    AvgUnitsPerTrip
```

**Fleet idle cost:**

```dax
Fleet Idle Cost = 
VAR IdleHours = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[IdleTimeMinutes] / 60
    )
VAR CostPerHour = 45 -- Configurable parameter
RETURN
    IdleHours * CostPerHour
```

**Warehouse gravity zone optimization score:**

```dax
Gravity Zone Score = 
VAR CurrentDwell = 
    AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR ProductGravity = 
    AVERAGE(DimProductGravity[GravityScore])
VAR OptimalDwell = 
    SWITCH(
        TRUE(),
        ProductGravity >= 8, 4,   -- High gravity: max 4 hours
        ProductGravity >= 5, 12,  -- Medium gravity: max 12 hours
        24                         -- Low gravity: max 24 hours
    )
VAR Score = 
    1 - (CurrentDwell / OptimalDwell)
RETURN
    MAX(MIN(Score, 1), 0) -- Clamp between 0 and 1
```

**Predictive delay risk:**

```dax
Delay Risk Index = 
VAR AvgDelay = 
    CALCULATE(
        AVERAGE(FactFleetTrips[TripDurationMinutes]),
        FactFleetTrips[IsDelayed] = TRUE
    )
VAR CurrentAvg = 
    AVERAGE(FactFleetTrips[TripDurationMinutes])
VAR WeatherRisk = 
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FactFleetTrips[WeatherCondition] IN {"Rain", "Snow", "Fog"}
    ) / COUNTROWS(FactFleetTrips)
RETURN
    (CurrentAvg / AvgDelay) * (1 + WeatherRisk)
```

### Row-Level Security Setup

Define roles in Power BI for regional access:

```dax
-- Role: Region Manager
[DimGeography[RegionName]] = USERNAME()

-- Role: Country Director  
[DimGeography[Country]] = LOOKUPVALUE(
    UserRegionMapping[Country],
    UserRegionMapping[Email],
    USERPRINCIPALNAME()
)

-- Role: Warehouse Supervisor
[DimGeography[LocationID]] IN VALUES(UserWarehouseAccess[WarehouseCode])
```

## Alerting System

### Automated threshold monitoring:

```sql
CREATE PROCEDURE sp_CheckAlertsAndNotify
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    DECLARE @Recipients VARCHAR(500) = '${ALERT_EMAIL_LIST}';
    
    -- Check for high idle time
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips
        WHERE TripEndTime >= DATEADD(HOUR, -2, GETDATE())
            AND (IdleTimeMinutes / NULLIF(TripDurationMinutes, 0)) > 0.15
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeds 15% threshold in last 2 hours.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = @Recipients,
            @subject = 'LogiFleet Pulse - Fleet Idle Alert',
            @body = @AlertMessage;
    END
    
    -- Check for warehouse dwell time
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations fwo
        INNER JOIN DimProductGravity dpg ON fwo.ProductKey = dpg.ProductKey
        WHERE fwo.OperationStartTime >= DATEADD(DAY, -1, GETDATE())
            AND fwo.DwellTimeHours > 72
            AND dpg.GravityScore >= 7
    )
    BEGIN
        SET @AlertMessage = 'ALERT: High-gravity products have exceeded 72-hour dwell time.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = @Recipients,
            @subject = 'LogiFleet Pulse - Dwell Time Alert',
            @body = @AlertMessage;
    END
    
    -- Check for bottleneck conditions
    INSERT INTO AlertHistory (AlertType, AlertMessage, Severity, CreatedAt)
    SELECT 
        'Bottleneck',
        'Bottleneck detected at ' + LocationName + ' for ' + OperationType,
        BottleneckSeverity,
        GETDATE()
    FROM vw_BottleneckPrediction
    WHERE BottleneckSeverity IN ('Critical', 'Warning');
END;
```

**Schedule the alert check:**

```sql
-- Create SQL Agent job to run every 15 minutes
EXEC msdb.dbo.sp_add_job
    @job_name = 'LogiFleet_AlertMonitoring',
    @enabled = 1;

EXEC msdb.dbo.sp_add_jobstep
    @job_name = 'LogiFleet_AlertMonitoring',
    @step_name = 'Check Thresholds',
    @subsystem = 'TSQL',
    @command = 'EXEC sp_CheckAlertsAndNotify;',
    @database_name = 'LogiFleetPulse';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = 'Every15Minutes',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = 'LogiFleet_AlertMonitoring',
    @schedule_name = 'Every15Minutes';

EXEC msdb.dbo.sp_add_jobserver
    @job_name = 'LogiFleet_AlertMonitoring';
```

## Common Patterns

### Pattern 1: Cross-Fact Correlation Query

Find correlation between warehouse dwell time and fleet delays:

```sql
SELECT 
    dg.RegionName,
    AVG(fwo.DwellTimeHours) AS AvgWarehouseDwell,
    AVG(CASE WHEN fft.IsDelayed = 1 THEN 1.0 ELSE 0.0 END) AS DelayRate,
    COUNT(DISTINCT fft.TripKey) AS TotalTrips
FROM FactWarehouseOperations fwo
INNER JOIN DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
INNER JOIN FactFleetTrips fft ON dg.GeographyKey = fft.OriginGeographyKey
    AND CAST(fwo.OperationStartTime AS DATE) = CAST(fft.TripStartTime AS DATE)
WHERE fwo.OperationStartTime >= DATEADD(MONTH, -1, GETDATE())
    AND fwo.OperationType = 'Shipping'
GROUP BY dg.RegionName
HAVING COUNT(DISTINCT fft.TripKey) > 10
ORDER BY DelayRate DESC;
```

### Pattern 2: Temporal Elasticity Analysis

Simulate capacity changes:

```sql
DECLARE @CapacityMultiplier DECIMAL(5,2) = 0.95; -- 95% capacity scenario

WITH SimulatedLoad AS (
    SELECT
        GeographyKey,
        OperationType,
        COUNT(*) AS CurrentOps,
        COUNT(*) / @CapacityMultiplier AS ProjectedOps,
        AVG(DurationMinutes) AS CurrentAvgDuration
    FROM FactWarehouseOperations
    WHERE OperationStartTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY GeographyKey, OperationType
)
SELECT
    dg.LocationName,
    sl.OperationType,
    sl.CurrentOps,
    sl.ProjectedOps,
    sl.CurrentAvgDuration,
    sl.CurrentAvgDuration * (sl.ProjectedOps / NULLIF(sl.CurrentOps, 0)) AS ProjectedAvgDuration,
    CASE 
        WHEN (sl.CurrentAvgDuration * (sl.ProjectedOps / NULLIF(sl.CurrentOps, 0))) > sl.CurrentAvgDuration * 1.3
        THEN 'High Risk'
        WHEN (sl.CurrentAvgDuration * (sl.ProjectedOps / NULLIF(sl.CurrentOps, 0))) > sl.CurrentAvgDuration * 1.15
        THEN 'Moderate Risk'
        ELSE 'Low Risk'
    END AS CapacityRisk
FROM SimulatedLoad sl
INNER JOIN DimGeography dg ON sl.GeographyKey = dg.GeographyKey;
```

### Pattern 3: Gravity Zone
