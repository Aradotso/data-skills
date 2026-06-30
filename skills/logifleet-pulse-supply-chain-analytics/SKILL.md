---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse for fleet, warehouse, and supply chain KPI modeling
triggers:
  - set up LogiFleet Pulse supply chain analytics
  - deploy SQL server logistics data warehouse
  - create Power BI logistics dashboard
  - implement multi-fact star schema for warehouse operations
  - configure fleet telemetry analytics
  - build cross-modal supply chain reporting
  - optimize warehouse gravity zones with LogiFleet
  - integrate logistics KPI harmonization model
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced MS SQL Server data warehousing solution with Power BI dashboards for unified logistics intelligence. It combines warehouse operations, fleet telemetry, inventory management, and external signals into a multi-fact star schema that enables cross-functional KPI analysis. The platform provides real-time visibility into supply chain operations, predictive bottleneck detection, and adaptive fleet optimization.

**Core Capabilities:**
- Multi-fact star schema with time-phased dimensions (15-minute granularity)
- Warehouse operations tracking (receiving, putaway, picking, packing, shipping)
- Fleet telemetry integration (GPS, fuel consumption, idle time, maintenance)
- Cross-dock transfer analytics
- Warehouse gravity zone optimization
- Predictive bottleneck indexing
- Role-based access control with row-level security

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- Access to source systems: WMS, TMS, or telemetry APIs
- Optional: Azure Synapse Analytics for external enrichment

### Deployment Steps

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**
```sql
-- Connect to your SQL Server instance in SSMS
-- Open and execute schema_deployment.sql
-- This creates all tables, relationships, views, and stored procedures

:r schema_deployment.sql
GO
```

3. **Configure data sources:**
```json
// Update config_sample.json with your connection strings
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}"
  },
  "dataSources": {
    "wms": "${WMS_API_ENDPOINT}",
    "telematics": "${FLEET_TELEMETRY_ENDPOINT}",
    "erp": "${ERP_CONNECTION_STRING}"
  }
}
```

4. **Import Power BI template:**
```powershell
# Open LogiFleet_Pulse_Master.pbit in Power BI Desktop
# Enter connection parameters when prompted
# Publish to Power BI Service workspace
```

## Data Model Architecture

### Core Fact Tables

**FactWarehouseOperations** - Warehouse activity metrics:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'RECEIVING', 'PUTAWAY', 'PICKING', 'PACKING', 'SHIPPING'
    DwellTimeMinutes INT,
    PickRateUnitsPerHour DECIMAL(10,2),
    PackingCycleSeconds INT,
    EmployeeKey INT,
    ZoneGravityScore DECIMAL(5,2),
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Recommended indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWH_Time_Product 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (DwellTimeMinutes, PickRateUnitsPerHour);

CREATE NONCLUSTERED INDEX IX_FactWH_Zone_Operation 
ON FactWarehouseOperations(WarehouseKey, OperationType, TimeKey);
```

**FactFleetTrips** - Fleet and route analytics:
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    OriginLocationKey INT,
    DestinationLocationKey INT,
    TripDistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    LoadWeightKg DECIMAL(10,2),
    OnTimeDeliveryFlag BIT,
    WeatherDelayFlag BIT,
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey)
);

-- Partitioning strategy for large datasets
CREATE PARTITION FUNCTION PF_TripsByMonth (INT)
AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301); -- YYYYMMDD format

CREATE PARTITION SCHEME PS_TripsByMonth
AS PARTITION PF_TripsByMonth
TO (FG_2026Q1, FG_2026Q2, FG_2026Q3, FG_2026Q4);
```

### Key Dimension Tables

**DimTime** - Time intelligence with 15-minute granularity:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY, -- Format: YYYYMMDDHHMM
    FullDateTime DATETIME NOT NULL,
    DateKey INT, -- YYYYMMDD
    TimeOfDay TIME,
    Hour TINYINT,
    MinuteBucket TINYINT, -- 0, 15, 30, 45
    DayOfWeek TINYINT,
    DayName VARCHAR(10),
    IsWeekend BIT,
    IsBusinessHour BIT,
    FiscalPeriod VARCHAR(10),
    FiscalYear INT
);

-- Populate DimTime for operational hours
DECLARE @StartDate DATETIME = '2026-01-01 00:00:00';
DECLARE @EndDate DATETIME = '2027-12-31 23:45:00';

