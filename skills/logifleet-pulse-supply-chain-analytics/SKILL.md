---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing system for logistics with multi-fact star schema, fleet telemetry, and warehouse operations analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "create multi-fact star schema for warehouse data"
  - "build fleet telemetry data model"
  - "implement logistics KPI tracking system"
  - "design warehouse gravity zones in SQL"
  - "connect WMS data to Power BI"
  - "optimize supply chain data warehouse"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What It Does

LogiFleet Pulse is a multi-modal logistics intelligence platform that combines:
- **MS SQL Server data warehousing** with multi-fact star schema
- **Power BI dashboards** for real-time fleet and warehouse visualization
- **Warehouse operations analytics** (putaway, picking, packing, dwell time)
- **Fleet telemetry integration** (GPS, fuel consumption, maintenance)
- **Cross-fact KPI harmonization** linking inventory with fleet metrics
- **Predictive bottleneck detection** using time-phased dimensions

The system uses a custom star schema with time-aware dimensions (15-minute granularity) and bridge tables for many-to-many relationships between routes and storage zones.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Database permissions: CREATE TABLE, CREATE VIEW, CREATE PROCEDURE
- Optional: Azure Synapse Analytics for external data enrichment

### Step 1: Clone Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Replace with your database name
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute schema deployment script
-- (Assumes schema.sql is in the repository root)
:r schema.sql
```

### Step 3: Configure Data Sources

Create a configuration file for connection strings:

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "auth": "integrated"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telemetry_api": "${FLEET_TELEMETRY_ENDPOINT}",
    "weather_api": "${WEATHER_API_ENDPOINT}"
  },
  "refresh_interval_minutes": 15
}
```

### Step 4: Import Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. When prompted, enter SQL Server connection details
3. Configure row-level security roles in Model view
4. Publish to Power BI Service (optional)

## Core Data Model Structure

### Fact Tables

#### FactWarehouseOperations

```sql
CREATE TABLE FactWarehouseOperations (
    OperationID INT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    EmployeeKey INT,
    OperationType VARCHAR(50), -- 'Putaway', 'Pick', 'Pack', 'Ship'
    Quantity DECIMAL(18,2),
    DwellTimeMinutes INT,
    ProcessTimeSeconds INT,
    ZoneGravityScore DECIMAL(5,2),
    BatchNumber VARCHAR(50),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
);

-- Recommended indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey) INCLUDE (Quantity, DwellTimeMinutes);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey) INCLUDE (OperationType, Quantity);
CREATE COLUMNSTORE INDEX CCI_FactWarehouse ON FactWarehouseOperations;
```

#### FactFleetTrips

```sql
CREATE TABLE FactFleetTrips (
    TripID INT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT,
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    AverageSpeed DECIMAL(5,2),
    WeatherCondition VARCHAR(50),
    DelayReasonCode VARCHAR(20),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey) INCLUDE (FuelConsumedGallons, IdleTimeMinutes);
CREATE NONCLUSTERED INDEX IX_FactFleet_Route ON FactFleetTrips(RouteKey) INCLUDE (DistanceMiles, DelayReasonCode);
```

### Dimension Tables

#### DimTime (15-minute granularity)

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    DayOfMonth INT,
    DayOfWeek INT,
    DayName VARCHAR(20),
    Hour INT,
    Minute INT,
    QuarterHour INT, -- 0, 15, 30, 45
    IsWeekend BIT,
    IsHoliday BIT,
    ShiftType VARCHAR(20), -- 'Morning', 'Evening', 'Night'
    FiscalYear INT,
    FiscalQuarter INT
);

-- Populate DimTime with 15-minute intervals
DECLARE @StartDate DATETIME2 = '2025-01-01 00:00:00';
DECLARE @EndDate DATETIME2 = '2027-12-31 23:45:00';

;WITH TimeSeries AS (
    SELECT @StartDate AS FullDateTime
    UNION ALL
    SELECT DATEADD(MINUTE, 15, FullDateTime)
    FROM TimeSeries
    WHERE DATEADD(MINUTE, 15, FullDateTime) <= @EndDate
)
INSERT INTO DimTime (TimeKey, FullDateTime, Date, Year, Month, Hour, Minute, QuarterHour)
SELECT 
    CONVERT(INT, FORMAT(FullDateTime, 'yyyyMMddHHmm')),
    FullDateTime,
    CAST(FullDateTime AS DATE),
    YEAR(FullDateTime),
    MONTH(FullDateTime),
    DATEPART(HOUR, FullDateTime),
    DATEPART(MINUTE, FullDateTime),
    (DATEPART(MINUTE, FullDateTime) / 15) * 15
