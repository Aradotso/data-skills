---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing platform for logistics intelligence, fleet tracking, and warehouse optimization with multi-fact star schema modeling
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy the logistics data warehouse schema"
  - "configure Power BI for fleet and warehouse KPIs"
  - "create cross-fact queries for supply chain analytics"
  - "implement warehouse gravity zone optimization"
  - "build real-time logistics dashboards"
  - "query fleet telemetry and warehouse operations"
  - "set up predictive bottleneck detection"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI coding agents to help developers implement **LogiFleet Pulse**, an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and external data signals into a multi-fact star schema data warehouse. Built on MS SQL Server with Power BI visualization, it provides cross-modal supply chain analytics and predictive insights.

## What LogiFleet Pulse Does

LogiFleet Pulse is a **multi-modal logistics intelligence engine** that:

- Consolidates warehouse operations (receiving, putaway, picking, packing, shipping)
- Tracks fleet telemetry (GPS, fuel consumption, driver behavior, maintenance)
- Monitors inventory aging, supplier reliability, and demand patterns
- Integrates external signals (weather, traffic) as risk factors
- Provides cross-fact KPI harmonization (e.g., dwell time vs. fleet idling cost)
- Enables temporal elasticity modeling and predictive bottleneck detection
- Delivers role-based Power BI dashboards with real-time alerts

**Key Differentiators:**
- Multi-fact star schema with time-phased dimensions
- Warehouse Gravity Zones™ for spatial optimization
- Adaptive Fleet Triage Engine for maintenance prioritization
- Cross-fact queries linking warehouse and fleet metrics

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, TMS, telematics APIs, ERP systems

### Database Deployment

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute schema deployment script
-- (Assumes schema.sql is in the repository)
:r schema.sql
GO
```

3. **Configure connection strings:**
```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "windows",
    "trusted_connection": true
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "fleet_telemetry": "${FLEET_API_ENDPOINT}",
    "weather_api": "${WEATHER_API_ENDPOINT}"
  }
}
```

4. **Import Power BI template:**
- Open `LogiFleet_Pulse_Master.pbit`
- Connect to your SQL Server instance
- Configure data refresh schedule

## Core Data Model

### Fact Tables

#### FactWarehouseOperations
Captures warehouse micro-operations with time-phased granularity.

```sql
-- Query warehouse operations by gravity zone
SELECT 
    dg.GravityZoneName,
    dt.FiscalQuarter,
    SUM(fwo.DwellTimeMinutes) AS TotalDwellTime,
    AVG(fwo.PickRatePerHour) AS AvgPickRate,
    COUNT(DISTINCT fwo.SKU) AS UniqueSKUs
FROM FactWarehouseOperations fwo
INNER JOIN DimProductGravity dg ON fwo.ProductGravityKey = dg.ProductGravityKey
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
WHERE dt.CalendarYear = 2026
GROUP BY dg.GravityZoneName, dt.FiscalQuarter
ORDER BY TotalDwellTime DESC;
```

#### FactFleetTrips
Tracks fleet movements, fuel consumption, and route efficiency.

```sql
-- Identify routes with high idle time and fuel waste
SELECT 
    dg.RegionName,
    ft.RouteID,
    SUM(ft.IdleTimeMinutes) AS TotalIdleTime,
    SUM(ft.FuelConsumedLiters) AS TotalFuel,
    (SUM(ft.IdleTimeMinutes) * 1.0 / NULLIF(SUM(ft.TripDurationMinutes), 0)) * 100 AS IdlePercentage
FROM FactFleetTrips ft
INNER JOIN DimGeography dg ON ft.OriginGeographyKey = dg.GeographyKey
INNER JOIN DimTime dt ON ft.DepartureTimeKey = dt.TimeKey
WHERE dt.CalendarMonth = '2026-06'
    AND ft.IdleTimeMinutes > 0
