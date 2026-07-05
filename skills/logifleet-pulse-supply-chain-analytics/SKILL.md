---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for warehouse operations, fleet telemetry, and cross-modal supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure warehouse and fleet data warehouse"
  - "create logistics Power BI dashboard"
  - "implement multi-fact star schema for supply chain"
  - "build warehouse gravity zone analytics"
  - "connect fleet telemetry to SQL Server"
  - "deploy logistics intelligence platform"
  - "create cross-dock analytics dashboard"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a multi-fact star schema data warehouse. Built on MS SQL Server with Power BI visualizations, it provides cross-modal analytics for supply chain optimization, predictive bottleneck detection, and real-time operational intelligence.

**Key Capabilities:**
- Multi-fact star schema with time-phased dimensions
- Warehouse gravity zone optimization
- Fleet telemetry integration and maintenance triage
- Cross-fact KPI harmonization (warehouse + fleet + inventory)
- Predictive bottleneck detection
- Role-based access control and row-level security
- Real-time dashboards with 15-minute refresh intervals

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise)
- Power BI Desktop (latest version)
- Data source access: WMS, TMS, telemetry APIs, ERP systems

### Deployment Steps

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**
```sql
-- Connect to your SQL Server instance
-- Execute the schema creation script
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the complete schema script (included in repository)
-- This creates all fact tables, dimensions, views, and stored procedures
```

3. **Configure data sources:**
```json
// config.json (create from config_sample.json)
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_connection": "${WMS_CONNECTION_STRING}",
    "fleet_telemetry_api": "${FLEET_API_ENDPOINT}",
    "erp_connection": "${ERP_CONNECTION_STRING}",
    "weather_api_key": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

4. **Import Power BI template:**
- Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
- Enter SQL Server connection details when prompted
- Publish to Power BI Service for scheduled refresh

## Core Data Model

### Fact Tables

**FactWarehouseOperations** - Warehouse activity metrics
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType NVARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes INT,
    PickRate DECIMAL(10,2), -- Units per hour
    PackingTimeMinutes INT,
    ErrorCount INT DEFAULT 0,
    RecordedAt DATETIME2 DEFAULT GETDATE(),
    CONSTRAINT FK_Warehouse_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Warehouse_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_Warehouse_Location FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
);

-- Recommended indexing strategy
CREATE NONCLUSTERED INDEX IX_WarehouseOps_Time ON FactWarehouseOperations(TimeKey) INCLUDE (ProductKey, DwellTimeMinutes);
CREATE NONCLUSTERED INDEX IX_WarehouseOps_Product ON FactWarehouseOperations(ProductKey, OperationType);
```

**FactFleetTrips** - Vehicle telemetry and route data
```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DistanceKM DECIMAL(10,2),
    AverageSpeedKMH DECIMAL(10,2),
    DelayMinutes INT DEFAULT 0,
    WeatherImpact BIT DEFAULT 0,
    RecordedAt DATETIME2 DEFAULT GETDATE(),
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_Fleet_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED INDEX IX_FleetTrips_Vehicle_Time ON FactFleetTrips(VehicleKey, TimeKey);
CREATE NONCLUSTERED INDEX IX_FleetTrips_Route ON FactFleetTrips(RouteKey) INCLUDE (FuelConsumedLiters, IdleTimeMinutes);
```

**FactCrossDock** - Cross-docking operations
```sql
CREATE TABLE FactCrossDock (
    CrossDockID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OriginWarehouseKey INT NOT NULL,
    DestinationWarehouseKey INT NOT NULL,
    TransferDwellMinutes INT,
    UnitsTransferred INT,
    RecordedAt DATETIME2 DEFAULT GETDATE(),
    CONSTRAINT FK_CrossDock_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);
```

### Dimension Tables

**DimTime** - Time-phased dimension (15-minute granularity)
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2,
    DateKey INT,
    TimeSlot TIME,
    HourOfDay INT,
    DayOfWeek NVARCHAR(10),
    WeekOfYear INT,
    MonthName NVARCHAR(10),
    Quarter INT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT
);

-- Populate time dimension
INSERT INTO DimTime (TimeKey, FullDateTime, DateKey, TimeSlot, HourOfDay, DayOfWeek, WeekOfYear, MonthName, Quarter, FiscalYear, IsWeekend, IsHoliday)
SELECT 
    CAST(FORMAT(dt, 'yyyyMMddHHmm') AS INT) AS TimeKey,
    dt AS FullDateTime,
    CAST(FORMAT(dt, 'yyyyMMdd') AS INT) AS DateKey,
    CAST(dt AS TIME) AS TimeSlot,
    DATEPART(HOUR, dt) AS HourOfDay,
    DATENAME(WEEKDAY, dt) AS DayOfWeek,
    DATEPART(WEEK, dt) AS WeekOfYear,
    DATENAME(MONTH, dt) AS MonthName,
    DATEPART(QUARTER, dt) AS Quarter,
    YEAR(dt) AS FiscalYear,
    CASE WHEN DATEPART(WEEKDAY, dt) IN (1, 7) THEN 1 ELSE 0 END AS IsWeekend,
    0 AS IsHoliday