FROM TimeSeries
OPTION (MAXRECURSION 0);
```

#### DimProductGravity (Warehouse Gravity Zones™)

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(255),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    UnitValue DECIMAL(10,2),
    WeightLbs DECIMAL(8,2),
    IsFragile BIT,
    -- Gravity Score Components
    PickFrequency INT, -- Picks per day average
    ReplenishmentLeadTimeDays INT,
    ValueDensity DECIMAL(10,2), -- UnitValue / WeightLbs
    GravityScore AS (
        (PickFrequency * 0.5) + 
        ((100.0 / NULLIF(ReplenishmentLeadTimeDays, 0)) * 0.3) +
        (ValueDensity * 0.2)
    ) PERSISTED,
    RecommendedZone VARCHAR(20) -- 'High-Gravity', 'Medium-Gravity', 'Low-Gravity'
);

-- Assign zones based on gravity score
UPDATE DimProductGravity
SET RecommendedZone = CASE
    WHEN GravityScore >= 75 THEN 'High-Gravity'
    WHEN GravityScore >= 40 THEN 'Medium-Gravity'
    ELSE 'Low-Gravity'
END;
```

## Key Queries & Patterns

### Cross-Fact KPI: Dwell Time vs Fleet Idle Cost

```sql
-- Find correlation between warehouse dwell time and fleet idling costs
WITH WarehouseDwell AS (
    SELECT 
        dt.Date,
        dp.Category,
        AVG(fwo.DwellTimeMinutes) AS AvgDwellMinutes,
        SUM(fwo.Quantity) AS TotalUnits
    FROM FactWarehouseOperations fwo
    JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    WHERE dt.Date >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY dt.Date, dp.Category
),
FleetIdle AS (
    SELECT 
        dt.Date,
        SUM(fft.IdleTimeMinutes) AS TotalIdleMinutes,
        SUM(fft.FuelConsumedGallons * 3.50) AS EstimatedIdleCost -- $3.50/gallon
    FROM FactFleetTrips fft
    JOIN DimTime dt ON fft.TimeKey = dt.TimeKey
    WHERE dt.Date >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY dt.Date
)
SELECT 
    wd.Date,
    wd.Category,
    wd.AvgDwellMinutes,
    fi.TotalIdleMinutes,
    fi.EstimatedIdleCost,
    -- Correlation indicator
    (wd.AvgDwellMinutes / NULLIF(wd.TotalUnits, 0)) * fi.EstimatedIdleCost AS ImpactScore
FROM WarehouseDwell wd
JOIN FleetIdle fi ON wd.Date = fi.Date
ORDER BY ImpactScore DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify storage zones approaching capacity with high-gravity items
CREATE PROCEDURE sp_DetectBottlenecks
AS
BEGIN
    SELECT 
        wz.ZoneName,
        wz.CurrentCapacityPercent,
        COUNT(DISTINCT fwo.ProductKey) AS UniqueHighGravitySKUs,
        AVG(dpg.GravityScore) AS AvgGravityScore,
        SUM(fwo.Quantity) AS TotalUnitsLastWeek,
        -- Bottleneck risk score
        CASE 
            WHEN wz.CurrentCapacityPercent > 85 AND AVG(dpg.GravityScore) > 70 THEN 'CRITICAL'
            WHEN wz.CurrentCapacityPercent > 75 AND AVG(dpg.GravityScore) > 50 THEN 'HIGH'
            WHEN wz.CurrentCapacityPercent > 65 THEN 'MEDIUM'
            ELSE 'LOW'
        END AS BottleneckRisk
    FROM FactWarehouseOperations fwo
    JOIN DimProductGravity dpg ON fwo.ProductKey = dpg.ProductKey
    JOIN DimWarehouseZone wz ON fwo.WarehouseKey = wz.WarehouseKey
    JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    WHERE dt.Date >= DATEADD(DAY, -7, GETDATE())
      AND dpg.GravityScore > 50 -- Focus on high-gravity items
    GROUP BY wz.ZoneName, wz.CurrentCapacityPercent
    HAVING wz.CurrentCapacityPercent > 60
    ORDER BY BottleneckRisk DESC, AvgGravityScore DESC;
END;
```

### Fleet Maintenance Triage Engine