GROUP BY dg.RegionName, ft.RouteID
HAVING (SUM(ft.IdleTimeMinutes) * 1.0 / NULLIF(SUM(ft.TripDurationMinutes), 0)) * 100 > 15
ORDER BY IdlePercentage DESC;
```

#### FactCrossDock
Tracks transfers between inbound and outbound without long-term storage.

```sql
-- Measure cross-dock efficiency
SELECT 
    dt.DayOfWeek,
    dt.HourOfDay,
    COUNT(*) AS CrossDockCount,
    AVG(fcd.TransferTimeMinutes) AS AvgTransferTime,
    SUM(CASE WHEN fcd.TransferTimeMinutes <= 30 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS FastTransferPercentage
FROM FactCrossDock fcd
INNER JOIN DimTime dt ON fcd.ArrivalTimeKey = dt.TimeKey
WHERE dt.CalendarMonth = '2026-06'
GROUP BY dt.DayOfWeek, dt.HourOfDay
ORDER BY dt.DayOfWeek, dt.HourOfDay;
```

### Dimension Tables

#### DimProductGravity
Novel dimension assigning "gravity scores" based on velocity, value, fragility.

```sql
-- Create product gravity dimension
CREATE TABLE DimProductGravity (
    ProductGravityKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) NOT NULL,
    ProductName NVARCHAR(255),
    GravityScore DECIMAL(5,2), -- Composite score
    VelocityRank INT, -- Pick frequency rank
    ValuePerUnit DECIMAL(10,2),
    FragilityIndex DECIMAL(3,2), -- 0.0 to 1.0
    GravityZoneName NVARCHAR(50), -- High, Medium, Low gravity
    RecommendedStorageLocation NVARCHAR(100),
    LastRecalculatedDate DATETIME2
);

-- Calculate gravity scores (example stored procedure)
CREATE PROCEDURE uspRecalculateProductGravity
AS
BEGIN
    UPDATE DimProductGravity
    SET GravityScore = (VelocityRank * 0.5) + (ValuePerUnit / 100 * 0.3) + (FragilityIndex * 0.2),
        LastRecalculatedDate = GETUTCDATE();
    
    -- Assign zones based on score
    UPDATE DimProductGravity
    SET GravityZoneName = CASE 
        WHEN GravityScore >= 75 THEN 'High Gravity'
        WHEN GravityScore >= 40 THEN 'Medium Gravity'
        ELSE 'Low Gravity'
    END;
END;
```

#### DimTime
Time-phased dimension with 15-minute granularity.

```sql
-- Query by time slices
SELECT 
    dt.CalendarDate,
    dt.HourOfDay,
    dt.QuarterHour, -- 0, 15, 30, 45
    COUNT(fwo.OperationID) AS OperationCount
FROM DimTime dt
LEFT JOIN FactWarehouseOperations fwo ON dt.TimeKey = fwo.TimeKey
WHERE dt.CalendarDate BETWEEN '2026-06-01' AND '2026-06-30'
    AND dt.IsBusinessDay = 1
GROUP BY dt.CalendarDate, dt.HourOfDay, dt.QuarterHour
ORDER BY dt.CalendarDate, dt.HourOfDay, dt.QuarterHour;
```

#### DimSupplierReliability
Tracks supplier performance metrics.

```sql
-- Identify unreliable suppliers impacting warehouse dwell
SELECT 
    ds.SupplierName,
    ds.LeadTimeVarianceDays,
    ds.DefectPercentage,
    ds.ComplianceScore,
    AVG(fwo.DwellTimeMinutes) AS AvgDwellTime
FROM DimSupplierReliability ds
INNER JOIN FactWarehouseOperations fwo ON ds.SupplierKey = fwo.SupplierKey
WHERE ds.ComplianceScore < 80
GROUP BY ds.SupplierName, ds.LeadTimeVarianceDays, ds.DefectPercentage, ds.ComplianceScore
ORDER BY AvgDwellTime DESC;
```

## Cross-Fact KPI Queries

### Dwell Time vs. Fleet Idling Cost

```sql
-- Correlate warehouse dwell with fleet inefficiency
WITH WarehouseDwell AS (
    SELECT 
        fwo.SKU,
        dt.CalendarMonth,
        SUM(fwo.DwellTimeMinutes) AS TotalDwellTime
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    GROUP BY fwo.SKU, dt.CalendarMonth
),
FleetIdle AS (
    SELECT 
        ft.CargoSKU,
        dt.CalendarMonth,
        SUM(ft.IdleTimeMinutes * ft.FuelCostPerMinute) AS IdleCost
    FROM FactFleetTrips ft
    INNER JOIN DimTime dt ON ft.DepartureTimeKey = dt.TimeKey
    GROUP BY ft.CargoSKU, dt.CalendarMonth
)
SELECT 
    wd.SKU,
    wd.CalendarMonth,
    wd.TotalDwellTime,
    ISNULL(fi.IdleCost, 0) AS FleetIdleCost,
    (wd.TotalDwellTime * 1.0 / NULLIF(fi.IdleCost, 0)) AS DwellToIdleRatio