WITH TimeCTE AS (
    SELECT @StartDate AS TimeValue
    UNION ALL
    SELECT DATEADD(MINUTE, 15, TimeValue)
    FROM TimeCTE
    WHERE TimeValue < @EndDate
)
INSERT INTO DimTime (TimeKey, FullDateTime, DateKey, TimeOfDay, Hour, MinuteBucket)
SELECT 
    CAST(FORMAT(TimeValue, 'yyyyMMddHHmm') AS INT),
    TimeValue,
    CAST(FORMAT(TimeValue, 'yyyyMMdd') AS INT),
    CAST(TimeValue AS TIME),
    DATEPART(HOUR, TimeValue),
    DATEPART(MINUTE, TimeValue),
FROM TimeCTE
OPTION (MAXRECURSION 0);
```

**DimProductGravity** - Warehouse gravity zone assignment:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    VelocityScore DECIMAL(5,2), -- Pick frequency weight
    ValueScore DECIMAL(5,2), -- Unit value weight
    FragilityScore DECIMAL(5,2), -- Handling care weight
    GravityZone VARCHAR(20), -- 'HIGH', 'MEDIUM', 'LOW'
    OptimalStorageDistance INT, -- Meters from shipping dock
    LastRecalculated DATETIME DEFAULT GETDATE()
);

-- Stored procedure to recalculate gravity scores
CREATE PROCEDURE SP_RecalculateGravityZones
AS
BEGIN
    UPDATE DimProductGravity
    SET 
        VelocityScore = (
            SELECT COUNT(*) * 1.0 / NULLIF(DATEDIFF(DAY, MIN(wh.OperationDate), MAX(wh.OperationDate)), 0)
            FROM FactWarehouseOperations wh
            WHERE wh.ProductKey = DimProductGravity.ProductKey
            AND wh.OperationType = 'PICKING'
            AND wh.OperationDate >= DATEADD(MONTH, -3, GETDATE())
        ),
        GravityZone = CASE 
            WHEN (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) > 7.5 THEN 'HIGH'
            WHEN (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) > 4.0 THEN 'MEDIUM'
            ELSE 'LOW'
        END,
        LastRecalculated = GETDATE();
END;
```

## Key SQL Views and Procedures

### Cross-Fact KPI Harmonization View

```sql
CREATE VIEW VW_UnifiedLogisticsKPI AS
SELECT 
    t.DateKey,
    t.FiscalPeriod,
    p.Category AS ProductCategory,
    g.GravityZone,
    
    -- Warehouse KPIs
    AVG(wh.DwellTimeMinutes) AS AvgDwellTime,
    AVG(wh.PickRateUnitsPerHour) AS AvgPickRate,
    COUNT(DISTINCT wh.OperationKey) AS TotalOperations,
    
    -- Fleet KPIs
    AVG(fl.IdleTimeMinutes * 1.0 / NULLIF(fl.TripDistanceKm, 0)) AS IdlePerKm,
    AVG(fl.FuelConsumedLiters * 1.0 / NULLIF(fl.TripDistanceKm, 0)) AS FuelEfficiency,
    SUM(CASE WHEN fl.OnTimeDeliveryFlag = 1 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS OnTimePercentage,
    
    -- Cross-fact correlation
    AVG(wh.DwellTimeMinutes * fl.IdleTimeMinutes) AS DwellIdleCorrelation

FROM DimTime t
INNER JOIN FactWarehouseOperations wh ON t.TimeKey = wh.TimeKey
INNER JOIN DimProductGravity p ON wh.ProductKey = p.ProductKey
INNER JOIN FactFleetTrips fl ON t.TimeKey = fl.TimeKey
WHERE t.IsBusinessHour = 1
GROUP BY t.DateKey, t.FiscalPeriod, p.Category, g.GravityZone;
```

### Predictive Bottleneck Detection