```sql
-- Prioritize maintenance based on revenue impact
WITH VehicleMetrics AS (
    SELECT 
        dv.VehicleID,
        dv.VehicleType,
        dv.LastMaintenanceDate,
        dv.TirePressurePSI,
        dv.EngineDiagnosticCode,
        AVG(fft.FuelConsumedGallons / NULLIF(fft.DistanceMiles, 0)) AS AvgMPG,
        SUM(CASE WHEN fft.DelayReasonCode = 'MECHANICAL' THEN 1 ELSE 0 END) AS MechanicalDelays,
        -- Estimate revenue impact based on cargo value
        AVG(cargo.TotalValue) AS AvgCargoValue
    FROM DimVehicle dv
    JOIN FactFleetTrips fft ON dv.VehicleKey = fft.VehicleKey
    LEFT JOIN (
        SELECT TripID, SUM(UnitValue * Quantity) AS TotalValue
        FROM TripCargo tc
        JOIN DimProductGravity dpg ON tc.ProductKey = dpg.ProductKey
        GROUP BY TripID
    ) cargo ON fft.TripID = cargo.TripID
    WHERE fft.TripStartTime >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY dv.VehicleID, dv.VehicleType, dv.LastMaintenanceDate, dv.TirePressurePSI, dv.EngineDiagnosticCode
)
SELECT 
    VehicleID,
    VehicleType,
    DATEDIFF(DAY, LastMaintenanceDate, GETDATE()) AS DaysSinceMaintenance,
    TirePressurePSI,
    EngineDiagnosticCode,
    AvgMPG,
    MechanicalDelays,
    AvgCargoValue,
    -- Priority score: higher = more urgent
    (MechanicalDelays * 10) + 
    (CASE WHEN TirePressurePSI < 30 THEN 15 ELSE 0 END) +
    (DATEDIFF(DAY, LastMaintenanceDate, GETDATE()) / 10) +
    (AvgCargoValue / 1000) AS MaintenancePriorityScore
FROM VehicleMetrics
WHERE 
    TirePressurePSI < 32 
    OR DATEDIFF(DAY, LastMaintenanceDate, GETDATE()) > 90
    OR EngineDiagnosticCode IS NOT NULL
ORDER BY MaintenancePriorityScore DESC;
```

## Power BI Integration

### DAX Measures for Cross-Fact Analytics

```dax
// Measure: Weighted Average Dwell Time
Weighted Dwell Time = 
DIVIDE(
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[DwellTimeMinutes] * FactWarehouseOperations[Quantity]
    ),
    SUM(FactWarehouseOperations[Quantity]),
    0
)

// Measure: Fleet Efficiency Index
Fleet Efficiency Index = 
VAR TotalDistance = SUM(FactFleetTrips[DistanceMiles])
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
VAR TotalFuel = SUM(FactFleetTrips[FuelConsumedGallons])
RETURN
    DIVIDE(
        TotalDistance,
        (TotalIdleTime / 60) + (TotalFuel * 0.5), // Penalize idle time and fuel consumption
        0
    )

// Measure: Gravity Zone Compliance
Gravity Zone Compliance = 
VAR ActualZone = SELECTEDVALUE(DimWarehouseZone[ZoneType])
VAR RecommendedZone = SELECTEDVALUE(DimProductGravity[RecommendedZone])
RETURN
    IF(
        ActualZone = RecommendedZone,
        1,
        0
    )

// Measure: Cross-Fact Revenue at Risk
Revenue at Risk = 
VAR DelayedTrips = 
    FILTER(
        FactFleetTrips,
        FactFleetTrips[DelayReasonCode] <> BLANK()
    )
VAR DelayedTripIDs = VALUES(FactFleetTrips[TripID])
RETURN
    CALCULATE(
        SUMX(
            TripCargo,
            TripCargo[Quantity] * RELATED(DimProductGravity[UnitValue])
        ),
        DelayedTripIDs
    )
```

### Row-Level Security (RLS)

```dax
// Role: Regional Manager (only sees data for assigned regions)
[RegionCode] = USERNAME()

// Role: Warehouse Supervisor (only sees specific warehouse)
[WarehouseKey] IN {
    LOOKUPVALUE(
        UserWarehouseAccess[WarehouseKey],
        UserWarehouseAccess[UserEmail],
        USERNAME()
    )
}
```

## Data Ingestion Patterns

### Incremental Load from WMS API