FROM WarehouseDwell wd
LEFT JOIN FleetIdle fi ON wd.SKU = fi.CargoSKU AND wd.CalendarMonth = fi.CalendarMonth
WHERE wd.TotalDwellTime > 500
ORDER BY FleetIdleCost DESC;
```

### Predictive Bottleneck Index

```sql
-- Generate bottleneck heatmap scores
CREATE VIEW vwBottleneckIndex AS
SELECT 
    dg.WarehouseName,
    dt.CalendarDate,
    dt.HourOfDay,
    COUNT(fwo.OperationID) AS OperationVolume,
    AVG(fwo.DwellTimeMinutes) AS AvgDwell,
    SUM(CASE WHEN fwo.DwellTimeMinutes > 180 THEN 1 ELSE 0 END) AS HighDwellCount,
    -- Bottleneck score: volume * dwell * high-dwell ratio
    (COUNT(fwo.OperationID) * AVG(fwo.DwellTimeMinutes) * 
     (SUM(CASE WHEN fwo.DwellTimeMinutes > 180 THEN 1 ELSE 0 END) * 1.0 / COUNT(*))) AS BottleneckScore
FROM FactWarehouseOperations fwo
INNER JOIN DimGeography dg ON fwo.WarehouseKey = dg.GeographyKey
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
GROUP BY dg.WarehouseName, dt.CalendarDate, dt.HourOfDay;

-- Query high-risk periods
SELECT * FROM vwBottleneckIndex
WHERE BottleneckScore > 5000
ORDER BY BottleneckScore DESC;
```

## Data Ingestion Patterns

### Incremental Loading

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE uspLoadWarehouseOperations
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductGravityKey, SupplierKey,
        OperationType, DwellTimeMinutes, PickRatePerHour, PackingTimeSeconds
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dpg.ProductGravityKey,
        ds.SupplierKey,
        src.OperationType,
        src.DwellTimeMinutes,
        src.PickRatePerHour,
        src.PackingTimeSeconds
    FROM ExternalWarehouseData src
    INNER JOIN DimTime dt ON CAST(src.OperationTimestamp AS DATE) = dt.CalendarDate 
        AND DATEPART(HOUR, src.OperationTimestamp) = dt.HourOfDay
        AND (DATEPART(MINUTE, src.OperationTimestamp) / 15) * 15 = dt.QuarterHour
    INNER JOIN DimGeography dg ON src.WarehouseCode = dg.LocationCode
    INNER JOIN DimProductGravity dpg ON src.SKU = dpg.SKU
    INNER JOIN DimSupplierReliability ds ON src.SupplierID = ds.SupplierID
    WHERE src.OperationTimestamp > @LastLoadTimestamp;
    
    -- Update last load timestamp
    UPDATE ETLControl
    SET LastLoadTimestamp = GETUTCDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
```

### External Data Integration

```sql
-- Create external table for weather data (SQL Server PolyBase)
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = HADOOP,
    LOCATION = '${WEATHER_API_ENDPOINT}'
);

CREATE EXTERNAL FILE FORMAT JSONFormat
WITH (FORMAT_TYPE = JSON);

CREATE EXTERNAL TABLE ExtWeatherData (
    LocationCode NVARCHAR(50),
    Timestamp DATETIME2,
    Temperature DECIMAL(5,2),
    Precipitation DECIMAL(5,2),
    RoadCondition NVARCHAR(50)
)
WITH (
    LOCATION = '/weather/current',
    DATA_SOURCE = WeatherAPI,
    FILE_FORMAT = JSONFormat
);

-- Join weather with fleet delays
SELECT 
    ft.RouteID,
    ft.DepartureTimestamp,
    w.Temperature,
    w.RoadCondition,
    ft.DelayMinutes
FROM FactFleetTrips ft
INNER JOIN ExtWeatherData w 
    ON ft.OriginLocationCode = w.LocationCode
    AND ABS(DATEDIFF(MINUTE, ft.DepartureTimestamp, w.Timestamp)) < 30
WHERE ft.DelayMinutes > 15;
```

