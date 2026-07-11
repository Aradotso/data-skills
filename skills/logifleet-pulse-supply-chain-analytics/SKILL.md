---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI data warehousing engine for logistics, fleet management, and supply chain analytics with multi-fact star schema modeling
triggers:
  - "set up supply chain analytics dashboard"
  - "create logistics data warehouse"
  - "build fleet management Power BI report"
  - "implement warehouse operations tracking"
  - "configure multi-fact star schema for logistics"
  - "analyze cross-dock and fleet telemetry data"
  - "deploy LogiFleet Pulse analytics"
  - "optimize warehouse gravity zones"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and business intelligence solution for logistics and supply chain operations. It combines:

- **Multi-fact star schema** with time-phased dimensions for warehouse operations, fleet trips, and cross-dock activities
- **MS SQL Server backend** for transactional data integrity and complex queries
- **Power BI dashboards** for real-time visualization and KPI tracking
- **Predictive analytics** for bottleneck detection and fleet optimization
- **Cross-modal data integration** linking warehouse velocity, fleet telemetry, inventory aging, and external signals

The platform enables unified analytics across traditionally siloed logistics functions, providing one version of truth for decision-makers.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems: WMS, TMS, telemetry APIs

### Database Deployment

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance
-- Run the main schema creation script
:r schema/01_create_database.sql
:r schema/02_create_dimensions.sql
:r schema/03_create_facts.sql
:r schema/04_create_views.sql
:r schema/05_create_procedures.sql
```

3. **Configure data source connections:**
```json
// config_sample.json - copy to config.json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "user": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "fleet_telemetry": "${FLEET_API_ENDPOINT}",
    "weather_api": "${WEATHER_API_KEY}"
  }
}
```

### Power BI Configuration

1. **Open the template:**
```powershell
# Open Power BI template
Start-Process "LogiFleet_Pulse_Master.pbit"
```

2. **Connect to SQL Server:**
- When prompted, enter your SQL Server connection string
- Select DirectQuery or Import mode (DirectQuery recommended for real-time data)
- Apply row-level security settings for user roles

## Key Database Objects

### Core Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
SELECT TOP 10
    TimeKey,
    FullDateTime,
    HourOfDay,
    DayOfWeek,
    FiscalPeriod
FROM DimTime
ORDER BY TimeKey DESC;

-- DimGeography: Hierarchical location dimension
SELECT 
    GeoKey,
    Continent,
    Country,
    Region,
    WarehouseCode,
    RouteNode
FROM DimGeography
WHERE Country = 'USA';

-- DimProductGravity: Velocity-based product classification
SELECT 
    ProductKey,
    SKU,
    ProductName,
    GravityScore, -- Higher = faster moving
    StorageZoneRecommendation
FROM DimProductGravity
WHERE GravityScore > 0.7;
```

### Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse metrics
SELECT 
    w.OperationKey,
    w.TimeKey,
    w.ProductKey,
    w.WarehouseKey,
    w.OperationType, -- 'RECEIVE', 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    w.Quantity,
    w.DwellTimeMinutes,
    w.PickRateUnitsPerHour,
    w.AccuracyPercent
FROM FactWarehouseOperations w
WHERE w.TimeKey >= CAST(FORMAT(GETDATE()-7, 'yyyyMMddHHmm') AS INT);

-- FactFleetTrips: Vehicle movement and performance
SELECT 
    f.TripKey,
    f.TimeKey,
    f.VehicleKey,
    f.RouteKey,
    f.DistanceMiles,
    f.FuelGallons,
    f.IdleTimeMinutes,
    f.LoadingTimeMinutes,
    f.OnTimeDeliveryFlag
FROM FactFleetTrips f
WHERE f.OnTimeDeliveryFlag = 0; -- Late deliveries

-- FactCrossDock: Transfer operations
SELECT 
    c.CrossDockKey,
    c.TimeKey,
    c.InboundTripKey,
    c.OutboundTripKey,
    c.ProductKey,
    c.Quantity,
    c.TransferTimeMinutes
