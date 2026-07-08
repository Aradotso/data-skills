---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI data warehousing engine for logistics, fleet management, and supply chain intelligence with multi-fact star schema
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "implement multi-fact star schema for warehouse data"
  - "create fleet management data warehouse"
  - "build supply chain intelligence platform"
  - "deploy LogiCore analytics warehouse"
  - "design logistics KPI dashboard with Power BI"
  - "integrate warehouse and fleet telemetry data"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced data warehousing and analytics platform for logistics operations, combining MS SQL Server backend with Power BI visualization. It implements a multi-fact star schema architecture that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer for real-time supply chain intelligence.

**Key capabilities:**
- Multi-fact data warehouse with time-phased dimensions
- Warehouse operations tracking (receiving, putaway, picking, packing, shipping)
- Fleet telemetry integration (GPS, fuel consumption, maintenance)
- Cross-fact KPI harmonization (linking inventory with fleet metrics)
- Predictive bottleneck detection and fleet triage
- Role-based Power BI dashboards with row-level security

## Installation

### Prerequisites

- MS SQL Server 2019+ (Enterprise or Developer Edition recommended)
- Power BI Desktop (latest version)
- SSMS (SQL Server Management Studio)
- Source data access: WMS, TMS, telemetry APIs

### Database Setup

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**

```sql
-- Connect to your SQL Server instance via SSMS
-- Create database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Execute schema creation scripts in order:
-- 1. Create dimension tables
-- 2. Create fact tables
-- 3. Create bridge tables
-- 4. Create views and stored procedures
-- 5. Set up indexing strategy
```

3. **Configure data connections:**

Create a `config.json` file:

```json
{
  "sql_server": {
    "server": "YOUR_SQL_SERVER",
    "database": "LogiFleetPulse",
    "username": "$(SQL_USER)",
    "password": "$(SQL_PASSWORD)",
    "connection_timeout": 30
  },
  "data_sources": {
    "wms_api": "$(WMS_API_URL)",
    "telemetry_endpoint": "$(FLEET_TELEMETRY_URL)",
    "weather_api_key": "$(WEATHER_API_KEY)"
  },
  "refresh_intervals": {
    "warehouse_operations": 15,
    "fleet_trips": 5,
    "external_feeds": 60
  }
}
```

## Core Schema Architecture

### Dimension Tables

**DimTime** - Time-aware dimension with 15-minute granularity:

```sql
CREATE TABLE dbo.DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2(0) NOT NULL,
    TimeMinute TINYINT,
    TimeHour TINYINT,
    DayOfWeek TINYINT,
    DayOfMonth TINYINT,
    MonthNumber TINYINT,
    QuarterNumber TINYINT,
    FiscalYear SMALLINT,
    IsWeekend BIT,
    IsHoliday BIT,
    ShiftCode VARCHAR(10)
);

-- Create clustered index on TimeKey
CREATE CLUSTERED INDEX IX_DimTime_TimeKey ON dbo.DimTime(TimeKey);
```

**DimGeography** - Hierarchical location dimension:

```sql
CREATE TABLE dbo.DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID VARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200),
    LocationType VARCHAR(50), -- Warehouse, DistributionCenter, Route, DeliveryZone
    AddressLine1 NVARCHAR(200),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    Country NVARCHAR(100),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    ParentLocationKey INT,
    TimeZone VARCHAR(50),
    IsActive BIT DEFAULT 1
);
```

**DimProductGravity** - Product classification with velocity scoring:

```sql
CREATE TABLE dbo.DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName NVARCHAR(200),
    CategoryLevel1 NVARCHAR(100),
    CategoryLevel2 NVARCHAR(100),
    CategoryLevel3 NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * (1/fragility)
    VelocityClass VARCHAR(20), -- Fast, Medium, Slow
    ValueTier VARCHAR(20), -- High, Medium, Low
    FragilityIndex TINYINT, -- 1-10 scale
    UnitWeight DECIMAL(10,3),
    StorageRequirement VARCHAR(50), -- Ambient, Refrigerated, Frozen
    OptimalZoneType VARCHAR(50),
    LastRecalculatedDate DATETIME2(0)
);
```