## Power BI Integration

### DAX Measures

```dax
// Total Dwell Time (Hours)
TotalDwellTimeHours = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeMinutes] / 60
)

// Fleet Utilization Rate
FleetUtilizationRate = 
DIVIDE(
    SUM(FactFleetTrips[ActiveDrivingMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes]),
    0
) * 100

// Cross-Fact: Dwell Impact on Delivery
DwellImpactScore = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR AvgDelay = AVERAGE(FactFleetTrips[DelayMinutes])
RETURN
    AvgDwell * AvgDelay / 100

// Gravity Zone Efficiency
GravityZoneEfficiency = 
DIVIDE(
    CALCULATE(
        SUM(FactWarehouseOperations[PickRatePerHour]),
        DimProductGravity[GravityZoneName] = "High Gravity"
    ),
    CALCULATE(
        SUM(FactWarehouseOperations[PickRatePerHour]),
        DimProductGravity[GravityZoneName] = "Low Gravity"
    ),
    0
)
```

### Row-Level Security

```dax
// Role: Regional Manager (only sees their region)
[RegionName] = USERNAME()

// Role: Warehouse Supervisor (only sees their warehouse)
[WarehouseName] = LOOKUPVALUE(
    UserWarehouseMapping[WarehouseName],
    UserWarehouseMapping[UserEmail],
    USERPRINCIPALNAME()
)
```

### Scheduled Refresh Configuration

```json
{
  "refreshSchedule": {
    "frequency": "15min",
    "retryAttempts": 3,
    "notifyOnFailure": true,
    "emailRecipients": ["${ALERT_EMAIL}"]
  },
  "incrementalRefresh": {
    "enabled": true,
    "rollingWindow": {
      "days": 90,
      "archiveAfterDays": 365
    }
  }
}
```

## Alerting & Monitoring

### SQL Server Agent Alert Job

```sql
-- Create alert for high dwell time
CREATE PROCEDURE uspAlertHighDwell
AS
BEGIN
    DECLARE @AlertThreshold INT = 240; -- 4 hours
    DECLARE @EmailRecipient NVARCHAR(255) = '${ALERT_EMAIL}';
    DECLARE @Subject NVARCHAR(255);
    DECLARE @Body NVARCHAR(MAX);
    
    IF EXISTS (
        SELECT 1 
        FROM FactWarehouseOperations fwo
        INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
        WHERE dt.CalendarDate = CAST(GETDATE() AS DATE)
            AND fwo.DwellTimeMinutes > @AlertThreshold
    )
    BEGIN
        SET @Subject = 'LogiFleet Alert: High Dwell Time Detected';
        SET @Body = 'Multiple SKUs exceeding ' + CAST(@AlertThreshold AS NVARCHAR) + ' minutes dwell time today.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = @EmailRecipient,
            @subject = @Subject,
            @body = @Body;
    END;
END;

-- Schedule job to run every 30 minutes
EXEC msdb.dbo.sp_add_job @job_name = 'LogiFleet_DwellAlert';
EXEC msdb.dbo.sp_add_jobstep 
    @job_name = 'LogiFleet_DwellAlert',
    @step_name = 'CheckDwell',
    @command = 'EXEC uspAlertHighDwell;';
EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = 'Every30Min',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 30;
EXEC msdb.dbo.sp_attach_schedule 
    @job_name = 'LogiFleet_DwellAlert',
    @schedule_name = 'Every30Min';
```

### Fleet Maintenance Triage

