---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for warehouse, fleet, and supply chain analytics
triggers:
  - set up logifleet pulse analytics warehouse
  - configure power bi logistics dashboard
  - deploy supply chain data model
  - implement warehouse fleet analytics
  - create logistics intelligence platform
  - build multi-fact star schema for logistics
  - integrate warehouse and fleet data sources
  - set up real-time logistics dashboards
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive MS SQL Server data warehouse and Power BI visualization platform for logistics and supply chain analytics. It provides:

- **Multi-fact star schema** linking warehouse operations, fleet telemetry, and supply chain data
- **Time-phased dimensions** with 15-minute granularity for real-time operational tracking
- **Cross-fact KPI harmonization** connecting inventory metrics with fleet performance
- **Power BI dashboards** for warehouse velocity, fleet optimization, and predictive bottleneck detection
- **Role-based security** with row-level filtering for enterprise deployments

The platform integrates data from warehouse management systems (WMS), telematics/GPS feeds, supplier portals, and external APIs to create a unified semantic layer for logistics intelligence.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, TMS, telematics APIs

### Database Deployment

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**

```sql
-- Connect to your SQL Server instance via SSMS
-- Execute the schema creation script
:r ./sql/01_CreateDatabase.sql
:r ./sql/02_CreateDimensions.sql
:r ./sql/03_CreateFacts.sql
:r ./sql/04_CreateViews.sql
:r ./sql/05_CreateStoredProcedures.sql
```

3. **Configure data connections:**

```sql
-- Update connection strings in config table
USE LogiFleetPulse;
GO

INSERT INTO ConfigDataSources (SourceName, ConnectionString, SourceType, RefreshIntervalMinutes)
VALUES 
  ('WMS_Primary', 'Server=$(WMS_SERVER);Database=$(WMS_DB);User=$(WMS_USER);Password=$(WMS_PASSWORD)', 'WMS', 15),
  ('FleetTelemetry', '$(TELEMETRY_API_ENDPOINT)', 'API', 5),
  ('SupplierPortal', 'Server=$(ERP_SERVER);Database=$(ERP_DB);Integrated Security=true', 'ERP', 60);
```

### Power BI Template Setup

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter connection parameters when prompted:
   - SQL Server instance name
   - Database name: `LogiFleetPulse`
   - Authentication method (Windows or SQL Server)

3. Configure refresh schedule in Power BI Service (if publishing):

```powerquery
// Edit parameters in Power Query Editor
let
    ServerName = #"SQL_SERVER_NAME",
    DatabaseName = "LogiFleetPulse",
    Source = Sql.Database(ServerName, DatabaseName)
in
    Source
```

## Core Data Model

### Key Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
SELECT 
    TimeKey,
    FullDateTime,
    DateKey,
    TimeOf15Min,
    HourOfDay,
    DayOfWeek,
    FiscalPeriod
FROM DimTime
WHERE DateKey = CONVERT(VARCHAR(8), GETDATE(), 112);

-- DimProductGravity: Products classified by velocity and value
SELECT 
    ProductKey,
    ProductSKU,
    ProductName,
    GravityScore,  -- 1-100, higher = faster moving
    StorageZoneRecommendation,
    PickFrequencyCategory
FROM DimProductGravity
WHERE GravityScore > 70;

-- DimGeography: Hierarchical location dimension
SELECT 
    GeographyKey,
    LocationCode,
    LocationName,
    LocationType,  -- 'Warehouse', 'RouteNode', 'CustomerSite'
    RegionName,
    CountryName
FROM DimGeography;
```

### Key Fact Tables

```sql
-- FactWarehouseOperations: Warehouse activity metrics
SELECT 
    OperationKey,
    TimeKey,
    ProductKey,
    GeographyKey,
    OperationType,  -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled,
    DwellTimeMinutes,
    CycleTimeSeconds,
    LabourHours
FROM FactWarehouseOperations
WHERE TimeKey >= CONVERT(VARCHAR(8), GETDATE()-1, 112) + '0000';

-- FactFleetTrips: Vehicle and route performance
SELECT 
    TripKey,
    TimeKey,
    VehicleKey,
    RouteKey,
    OriginGeographyKey,
    DestinationGeographyKey,
    DistanceKM,
    FuelLitres,
    IdleTimeMinutes,
    LoadWeightKG,
    TripDurationMinutes