```sql
CREATE PROCEDURE SP_CalculateBottleneckIndex
    @ForecastHorizonHours INT = 24
AS
BEGIN
    WITH CurrentMetrics AS (
        SELECT 
            WarehouseKey,
            AVG(DwellTimeMinutes) AS AvgDwell,
            STDEV(DwellTimeMinutes) AS StdDevDwell,
            COUNT(*) AS OperationCount
        FROM FactWarehouseOperations
        WHERE TimeKey >= CAST(FORMAT(DATEADD(HOUR, -@ForecastHorizonHours, GETDATE()), 'yyyyMMddHHmm') AS INT)
        GROUP BY WarehouseKey
    ),
    FleetLoad AS (
        SELECT 
            OriginLocationKey,
            COUNT(*) AS InboundTrips,
            AVG(LoadWeightKg) AS AvgInboundLoad
        FROM FactFleetTrips
        WHERE TimeKey >= CAST(FORMAT(DATEADD(HOUR, -@ForecastHorizonHours, GETDATE()), 'yyyyMMddHHmm') AS INT)
        GROUP BY OriginLocationKey
    )
    SELECT 
        cm.WarehouseKey,
        cm.AvgDwell,
        fl.InboundTrips,
        -- Bottleneck Index: weighted score of dwell variance + inbound pressure
        (cm.StdDevDwell / NULLIF(cm.AvgDwell, 0) * 0.6 + 
         fl.InboundTrips * 1.0 / cm.OperationCount * 0.4) AS BottleneckIndex,
        CASE 
            WHEN (cm.StdDevDwell / NULLIF(cm.AvgDwell, 0) * 0.6 + 
                  fl.InboundTrips * 1.0 / cm.OperationCount * 0.4) > 0.75 THEN 'CRITICAL'
            WHEN (cm.StdDevDwell / NULLIF(cm.AvgDwell, 0) * 0.6 + 
                  fl.InboundTrips * 1.0 / cm.OperationCount * 0.4) > 0.5 THEN 'WARNING'
            ELSE 'NORMAL'
        END AS AlertLevel
    FROM CurrentMetrics cm
    LEFT JOIN FleetLoad fl ON cm.WarehouseKey = fl.OriginLocationKey
    ORDER BY BottleneckIndex DESC;
END;
```

### Automated Alert Configuration

```sql
CREATE PROCEDURE SP_ConfigureAlerts
    @KPIName VARCHAR(100),
    @ThresholdValue DECIMAL(10,2),
    @ComparisonOperator VARCHAR(10), -- '>', '<', '='
    @NotificationEmail VARCHAR(255),
    @NotificationTeamsWebhook VARCHAR(500) = NULL
AS
BEGIN
    -- Create alert rule
    INSERT INTO AlertRules (KPIName, ThresholdValue, ComparisonOperator, NotificationEmail, NotificationTeamsWebhook)
    VALUES (@KPIName, @ThresholdValue, @ComparisonOperator, @NotificationEmail, @NotificationTeamsWebhook);
    
    -- Create SQL Server Agent job for monitoring
    DECLARE @JobName VARCHAR(200) = 'Alert_' + @KPIName;
    DECLARE @StepCommand NVARCHAR(MAX) = N'
    DECLARE @CurrentValue DECIMAL(10,2);
    DECLARE @AlertTriggered BIT = 0;
    
    -- Get current KPI value (example for fleet idle time)
    SELECT @CurrentValue = AVG(IdleTimeMinutes * 100.0 / NULLIF(TripDistanceKm, 0))
    FROM FactFleetTrips
    WHERE TimeKey >= CAST(FORMAT(DATEADD(HOUR, -1, GETDATE()), ''yyyyMMddHHmm'') AS INT);
    
    IF ' + @ComparisonOperator + ' @CurrentValue ' + @ComparisonOperator + ' ' + CAST(@ThresholdValue AS VARCHAR) + '
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = ''' + @NotificationEmail + ''',
            @subject = ''LogiFleet Alert: ' + @KPIName + ' threshold breached'',
            @body = ''Current value: '' + CAST(@CurrentValue AS VARCHAR);
    END';
    
    EXEC msdb.dbo.sp_add_job @job_name = @JobName;
    EXEC msdb.dbo.sp_add_jobstep @job_name = @JobName, @step_name = 'Check_Threshold', @command = @StepCommand;
    EXEC msdb.dbo.sp_add_schedule @schedule_name = @JobName + '_Schedule', @freq_type = 4, @freq_interval = 1, @freq_subday_type = 8, @freq_subday_interval = 1;
    EXEC msdb.dbo.sp_attach_schedule @job_name = @JobName, @schedule_name = @JobName + '_Schedule';
END;
```

## Power BI Configuration

### DAX Measures for Cross-Fact Analysis