### Fact Tables

**FactWarehouseOperations** - Core warehouse activity tracking:

```sql
CREATE TABLE dbo.FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OrderID VARCHAR(100),
    OperationStartTime DATETIME2(0),
    OperationEndTime DATETIME2(0),
    DwellTimeMinutes INT,
    UnitsProcessed INT,
    OperatorID VARCHAR(50),
    ZoneID VARCHAR(50),
    StorageLocationID VARCHAR(100),
    EquipmentID VARCHAR(50),
    QualityCheckPassed BIT,
    ExceptionCode VARCHAR(50),
    CONSTRAINT FK_Warehouse_Time FOREIGN KEY (TimeKey) REFERENCES dbo.DimTime(TimeKey),
    CONSTRAINT FK_Warehouse_Geography FOREIGN KEY (GeographyKey) REFERENCES dbo.DimGeography(GeographyKey),
    CONSTRAINT FK_Warehouse_Product FOREIGN KEY (ProductKey) REFERENCES dbo.DimProductGravity(ProductKey)
);

-- Partitioning strategy for large datasets
CREATE PARTITION FUNCTION PF_WarehouseByMonth (INT)
AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301, 20260401); -- Extend monthly

-- Columnstore index for analytics queries
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse 
ON dbo.FactWarehouseOperations(TimeKey, GeographyKey, ProductKey, OperationType, DwellTimeMinutes, UnitsProcessed);
```

**FactFleetTrips** - Fleet telemetry and route tracking:

```sql
CREATE TABLE dbo.FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TripID VARCHAR(100) NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    TripStartTimeKey INT NOT NULL,
    TripEndTimeKey INT,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT,
    RouteID VARCHAR(100),
    PlannedDistanceKm DECIMAL(10,2),
    ActualDistanceKm DECIMAL(10,2),
    PlannedDurationMinutes INT,
    ActualDurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    StopCount TINYINT,
    AverageSpeedKmh DECIMAL(5,2),
    MaxSpeedKmh DECIMAL(5,2),
    HarshBrakingEvents TINYINT,
    HarshAccelerationEvents TINYINT,
    MaintenanceAlertCount TINYINT,
    WeatherCondition VARCHAR(50),
    TrafficDelayMinutes INT,
    OnTimeDelivery BIT,
    CONSTRAINT FK_Fleet_StartTime FOREIGN KEY (TripStartTimeKey) REFERENCES dbo.DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES dbo.DimGeography(GeographyKey)
);
```

**FactCrossDock** - Cross-docking operations:

```sql
CREATE TABLE dbo.FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    TransferStartTime DATETIME2(0),
    TransferEndTime DATETIME2(0),
    DockDoorInbound VARCHAR(20),
    DockDoorOutbound VARCHAR(20),
    UnitsTransferred INT,
    TransferDurationMinutes INT,
    TemperatureCompliant BIT,
    CONSTRAINT FK_CrossDock_Time FOREIGN KEY (TimeKey) REFERENCES dbo.DimTime(TimeKey),
    CONSTRAINT FK_CrossDock_Geography FOREIGN KEY (GeographyKey) REFERENCES dbo.DimGeography(GeographyKey),
    CONSTRAINT FK_CrossDock_Product FOREIGN KEY (ProductKey) REFERENCES dbo.DimProductGravity(ProductKey)
);
```

## Data Loading Patterns

### Incremental ETL Stored Procedure