FROM FactFleetTrips
WHERE TripStatus = 'Completed';

-- FactCrossDock: Transfer operations
SELECT 
    CrossDockKey,
    TimeKey,
    ProductKey,
    InboundShipmentKey,
    OutboundShipmentKey,
    TransferTimeMinutes,
    QuantityTransferred
FROM FactCrossDock;
```

## Common Analytical Patterns

### Cross-Fact KPI: Dwell Time vs Fleet Idle Cost

```sql
-- Correlate warehouse dwell with fleet idle time
WITH WarehouseDwell AS (
    SELECT 
        w.ProductKey,
        w.GeographyKey,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateKey >= CONVERT(VARCHAR(8), GETDATE()-7, 112)
        AND w.OperationType IN ('Receiving', 'Putaway')
    GROUP BY w.ProductKey, w.GeographyKey
),
FleetIdle AS (
    SELECT 
        f.OriginGeographyKey,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        SUM(f.IdleTimeMinutes * 0.45) AS TotalIdleCost  -- $0.45/min cost assumption
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.DateKey >= CONVERT(VARCHAR(8), GETDATE()-7, 112)
    GROUP BY f.OriginGeographyKey
)
SELECT 
    g.LocationName,
    wd.AvgDwellTime,
    fi.AvgIdleTime,
    fi.TotalIdleCost,
    CASE 
        WHEN wd.AvgDwellTime > 120 AND fi.AvgIdleTime > 30 
        THEN 'High Risk - Both Elevated'
        WHEN wd.AvgDwellTime > 120 
        THEN 'Warehouse Bottleneck'
        WHEN fi.AvgIdleTime > 30 
        THEN 'Fleet Inefficiency'
        ELSE 'Normal'
    END AS AlertStatus
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.GeographyKey = fi.OriginGeographyKey
INNER JOIN DimGeography g ON wd.GeographyKey = g.GeographyKey
ORDER BY fi.TotalIdleCost DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products in wrong storage zones
SELECT 
    p.ProductSKU,
    p.ProductName,
    p.GravityScore,
    p.StorageZoneRecommendation AS RecommendedZone,
    g.LocationName AS CurrentZone,
    COUNT(w.OperationKey) AS PickCount,
    AVG(w.CycleTimeSeconds) AS AvgPickTime,
    CASE 
        WHEN p.GravityScore > 80 AND g.LocationName NOT LIKE '%Fast%' 
        THEN 'Relocate to Fast Zone'
        WHEN p.GravityScore < 30 AND g.LocationName LIKE '%Fast%' 
        THEN 'Relocate to Slow Zone'
        ELSE 'Optimal'
    END AS Action
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE t.DateKey >= CONVERT(VARCHAR(8), GETDATE()-30, 112)
    AND w.OperationType = 'Picking'
GROUP BY p.ProductSKU, p.ProductName, p.GravityScore, 
         p.StorageZoneRecommendation, g.LocationName
HAVING COUNT(w.OperationKey) > 10
ORDER BY p.GravityScore DESC;
```

### Predictive Bottleneck Detection

```sql
-- Stored procedure for bottleneck prediction
CREATE PROCEDURE usp_PredictBottlenecks
    @ForecastHours INT = 24