```dax
// Composite KPI: Dwell-Adjusted Fleet Cost
DwellFleetCost = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR AvgIdleCost = AVERAGE(FactFleetTrips[IdleTimeMinutes]) * [CostPerIdleMinute]
VAR DwellPenalty = IF(AvgDwell > 72, (AvgDwell - 72) * 0.5, 0)
RETURN
    AvgIdleCost + DwellPenalty

// Warehouse Gravity Zone Efficiency
GravityZoneEfficiency = 
CALCULATE(
    DIVIDE(
        SUM(FactWarehouseOperations[PickRateUnitsPerHour]),
        SUM(FactWarehouseOperations[DwellTimeMinutes])
    ),
    FILTER(
        DimProductGravity,
        DimProductGravity[GravityZone] = SELECTEDVALUE(DimProductGravity[GravityZone])
    )
)

// On-Time Delivery with Weather Adjustment
AdjustedOTD = 
VAR BaseOTD = 
    DIVIDE(
        CALCULATE(COUNTROWS(FactFleetTrips), FactFleetTrips[OnTimeDeliveryFlag] = TRUE()),
        COUNTROWS(FactFleetTrips)
    )
VAR WeatherDelays = 
    CALCULATE(COUNTROWS(FactFleetTrips), FactFleetTrips[WeatherDelayFlag] = TRUE())
VAR TotalTrips = COUNTROWS(FactFleetTrips)
RETURN
    IF(WeatherDelays > TotalTrips * 0.1, BaseOTD * 1.05, BaseOTD) // 5% allowance for severe weather

// Temporal Elasticity - Capacity Simulation
CapacityElasticity = 
VAR CurrentCapacity = SUM(FactWarehouseOperations[OperationCount])
VAR SimulatedCapacity = CurrentCapacity * 1.2 // 20% increase
VAR HistoricalElasticity = 
    CALCULATE(
        DIVIDE(
            SUM(FactWarehouseOperations[PickRateUnitsPerHour]),
            SUM(FactWarehouseOperations[DwellTimeMinutes])
        ),
        DATESINPERIOD(DimTime[FullDateTime], MAX(DimTime[FullDateTime]), -90, DAY)
    )
RETURN
    SimulatedCapacity * HistoricalElasticity * 0.85 // 15% efficiency loss at higher capacity
```

### Power BI Report Configuration

```json
// Report settings for LogiFleet_Pulse_Master.pbit
{
  "dataRefresh": {
    "schedule": "every15Minutes",
    "incrementalRefresh": true,
    "partitionStrategy": "rollingMonth"
  },
  "rowLevelSecurity": {
    "enabled": true,
    "roles": [
      {
        "name": "WarehouseManager",
        "filter": "[DimWarehouse][WarehouseKey] IN UserWarehouseAccess(USERNAME())"
      },
      {
        "name": "FleetSupervisor",
        "filter": "[DimVehicle][FleetRegion] = UserRegion(USERNAME())"
      },
      {
        "name": "Executive",
        "filter": "1=1"
      }
    ]
  },
  "dashboardPages": [
    "ExecutiveSummary",
    "WarehouseOperations",
    "FleetPerformance",
    "CrossModalAnalysis",
    "PredictiveAlerts"
  ]
}
```

## Data Integration Patterns

### Incremental Load from WMS

```sql
CREATE PROCEDURE SP_IncrementalLoadWarehouseOps
    @LastLoadTimestamp DATETIME = NULL
AS
BEGIN
    SET @LastLoadTimestamp = ISNULL(@LastLoadTimestamp, (SELECT MAX(LoadedAt) FROM ETL_LoadLog WHERE TableName = 'FactWarehouseOperations'));
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType, 
        DwellTimeMinutes, PickRateUnitsPerHour, PackingCycleSeconds
    )
    SELECT 
        CAST(FORMAT(wms.OperationDateTime, 'yyyyMMddHHmm') AS INT),
        p.ProductKey,
        w.WarehouseKey,
        wms.ActivityType,
        DATEDIFF(MINUTE, wms.StartTime, wms.EndTime),
        wms.UnitsHandled * 60.0 / NULLIF(DATEDIFF(MINUTE, wms.StartTime, wms.EndTime), 0),
        DATEDIFF(SECOND, wms.PackStart, wms.PackEnd)
    FROM OPENQUERY(WMS_LinkedServer, 
        'SELECT * FROM warehouse_operations WHERE last_modified > ''' + CONVERT(VARCHAR, @LastLoadTimestamp, 120) + '''') wms
    INNER JOIN DimProduct p ON wms.sku = p.SKU
    INNER JOIN DimWarehouse w ON wms.warehouse_code = w.WarehouseCode;
    
    INSERT INTO ETL_LoadLog (TableName, LoadedAt, RowsInserted)
    VALUES ('FactWarehouseOperations', GETDATE(), @@ROWCOUNT);
END;
```

