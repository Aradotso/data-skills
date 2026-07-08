---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehousing with multi-fact star schema for warehouse, fleet, and supply chain intelligence
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics data warehouse with power bi
  - configure warehouse fleet multi-fact star schema
  - implement supply chain intelligence dashboard
  - build real-time logistics analytics platform
  - create cross-modal supply chain orchestrator
  - integrate warehouse and fleet telemetry data
  - design logistics KPI harmonization system
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that combines MS SQL Server data warehousing with Power BI dashboards to provide unified visibility across warehouse operations, fleet management, and supply chain metrics. It uses a multi-fact star schema architecture to harmonize cross-domain KPIs (warehouse dwell time vs. fleet fuel consumption, inventory turnover vs. delivery routes) into a single semantic layer.

**Core capabilities:**
- Multi-fact data warehouse (warehouse operations, fleet trips, cross-dock transfers)
- Time-phased dimensional modeling (15-minute granularity)
- Real-time Power BI dashboards with role-based access
- Predictive bottleneck detection and fleet triage
- Warehouse "gravity zone" spatial optimization
- Cross-fact KPI correlation and anomaly detection

## Installation & Setup

### Prerequisites

```bash
# Required software
- MS SQL Server 2019+ (Standard or Enterprise edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) 18+
```

### 1. Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### 2. Deploy SQL Schema

```sql
-- Connect to your SQL Server instance via SSMS
-- Execute the schema deployment script

USE master;
GO

-- Create the database
CREATE DATABASE LogiFleetPulse
ON PRIMARY (
    NAME = LogiFleet_Data,
    FILENAME = 'C:\SQLData\LogiFleetPulse.mdf',
    SIZE = 10GB,
    FILEGROWTH = 1GB
)
LOG ON (
    NAME = LogiFleet_Log,
    FILENAME = 'C:\SQLData\LogiFleetPulse_log.ldf',
    SIZE = 2GB,
    FILEGROWTH = 512MB
);
GO

USE LogiFleetPulse;
GO

-- Run the complete schema script
-- (Located in /sql/schema_deployment.sql in the repository)
```

### 3. Configure Data Sources

Create a configuration file for your data source connections:

```json
{
  "dataSources": {
    "wms": {
      "type": "SQL",
      "connectionString": "Server=${WMS_SERVER};Database=${WMS_DB};Integrated Security=true;"
    },
    "telematics": {
      "type": "REST_API",
      "endpoint": "${FLEET_API_ENDPOINT}",
      "authHeader": "Bearer ${FLEET_API_TOKEN}"
    },
    "weatherAPI": {
      "type": "REST_API",
      "endpoint": "https://api.weather.provider/v2/",
      "apiKey": "${WEATHER_API_KEY}"
    }
  },
  "refreshSchedule": {
    "incrementalLoad": "*/15 * * * *",
    "fullRefresh": "0 2 * * 0"
  }
}
```

### 4. Load Power BI Template

```bash
# Open Power BI Desktop
# File > Open > LogiFleet_Pulse_Master.pbit
# Enter connection parameters when prompted:
# - SQL Server instance name
# - Database: LogiFleetPulse
# - Authentication method (Windows/SQL)
```

## Key Schema Components

### Fact Tables

#### FactWarehouseOperations

```sql
-- Core warehouse activity tracking
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    DateKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType NVARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled DECIMAL(18,2),
    DwellTimeMinutes INT,
    CycleTimeSeconds INT,
    ZoneKey INT,
    EmployeeKey INT,
    CONSTRAINT FK_WarehouseOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WarehouseOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Create columnstore index for analytics performance
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouseOps
ON FactWarehouseOperations (
    TimeKey, DateKey, WarehouseKey, ProductKey, 
    QuantityHandled, DwellTimeMinutes, CycleTimeSeconds
);
```

#### FactFleetTrips

```sql
-- Fleet telemetry and trip tracking
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TripStartTimeKey INT NOT NULL,
    TripEndTimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightKG DECIMAL(12,2),
    DeliveryStatus NVARCHAR(50), -- 'OnTime', 'Delayed', 'InTransit'
    WeatherCondition NVARCHAR(100),
    CONSTRAINT FK_FleetTrips_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_FleetTrips_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleetTrips
ON FactFleetTrips (
    TripStartTimeKey, VehicleKey, RouteKey, 
    DistanceKM, FuelConsumedLiters, IdleTimeMinutes
);
```