```sql
CREATE PROCEDURE dbo.usp_IncrementalLoad_WarehouseOperations
    @LastLoadDateTime DATETIME2(0),
    @CurrentLoadDateTime DATETIME2(0)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new warehouse operations
    INSERT INTO dbo.FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OrderID, OperationStartTime, OperationEndTime,
        DwellTimeMinutes, UnitsProcessed, OperatorID,
        ZoneID, StorageLocationID, EquipmentID,
        QualityCheckPassed, ExceptionCode
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        src.OperationType,
        src.OrderID,
        src.OperationStartTime,
        src.OperationEndTime,
        DATEDIFF(MINUTE, src.OperationStartTime, src.OperationEndTime) AS DwellTimeMinutes,
        src.UnitsProcessed,
        src.OperatorID,
        src.ZoneID,
        src.StorageLocationID,
        src.EquipmentID,
        src.QualityCheckPassed,
        src.ExceptionCode
    FROM StagingWarehouseOperations src
    INNER JOIN dbo.DimTime dt ON CAST(src.OperationStartTime AS DATE) = CAST(dt.FullDateTime AS DATE)
        AND DATEPART(HOUR, src.OperationStartTime) = dt.TimeHour
        AND (DATEPART(MINUTE, src.OperationStartTime) / 15) * 15 = dt.TimeMinute
    INNER JOIN dbo.DimGeography dg ON src.LocationID = dg.LocationID
    INNER JOIN dbo.DimProductGravity dp ON src.SKU = dp.SKU
    WHERE src.OperationStartTime > @LastLoadDateTime
        AND src.OperationStartTime <= @CurrentLoadDateTime;
    
    -- Update gravity scores based on new velocity patterns
    EXEC dbo.usp_RecalculateProductGravity;
    
    RETURN @@ROWCOUNT;
END;
GO
```

### Gravity Score Calculation

```sql
CREATE PROCEDURE dbo.usp_RecalculateProductGravity
AS
BEGIN
    -- Recalculate product gravity scores based on last 90 days activity
    WITH ProductVelocity AS (
        SELECT 
            ProductKey,
            COUNT(*) AS OperationCount,
            SUM(UnitsProcessed) AS TotalUnits,
            AVG(DwellTimeMinutes) AS AvgDwellTime
        FROM dbo.FactWarehouseOperations
        WHERE OperationStartTime >= DATEADD(DAY, -90, GETDATE())
        GROUP BY ProductKey
    ),
    VelocityScores AS (
        SELECT 
            pv.ProductKey,
            NTILE(5) OVER (ORDER BY pv.TotalUnits DESC) AS VelocityQuintile,
            CASE 
                WHEN NTILE(5) OVER (ORDER BY pv.TotalUnits DESC) IN (1,2) THEN 'Fast'
                WHEN NTILE(5) OVER (ORDER BY pv.TotalUnits DESC) = 3 THEN 'Medium'
                ELSE 'Slow'
            END AS VelocityClass
        FROM ProductVelocity pv
    )
    UPDATE dp
    SET 
        dp.GravityScore = (vs.VelocityQuintile * 20) * 
                          (CASE dp.ValueTier WHEN 'High' THEN 1.5 WHEN 'Medium' THEN 1.0 ELSE 0.7 END) * 
                          (1.0 / NULLIF(dp.FragilityIndex, 0)),
        dp.VelocityClass = vs.VelocityClass,
        dp.LastRecalculatedDate = GETDATE()
    FROM dbo.DimProductGravity dp
    INNER JOIN VelocityScores vs ON dp.ProductKey = vs.ProductKey;
END;
GO
```

## Power BI Integration

### Connection Setup

1. **Open Power BI Desktop**
2. **Import the template:** `LogiFleet_Pulse_Master.pbit`
3. **Configure connection:**

```m
// Power Query M - SQL Server connection
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER"), 
        "LogiFleetPulse",
        [
            Query = null,
            CommandTimeout = #duration(0, 0, 10, 0),
            ConnectionTimeout = #duration(0, 0, 1, 0)
        ]
    )
in
    Source
```

### Key DAX Measures

**Cross-Fact KPI: Dwell Time vs Fleet Idle Cost**

```dax
DwellTime_vs_FleetIdleCost = 
VAR AvgDwellTime = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        FactWarehouseOperations[OperationType] = "Picking"
    )
VAR TotalIdleMinutes = 
    CALCULATE(
        SUM(FactFleetTrips[IdleTimeMinutes])
    )
VAR IdleCostPerMinute = 0.85 // Cost parameter
RETURN
    DIVIDE(AvgDwellTime, TotalIdleMinutes * IdleCostPerMinute, 0)
```

**Predictive Bottleneck Index**