AS
BEGIN
    -- Historical pattern analysis
    WITH HourlyPatterns AS (
        SELECT 
            t.HourOfDay,
            t.DayOfWeek,
            AVG(w.QuantityHandled) AS AvgVolume,
            MAX(w.QuantityHandled) AS PeakVolume,
            COUNT(DISTINCT w.GeographyKey) AS ActiveZones
        FROM FactWarehouseOperations w
        INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
        WHERE t.DateKey >= CONVERT(VARCHAR(8), GETDATE()-90, 112)
        GROUP BY t.HourOfDay, t.DayOfWeek
    ),
    CurrentCapacity AS (
        SELECT 
            GeographyKey,
            SUM(LabourHours) AS AvailableLabourHours
        FROM FactWarehouseOperations
        WHERE TimeKey >= CONVERT(VARCHAR(8), GETDATE(), 112) + 
              RIGHT('0' + CAST(DATEPART(HOUR, GETDATE()) AS VARCHAR), 2) + '00'
        GROUP BY GeographyKey
    )
    SELECT 
        DATEADD(HOUR, ROW_NUMBER() OVER (ORDER BY hp.HourOfDay), GETDATE()) AS PredictedTime,
        hp.HourOfDay,
        hp.AvgVolume,
        hp.PeakVolume,
        cc.AvailableLabourHours,
        CASE 
            WHEN hp.PeakVolume > cc.AvailableLabourHours * 15  -- 15 units/labour hour threshold
            THEN 'Critical Bottleneck Predicted'
            WHEN hp.AvgVolume > cc.AvailableLabourHours * 12
            THEN 'Potential Bottleneck'
            ELSE 'Normal'
        END AS BottleneckRisk
    FROM HourlyPatterns hp
    CROSS JOIN CurrentCapacity cc
    WHERE hp.DayOfWeek = DATEPART(WEEKDAY, GETDATE())
    ORDER BY hp.HourOfDay;
END;
GO

-- Execute bottleneck prediction
EXEC usp_PredictBottlenecks @ForecastHours = 24;
```

## Data Loading & ETL

### Incremental Load Pattern

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTimeKey INT = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Get last load time if not provided
    IF @LastLoadTimeKey IS NULL
        SELECT @LastLoadTimeKey = MAX(TimeKey) FROM FactWarehouseOperations;
    
    -- Load new records from WMS staging
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, GeographyKey, OperationType,
        QuantityHandled, DwellTimeMinutes, CycleTimeSeconds, LabourHours
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        stg.OperationType,
        stg.Quantity,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DwellTime,
        DATEDIFF(SECOND, stg.StartTime, stg.EndTime) AS CycleTime,
        stg.LabourHours
    FROM StagingWarehouseOperations stg
    INNER JOIN DimTime t ON CONVERT(VARCHAR(12), stg.OperationTime, 114) = t.FullDateTime
    INNER JOIN DimProductGravity p ON stg.SKU = p.ProductSKU
    INNER JOIN DimGeography g ON stg.LocationCode = g.LocationCode
    WHERE t.TimeKey > @LastLoadTimeKey;
    
    -- Log load completion
    INSERT INTO ETLLog (ProcedureName, RowsLoaded, LoadTime)
    VALUES ('usp_LoadWarehouseOperations', @@ROWCOUNT, GETDATE());
END;
GO
```

### External API Integration

```sql
-- Load fleet telemetry from REST API (using OPENROWSET)
INSERT INTO FactFleetTrips (
    TimeKey, VehicleKey, RouteKey, OriginGeographyKey, 
    DestinationGeographyKey, DistanceKM, FuelLitres, IdleTimeMinutes
)
SELECT 
    t.TimeKey,
    v.VehicleKey,
    r.RouteKey,
    og.GeographyKey AS OriginKey,
    dg.GeographyKey AS DestKey,
    json.distance,
    json.fuel_consumed,
    json.idle_minutes
FROM OPENROWSET(
    BULK '$(TELEMETRY_API_ENDPOINT)/trips/latest',
    FORMATFILE = 'json',
    FORMATFILE_DATA_SOURCE = 'TelemetryAPI'
) AS json
INNER JOIN DimTime t ON json.timestamp = t.FullDateTime
INNER JOIN DimVehicle v ON json.vehicle_id = v.VehicleID
INNER JOIN DimRoute r ON json.route_id = r.RouteID
INNER JOIN DimGeography og ON json.origin_code = og.LocationCode
INNER JOIN DimGeography dg ON json.destination_code = dg.LocationCode;
```

## Power BI DAX Measures

### Fleet Efficiency Ratio

```dax
FleetEfficiencyRatio = 
VAR TotalTripTime = SUM(FactFleetTrips[TripDurationMinutes])
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN
DIVIDE(
    TotalTripTime - TotalIdleTime,
    TotalTripTime,
    0
)
```

### Warehouse Throughput Rate

```dax
ThroughputRate = 
DIVIDE(
    SUM(FactWarehouseOperations[QuantityHandled]),
    SUM(FactWarehouseOperations[LabourHours]),
    0
)
```

### Cross-Dock Efficiency