```sql
-- Adaptive fleet triage scoring
CREATE VIEW vwFleetMaintenancePriority AS
SELECT 
    ft.VehicleID,
    ft.EngineDiagnosticCode,
    ft.TirePressurePSI,
    ft.LoadWeightKg,
    dpg.ValuePerUnit AS CargoValue,
    -- Priority score: diagnostic severity + cargo value weight
    (CASE 
        WHEN ft.EngineDiagnosticCode LIKE 'P0%' THEN 100 -- Critical
        WHEN ft.EngineDiagnosticCode LIKE 'P1%' THEN 50  -- Warning
        ELSE 0 
    END +
    CASE 
        WHEN ft.TirePressurePSI < 30 THEN 75
        WHEN ft.TirePressurePSI < 35 THEN 25
        ELSE 0
    END +
    (dpg.ValuePerUnit * ft.LoadWeightKg / 1000)) AS PriorityScore
FROM FactFleetTrips ft
INNER JOIN DimProductGravity dpg ON ft.CargoSKU = dpg.SKU
WHERE ft.EngineDiagnosticCode IS NOT NULL
    OR ft.TirePressurePSI < 35;

-- Get top 10 maintenance priorities
SELECT TOP 10 *
FROM vwFleetMaintenancePriority
ORDER BY PriorityScore DESC;
```

## Temporal Elasticity Simulation

```sql
-- Simulate impact of warehouse capacity change
CREATE PROCEDURE uspSimulateCapacityChange
    @NewCapacityPercent DECIMAL(5,2),
    @SimulationDays INT = 30
AS
BEGIN
    WITH HistoricalBaseline AS (
        SELECT 
            AVG(DwellTimeMinutes) AS AvgDwell,
            AVG(PickRatePerHour) AS AvgPickRate,
            COUNT(*) AS OperationCount
        FROM FactWarehouseOperations fwo
        INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
        WHERE dt.CalendarDate >= DATEADD(DAY, -@SimulationDays, GETDATE())
    ),
    ProjectedMetrics AS (
        SELECT 
            AvgDwell * (1 + (@NewCapacityPercent - 80) / 100 * 0.3) AS ProjectedDwell,
            AvgPickRate * (1 - (@NewCapacityPercent - 80) / 100 * 0.2) AS ProjectedPickRate,
            OperationCount
        FROM HistoricalBaseline
    )
    SELECT 
        @NewCapacityPercent AS NewCapacityPercent,
        h.AvgDwell AS CurrentAvgDwell,
        p.ProjectedDwell,
        h.AvgPickRate AS CurrentAvgPickRate,
        p.ProjectedPickRate,
        (p.ProjectedDwell - h.AvgDwell) / h.AvgDwell * 100 AS DwellChangePercent,
        (p.ProjectedPickRate - h.AvgPickRate) / h.AvgPickRate * 100 AS PickRateChangePercent
    FROM HistoricalBaseline h
    CROSS JOIN ProjectedMetrics p;
END;

-- Run simulation
EXEC uspSimulateCapacityChange @NewCapacityPercent = 95, @SimulationDays = 30;
```

## Common Troubleshooting

### Slow Cross-Fact Queries

```sql
-- Add covering indexes on fact tables
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_TimeProduct
ON FactWarehouseOperations (TimeKey, ProductGravityKey)
INCLUDE (DwellTimeMinutes, PickRatePerHour);

CREATE NONCLUSTERED INDEX IX_FactFleetTrips_TimeRoute
ON FactFleetTrips (DepartureTimeKey, RouteID)
INCLUDE (IdleTimeMinutes, FuelConsumedLiters);

-- Partition large fact tables by month
ALTER PARTITION SCHEME ps_MonthlyPartition
NEXT USED [PRIMARY];

ALTER TABLE FactWarehouseOperations
SWITCH PARTITION $PARTITION.pf_Monthly('2026-07-01')
TO FactWarehouseOperations_Archive PARTITION $PARTITION.pf_Monthly('2026-07-01');
```

### Power BI Refresh Failures

```powershell
# PowerShell script to diagnose refresh issues
$workspaceId = "${POWERBI_WORKSPACE_ID}"
$datasetId = "${POWERBI_DATASET_ID}"

# Get refresh history
$refreshes = Invoke-PowerBIRestMethod `
    -Url "groups/$workspaceId/datasets/$datasetId/refreshes" `
    -Method Get | ConvertFrom-Json

$refreshes.value | Where-Object { $_.status -eq "Failed" } | ForEach-Object {
    Write-Host "Failed refresh at $($_.endTime): $($_.serviceExceptionJson)"
}
```