### Fleet Telemetry API Integration

```sql
-- External table for real-time telemetry (requires PolyBase)
CREATE EXTERNAL DATA SOURCE FleetTelemetryAPI
WITH (
    TYPE = REST_API,
    LOCATION = '${FLEET_TELEMETRY_ENDPOINT}',
    CREDENTIAL = FleetAPICredential
);

CREATE EXTERNAL TABLE ExtFleetTelemetry (
    VehicleID VARCHAR(50),
    Timestamp DATETIME,
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6),
    FuelLevel DECIMAL(5,2),
    EngineIdleTime INT,
    LoadWeight DECIMAL(10,2)
)
WITH (
    DATA_SOURCE = FleetTelemetryAPI,
    FILE_FORMAT = JSON_FORMAT
);

-- Stored procedure to aggregate telemetry into trips
CREATE PROCEDURE SP_ProcessFleetTelemetry
AS
BEGIN
    WITH TripSegments AS (
        SELECT 
            VehicleID,
            Timestamp,
            LAG(Timestamp) OVER (PARTITION BY VehicleID ORDER BY Timestamp) AS PrevTimestamp,
            Latitude,
            Longitude,
            CASE 
                WHEN DATEDIFF(MINUTE, LAG(Timestamp) OVER (PARTITION BY VehicleID ORDER BY Timestamp), Timestamp) > 30 
                THEN 1 ELSE 0 
            END AS NewTripFlag
        FROM ExtFleetTelemetry
        WHERE Timestamp >= DATEADD(HOUR, -1, GETDATE())
    )
    INSERT INTO FactFleetTrips (TimeKey, VehicleKey, TripDistanceKm, IdleTimeMinutes, LoadWeightKg)
    SELECT 
        CAST(FORMAT(MIN(Timestamp), 'yyyyMMddHHmm') AS INT),
        v.VehicleKey,
        SUM(geography::Point(Latitude, Longitude, 4326).STDistance(
            geography::Point(LAG(Latitude) OVER (ORDER BY Timestamp), LAG(Longitude) OVER (ORDER BY Timestamp), 4326)
        )) / 1000.0,
        SUM(EngineIdleTime),
        AVG(LoadWeight)
    FROM TripSegments ts
    INNER JOIN DimVehicle v ON ts.VehicleID = v.VehicleID
    WHERE NewTripFlag = 0
    GROUP BY v.VehicleKey, TripID;
END;
```

## Common Usage Patterns

### Scenario 1: Identify Slow-Moving SKUs in Wrong Zones

```sql
-- Find products in high-gravity zones with low pick velocity
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityZone,
    COUNT(wh.OperationKey) AS PickCount,
    AVG(wh.DwellTimeMinutes) AS AvgDwell
FROM DimProductGravity p
INNER JOIN FactWarehouseOperations wh ON p.ProductKey = wh.ProductKey
WHERE 
    p.GravityZone = 'HIGH'
    AND wh.OperationType = 'PICKING'
    AND wh.TimeKey >= CAST(FORMAT(DATEADD(MONTH, -1, GETDATE()), 'yyyyMMddHHmm') AS INT)
GROUP BY p.SKU, p.ProductName, p.GravityZone
HAVING COUNT(wh.OperationKey) < 10
ORDER BY AvgDwell DESC;

-- Recommend zone reassignment
UPDATE DimProductGravity
SET 
    GravityZone = 'MEDIUM',
    OptimalStorageDistance = OptimalStorageDistance + 50
WHERE ProductKey IN (
    SELECT ProductKey FROM (
        -- Subquery from above
    ) AS LowVelocity
);
```

### Scenario 2: Fleet Route Optimization Based on Idle Time

```sql
-- Analyze routes with high idle-to-distance ratio
WITH RouteMetrics AS (
    SELECT 
        r.RouteName,
        AVG(ft.IdleTimeMinutes * 1.0 / NULLIF(ft.TripDistanceKm, 0)) AS IdlePerKm,
        AVG(ft.FuelConsumedLiters * 1.0 / NULLIF(ft.TripDistanceKm, 0)) AS FuelEfficiency,
        COUNT(*) AS TripCount
    FROM FactFleetTrips ft
    INNER JOIN DimRoute r ON ft.RouteKey = r.RouteKey
    WHERE ft.TimeKey >= CAST(FORMAT(DATEADD(MONTH, -3, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY r.RouteName
)
SELECT 
    RouteName,
    IdlePerKm,
    FuelEfficiency,
    TripCount,
    -- Recommendation flag
    CASE 
        WHEN IdlePerKm > 2.0 THEN 'Review for route consolidation or traffic avoidance'
        WHEN FuelEfficiency > 0.12 THEN 'Optimize for fuel efficiency (consider alternative paths)'
        ELSE 'Route performing within targets'
    END AS Recommendation
FROM RouteMetrics
ORDER BY IdlePerKm DESC;
```