### Dimension Tables

#### DimTime (Time-Phased Dimension)

```sql
-- 15-minute grain time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2(0) NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME(0),
    Hour TINYINT,
    Minute TINYINT,
    QuarterHour TINYINT, -- 1-4 (0-15, 16-30, 31-45, 46-59)
    DayPartition NVARCHAR(20), -- 'Morning', 'Afternoon', 'Evening', 'Night'
    IsBusinessHour BIT,
    IsWeekend BIT,
    ShiftType NVARCHAR(20) -- 'Day', 'Swing', 'Night'
);

-- Populate time dimension
WITH TimeSeries AS (
    SELECT CAST('2024-01-01 00:00:00' AS DATETIME2) AS TimeValue
    UNION ALL
    SELECT DATEADD(MINUTE, 15, TimeValue)
    FROM TimeSeries
    WHERE TimeValue < '2027-12-31 23:45:00'
)
INSERT INTO DimTime (TimeKey, FullDateTime, DateKey, TimeOfDay, Hour, Minute, QuarterHour, DayPartition, IsBusinessHour, IsWeekend, ShiftType)
SELECT 
    CAST(FORMAT(TimeValue, 'yyyyMMddHHmm') AS INT) AS TimeKey,
    TimeValue,
    CAST(FORMAT(TimeValue, 'yyyyMMdd') AS INT) AS DateKey,
    CAST(TimeValue AS TIME) AS TimeOfDay,
    DATEPART(HOUR, TimeValue) AS Hour,
    DATEPART(MINUTE, TimeValue) AS Minute,
    ((DATEPART(MINUTE, TimeValue) / 15) + 1) AS QuarterHour,
    CASE 
        WHEN DATEPART(HOUR, TimeValue) BETWEEN 6 AND 11 THEN 'Morning'
        WHEN DATEPART(HOUR, TimeValue) BETWEEN 12 AND 17 THEN 'Afternoon'
        WHEN DATEPART(HOUR, TimeValue) BETWEEN 18 AND 21 THEN 'Evening'
        ELSE 'Night'
    END AS DayPartition,
    CASE WHEN DATEPART(HOUR, TimeValue) BETWEEN 8 AND 17 THEN 1 ELSE 0 END AS IsBusinessHour,
    CASE WHEN DATEPART(WEEKDAY, TimeValue) IN (1,7) THEN 1 ELSE 0 END AS IsWeekend,
    CASE 
        WHEN DATEPART(HOUR, TimeValue) BETWEEN 6 AND 14 THEN 'Day'
        WHEN DATEPART(HOUR, TimeValue) BETWEEN 14 AND 22 THEN 'Swing'
        ELSE 'Night'
    END AS ShiftType
FROM TimeSeries
OPTION (MAXRECURSION 0);
```

#### DimProductGravity (Warehouse Gravity Zones)

```sql
-- Product dimension with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    ProductSKU NVARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    UnitWeight DECIMAL(10,4),
    Fragility NVARCHAR(20), -- 'Low', 'Medium', 'High'
    ValuePerUnit DECIMAL(12,2),
    AveragePickFrequency DECIMAL(10,2), -- picks per day
    GravityScore AS (
        (AveragePickFrequency * 0.5) + 
        (ValuePerUnit / 100 * 0.3) + 
        (CASE Fragility WHEN 'High' THEN 20 WHEN 'Medium' THEN 10 ELSE 0 END * 0.2)
    ) PERSISTED,
    RecommendedZone AS (
        CASE 
            WHEN (AveragePickFrequency * 0.5) + (ValuePerUnit / 100 * 0.3) > 15 THEN 'HighGravity'
            WHEN (AveragePickFrequency * 0.5) + (ValuePerUnit / 100 * 0.3) > 8 THEN 'MediumGravity'
            ELSE 'LowGravity'
        END
    ) PERSISTED
);
```

## Common Query Patterns

### Cross-Fact KPI Analysis