FROM FactCrossDock c
WHERE c.TransferTimeMinutes > 30; -- Bottleneck threshold
```

## Common Query Patterns

### Cross-Fact KPI Analysis

```sql
-- Link warehouse dwell time with fleet idle time
WITH WarehouseDwell AS (
    SELECT 
        w.ProductKey,
        w.TimeKey,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime
    FROM FactWarehouseOperations w
    WHERE w.OperationType = 'PUTAWAY'
    GROUP BY w.ProductKey, w.TimeKey
),
FleetIdle AS (
    SELECT 
        f.RouteKey,
        f.TimeKey,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime
    FROM FactFleetTrips f
    GROUP BY f.RouteKey, f.TimeKey
)
SELECT 
    t.FullDateTime,
    p.SKU,
    p.ProductName,
    wd.AvgDwellTime,
    fi.AvgIdleTime,
    -- Correlation score (custom business logic)
    CASE 
        WHEN wd.AvgDwellTime > 72 AND fi.AvgIdleTime > 15 THEN 'High Risk'
        WHEN wd.AvgDwellTime > 48 OR fi.AvgIdleTime > 10 THEN 'Medium Risk'
        ELSE 'Normal'
    END AS RiskLevel
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.TimeKey = fi.TimeKey
INNER JOIN DimTime t ON wd.TimeKey = t.TimeKey
INNER JOIN DimProductGravity p ON wd.ProductKey = p.ProductKey
WHERE t.FullDateTime >= DATEADD(day, -30, GETDATE())
ORDER BY wd.AvgDwellTime DESC, fi.AvgIdleTime DESC;
```

### Predictive Bottleneck Detection

```sql
-- Stored procedure for bottleneck scoring
CREATE PROCEDURE usp_CalculateBottleneckIndex
    @LookbackDays INT = 7,
    @ThresholdScore DECIMAL(5,2) = 0.75
AS
BEGIN
    SELECT 
        g.WarehouseCode,
        g.Region,
        COUNT(DISTINCT w.OperationKey) AS OperationCount,
        AVG(w.DwellTimeMinutes) AS AvgDwell,
        MAX(w.DwellTimeMinutes) AS MaxDwell,
        STDEV(w.DwellTimeMinutes) AS DwellVariance,
        -- Weighted bottleneck index (0-1 scale)
        (
            (AVG(w.DwellTimeMinutes) / NULLIF(MAX(w.DwellTimeMinutes), 0)) * 0.4 +
            (STDEV(w.DwellTimeMinutes) / NULLIF(AVG(w.DwellTimeMinutes), 0)) * 0.3 +
            (COUNT(CASE WHEN w.DwellTimeMinutes > 72 THEN 1 END) * 1.0 / COUNT(*)) * 0.3
        ) AS BottleneckIndex
    FROM FactWarehouseOperations w
    INNER JOIN DimGeography g ON w.WarehouseKey = g.GeoKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(day, -@LookbackDays, GETDATE())
    GROUP BY g.WarehouseCode, g.Region
    HAVING (
        (AVG(w.DwellTimeMinutes) / NULLIF(MAX(w.DwellTimeMinutes), 0)) * 0.4 +
        (STDEV(w.DwellTimeMinutes) / NULLIF(AVG(w.DwellTimeMinutes), 0)) * 0.3 +
        (COUNT(CASE WHEN w.DwellTimeMinutes > 72 THEN 1 END) * 1.0 / COUNT(*)) * 0.3
    ) >= @ThresholdScore
    ORDER BY BottleneckIndex DESC;
END;
GO

-- Execute bottleneck analysis
EXEC usp_CalculateBottleneckIndex 
    @LookbackDays = 14, 
    @ThresholdScore = 0.70;
```

### Warehouse Gravity Zone Optimization

```sql
-- Recommend storage zone reassignments
WITH ProductVelocity AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        p.GravityScore AS CurrentGravity,
        p.StorageZoneRecommendation AS CurrentZone,
        COUNT(w.OperationKey) AS PickFrequency,
        AVG(w.PickRateUnitsPerHour) AS AvgPickRate,
        -- Recalculate gravity based on recent activity
        (
            (COUNT(w.OperationKey) * 1.0 / NULLIF(DATEDIFF(day, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0)) * 0.5 +
            (AVG(w.PickRateUnitsPerHour) / 100.0) * 0.3 +
            (SUM(w.Quantity) / 1000.0) * 0.2
        ) AS RecalculatedGravity
    FROM DimProductGravity p
    LEFT JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
    LEFT JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE w.OperationType = 'PICK'
      AND t.FullDateTime >= DATEADD(day, -30, GETDATE())
    GROUP BY p.ProductKey, p.SKU, p.ProductName, p.GravityScore, p.StorageZoneRecommendation
)
SELECT 
    SKU,
    ProductName,
    CurrentZone,
    CurrentGravity,
    RecalculatedGravity,
    CASE 
        WHEN RecalculatedGravity >= 0.8 THEN 'Zone A (High Velocity)'
        WHEN RecalculatedGravity >= 0.5 THEN 'Zone B (Medium Velocity)'
        ELSE 'Zone C (Low Velocity)'
    END AS RecommendedZone,
    CASE 
        WHEN ABS(RecalculatedGravity - CurrentGravity) > 0.2 THEN 'REASSIGN'
        ELSE 'MAINTAIN'
    END AS Action