```dax
BottleneckIndex = 
VAR CurrentCapacity = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[OperationType] IN {"Picking", "Packing"}
    )
VAR HistoricalMax = 
    CALCULATE(
        MAXX(
            SUMMARIZE(
                FactWarehouseOperations,
                DimTime[TimeHour],
                "HourlyOps", COUNTROWS(FactWarehouseOperations)
            ),
            [HourlyOps]
        ),
        DATESINPERIOD(DimTime[FullDateTime], MAX(DimTime[FullDateTime]), -90, DAY)
    )
VAR CapacityUtilization = DIVIDE(CurrentCapacity, HistoricalMax, 0)
VAR WeightedScore = 
    SWITCH(
        TRUE(),
        CapacityUtilization >= 0.95, 10,
        CapacityUtilization >= 0.85, 7,
        CapacityUtilization >= 0.70, 4,
        1
    )
RETURN WeightedScore
```

**Fleet Triage Priority Score**

```dax
FleetTriagePriority = 
VAR MaintenanceAlerts = FactFleetTrips[MaintenanceAlertCount]
VAR LoadValue = 
    CALCULATE(
        SUMX(
            RELATEDTABLE(FactWarehouseOperations),
            FactWarehouseOperations[UnitsProcessed] * 
            RELATED(DimProductGravity[GravityScore])
        )
    )
VAR HarshEvents = 
    FactFleetTrips[HarshBrakingEvents] + 
    FactFleetTrips[HarshAccelerationEvents]
RETURN
    (MaintenanceAlerts * 5) + (LoadValue / 1000) + (HarshEvents * 2)
```

### Row-Level Security

```dax
// RLS expression for geography-based access
[GeographyKey] IN (
    SELECTCOLUMNS(
        FILTER(
            UserRegionAccess,
            UserRegionAccess[UserEmail] = USERPRINCIPALNAME()
        ),
        "GeographyKey", UserRegionAccess[GeographyKey]
    )
)
```

## Alerting System

### Automated Alert Stored Procedure

```sql
CREATE PROCEDURE dbo.usp_GenerateKPIAlerts
AS
BEGIN
    DECLARE @AlertThreshold_DwellTime INT = 120; -- minutes
    DECLARE @AlertThreshold_FleetIdle DECIMAL(5,2) = 0.15; -- 15% of trip
    DECLARE @AlertEmail NVARCHAR(255) = '$(ALERT_EMAIL)';
    
    -- High dwell time alert
    INSERT INTO AlertLog (AlertType, Severity, Message, CreatedDate)
    SELECT 
        'HighDwellTime',
        'Warning',
        CONCAT('SKU ', dp.SKU, ' at ', dg.LocationName, 
               ' has dwell time of ', fwo.DwellTimeMinutes, ' minutes'),
        GETDATE()
    FROM dbo.FactWarehouseOperations fwo
    INNER JOIN dbo.DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    INNER JOIN dbo.DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
    WHERE fwo.DwellTimeMinutes > @AlertThreshold_DwellTime
        AND fwo.OperationStartTime >= DATEADD(HOUR, -1, GETDATE());
    
    -- Fleet idle time alert
    INSERT INTO AlertLog (AlertType, Severity, Message, CreatedDate)
    SELECT 
        'ExcessiveFleetIdle',
        'Critical',
        CONCAT('Vehicle ', VehicleID, ' on route ', RouteID,
               ' idle time: ', IdleTimeMinutes, ' minutes (', 
               CAST((IdleTimeMinutes * 100.0 / NULLIF(ActualDurationMinutes, 0)) AS DECIMAL(5,2)),
               '% of trip)'),
        GETDATE()
    FROM dbo.FactFleetTrips
    WHERE (IdleTimeMinutes * 1.0 / NULLIF(ActualDurationMinutes, 0)) > @AlertThreshold_FleetIdle
        AND TripEndTimeKey IS NOT NULL
        AND TripEndTimeKey >= (SELECT TimeKey FROM dbo.DimTime WHERE FullDateTime >= DATEADD(HOUR, -1, GETDATE()));
    
    -- Send consolidated email via Database Mail
    EXEC msdb.dbo.sp_send_dbmail
        @profile_name = 'LogiFleetPulse',
        @recipients = @AlertEmail,
        @subject = 'LogiFleet Pulse - KPI Alerts',
        @query = 'SELECT TOP 50 * FROM AlertLog WHERE CreatedDate >= DATEADD(HOUR, -1, GETDATE()) ORDER BY Severity DESC, CreatedDate DESC',
        @attach_query_result_as_file = 0;
END;
GO

-- Schedule via SQL Server Agent
EXEC msdb.dbo.sp_add_job
    @job_name = N'LogiFleet_KPI_Alerts',
    @enabled = 1;

EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'LogiFleet_KPI_Alerts',
    @step_name = N'Execute Alert Generation',
    @subsystem = N'TSQL',
    @command = N'EXEC dbo.usp_GenerateKPIAlerts',
    @database_name = N'LogiFleetPulse';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = N'LogiFleet_KPI_Alerts',
    @schedule_name = N'Every15Minutes';
```