```sql
-- Correlate warehouse dwell time with fleet idle time
WITH WarehouseDwell AS (
    SELECT 
        dt.DateKey,
        dw.WarehouseName,
        dp.Category,
        AVG(fwo.DwellTimeMinutes) AS AvgDwellTime,
        SUM(fwo.QuantityHandled) AS TotalQuantity
    FROM FactWarehouseOperations fwo
    JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    JOIN DimWarehouse dw ON fwo.WarehouseKey = dw.WarehouseKey
    JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    WHERE dt.DateKey >= 20260601 AND dt.DateKey < 20260701
        AND fwo.OperationType = 'Picking'
    GROUP BY dt.DateKey, dw.WarehouseName, dp.Category
),
FleetIdle AS (
    SELECT 
        dt.DateKey,
        dg.WarehouseName AS DestinationWarehouse,
        AVG(fft.IdleTimeMinutes) AS AvgIdleTime,
        AVG(fft.FuelConsumedLiters) AS AvgFuel
    FROM FactFleetTrips fft
    JOIN DimTime dt ON fft.TripStartTimeKey = dt.TimeKey
    JOIN DimGeography dg ON fft.DestinationGeographyKey = dg.GeographyKey
    WHERE dt.DateKey >= 20260601 AND dt.DateKey < 20260701
    GROUP BY dt.DateKey, dg.WarehouseName
)
SELECT 
    wd.DateKey,
    wd.WarehouseName,
    wd.Category,
    wd.AvgDwellTime,
    fi.AvgIdleTime,
    fi.AvgFuel,
    -- Correlation metric: high dwell + high idle = bottleneck
    (wd.AvgDwellTime / 60.0 * fi.AvgIdleTime / 60.0) AS BottleneckScore
FROM WarehouseDwell wd
LEFT JOIN FleetIdle fi 
    ON wd.DateKey = fi.DateKey 
    AND wd.WarehouseName = fi.DestinationWarehouse
WHERE (wd.AvgDwellTime / 60.0 * fi.AvgIdleTime / 60.0) > 5 -- Threshold for alert
ORDER BY BottleneckScore DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products in suboptimal zones
SELECT 
    dp.ProductSKU,
    dp.ProductName,
    dp.GravityScore,
    dp.RecommendedZone,
    dz.ZoneName AS CurrentZone,
    dz.DistanceFromShippingDock AS CurrentDistance,
    COUNT(fwo.OperationKey) AS PickCount,
    AVG(fwo.CycleTimeSeconds) AS AvgCycleTime,
    -- Estimated time savings if moved to recommended zone
    CASE 
        WHEN dp.RecommendedZone = 'HighGravity' AND dz.DistanceFromShippingDock > 50 
            THEN AVG(fwo.CycleTimeSeconds) * 0.25 -- 25% improvement
        WHEN dp.RecommendedZone = 'MediumGravity' AND dz.DistanceFromShippingDock > 100 
            THEN AVG(fwo.CycleTimeSeconds) * 0.15
        ELSE 0
    END AS EstimatedTimeSavings
FROM FactWarehouseOperations fwo
JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
JOIN DimZone dz ON fwo.ZoneKey = dz.ZoneKey
WHERE fwo.OperationType = 'Picking'
    AND fwo.DateKey >= 20260601
GROUP BY 
    dp.ProductSKU, dp.ProductName, dp.GravityScore, 
    dp.RecommendedZone, dz.ZoneName, dz.DistanceFromShippingDock
HAVING COUNT(fwo.OperationKey) > 50 -- Minimum pick frequency
    AND (
        (dp.RecommendedZone = 'HighGravity' AND dz.DistanceFromShippingDock > 50)
        OR (dp.RecommendedZone = 'MediumGravity' AND dz.DistanceFromShippingDock > 100)
    )
ORDER BY EstimatedTimeSavings DESC;
```

### Fleet Triage Scoring