```dax
CrossDockEfficiency = 
VAR AvgTransferTime = AVERAGE(FactCrossDock[TransferTimeMinutes])
VAR TargetTime = 45  // Target: 45 minutes
RETURN
IF(
    AvgTransferTime <= TargetTime,
    1,
    DIVIDE(TargetTime, AvgTransferTime, 0)
)
```

### Time Intelligence: Rolling 7-Day Average

```dax
DwellTime_7DayAvg = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATESINPERIOD(
        DimTime[Date],
        LASTDATE(DimTime[Date]),
        -7,
        DAY
    )
)
```

## Configuration Management

### Environment Variables

Set these environment variables for deployment automation:

```bash
# SQL Server connection
export SQL_SERVER_NAME="your-server.database.windows.net"
export SQL_DATABASE="LogiFleetPulse"
export SQL_USER="sqladmin"
export SQL_PASSWORD="your-secure-password"

# Data source endpoints
export WMS_SERVER="wms-server.company.local"
export WMS_DB="WarehouseManagement"
export TELEMETRY_API_ENDPOINT="https://api.telematics.company.com/v2"
export ERP_SERVER="erp-server.company.local"

# Power BI Service
export POWERBI_WORKSPACE_ID="your-workspace-guid"
export POWERBI_CLIENT_ID="your-app-registration-id"
export POWERBI_CLIENT_SECRET="your-client-secret"
```

### Row-Level Security Setup

```sql
-- Create security roles in Power BI
-- Define security table in SQL
CREATE TABLE SecurityUserRoles (
    UserEmail NVARCHAR(255),
    RoleName NVARCHAR(100),
    GeographyKey INT,
    CONSTRAINT FK_Security_Geography FOREIGN KEY (GeographyKey) 
        REFERENCES DimGeography(GeographyKey)
);

-- Populate with user assignments
INSERT INTO SecurityUserRoles (UserEmail, RoleName, GeographyKey)
VALUES 
    ('warehouse.manager@company.com', 'WarehouseManager', 1001),
    ('fleet.supervisor@company.com', 'FleetSupervisor', NULL),  -- NULL = all locations
    ('executive@company.com', 'Executive', NULL);

-- Create RLS function
CREATE FUNCTION fn_SecurityFilter(@UserEmail NVARCHAR(255))
RETURNS TABLE
AS
RETURN
(
    SELECT GeographyKey
    FROM SecurityUserRoles
    WHERE UserEmail = @UserEmail
        OR RoleName = 'Executive'  -- Executives see all
);
```

## Alerting & Automation

### SQL Server Agent Job for Threshold Alerts

```sql
-- Create alert check procedure
CREATE PROCEDURE usp_CheckAlertThresholds
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check for excessive dwell time
    IF EXISTS (
        SELECT 1 
        FROM FactWarehouseOperations w
        INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
        WHERE t.DateKey = CONVERT(VARCHAR(8), GETDATE(), 112)
            AND w.DwellTimeMinutes > 180
        GROUP BY w.GeographyKey
        HAVING COUNT(*) > 5
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Excessive dwell time detected at multiple locations';
        
        -- Send via Database Mail
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'operations@company.com',
            @subject = 'LogiFleet Pulse: Dwell Time Alert',
            @body = @AlertMessage;
    END;
    
    -- Check for fleet idle time
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips f
        INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
        WHERE t.DateKey = CONVERT(VARCHAR(8), GETDATE(), 112)
            AND CAST(f.IdleTimeMinutes AS FLOAT) / f.TripDurationMinutes > 0.20
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeds 20% on multiple trips today';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'fleet@company.com',
            @subject = 'LogiFleet Pulse: Fleet Idle Alert',
            @body = @AlertMessage;
    END;
END;
GO

-- Schedule job to run every 30 minutes
USE msdb;
GO
EXEC sp_add_job @job_name = 'LogiFleet_AlertCheck';
EXEC sp_add_jobstep 
    @job_name = 'LogiFleet_AlertCheck',
    @step_name = 'Check Thresholds',
    @subsystem = 'TSQL',
    @command = 'EXEC LogiFleetPulse.dbo.usp_CheckAlertThresholds',
    @database_name = 'LogiFleetPulse';
EXEC sp_add_schedule 
    @schedule_name = 'Every30Minutes',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 30;
EXEC sp_attach_schedule 
    @job_name = 'LogiFleet_AlertCheck',
    @schedule_name = 'Every30Minutes';
EXEC sp_add_jobserver 
    @job_name = 'LogiFleet_AlertCheck';
```