FROM (
    SELECT DATEADD(MINUTE, ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) * 15, '2025-01-01') AS dt
    FROM sys.all_objects a CROSS JOIN sys.all_objects b
) AS DateSequence
WHERE dt < '2030-01-01';
```

**DimProductGravity** - Product classification with gravity scoring
```sql
CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    ProductSKU NVARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * (1/fragility)
    IsPerishable BIT DEFAULT 0,
    AverageWeight DECIMAL(10,2),
    WarehouseZone NVARCHAR(50), -- Assigned based on gravity
    LastGravityUpdate DATETIME2
);

-- Calculate and update gravity scores
CREATE PROCEDURE UpdateProductGravity
AS
BEGIN
    UPDATE p
    SET GravityScore = (
        -- Velocity component (picks per day)
        (SELECT COUNT(*) / 30.0 FROM FactWarehouseOperations WHERE ProductKey = p.ProductKey 
         AND OperationType = 'Picking' AND TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd') AS INT))
        * 
        -- Value component (normalized price)
        (p.UnitPrice / 100.0)
        *
        -- Fragility factor (inverted)
        (1.0 / CASE p.FragilityRating WHEN 'High' THEN 3 WHEN 'Medium' THEN 2 ELSE 1 END)
    ),
    LastGravityUpdate = GETDATE()
    FROM DimProduct p;
    
    -- Reassign warehouse zones based on gravity quartiles
    UPDATE DimProduct
    SET WarehouseZone = CASE 
        WHEN GravityScore >= (SELECT PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY GravityScore) OVER() FROM DimProduct) THEN 'HighGravity'
        WHEN GravityScore >= (SELECT PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY GravityScore) OVER() FROM DimProduct) THEN 'MediumGravity'
        WHEN GravityScore >= (SELECT PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY GravityScore) OVER() FROM DimProduct) THEN 'LowGravity'
        ELSE 'SlowMover'
    END;
END;
```

**DimVehicle** - Fleet assets with maintenance tracking
```sql
CREATE TABLE DimVehicle (
    VehicleKey INT IDENTITY(1,1) PRIMARY KEY,
    VehicleID NVARCHAR(50) UNIQUE NOT NULL,
    VehicleType NVARCHAR(50), -- 'Box Truck', 'Refrigerated', 'Flatbed'
    Capacity DECIMAL(10,2),
    MaintenanceScore DECIMAL(5,2), -- 0-100, calculated from telemetry
    LastMaintenanceDate DATE,
    TirePressureStatus NVARCHAR(20),
    EngineHealthStatus NVARCHAR(20),
    IsActive BIT DEFAULT 1
);
```

## Common Queries and Patterns

### Cross-Fact KPI Analysis

**Query: Dwell time vs. fleet idling cost per route**
```sql
CREATE VIEW vw_DwellTimeVsFleetCost AS
SELECT 
    t.DateKey,
    r.RouteName,
    w.WarehouseName,
    -- Warehouse metrics
    AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
    SUM(CASE WHEN wo.DwellTimeMinutes > 72 THEN 1 ELSE 0 END) AS HighDwellCount,
    -- Fleet metrics
    AVG(ft.IdleTimeMinutes) AS AvgIdleTimeMinutes,
    SUM(ft.FuelConsumedLiters * 1.50) AS EstimatedIdlingCostUSD, -- $1.50 per liter average
    -- Correlation indicator
    CASE 
        WHEN AVG(wo.DwellTimeMinutes) > 72 AND AVG(ft.IdleTimeMinutes) > 30 THEN 'HighRisk'
        WHEN AVG(wo.DwellTimeMinutes) > 48 OR AVG(ft.IdleTimeMinutes) > 20 THEN 'MediumRisk'
        ELSE 'Normal'
    END AS CorrelationRisk
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
LEFT JOIN FactFleetTrips ft ON t.DateKey = CAST(FORMAT(ft.RecordedAt, 'yyyyMMdd') AS INT)
LEFT JOIN DimRoute r ON ft.RouteKey = r.RouteKey
WHERE t.DateKey >= CAST(FORMAT(DATEADD(MONTH, -3, GETDATE()), 'yyyyMMdd') AS INT)
GROUP BY t.DateKey, r.RouteName, w.WarehouseName;
```

### Predictive Bottleneck Detection

**Stored procedure: Identify high-risk congestion points**
```sql
CREATE PROCEDURE DetectBottlenecks
    @ThresholdHours INT = 72,
    @MinOccurrences INT = 5