### Missing Dimension Lookups

```sql
-- Audit orphaned fact records
SELECT 
    'FactWarehouseOperations' AS TableName,
    COUNT(*) AS OrphanedRecords
FROM FactWarehouseOperations fwo
LEFT JOIN DimProductGravity dpg ON fwo.ProductGravityKey = dpg.ProductGravityKey
WHERE dpg.ProductGravityKey IS NULL

UNION ALL

SELECT 
    'FactFleetTrips',
    COUNT(*)
FROM FactFleetTrips ft
LEFT JOIN DimGeography dg ON ft.OriginGeographyKey = dg.GeographyKey
WHERE dg.GeographyKey IS NULL;

-- Auto-insert missing dimension members (Type 1 SCD)
INSERT INTO DimProductGravity (SKU, ProductName, GravityScore, GravityZoneName)
SELECT DISTINCT 
    src.SKU,
    src.ProductName,
    50, -- Default gravity
    'Medium Gravity'
FROM ExternalWarehouseData src
WHERE NOT EXISTS (
    SELECT 1 FROM DimProductGravity WHERE SKU = src.SKU
);
```

### Performance Monitoring

```sql
-- Identify expensive queries
SELECT 
    qs.execution_count,
    qs.total_worker_time / qs.execution_count AS avg_cpu_time,
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed_time,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS statement_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE st.text LIKE '%FactWarehouseOperations%'
    OR st.text LIKE '%FactFleetTrips%'
ORDER BY avg_elapsed_time DESC;
```

## Best Practices

1. **Always use parameterized queries** for dynamic filtering to prevent SQL injection
2. **Refresh DimProductGravity weekly** to adapt to changing SKU velocity patterns
3. **Partition fact tables by month** once they exceed 10 million rows
4. **Use columnstore indexes** on large fact tables for analytical queries
5. **Implement slowly changing dimensions (SCD Type 2)** for supplier and geography to track historical changes
6. **Set up database maintenance plans** for index rebuilds and statistics updates
7. **Test alerts in non-production environments** before enabling in production
8. **Document all custom DAX measures** with business context for Power BI consumers
9. **Use environment variables** for all connection strings and API endpoints
10. **Schedule gravity recalculation** during off-peak hours to minimize impact

## Advanced Pattern: Collaborative Anomaly Tagging

```sql
-- User annotation table for anomaly detection
CREATE TABLE AnomalyAnnotations (
    AnnotationID INT PRIMARY KEY IDENTITY(1,1),
    FactTableName NVARCHAR(100),
    FactRecordID INT,
    AnomalyType NVARCHAR(50), -- 'Supplier Strike', 'Weather Delay', etc.
    UserEmail NVARCHAR(255),
    AnnotationText NVARCHAR(MAX),
    CreatedDate DATETIME2 DEFAULT GETUTCDATE(),
    Verified BIT DEFAULT 0
);

-- Tag an anomaly
INSERT INTO AnomalyAnnotations (FactTableName, FactRecordID, AnomalyType, UserEmail, AnnotationText)
VALUES ('FactWarehouseOperations', 12345, 'Supplier Strike', 'analyst@company.com', 
        'Major supplier strike caused 300% increase in dwell time for electronics category.');

-- Query annotated anomalies
SELECT 
    aa.AnomalyType,
    aa.AnnotationText,
    fwo.DwellTimeMinutes,
    fwo.SKU,
    dt.CalendarDate
FROM AnomalyAnnotations aa
INNER JOIN FactWarehouseOperations fwo ON aa.FactRecordID = fwo.OperationID
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
WHERE aa.FactTableName = 'FactWarehouseOperations'
    AND aa.Verified = 1
ORDER BY dt.CalendarDate DESC;
```

This skill provides comprehensive guidance for deploying and using LogiFleet Pulse to build advanced supply chain analytics solutions with MS SQL Server and Power BI.