## Troubleshooting

### Performance Optimization

**Issue**: Slow cross-fact queries spanning warehouse and fleet data

**Solution**: Create indexed views for common join patterns

```sql
-- Create indexed view for cross-fact analysis
CREATE VIEW vw_WarehouseFleetCorrelation
WITH SCHEMABINDING
AS
SELECT 
    w.GeographyKey,
    w.TimeKey,
    SUM(w.DwellTimeMinutes) AS TotalDwellTime,
    COUNT_BIG(*) AS OperationCount,
    SUM(f.IdleTimeMinutes) AS TotalFleetIdle
FROM dbo.FactWarehouseOperations w
INNER JOIN dbo.FactFleetTrips f 
    ON w.GeographyKey = f.OriginGeographyKey
    AND w.TimeKey = f.TimeKey
GROUP BY w.GeographyKey, w.TimeKey;
GO

-- Create clustered index on view
CREATE UNIQUE CLUSTERED INDEX IX_WarehouseFleet 
ON vw_WarehouseFleetCorrelation (GeographyKey, TimeKey);
```

### Data Quality Issues

**Issue**: Missing dimension keys causing orphaned fact records

**Solution**: Implement ETL validation checks

```sql
-- Validate staging data before load
CREATE PROCEDURE usp_ValidateStagingData
AS
BEGIN
    -- Check for invalid product SKUs
    SELECT 'Invalid Products' AS IssueType, COUNT(*) AS RecordCount
    FROM StagingWarehouseOperations s
    LEFT JOIN DimProductGravity p ON s.SKU = p.ProductSKU
    WHERE p.ProductKey IS NULL
    
    UNION ALL
    
    -- Check for invalid locations
    SELECT 'Invalid Locations', COUNT(*)
    FROM StagingWarehouseOperations s
    LEFT JOIN DimGeography g ON s.LocationCode = g.LocationCode
    WHERE g.GeographyKey IS NULL
    
    UNION ALL
    
    -- Check for future-dated transactions
    SELECT 'Future Dates', COUNT(*)
    FROM StagingWarehouseOperations
    WHERE OperationTime > GETDATE();
END;
GO
```

### Power BI Refresh Failures

**Issue**: Scheduled refresh fails with timeout errors

**Solution**: Implement incremental refresh in Power BI

```powerquery
// Add RangeStart and RangeEnd parameters
let
    Source = Sql.Database(ServerName, DatabaseName),
    FilteredData = Table.SelectRows(
        Source, 
        each [OperationTime] >= RangeStart and [OperationTime] < RangeEnd
    )
in
    FilteredData
```

Configure incremental refresh policy in Power BI Desktop:
- Rows to archive: 2 years
- Rows to refresh: 7 days
- Detect data changes: Yes (based on TimeKey column)

### Connection String Issues

**Issue**: Authentication failures when connecting from Power BI Service

**Solution**: Use gateway with service principal authentication

```sql
-- Grant access to service principal
CREATE USER [powerbi-service-principal] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [powerbi-service-principal];
GRANT EXECUTE ON SCHEMA::dbo TO [powerbi-service-principal];
```

## Best Practices

1. **Partition large fact tables** by date for better query performance:

```sql
ALTER TABLE FactWarehouseOperations
ADD CONSTRAINT PK_WarehouseOps_Partitioned 
PRIMARY KEY (OperationKey, TimeKey)
ON PartitionSchemeMonthly(TimeKey);
```

2. **Use stored procedures** for all data loads to maintain audit trail

3. **Implement slowly changing dimensions** (SCD Type 2) for product and geography changes:

```sql
ALTER TABLE DimProductGravity
ADD ValidFrom DATETIME2, ValidTo DATETIME2, IsCurrent BIT;
```

4. **Schedule Power BI refresh** during off-peak hours (2 AM - 4 AM)

5. **Monitor query performance** using SQL Server Query Store:

```sql
ALTER DATABASE LogiFleetPulse 
SET QUERY_STORE = ON (OPERATION_MODE = READ_WRITE);
```