AS
BEGIN
    -- Identify warehouse operations with recurring high dwell times
    SELECT 
        p.ProductSKU,
        p.ProductName,
        w.WarehouseName,
        COUNT(*) AS HighDwellOccurrences,
        AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes,
        MAX(wo.DwellTimeMinutes) AS MaxDwellMinutes,
        p.GravityScore,
        CASE 
            WHEN p.GravityScore > 50 THEN 'CRITICAL - High-value product affected'
            WHEN COUNT(*) > @MinOccurrences * 2 THEN 'WARNING - Frequent congestion'
            ELSE 'MONITOR'
        END AS SeverityLevel
    FROM FactWarehouseOperations wo
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE wo.DwellTimeMinutes > (@ThresholdHours * 60)
      AND t.DateKey >= CAST(FORMAT(DATEADD(MONTH, -1, GETDATE()), 'yyyyMMdd') AS INT)
    GROUP BY p.ProductSKU, p.ProductName, w.WarehouseName, p.GravityScore
    HAVING COUNT(*) >= @MinOccurrences
    ORDER BY 
        CASE 
            WHEN p.GravityScore > 50 THEN 1
            WHEN COUNT(*) > @MinOccurrences * 2 THEN 2
            ELSE 3
        END,
        AvgDwellMinutes DESC;
END;
```

### Fleet Maintenance Triage

**Query: Prioritize fleet maintenance by revenue impact**
```sql
SELECT 
    v.VehicleID,
    v.VehicleType,
    v.MaintenanceScore,
    v.TirePressureStatus,
    v.EngineHealthStatus,
    -- Calculate revenue impact
    SUM(ft.DistanceKM * 2.50) AS RevenueAtRiskUSD, -- $2.50 per KM average margin
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    COUNT(DISTINCT ft.RouteKey) AS ActiveRoutes,
    -- Priority scoring
    (
        (100 - v.MaintenanceScore) * 0.4 + -- Maintenance urgency weight
        (AVG(ft.IdleTimeMinutes) / 60.0) * 0.3 + -- Idle time weight
        (SUM(ft.DistanceKM * 2.50) / 10000.0) * 0.3 -- Revenue weight
    ) AS PriorityScore