```sql
-- Proactive maintenance prioritization based on revenue impact
WITH VehicleHealth AS (
    SELECT 
        v.VehicleKey,
        v.VehicleID,
        v.TirePressurePSI,
        v.EngineHoursTotal,
        v.LastMaintenanceDate,
        CASE 
            WHEN v.TirePressurePSI < 30 THEN 20
            WHEN v.TirePressurePSI < 35 THEN 10
            ELSE 0
        END AS TireRiskScore,
        CASE 
            WHEN DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) > 90 THEN 30
            WHEN DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) > 60 THEN 15
            ELSE 0
        END AS MaintenanceRiskScore
    FROM DimVehicle v
),
RevenueImpact AS (
    SELECT 
        fft.VehicleKey,
        AVG(ffo.TotalOrderValue) AS AvgOrderValue,
        COUNT(DISTINCT fft.TripKey) AS TripsLast30Days
    FROM FactFleetTrips fft
    JOIN FactOrders ffo ON fft.TripKey = ffo.TripKey
    JOIN DimTime dt ON fft.TripStartTimeKey = dt.TimeKey
    WHERE dt.DateKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd') AS INT)
    GROUP BY fft.VehicleKey
)
SELECT 
    vh.VehicleID,
    vh.TirePressurePSI,
    DATEDIFF(DAY, vh.LastMaintenanceDate, GETDATE()) AS DaysSinceMaintenance,
    ri.AvgOrderValue,
    ri.TripsLast30Days,
    vh.TireRiskScore,
    vh.MaintenanceRiskScore,
    -- Composite triage score (higher = more urgent)
    ((vh.TireRiskScore + vh.MaintenanceRiskScore) * (ri.AvgOrderValue / 1000.0)) AS TriageScore
FROM VehicleHealth vh
LEFT JOIN RevenueImpact ri ON vh.VehicleKey = ri.VehicleKey
WHERE (vh.TireRiskScore + vh.MaintenanceRiskScore) > 10
ORDER BY TriageScore DESC;
```

## Data Loading Procedures

### Incremental Load for Warehouse Operations

```sql
CREATE PROCEDURE usp_IncrementalLoad_WarehouseOperations
    @LastLoadTimeKey INT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Staging table for incoming data
    CREATE TABLE #StagingWarehouseOps (
        OperationTimestamp DATETIME2,
        WarehouseCode NVARCHAR(50),
        ProductSKU NVARCHAR(50),
        OperationType NVARCHAR(50),
        Quantity DECIMAL(18,2),
        DwellTime INT,
        CycleTime INT,
        ZoneCode NVARCHAR(20),
        EmployeeID NVARCHAR(50)
    );
    
    -- Load from WMS external table (configure via PolyBase or SSIS)
    -- Example: INSERT INTO #StagingWarehouseOps SELECT * FROM ExternalWMSData
    -- WHERE OperationTimestamp > (SELECT FullDateTime FROM DimTime WHERE TimeKey = @LastLoadTimeKey)
    
    -- Transform and load into fact table
    INSERT INTO FactWarehouseOperations (
        TimeKey, DateKey, WarehouseKey, ProductKey, 
        OperationType, QuantityHandled, DwellTimeMinutes, 
        CycleTimeSeconds, ZoneKey, EmployeeKey
    )
    SELECT 
        CAST(FORMAT(s.OperationTimestamp, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        CAST(FORMAT(s.OperationTimestamp, 'yyyyMMdd') AS INT) AS DateKey,
        dw.WarehouseKey,
        dp.ProductKey,
        s.OperationType,
        s.Quantity,
        s.DwellTime,
        s.CycleTime,
        dz.ZoneKey,
        de.EmployeeKey
    FROM #StagingWarehouseOps s
    JOIN DimWarehouse dw ON s.WarehouseCode = dw.WarehouseCode
    JOIN DimProductGravity dp ON s.ProductSKU = dp.ProductSKU
    LEFT JOIN DimZone dz ON s.ZoneCode = dz.ZoneCode
    LEFT JOIN DimEmployee de ON s.EmployeeID = de.EmployeeID
    WHERE NOT EXISTS (
        -- Prevent duplicates
        SELECT 1 FROM FactWarehouseOperations fwo
        WHERE fwo.TimeKey = CAST(FORMAT(s.OperationTimestamp, 'yyyyMMddHHmm') AS INT)
            AND fwo.ProductKey = dp.ProductKey
            AND fwo.OperationType = s.OperationType
    );
    
    DROP TABLE #StagingWarehouseOps;
END;
GO
```

### Scheduled Job for Automated Refresh