FROM ProductVelocity
WHERE ABS(RecalculatedGravity - CurrentGravity) > 0.15
ORDER BY RecalculatedGravity DESC;
```

## Data Loading Procedures

### Incremental ETL Pattern

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last hour if not specified
    IF @LastLoadDateTime IS NULL
        SET @LastLoadDateTime = DATEADD(hour, -1, GETDATE());
    
    -- Stage new records
    INSERT INTO FactWarehouseOperations (
        TimeKey,
        ProductKey,
        WarehouseKey,
        OperationType,
        Quantity,
        DwellTimeMinutes,
        PickRateUnitsPerHour,
        AccuracyPercent,
        LoadedDateTime
    )
    SELECT 
        CAST(FORMAT(src.OperationDateTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        p.ProductKey,
        g.GeoKey AS WarehouseKey,
        src.OperationType,
        src.Quantity,
        src.DwellTimeMinutes,
        src.PickRateUnitsPerHour,
        src.AccuracyPercent,
        GETDATE() AS LoadedDateTime
    FROM StagingWarehouseOperations src
    INNER JOIN DimProductGravity p ON src.SKU = p.SKU
    INNER JOIN DimGeography g ON src.WarehouseCode = g.WarehouseCode
    WHERE src.OperationDateTime > @LastLoadDateTime
      AND NOT EXISTS (
          SELECT 1 
          FROM FactWarehouseOperations f 
          WHERE f.TimeKey = CAST(FORMAT(src.OperationDateTime, 'yyyyMMddHHmm') AS INT)
            AND f.ProductKey = p.ProductKey
            AND f.WarehouseKey = g.GeoKey
      );
    
    -- Log load metadata
    INSERT INTO ETLLoadLog (ProcedureName, RowsLoaded, LoadDateTime)
    VALUES ('usp_LoadWarehouseOperations', @@ROWCOUNT, GETDATE());
END;
GO
```

### Fleet Telemetry Integration

```sql
-- Real-time fleet data ingestion from external API
CREATE PROCEDURE usp_IngestFleetTelemetry
AS
BEGIN
    -- Assumes external table or linked server to telemetry API
    MERGE INTO FactFleetTrips AS target
    USING (
        SELECT 
            CAST(FORMAT(TelemetryTimestamp, 'yyyyMMddHHmm') AS INT) AS TimeKey,
            v.VehicleKey,
            r.RouteKey,
            DistanceMiles,
            FuelGallons,
            IdleTimeMinutes,
            LoadingTimeMinutes,
            CASE WHEN ActualArrival <= ScheduledArrival THEN 1 ELSE 0 END AS OnTimeDeliveryFlag
        FROM ExternalFleetTelemetry ext
        INNER JOIN DimVehicle v ON ext.VehicleID = v.VehicleID
        INNER JOIN DimRoute r ON ext.RouteID = r.RouteID
        WHERE ext.TelemetryTimestamp > DATEADD(minute, -30, GETDATE())
    ) AS source
    ON target.TimeKey = source.TimeKey 
       AND target.VehicleKey = source.VehicleKey
    WHEN MATCHED THEN
        UPDATE SET 
            target.DistanceMiles = source.DistanceMiles,
            target.FuelGallons = source.FuelGallons,
            target.IdleTimeMinutes = source.IdleTimeMinutes
    WHEN NOT MATCHED THEN
        INSERT (TimeKey, VehicleKey, RouteKey, DistanceMiles, FuelGallons, IdleTimeMinutes, LoadingTimeMinutes, OnTimeDeliveryFlag)
        VALUES (source.TimeKey, source.VehicleKey, source.RouteKey, source.DistanceMiles, source.FuelGallons, source.IdleTimeMinutes, source.LoadingTimeMinutes, source.OnTimeDeliveryFlag);
END;
GO
```