## Common Queries and Patterns

### Cross-Fact Analysis: Warehouse Zone Performance by Fleet Route

```sql
-- Link warehouse gravity zones with delivery route efficiency
SELECT 
    dg_wh.LocationName AS WarehouseZone,
    dp.VelocityClass,
    COUNT(DISTINCT fwo.OrderID) AS OrdersProcessed,
    AVG(fwo.DwellTimeMinutes) AS AvgWarehouseDwellTime,
    AVG(fft.IdleTimeMinutes) AS AvgFleetIdleTime,
    AVG(fft.ActualDurationMinutes - fft.PlannedDurationMinutes) AS AvgDeliveryDelay,
    SUM(CASE WHEN fft.OnTimeDelivery = 1 THEN 1 ELSE 0 END) * 100.0 / COUNT(fft.TripKey) AS OnTimePercentage
FROM dbo.FactWarehouseOperations fwo
INNER JOIN dbo.DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
INNER JOIN dbo.DimGeography dg_wh ON fwo.GeographyKey = dg_wh.GeographyKey
INNER JOIN dbo.FactFleetTrips fft ON fwo.OrderID = fft.RouteID -- Join via business key
INNER JOIN dbo.DimGeography dg_route ON fft.OriginGeographyKey = dg_route.GeographyKey
WHERE fwo.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
    AND fwo.OperationType = 'Shipping'
GROUP BY dg_wh.LocationName, dp.VelocityClass
HAVING COUNT(DISTINCT fwo.OrderID) > 10
ORDER BY AvgDeliveryDelay DESC;
```

### Temporal Elasticity Simulation Query

```sql
-- Simulate impact of capacity change on operations
WITH CurrentCapacity AS (
    SELECT 
        dt.TimeHour,
        COUNT(*) AS CurrentOps,
        MAX(COUNT(*)) OVER () AS PeakOps
    FROM dbo.FactWarehouseOperations fwo
    INNER JOIN dbo.DimTime dt ON fwo.TimeKey = dt.TimeKey
    WHERE fwo.OperationStartTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY dt.TimeHour
),
SimulatedCapacity AS (
    SELECT 
        TimeHour,
        CurrentOps,
        PeakOps,
        CurrentOps * 1.0 / NULLIF(PeakOps, 0) AS CurrentUtilization,
        CASE 
            WHEN (CurrentOps * 1.0 / NULLIF(PeakOps, 0)) >= 0.95 THEN CurrentOps * 1.15
            ELSE CurrentOps
        END AS SimulatedOps95Percent
    FROM CurrentCapacity
)
SELECT 
    TimeHour,
    CurrentOps,
    CurrentUtilization,
    SimulatedOps95Percent,
    SimulatedOps95Percent - CurrentOps AS AdditionalCapacityNeeded,
    (SimulatedOps95Percent - CurrentOps) * 12.50 AS EstimatedLaborCostUSD -- $12.50/operation avg
FROM SimulatedCapacity
ORDER BY TimeHour;
```

### Natural Language Query Support (Power BI Q&A)

Configure Q&A synonyms in Power BI:

```
// In Power BI Desktop: Modeling > Q&A Setup
Table: FactWarehouseOperations
Synonyms for "DwellTimeMinutes": 
  - wait time
  - storage duration
  - holding period
  - time in warehouse

Table: FactFleetTrips
Synonyms for "OnTimeDelivery":
  - delivered on time
  - punctual delivery
  - timely arrival
```

## Troubleshooting

### Performance Optimization

**Slow dashboard refresh:**

```sql
-- Check query execution plans
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- Rebuild columnstore indexes monthly
ALTER INDEX NCCI_FactWarehouse ON dbo.FactWarehouseOperations REBUILD;
ALTER INDEX NCCI_FactFleet ON dbo.FactFleetTrips REBUILD;

-- Update statistics after large data loads
UPDATE STATISTICS dbo.FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS dbo.FactFleetTrips WITH FULLSCAN;
```

**Power BI performance tuning:**

```dax
// Use variables to cache calculations
Measure_Optimized = 
VAR FilteredOperations = 
    CALCULATETABLE(
        FactWarehouseOperations,
        FactWarehouseOperations[OperationType] = "Picking"
    )
VAR TotalDwellTime = SUMX(FilteredOperations, [DwellTimeMinutes])
RETURN TotalDwellTime

// Replace iterators with aggregations where possible
// Avoid: SUMX(FILTER(...), ...) 
// Prefer: CALCULATE(SUM(...), ...)
```

### Data Quality Issues

**Missing dimension keys:**

```sql
-- Find orphaned fact records
SELECT 'WarehouseOperations' AS FactTable, COUNT(*) AS OrphanCount
FROM dbo.FactWarehouseOperations fwo
WHERE NOT EXISTS (SELECT 1 FROM dbo.DimProductGravity dp WHERE dp.ProductKey = fwo.ProductKey)

UNION ALL

SELECT 'FleetTrips', COUNT(*)
FROM dbo.FactFleetTrips fft
WHERE NOT EXISTS (SELECT 1 FROM dbo.DimGeography dg WHERE dg.GeographyKey = fft.OriginGeographyKey);

-- Auto-resolve with default/unknown dimension member
UPDATE fwo
SET fwo.ProductKey = -1 -- Unknown product key
FROM dbo.FactWarehouseOperations fwo
WHERE NOT EXISTS (SELECT 1 FROM dbo.DimProductGravity dp WHERE dp.ProductKey = fwo.ProductKey);
```

### Connection Failures

**Power BI gateway issues:**

```powershell
# Test SQL Server connectivity from gateway machine
Test-NetConnection -ComputerName YOUR_SQL_SERVER -Port 1433

# Verify service account permissions
# Gateway service account must have db_datareader on LogiFleetPulse
```

**Timeout errors:**

```sql
-- Increase query timeout in connection string
-- Power BI Desktop: File > Options > Data Load
-- Command Timeout: 600 seconds (10 minutes)

-- Or modify stored procedures
ALTER PROCEDURE dbo.usp_IncrementalLoad_WarehouseOperations
WITH RECOMPILE -- Force new execution plan
AS
BEGIN
    SET LOCK_TIMEOUT 60000; -- 60 seconds
    -- procedure body
END;
```

## Best Practices

1. **Partition large fact tables** by month/quarter for faster queries
2. **Use columnstore indexes** on fact tables for analytical workloads
3. **Implement incremental refresh** in Power BI (Premium required)
4. **Schedule nightly maintenance** for index rebuilds and stats updates
5. **Monitor tempdb usage** during heavy ETL operations
6. **Use query folding** in Power BI to push transformations to SQL Server
7. **Implement data retention policies** (archive data older than 3 years)
8. **Test row-level security** thoroughly before production deployment
9. **Document business logic** in measure descriptions
10. **Version control** all SQL scripts and Power BI templates

## Environment Variables Reference

- `SQL_SERVER` - SQL Server instance name
- `SQL_USER` - Database authentication username
- `SQL_PASSWORD` - Database authentication password
- `WMS_API_URL` - Warehouse Management System API endpoint
- `FLEET_TELEMETRY_URL` - Fleet telematics data feed
- `WEATHER_API_KEY` - External weather service API key
- `ALERT_EMAIL` - Email address for automated alerts