```sql
-- Create SQL Agent job for 15-minute incremental loads
USE msdb;
GO

EXEC dbo.sp_add_job
    @job_name = N'LogiFleet_IncrementalLoad',
    @enabled = 1;

EXEC sp_add_jobstep
    @job_name = N'LogiFleet_IncrementalLoad',
    @step_name = N'Load Warehouse Operations',
    @subsystem = N'TSQL',
    @command = N'
        DECLARE @LastLoad INT;
        SELECT @LastLoad = MAX(TimeKey) FROM FactWarehouseOperations;
        EXEC usp_IncrementalLoad_WarehouseOperations @LastLoad;
    ',
    @database_name = N'LogiFleetPulse';

EXEC sp_add_schedule
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC sp_attach_schedule
    @job_name = N'LogiFleet_IncrementalLoad',
    @schedule_name = N'Every15Minutes';

EXEC sp_add_jobserver
    @job_name = N'LogiFleet_IncrementalLoad';
GO
```

## Power BI Configuration

### Key DAX Measures

#### Cross-Fact Bottleneck Index

```dax
Bottleneck Index = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR AvgIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR DwellHours = DIVIDE(AvgDwell, 60, 0)
VAR IdleHours = DIVIDE(AvgIdle, 60, 0)
RETURN
    DwellHours * IdleHours * 10 -- Scale factor for visualization
```

#### Fleet Fuel Efficiency

```dax
Fuel Efficiency (KM/L) = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKM]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)
```

#### Warehouse Gravity Compliance

```dax
Gravity Zone Compliance % = 
VAR TotalPicks = COUNTROWS(FactWarehouseOperations)
VAR OptimalPicks = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProductGravity[RecommendedZone] = DimZone[ZoneType]
    )
RETURN
    DIVIDE(OptimalPicks, TotalPicks, 0) * 100
```

### Row-Level Security (RLS)

```dax
-- Create roles in Power BI Desktop (Modeling > Manage Roles)

-- Role: Warehouse Manager (sees only their warehouse)
[WarehouseCode] = USERNAME()

-- Role: Regional Manager (sees multiple warehouses by region)
DimWarehouse[Region] IN { "North", "East" }

-- Role: Executive (sees all data)
-- No filter applied
```

### Scheduled Refresh

Configure in Power BI Service:
1. Publish the report to your workspace
2. Navigate to dataset settings
3. Set refresh schedule: Every 15 minutes (Premium capacity required)
4. Configure credentials for SQL Server connection (use service account with read access)

```json
{
  "refreshSchedule": {
    "days": ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday"],
    "times": ["00:15", "00:30", "00:45", "01:00", "..."],
    "enabled": true,
    "timeZone": "UTC"
  }
}
```

## Alerting Configuration

### SQL-Based Alert Procedure

```sql
CREATE PROCEDURE usp_AlertCheck_BottleneckThreshold
AS
BEGIN
    DECLARE @AlertRecipients NVARCHAR(MAX) = '${ALERT_EMAIL_RECIPIENTS}'; -- Env var
    
    -- Check for high bottleneck scores in last hour
    IF EXISTS (
        SELECT 1
        FROM (
            SELECT 
                dw.WarehouseName,
                AVG(fwo.DwellTimeMinutes) / 60.0 * AVG(fft.IdleTimeMinutes) / 60.0 AS BottleneckScore
            FROM FactWarehouseOperations fwo
            JOIN FactFleetTrips fft ON fwo.DateKey = fft.DateKey
            JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
            JOIN DimWarehouse dw ON fwo.WarehouseKey = dw.WarehouseKey
            WHERE dt.FullDateTime >= DATEADD(HOUR, -1, GETDATE())
            GROUP BY dw.WarehouseName
        ) AS Metrics
        WHERE BottleneckScore > 5 -- Alert threshold
    )
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = @AlertRecipients,
            @subject = 'LogiFleet Alert: High Bottleneck Score Detected',
            @body = 'Warehouse operations show elevated bottleneck risk. Review dashboard immediately.',
            @importance = 'High';
    END
END;
GO

-- Schedule this to run every 15 minutes via SQL Agent
```

### Power BI Alert (via Power Automate)