FROM DimVehicle v
INNER JOIN FactFleetTrips ft ON v.VehicleKey = ft.VehicleKey
INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
WHERE t.DateKey >= CAST(FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMdd') AS INT)
  AND (v.MaintenanceScore < 70 OR v.TirePressureStatus = 'Low' OR v.EngineHealthStatus != 'Good')
GROUP BY v.VehicleID, v.VehicleType, v.MaintenanceScore, v.TirePressureStatus, v.EngineHealthStatus
ORDER BY PriorityScore DESC;
```

## Data Integration Patterns

### Incremental Loading from WMS

```sql
CREATE PROCEDURE LoadWarehouseOperations
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    INSERT INTO FactWarehouseOperations (TimeKey, ProductKey, WarehouseKey, OperationType, DwellTimeMinutes, PickRate, PackingTimeMinutes, ErrorCount)
    SELECT 
        CAST(FORMAT(wms.EventTimestamp, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        p.ProductKey,
        w.WarehouseKey,
        wms.OperationType,
        DATEDIFF(MINUTE, wms.StartTime, wms.EndTime) AS DwellTimeMinutes,
        CASE WHEN wms.OperationType = 'Picking' THEN wms.UnitsProcessed / NULLIF(DATEDIFF(HOUR, wms.StartTime, wms.EndTime), 0) END AS PickRate,
        CASE WHEN wms.OperationType = 'Packing' THEN DATEDIFF(MINUTE, wms.StartTime, wms.EndTime) END AS PackingTimeMinutes,
        wms.ErrorCount
    FROM OPENQUERY([WMS_SERVER], '
        SELECT * FROM WMSEvents 
        WHERE LastModified > ''' + CONVERT(NVARCHAR, @LastLoadTimestamp, 120) + '''
    ') AS wms
    INNER JOIN DimProduct p ON wms.SKU = p.ProductSKU
    INNER JOIN DimWarehouse w ON wms.WarehouseCode = w.WarehouseCode
    WHERE NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations f
        WHERE f.TimeKey = CAST(FORMAT(wms.EventTimestamp, 'yyyyMMddHHmm') AS INT)
          AND f.ProductKey = p.ProductKey
    );
    
    -- Update last load timestamp
    UPDATE ETLControl SET LastLoadTimestamp = GETDATE() WHERE TableName = 'FactWarehouseOperations';
END;
```

### Fleet Telemetry API Integration

```sql
-- External table for streaming telemetry (requires PolyBase)
CREATE EXTERNAL DATA SOURCE FleetTelemetryAPI
WITH (
    TYPE = REST_API,
    LOCATION = '${FLEET_API_ENDPOINT}',
    CREDENTIAL = FleetAPICredential
);

-- Staging procedure for API data
CREATE PROCEDURE StageFleetTelemetry
AS
BEGIN
    -- Fetch from API and stage
    INSERT INTO StagingFleetTelemetry (VehicleID, Timestamp, Latitude, Longitude, FuelLevel, Speed, EngineTemp)
    SELECT * FROM OPENROWSET(
        BULK '',
        DATA_SOURCE = 'FleetTelemetryAPI',
        FORMAT = 'CSV',
        PARSER_VERSION = '2.0'
    ) AS TelemetryData;
    
    -- Transform and load into fact table
    INSERT INTO FactFleetTrips (TimeKey, VehicleKey, RouteKey, DriverKey, FuelConsumedLiters, IdleTimeMinutes, DistanceKM, AverageSpeedKMH)
    SELECT 
        CAST(FORMAT(st.Timestamp, 'yyyyMMddHHmm') AS INT),
        v.VehicleKey,
        r.RouteKey,
        d.DriverKey,
        LAG(st.FuelLevel) OVER (PARTITION BY st.VehicleID ORDER BY st.Timestamp) - st.FuelLevel AS FuelConsumed,
        SUM(CASE WHEN st.Speed = 0 THEN 1 ELSE 0 END) AS IdleMinutes,
        SUM(st.DistanceDelta) AS TotalDistance,
        AVG(st.Speed) AS AvgSpeed
    FROM StagingFleetTelemetry st
    INNER JOIN DimVehicle v ON st.VehicleID = v.VehicleID
    LEFT JOIN DimRoute r ON st.RouteID = r.RouteID
    LEFT JOIN DimDriver d ON st.DriverID = d.DriverID
    GROUP BY st.VehicleID, v.VehicleKey, r.RouteKey, d.DriverKey, st.Timestamp;
END;
```

## Power BI Configuration

### DAX Measures for Cross-Fact Analysis

**Composite Dwell-to-Idle Ratio:**
```dax
DwellIdleRatio = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR AvgIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
RETURN
DIVIDE(AvgDwell, AvgIdle, 0)
```

**Warehouse Gravity Efficiency:**
```dax
GravityEfficiency = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[PickRate]),
    DimProduct[WarehouseZone] = "HighGravity"
) / 
CALCULATE(
    AVERAGE(FactWarehouseOperations[PickRate]),
    DimProduct[WarehouseZone] = "SlowMover"
)
```

**Fleet Utilization Index:**
```dax
FleetUtilization = 
VAR TotalTripTime = SUM(FactFleetTrips[DistanceKM]) / AVERAGE(FactFleetTrips[AverageSpeedKMH]) * 60
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN
DIVIDE(TotalTripTime, TotalTripTime + TotalIdleTime, 0) * 100
```

### Row-Level Security

```dax
-- Create role: Regional Managers
[DimWarehouse[Region]] = USERPRINCIPALNAME()

-- Create role: Fleet Supervisors
[DimVehicle[AssignedSupervisor]] = USERNAME()
```

## Alerts and Automation

### SQL Server Agent Job for Threshold Alerts

```sql
CREATE PROCEDURE SendBottleneckAlerts
AS
BEGIN
    DECLARE @AlertBody NVARCHAR(MAX);
    
    -- Build alert message
    SELECT @AlertBody = STRING_AGG(
        'ALERT: ' + ProductSKU + ' in ' + WarehouseName + 
        ' - Dwell: ' + CAST(AvgDwellMinutes AS NVARCHAR) + ' min - Severity: ' + SeverityLevel,
        CHAR(13) + CHAR(10)
    )
    FROM (
        EXEC DetectBottlenecks @ThresholdHours = 48, @MinOccurrences = 3
    ) AS Bottlenecks
    WHERE SeverityLevel IN ('CRITICAL - High-value product affected', 'WARNING - Frequent congestion');
    
    -- Send email via Database Mail
    IF @AlertBody IS NOT NULL
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetPulseAlerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'LogiFleet Pulse - Bottleneck Detection Alert',
            @body = @AlertBody;
    END;
END;

-- Schedule job to run every 4 hours
EXEC msdb.dbo.sp_add_job @job_name = 'LogiFleet_BottleneckMonitoring';
EXEC msdb.dbo.sp_add_jobstep @job_name = 'LogiFleet_BottleneckMonitoring', @step_name = 'RunDetection', @command = 'EXEC SendBottleneckAlerts';
EXEC msdb.dbo.sp_add_schedule @schedule_name = 'Every4Hours', @freq_type = 4, @freq_interval = 1, @freq_subday_type = 8, @freq_subday_interval = 4;
EXEC msdb.dbo.sp_attach_schedule @job_name = 'LogiFleet_BottleneckMonitoring', @schedule_name = 'Every4Hours';
```

## Troubleshooting

### Performance Optimization

**Issue: Slow cross-fact queries**
```sql
-- Ensure columnstore indexes on large fact tables
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_WarehouseOps 
ON FactWarehouseOperations (TimeKey, ProductKey, DwellTimeMinutes, PickRate);

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FleetTrips
ON FactFleetTrips (TimeKey, VehicleKey, FuelConsumedLiters, IdleTimeMinutes);

-- Partition large tables by date range
ALTER DATABASE LogiFleetPulse 
ADD FILEGROUP FG_2025, FG_2026;

CREATE PARTITION FUNCTION PF_DateKey (INT)
AS RANGE RIGHT FOR VALUES (20250101, 20260101);

CREATE PARTITION SCHEME PS_DateKey
AS PARTITION PF_DateKey TO (FG_2025, FG_2026, [PRIMARY]);
```

**Issue: Power BI refresh timeouts**
```powerquery
// In Power Query, use incremental refresh
// Set RangeStart and RangeEnd parameters
let
    Source = Sql.Database("${SQL_SERVER_HOST}", "LogiFleetPulse"),
    FilteredRows = Table.SelectRows(Source, each [TimeKey] >= Number.From(DateTime.ToText(RangeStart, "yyyyMMddHHmm")) and [TimeKey] < Number.From(DateTime.ToText(RangeEnd, "yyyyMMddHHmm")))
in
    FilteredRows
```

### Data Quality Checks

**Identify missing dimension relationships:**
```sql
-- Orphaned warehouse operations (no product match)
SELECT COUNT(*) AS OrphanedRecords
FROM FactWarehouseOperations wo
LEFT JOIN DimProduct p ON wo.ProductKey = p.ProductKey
WHERE p.ProductKey IS NULL;

-- Fleet trips with invalid time keys
SELECT COUNT(*) AS InvalidTimeKeys
FROM FactFleetTrips ft
LEFT JOIN DimTime t ON ft.TimeKey = t.TimeKey
WHERE t.TimeKey IS NULL;
```

**Monitor data freshness:**
```sql
SELECT 
    'FactWarehouseOperations' AS TableName,
    MAX(RecordedAt) AS LastRecord,
    DATEDIFF(MINUTE, MAX(RecordedAt), GETDATE()) AS MinutesSinceLastUpdate
FROM FactWarehouseOperations
UNION ALL
SELECT 
    'FactFleetTrips',
    MAX(RecordedAt),
    DATEDIFF(MINUTE, MAX(RecordedAt), GETDATE())
FROM FactFleetTrips;
```

## Best Practices

1. **Incremental Loading**: Always use watermark-based incremental loads with `LastLoadTimestamp` tracking
2. **Gravity Score Updates**: Run `UpdateProductGravity` weekly during off-peak hours
3. **Index Maintenance**: Rebuild columnstore indexes monthly; update statistics weekly
4. **Security**: Store all credentials in environment variables; never hardcode in scripts
5. **Testing**: Use separate schema (e.g., `Staging`) for ETL testing before production merge
6. **Documentation**: Annotate all custom DAX measures with business logic comments
7. **Monitoring**: Set up SQL Server Extended Events for query performance tracking above 5 seconds

## Environment Variables Reference

Required environment variables for deployment:

```bash
SQL_SERVER_HOST=your-sql-server.database.windows.net
SQL_USER=logifleet_admin
SQL_PASSWORD=<secure-password>
WMS_CONNECTION_STRING=<wms-odbc-or-api-connection>
FLEET_API_ENDPOINT=https://fleet-telemetry.example.com/api/v1
ERP_CONNECTION_STRING=<erp-connection-string>
WEATHER_API_KEY=<external-weather-api-key>
ALERT_EMAIL_RECIPIENTS=logistics-team@company.com;operations@company.com
```