```sql
CREATE PROCEDURE sp_IncrementalLoadWarehouseOps
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Stage external data
    INSERT INTO StageWarehouseOps (
        OperationTimestamp,
        SKU,
        WarehouseCode,
        OperationType,
        Quantity,
        ProcessTimeSeconds
    )
    SELECT 
        op.Timestamp,
        op.SKU,
        op.WarehouseCode,
        op.Type,
        op.Qty,
        op.ProcessTime
    FROM OPENROWSET(
        BULK '${WMS_API_ENDPOINT}',
        FORMAT = 'JSON',
        FORMATFILE = 'wms_schema.json'
    ) AS op
    WHERE op.Timestamp > @LastLoadTimestamp;
    
    -- Transform and load to fact table
    INSERT INTO FactWarehouseOperations (
        TimeKey,
        ProductKey,
        WarehouseKey,
        OperationType,
        Quantity,
        ProcessTimeSeconds
    )
    SELECT 
        CONVERT(INT, FORMAT(s.OperationTimestamp, 'yyyyMMddHHmm')),
        dp.ProductKey,
        dw.WarehouseKey,
        s.OperationType,
        s.Quantity,
        s.ProcessTimeSeconds
    FROM StageWarehouseOps s
    JOIN DimProductGravity dp ON s.SKU = dp.SKU
    JOIN DimWarehouse dw ON s.WarehouseCode = dw.WarehouseCode
    WHERE NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations f
        WHERE f.TimeKey = CONVERT(INT, FORMAT(s.OperationTimestamp, 'yyyyMMddHHmm'))
          AND f.ProductKey = dp.ProductKey
          AND f.WarehouseKey = dw.WarehouseKey
    );
    
    -- Update last load timestamp
    UPDATE ETLControl
    SET LastLoadTimestamp = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
```

### Real-Time Fleet Telemetry Integration

```sql
-- External table for streaming data (requires SQL Server 2019+)
CREATE EXTERNAL DATA SOURCE FleetTelemetryAPI
WITH (
    TYPE = REST,
    LOCATION = '${FLEET_TELEMETRY_ENDPOINT}',
    CREDENTIAL = FleetAPICredential
);

-- Query live telemetry
SELECT 
    VehicleID,
    CurrentLocation,
    FuelLevel,
    Speed,
    EngineTemp,
    TirePressure,
    EventTimestamp
FROM OPENROWSET(
    BULK '/telemetry/live',
    DATA_SOURCE = 'FleetTelemetryAPI',
    FORMAT = 'JSON'
) AS Telemetry
WHERE EventTimestamp > DATEADD(MINUTE, -15, GETDATE());
```

## Alerting & Automation

### SQL Agent Job for Anomaly Detection

```sql
-- Create alert for critical bottlenecks
CREATE PROCEDURE sp_AlertCriticalBottlenecks
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    SELECT @AlertMessage = STRING_AGG(
        'Zone: ' + ZoneName + 
        ' | Capacity: ' + CAST(CurrentCapacityPercent AS VARCHAR) + '%' +
        ' | Risk: ' + BottleneckRisk,
        CHAR(13) + CHAR(10)
    )
    FROM (
        EXEC sp_DetectBottlenecks
    ) AS Bottlenecks
    WHERE BottleneckRisk = 'CRITICAL';
    
    IF @AlertMessage IS NOT NULL
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = 'CRITICAL: Warehouse Bottleneck Detected',
            @body = @AlertMessage;
    END;
END;

-- Schedule job to run every 15 minutes
EXEC msdb.dbo.sp_add_job @job_name = 'BottleneckMonitor';
EXEC msdb.dbo.sp_add_jobstep 
    @job_name = 'BottleneckMonitor',
    @step_name = 'Check Bottlenecks',
    @subsystem = 'TSQL',
    @command = 'EXEC sp_AlertCriticalBottlenecks';
EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = 'Every15Min',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;
EXEC msdb.dbo.sp_attach_schedule 
    @job_name = 'BottleneckMonitor',
    @schedule_name = 'Every15Min';
```

## Common Troubleshooting

### Issue: Power BI Refresh Timeout

**Cause**: Large fact tables causing query timeouts (>30 minutes)

**Solution**: Implement incremental refresh in Power BI

```powerquery
// Power Query M: Incremental Refresh Parameter
let
    Source = Sql.Database(
        "${SQL_SERVER_HOST}", 
        "LogiFleetPulse",
        [Query="
            SELECT * FROM FactWarehouseOperations
            WHERE FullDateTime >= #datetime(2026, 1, 1, 0, 0, 0)
              AND FullDateTime < #datetime(2026, 12, 31, 23, 59, 59)
        "]
    )
in
    Source
```