1. Create a Power BI alert on the "Bottleneck Index" card
2. Set threshold: > 5
3. Configure Power Automate flow:
   - Trigger: When a data driven alert is triggered
   - Action: Send email to ${OPERATIONS_TEAM_EMAIL}
   - Include: Screenshot of dashboard, direct link to report

## Troubleshooting

### Performance Issues

**Symptom:** Queries take >5 seconds

**Solution:**
```sql
-- Check missing indexes
SELECT 
    OBJECT_NAME(ips.object_id) AS TableName,
    ips.avg_user_impact,
    ips.avg_total_user_cost,
    ips.user_seeks + ips.user_scans AS TotalSeeks
FROM sys.dm_db_missing_index_group_stats AS ips
JOIN sys.dm_db_missing_index_groups AS ig ON ips.group_handle = ig.index_group_handle
WHERE ips.avg_user_impact > 50
ORDER BY ips.avg_user_impact DESC;

-- Update statistics
EXEC sp_updatestats;

-- Rebuild columnstore indexes
ALTER INDEX NCCI_FactWarehouseOps ON FactWarehouseOperations REBUILD;
ALTER INDEX NCCI_FactFleetTrips ON FactFleetTrips REBUILD;
```

### Power BI Refresh Failures

**Symptom:** "Data source error" in refresh history

**Check:**
```sql
-- Verify service account has access
USE LogiFleetPulse;
SELECT 
    dp.name AS Username,
    dp.type_desc AS AccountType,
    o.name AS ObjectName,
    p.permission_name
FROM sys.database_principals dp
JOIN sys.database_permissions p ON dp.principal_id = p.grantee_principal_id
JOIN sys.objects o ON p.major_id = o.object_id
WHERE dp.name = '${POWERBI_SERVICE_ACCOUNT}'; -- Replace with actual account
```

**Grant necessary permissions:**
```sql
GRANT SELECT ON SCHEMA::dbo TO [${POWERBI_SERVICE_ACCOUNT}];
```

### Data Quality Issues

**Symptom:** Missing records or orphaned keys

**Validation script:**
```sql
-- Check for orphaned fact records
SELECT 'FactWarehouseOperations' AS FactTable, COUNT(*) AS OrphanCount
FROM FactWarehouseOperations fwo
WHERE NOT EXISTS (SELECT 1 FROM DimProductGravity dp WHERE dp.ProductKey = fwo.ProductKey)
UNION ALL
SELECT 'FactFleetTrips', COUNT(*)
FROM FactFleetTrips fft
WHERE NOT EXISTS (SELECT 1 FROM DimVehicle dv WHERE dv.VehicleKey = fft.VehicleKey);

-- Resolve by adding missing dimension records or cleaning fact table
```

## Best Practices

1. **Always use TimeKey for joins** (not FullDateTime) for index efficiency
2. **Partition large fact tables** by DateKey (monthly partitions recommended)
3. **Use columnstore indexes** for analytical queries, B-tree for operational lookups
4. **Implement SCD Type 2** for dimensions that change over time (e.g., warehouse zone assignments)
5. **Test RLS performance** before deploying to production (can impact query times significantly)
6. **Monitor storage growth** — columnar compression typically achieves 10:1 ratio but varies by data cardinality

## Environment Variables Reference

```bash
# SQL Server
export SQL_SERVER="your-server.database.windows.net"
export SQL_DATABASE="LogiFleetPulse"
export SQL_USER="${SQL_ADMIN_USER}"
export SQL_PASSWORD="${SQL_ADMIN_PASSWORD}"

# Data Sources
export WMS_SERVER="wms-db-server"
export WMS_DB="WarehouseManagementDB"
export FLEET_API_ENDPOINT="https://api.fleet-provider.com/v2"
export FLEET_API_TOKEN="${FLEET_TELEMETRY_TOKEN}"
export WEATHER_API_KEY="${WEATHER_SERVICE_KEY}"

# Alerting
export ALERT_EMAIL_RECIPIENTS="ops-team@company.com;managers@company.com"
export OPERATIONS_TEAM_EMAIL="operations@company.com"

# Power BI Service
export POWERBI_SERVICE_ACCOUNT="svc-powerbi@company.com"
```