## Power BI DAX Measures

### Key Performance Indicators

```dax
// Warehouse Dwell Time (weighted average)
Avg Dwell Time = 
CALCULATE(
    AVERAGEX(
        FactWarehouseOperations,
        FactWarehouseOperations[DwellTimeMinutes]
    ),
    FactWarehouseOperations[OperationType] IN {"PUTAWAY", "PICK"}
)

// Fleet On-Time Delivery Rate
OTD Rate = 
DIVIDE(
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FactFleetTrips[OnTimeDeliveryFlag] = 1
    ),
    COUNTROWS(FactFleetTrips),
    0
) * 100

// Cross-Dock Efficiency Score
CrossDock Efficiency = 
VAR AvgTransferTime = AVERAGE(FactCrossDock[TransferTimeMinutes])
VAR TargetTransferTime = 30
RETURN
    SWITCH(
        TRUE(),
        AvgTransferTime <= TargetTransferTime * 0.8, "Excellent",
        AvgTransferTime <= TargetTransferTime, "Good",
        AvgTransferTime <= TargetTransferTime * 1.2, "Fair",
        "Poor"
    )

// Composite Logistics Health Index
Logistics Health Index = 
VAR DwellScore = 1 - ([Avg Dwell Time] / 168) // 168 hours = 1 week
VAR OTDScore = [OTD Rate] / 100
VAR IdleScore = 1 - (AVERAGE(FactFleetTrips[IdleTimeMinutes]) / 60)
RETURN
    (DwellScore * 0.4 + OTDScore * 0.4 + IdleScore * 0.2) * 100
```

### Time Intelligence Patterns

```dax
// Period-over-Period Comparison
Dwell Time vs Last Week = 
VAR CurrentDwell = [Avg Dwell Time]
VAR LastWeekDwell = 
    CALCULATE(
        [Avg Dwell Time],
        DATEADD(DimTime[FullDateTime], -7, DAY)
    )
RETURN
    CurrentDwell - LastWeekDwell

// Rolling 30-Day Average
Dwell Time 30D MA = 
CALCULATE(
    [Avg Dwell Time],
    DATESINPERIOD(
        DimTime[FullDateTime],
        LASTDATE(DimTime[FullDateTime]),
        -30,
        DAY
    )
)
```

## Configuration & Security

### Row-Level Security (RLS)

```sql
-- Create security roles based on geography
CREATE TABLE SecurityUserRole (
    UserEmail NVARCHAR(255),
    AllowedRegion NVARCHAR(100),
    RoleType NVARCHAR(50) -- 'User', 'Supervisor', 'Executive'
);

-- Sample RLS policy
CREATE FUNCTION fn_SecurityPredicate(@Region AS NVARCHAR(100))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN 
    SELECT 1 AS AccessAllowed
    FROM SecurityUserRole
    WHERE UserEmail = USER_NAME()
      AND (AllowedRegion = @Region OR RoleType = 'Executive');
GO

-- Apply policy to geography dimension
CREATE SECURITY POLICY RegionSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(Region)
ON dbo.DimGeography
WITH (STATE = ON);
```

### Automated Alerting

```sql
-- Configure threshold-based alerts
CREATE PROCEDURE usp_CheckKPIThresholds
AS
BEGIN
    DECLARE @AlertRecipients NVARCHAR(500) = '${ALERT_EMAIL_RECIPIENTS}';
    
    -- Fleet idle time alert
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips 
        WHERE IdleTimeMinutes > 0.15 * (IdleTimeMinutes + LoadingTimeMinutes + DistanceMiles * 2)
    )
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = @AlertRecipients,
            @subject = 'Alert: High Fleet Idle Time Detected',
            @body = 'One or more vehicles exceeded 15% idle time threshold.';
    END;
    
    -- Warehouse bottleneck alert
    DECLARE @BottleneckCount INT;
    SELECT @BottleneckCount = COUNT(*)
    FROM (
        EXEC usp_CalculateBottleneckIndex @LookbackDays = 1, @ThresholdScore = 0.80
    ) sub;
    
    IF @BottleneckCount > 0
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = @AlertRecipients,
            @subject = 'Alert: Warehouse Bottleneck Detected',
            @body_format = 'HTML',
            @body = 'Critical bottleneck detected. Review dashboard for details.';
    END;
END;
GO

-- Schedule hourly execution
EXEC msdb.dbo.sp_add_job
    @job_name = N'LogiFleet KPI Monitoring';
EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'LogiFleet KPI Monitoring',
    @step_name = N'Check Thresholds',
    @subsystem = N'TSQL',
    @command = N'EXEC usp_CheckKPIThresholds';
EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'Hourly',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 8, -- Hours
    @freq_subday_interval = 1;
```