### Scenario 3: Spoilage Prevention with Temperature Monitoring

```sql
-- Create alert for temperature-sensitive shipments
CREATE PROCEDURE SP_MonitorColdChain
AS
BEGIN
    DECLARE @CriticalShipments TABLE (
        ShipmentID INT,
        ProductName VARCHAR(200),
        CurrentTemp DECIMAL(5,2),
        TargetTemp DECIMAL(5,2),
        MinutesOutOfRange INT
    );
    
    INSERT INTO @CriticalShipments
    SELECT 
        ft.TripKey,
        p.ProductName,
        tm.CurrentTemperature,
        p.RequiredTempCelsius,
        DATEDIFF(MINUTE, tm.ThresholdBreachTime, GETDATE())
    FROM FactFleetTrips ft
    INNER JOIN TelemetryTemperature tm ON ft.VehicleKey = tm.VehicleKey
    INNER JOIN TripManifest man ON ft.TripKey = man.TripKey
    INNER JOIN DimProduct p ON man.ProductKey = p.ProductKey
    WHERE 
        p.TemperatureSensitive = 1
        AND ABS(tm.CurrentTemperature - p.RequiredTempCelsius) > 2.0
        AND tm.ThresholdBreachTime IS NOT NULL
        AND DATEDIFF(MINUTE, tm.ThresholdBreachTime, GETDATE()) > 20;
    
    IF EXISTS (SELECT 1 FROM @CriticalShipments)
    BEGIN
        DECLARE @AlertBody NVARCHAR(MAX) = (
            SELECT 
                ShipmentID, ProductName, CurrentTemp, TargetTemp, MinutesOutOfRange
            FROM @CriticalShipments
            FOR JSON PATH
        );
        
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = '${COLD_CHAIN_ALERT_EMAIL}',
            @subject = 'URGENT: Cold Chain Breach Detected',
            @body = @AlertBody,
            @body_format = 'JSON';
    END;
END;
```

## Troubleshooting

### Issue: Slow Dashboard Refresh Times

**Symptom:** Power BI reports take >30 seconds to refresh

**Solutions:**
```sql
-- 1. Check for missing indexes
SELECT 
    OBJECT_NAME(ips.object_id) AS TableName,
    ips.avg_fragmentation_in_percent,
    ips.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
WHERE ips.avg_fragmentation_in_percent > 30
AND ips.page_count > 1000;

-- 2. Rebuild fragmented indexes
ALTER INDEX ALL ON FactWarehouseOperations REBUILD WITH (ONLINE = ON);
ALTER INDEX ALL ON FactFleetTrips REBUILD WITH (ONLINE = ON);

-- 3. Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;

-- 4. Implement incremental refresh in Power BI
-- Edit dataset parameters in Power BI Service:
-- RangeStart = DateTime.LocalNow() - #duration(90, 0, 0, 0)
-- RangeEnd = DateTime.LocalNow()
```

### Issue: Row-Level Security Not Filtering Correctly

**Symptom:** Users see data outside their assigned scope

**Solution:**
```sql
-- Verify RLS function returns correct values
SELECT 
    SUSER_SNAME() AS CurrentUser,
    dbo.UserWarehouseAccess(SUSER_SNAME()) AS AssignedWarehouses;

-- Test RLS filter directly
SELECT *
FROM FactWarehouseOperations
WHERE WarehouseKey IN (SELECT WarehouseKey FROM dbo.UserWarehouseAccess(SUSER_SNAME()));

-- Ensure RLS is active in Power BI
-- In Power BI Desktop: Modeling > Manage Roles > View as Role
```

### Issue: Bottleneck Index Returns NULL Values

**Symptom:** SP_CalculateBottleneckIndex produces NULL results

**Solution:**
```sql
-- Check for division by zero and missing joins
SELECT 
    cm.WarehouseKey,
    cm.AvgDwell,
    cm.StdDevDwell,
    cm.OperationCount,
    fl.InboundTrips,
    CASE 