Enable incremental refresh in Power BI Desktop:
- Right-click table → Incremental refresh
- Set date range parameters: `RangeStart`, `RangeEnd`
- Archive last 2 years, refresh last 7 days

### Issue: Slow Cross-Fact Queries

**Cause**: Missing indexes on foreign keys

**Solution**: Apply recommended indexes

```sql
-- Analyze missing indexes
SELECT 
    dm.database_id,
    dm.object_id,
    dm.index_handle,
    dm.avg_user_impact,
    dm.avg_total_user_cost,
    dm.user_seeks,
    dm.user_scans,
    -- Generate CREATE INDEX statement
    'CREATE NONCLUSTERED INDEX IX_' + 
    OBJECT_NAME(dm.object_id) + '_' + 
    REPLACE(REPLACE(mid.equality_columns, '[', ''), ']', '') +
    ' ON ' + OBJECT_NAME(dm.object_id) + 
    ' (' + mid.equality_columns + ') INCLUDE (' + mid.included_columns + ');'
FROM sys.dm_db_missing_index_details mid
JOIN sys.dm_db_missing_index_groups mig ON mid.index_handle = mig.index_handle
JOIN sys.dm_db_missing_index_group_stats dm ON mig.index_group_handle = dm.group_handle
WHERE dm.avg_user_impact > 50
ORDER BY dm.avg_user_impact DESC;
```

### Issue: Gravity Score Not Updating

**Cause**: Computed column not recalculating after source data change

**Solution**: Force recalculation

```sql
-- Drop and recreate computed column
ALTER TABLE DimProductGravity DROP COLUMN GravityScore;

ALTER TABLE DimProductGravity ADD GravityScore AS (
    (PickFrequency * 0.5) + 
    ((100.0 / NULLIF(ReplenishmentLeadTimeDays, 0)) * 0.3) +
    (ValueDensity * 0.2)
) PERSISTED;

-- Update recommended zones
UPDATE DimProductGravity
SET RecommendedZone = CASE
    WHEN GravityScore >= 75 THEN 'High-Gravity'
    WHEN GravityScore >= 40 THEN 'Medium-Gravity'
    ELSE 'Low-Gravity'
END;
```

## Best Practices

### Partitioning Strategy

```sql
-- Partition FactWarehouseOperations by month
CREATE PARTITION FUNCTION PF_MonthlyRange (INT)
AS RANGE RIGHT FOR VALUES (
    20260101, 20260201, 20260301, 20260401, 20260501, 20260601,
    20260701, 20260801, 20260901, 20261001, 20261101, 20261201
);

CREATE PARTITION SCHEME PS_MonthlyRange
AS PARTITION PF_MonthlyRange
ALL TO ([PRIMARY]);

-- Apply to fact table
CREATE TABLE FactWarehouseOperations_Partitioned (
    -- ... same columns ...
) ON PS_MonthlyRange(TimeKey);
```

### Security Hardening

```sql
-- Create read-only role for Power BI service account
CREATE ROLE PowerBIReader;
GRANT SELECT ON SCHEMA::dbo TO PowerBIReader;
DENY INSERT, UPDATE, DELETE ON SCHEMA::dbo TO PowerBIReader;

CREATE USER [${POWERBI_SERVICE_ACCOUNT}] FOR LOGIN [${POWERBI_SERVICE_ACCOUNT}];
ALTER ROLE PowerBIReader ADD MEMBER [${POWERBI_SERVICE_ACCOUNT}];
```

### Monitoring Query Performance

```sql
-- Track top 10 slowest queries
SELECT TOP 10
    qs.execution_count,
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed_time,
    qs.total_worker_time / qs.execution_count AS avg_cpu_time,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE st.text LIKE '%FactWarehouse%' OR st.text LIKE '%FactFleet%'
ORDER BY avg_elapsed_time DESC;
```

## Environment Variables

Required environment variables for secure configuration:

- `SQL_SERVER_HOST`: SQL Server instance hostname/IP
- `SQL_SERVER_DATABASE`: Database name (default: LogiFleetPulse)
- `WMS_API_ENDPOINT`: Warehouse Management System API URL
- `FLEET_TELEMETRY_ENDPOINT`: Fleet GPS/telemetry API URL
- `WEATHER_API_ENDPOINT`: External weather data API
- `ALERT_EMAIL`: Email address for critical alerts
- `POWERBI_SERVICE_ACCOUNT`: Service account for Power BI refresh

Store credentials securely using SQL Server Credential objects or Azure Key Vault.