## Troubleshooting

### Performance Optimization

```sql
-- Add recommended indexes
CREATE NONCLUSTERED INDEX IX_Warehouse_Time_Product
ON FactWarehouseOperations (TimeKey, ProductKey)
INCLUDE (DwellTimeMinutes, PickRateUnitsPerHour);

CREATE NONCLUSTERED INDEX IX_Fleet_Time_Vehicle
ON FactFleetTrips (TimeKey, VehicleKey)
INCLUDE (IdleTimeMinutes, OnTimeDeliveryFlag);

-- Enable query store for performance tracking
ALTER DATABASE LogiFleetPulse
SET QUERY_STORE = ON
(
    OPERATION_MODE = READ_WRITE,
    DATA_FLUSH_INTERVAL_SECONDS = 900,
    INTERVAL_LENGTH_MINUTES = 60
);

-- Partition large fact tables by time
CREATE PARTITION FUNCTION pf_TimeKey (INT)
AS RANGE RIGHT FOR VALUES 
(202601010000, 202602010000, 202603010000); -- Monthly partitions
```

### Common Data Quality Issues

```sql
-- Detect missing time dimension entries
SELECT DISTINCT w.TimeKey
FROM FactWarehouseOperations w
WHERE NOT EXISTS (SELECT 1 FROM DimTime t WHERE t.TimeKey = w.TimeKey)
ORDER BY w.TimeKey DESC;

-- Identify orphaned product references
SELECT DISTINCT w.ProductKey
FROM FactWarehouseOperations w
WHERE NOT EXISTS (SELECT 1 FROM DimProductGravity p WHERE p.ProductKey = w.ProductKey);

-- Validate fleet telemetry data gaps
WITH ExpectedTimeSlots AS (
    SELECT TimeKey
    FROM DimTime
    WHERE FullDateTime >= DATEADD(day, -1, GETDATE())
)
SELECT t.TimeKey, t.FullDateTime
FROM ExpectedTimeSlots t
LEFT JOIN FactFleetTrips f ON t.TimeKey = f.TimeKey
WHERE f.TripKey IS NULL;
```

### Power BI Refresh Issues

```powershell
# Refresh Power BI dataset via REST API
$workspaceId = "${POWERBI_WORKSPACE_ID}"
$datasetId = "${POWERBI_DATASET_ID}"
$accessToken = "${POWERBI_ACCESS_TOKEN}"

$headers = @{
    "Authorization" = "Bearer $accessToken"
}

Invoke-RestMethod `
    -Uri "https://api.powerbi.com/v1.0/myorg/groups/$workspaceId/datasets/$datasetId/refreshes" `
    -Method Post `
    -Headers $headers
```

## Best Practices

1. **Incremental Loading**: Always use watermark-based ETL to avoid full table scans
2. **Dimension Management**: Update gravity scores weekly based on rolling 30-day averages
3. **Fact Table Retention**: Archive fact data older than 24 months to separate historical database
4. **Index Maintenance**: Rebuild fragmented indexes weekly during off-peak hours
5. **Security**: Never store credentials in code—use Azure Key Vault or environment variables
6. **Dashboard Performance**: Use aggregation tables for frequently accessed KPIs
7. **Data Validation**: Run quality checks before each ETL load to prevent cascading errors

## Additional Resources

- **Schema Documentation**: See `docs/database_schema.md` for complete ERD
- **API Integration Guide**: Reference `docs/api_integration.md` for external data source setup
- **Power BI Template Guide**: See `docs/powerbi_customization.md` for dashboard modifications
- **Sample Queries**: Browse `queries/examples/` for additional use case patterns
