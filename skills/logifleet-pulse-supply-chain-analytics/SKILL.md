---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing solution for logistics intelligence with multi-fact star schema and fleet optimization
triggers:
  - set up logifleet pulse logistics dashboard
  - deploy supply chain analytics warehouse
  - configure power bi fleet optimization
  - implement warehouse gravity zone modeling
  - create multi-fact logistics star schema
  - build cross-modal supply chain reports
  - optimize fleet telemetry analytics
  - integrate warehouse and fleet kpis
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform combining MS SQL Server data warehousing with Power BI visualization. It provides:

- **Multi-fact star schema** linking warehouse operations, fleet trips, and cross-dock activities
- **Time-phased dimensions** with 15-minute granularity
- **Warehouse Gravity Zones** for spatial optimization based on pick frequency and item value
- **Fleet triage engine** prioritizing maintenance by revenue impact
- **Cross-fact KPI harmonization** correlating inventory turnover with fuel consumption
- **Predictive bottleneck detection** using historical pattern analysis

Primary use cases: 3PL operators, retail chains, food distributors, any organization managing warehouse + fleet operations.

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, TMS, telematics APIs

### Step 1: Deploy SQL Schema

Clone the repository and navigate to the SQL scripts directory:

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse/sql
```

Execute the schema deployment script in SSMS:

```sql
-- Run this in SSMS connected to your target database
:r deploy_schema.sql

-- Verify deployment
SELECT 
    TABLE_SCHEMA,
    TABLE_NAME,
    TABLE_TYPE
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA IN ('fact', 'dim', 'bridge')
ORDER BY TABLE_SCHEMA, TABLE_NAME;
```

### Step 2: Configure Data Sources

Update the configuration file with your environment details:

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "integrated"
  },
  "data_sources": {
    "wms_connection": "${WMS_CONNECTION_STRING}",
    "telemetry_api": "${TELEMETRY_API_ENDPOINT}",
    "weather_api_key": "${WEATHER_API_KEY}"
  },
  "refresh_intervals": {
    "warehouse_ops": "15min",
    "fleet_telemetry": "5min",
    "supplier_data": "1hour"
  }
}
```

### Step 3: Import Power BI Template

Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop and configure the connection:

```powerquery
// Connection parameters
let
    Source = Sql.Database(
        "${SQL_SERVER_HOST}", 
        "LogiFleetPulse",
        [Query="EXEC sp_GetWarehouseFleetSummary @StartDate='" & Text.From(Date.AddDays(DateTime.LocalNow(), -30)) & "'"]
    )
in
    Source
```

## Core Database Schema

### Fact Tables

**FactWarehouseOperations** - Tracks all warehouse activities:

```sql
CREATE TABLE fact.FactWarehouseOperations (
    OperationID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    DwellTimeMinutes INT,
    PickRate DECIMAL(10,2), -- items per hour
    PackingTimeSeconds INT,
    GravityZoneID INT,
    CONSTRAINT FK_WO_Time FOREIGN KEY (TimeKey) REFERENCES dim.DimTime(TimeKey),
    CONSTRAINT FK_WO_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES dim.DimWarehouse(WarehouseKey),
    CONSTRAINT FK_WO_Product FOREIGN KEY (ProductKey) REFERENCES dim.DimProduct(ProductKey)
);

-- Columnstore index for analytics queries
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouseOperations
ON fact.FactWarehouseOperations (
    TimeKey, WarehouseKey, ProductKey, OperationType, DwellTimeMinutes
);
```

**FactFleetTrips** - Fleet telemetry and route data:

```sql
CREATE TABLE fact.FactFleetTrips (
    TripID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightKg DECIMAL(10,2),
    AverageSpeedKmh DECIMAL(5,2),
    MaintenanceScore DECIMAL(5,2), -- 0-100, higher = better
    DelayMinutes INT,
    DelayReason VARCHAR(100),
    CONSTRAINT FK_FT_Time FOREIGN KEY (TimeKey) REFERENCES dim.DimTime(TimeKey),
    CONSTRAINT FK_FT_Vehicle FOREIGN KEY (VehicleKey) REFERENCES dim.DimVehicle(VehicleKey)
);

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleetTrips
ON fact.FactFleetTrips (
    TimeKey, VehicleKey, RouteKey, FuelConsumedLiters, IdleTimeMinutes
);
```

### Dimension Tables

**DimTime** - Time intelligence with 15-minute buckets:

```sql
CREATE TABLE dim.DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    Hour TINYINT NOT NULL,
    QuarterHour TINYINT NOT NULL, -- 0, 15, 30, 45
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    IsWeekend BIT NOT NULL,
    FiscalPeriod VARCHAR(10),
    IsHoliday BIT DEFAULT 0,
    ShiftName VARCHAR(20) -- 'MORNING', 'AFTERNOON', 'NIGHT'
);

-- Populate time dimension
EXEC sp_PopulateTimeDimension 
    @StartDate = '2024-01-01', 
    @EndDate = '2027-12-31';
```

**DimProductGravity** - Product classification with gravity scoring:

```sql
CREATE TABLE dim.DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200),
    Category VARCHAR(50),
    Subcategory VARCHAR(50),
    UnitValue DECIMAL(10,2),
    FragilityScore DECIMAL(3,2), -- 0-1, higher = more fragile
    VelocityClass VARCHAR(20), -- 'FAST', 'MEDIUM', 'SLOW'
    GravityScore DECIMAL(5,2), -- Calculated: (velocity * value) / fragility
    OptimalZoneID INT,
    CONSTRAINT CHK_GravityScore CHECK (GravityScore >= 0)
);

-- Function to calculate gravity score
CREATE FUNCTION dbo.fn_CalculateGravityScore(
    @VelocityClass VARCHAR(20),
    @UnitValue DECIMAL(10,2),
    @FragilityScore DECIMAL(3,2)
)
RETURNS DECIMAL(5,2)
AS
BEGIN
    DECLARE @VelocityMultiplier DECIMAL(3,2);
    
    SET @VelocityMultiplier = CASE @VelocityClass
        WHEN 'FAST' THEN 3.0
        WHEN 'MEDIUM' THEN 1.5
        WHEN 'SLOW' THEN 0.5
        ELSE 1.0
    END;
    
    RETURN (@VelocityMultiplier * @UnitValue) / NULLIF(@FragilityScore, 0);
END;
```

## Key Stored Procedures

### Cross-Fact KPI Query

```sql
CREATE PROCEDURE sp_GetWarehouseFleetSummary
    @StartDate DATE,
    @EndDate DATE,
    @WarehouseKey INT = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH WarehouseSummary AS (
        SELECT 
            w.WarehouseKey,
            w.WarehouseName,
            COUNT(DISTINCT wo.OperationID) AS TotalOperations,
            AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
            SUM(CASE WHEN wo.OperationType = 'PICK' THEN 1 ELSE 0 END) AS TotalPicks,
            AVG(wo.PickRate) AS AvgPickRate
        FROM fact.FactWarehouseOperations wo
        INNER JOIN dim.DimTime t ON wo.TimeKey = t.TimeKey
        INNER JOIN dim.DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
        WHERE t.Date BETWEEN @StartDate AND @EndDate
            AND (@WarehouseKey IS NULL OR w.WarehouseKey = @WarehouseKey)
        GROUP BY w.WarehouseKey, w.WarehouseName
    ),
    FleetSummary AS (
        SELECT 
            g.WarehouseKey, -- Assuming origin is warehouse
            COUNT(DISTINCT ft.TripID) AS TotalTrips,
            SUM(ft.FuelConsumedLiters) AS TotalFuelConsumed,
            AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
            SUM(ft.DelayMinutes) AS TotalDelayMinutes
        FROM fact.FactFleetTrips ft
        INNER JOIN dim.DimTime t ON ft.TimeKey = t.TimeKey
        INNER JOIN dim.DimGeography g ON ft.OriginGeographyKey = g.GeographyKey
        WHERE t.Date BETWEEN @StartDate AND @EndDate
            AND (@WarehouseKey IS NULL OR g.WarehouseKey = @WarehouseKey)
        GROUP BY g.WarehouseKey
    )
    SELECT 
        ws.WarehouseKey,
        ws.WarehouseName,
        ws.TotalOperations,
        ws.AvgDwellTime,
        ws.TotalPicks,
        ws.AvgPickRate,
        ISNULL(fs.TotalTrips, 0) AS TotalTrips,
        ISNULL(fs.TotalFuelConsumed, 0) AS TotalFuelConsumed,
        ISNULL(fs.AvgIdleTime, 0) AS AvgIdleTime,
        ISNULL(fs.TotalDelayMinutes, 0) AS TotalDelayMinutes,
        -- Cross-fact KPI: Fuel efficiency per pick
        CASE 
            WHEN ws.TotalPicks > 0 THEN fs.TotalFuelConsumed / ws.TotalPicks
            ELSE 0 
        END AS FuelPerPick
    FROM WarehouseSummary ws
    LEFT JOIN FleetSummary fs ON ws.WarehouseKey = fs.WarehouseKey
    ORDER BY ws.WarehouseKey;
END;
GO
```

### Predictive Bottleneck Detection

```sql
CREATE PROCEDURE sp_PredictBottlenecks
    @ForecastDays INT = 7,
    @ThresholdPercentile DECIMAL(5,2) = 0.90
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate historical patterns
    WITH HistoricalPatterns AS (
        SELECT 
            t.DayOfWeek,
            t.Hour,
            p.GravityScore,
            AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
            STDEV(wo.DwellTimeMinutes) AS StdDevDwellTime,
            PERCENTILE_CONT(@ThresholdPercentile) WITHIN GROUP (ORDER BY wo.DwellTimeMinutes) 
                OVER (PARTITION BY t.DayOfWeek, t.Hour) AS P90DwellTime
        FROM fact.FactWarehouseOperations wo
        INNER JOIN dim.DimTime t ON wo.TimeKey = t.TimeKey
        INNER JOIN dim.DimProductGravity p ON wo.ProductKey = p.ProductKey
        WHERE t.Date >= DATEADD(DAY, -90, GETDATE())
        GROUP BY t.DayOfWeek, t.Hour, p.GravityScore
    ),
    ForecastPeriods AS (
        SELECT 
            DATEADD(DAY, n, CAST(GETDATE() AS DATE)) AS ForecastDate,
            DATEPART(WEEKDAY, DATEADD(DAY, n, GETDATE())) AS DayOfWeek,
            h.Hour
        FROM (SELECT TOP (@ForecastDays) ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1 AS n FROM sys.objects) d
        CROSS JOIN (SELECT DISTINCT Hour FROM dim.DimTime) h
    )
    SELECT 
        fp.ForecastDate,
        fp.Hour,
        hp.AvgDwellTime,
        hp.P90DwellTime,
        CASE 
            WHEN hp.P90DwellTime > hp.AvgDwellTime * 1.5 THEN 'HIGH'
            WHEN hp.P90DwellTime > hp.AvgDwellTime * 1.2 THEN 'MEDIUM'
            ELSE 'LOW'
        END AS BottleneckRisk,
        hp.GravityScore AS AffectedGravityZone
    FROM ForecastPeriods fp
    INNER JOIN HistoricalPatterns hp ON fp.DayOfWeek = hp.DayOfWeek AND fp.Hour = hp.Hour
    WHERE hp.P90DwellTime > hp.AvgDwellTime * 1.2
    ORDER BY fp.ForecastDate, fp.Hour, hp.P90DwellTime DESC;
END;
GO
```

## Power BI DAX Measures

### Fleet Idle Cost Calculation

```dax
Fleet Idle Cost = 
VAR IdleCostPerHour = 45 -- USD per hour (fuel + labor)
VAR TotalIdleMinutes = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN
    (TotalIdleMinutes / 60) * IdleCostPerHour
```

### Warehouse Efficiency Score

```dax
Warehouse Efficiency Score = 
VAR TargetPickRate = 120 -- items per hour
VAR ActualPickRate = AVERAGE(FactWarehouseOperations[PickRate])
VAR TargetDwellTime = 48 -- hours
VAR ActualDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) / 60
VAR PickEfficiency = DIVIDE(ActualPickRate, TargetPickRate, 0)
VAR DwellEfficiency = DIVIDE(TargetDwellTime, ActualDwellTime, 0)
RETURN
    (PickEfficiency * 0.6) + (DwellEfficiency * 0.4)
```

### Cross-Fact Correlation Measure

```dax
Dwell vs Idle Correlation = 
VAR SummaryTable = 
    SUMMARIZE(
        FactWarehouseOperations,
        DimTime[Date],
        DimWarehouse[WarehouseKey],
        "AvgDwell", AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
    )
VAR FleetTable = 
    SUMMARIZE(
        FactFleetTrips,
        DimTime[Date],
        DimGeography[WarehouseKey],
        "AvgIdle", AVERAGE(FactFleetTrips[IdleTimeMinutes])
    )
VAR JoinedTable = 
    NATURALLEFTOUTERJOIN(SummaryTable, FleetTable)
RETURN
    CORRELATIONX(JoinedTable, [AvgDwell], [AvgIdle])
```

## Data Ingestion Patterns

### Incremental Load from WMS

```sql
CREATE PROCEDURE sp_IncrementalLoadWarehouseOps
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRANSACTION;
    
    -- Load new operations from external WMS table
    INSERT INTO fact.FactWarehouseOperations (
        TimeKey,
        WarehouseKey,
        ProductKey,
        OperationType,
        DwellTimeMinutes,
        PickRate,
        PackingTimeSeconds,
        GravityZoneID
    )
    SELECT 
        t.TimeKey,
        w.WarehouseKey,
        p.ProductKey,
        ext.operation_type,
        DATEDIFF(MINUTE, ext.start_time, ext.end_time) AS DwellTimeMinutes,
        ext.pick_rate,
        ext.packing_seconds,
        p.OptimalZoneID
    FROM OPENQUERY(WMS_LINKED_SERVER, 
        'SELECT * FROM warehouse_operations WHERE last_updated > ?', @LastLoadTimestamp) ext
    INNER JOIN dim.DimTime t ON CAST(ext.operation_time AS DATETIME2) = t.FullDateTime
    INNER JOIN dim.DimWarehouse w ON ext.warehouse_code = w.WarehouseCode
    INNER JOIN dim.DimProductGravity p ON ext.sku = p.SKU
    WHERE NOT EXISTS (
        SELECT 1 FROM fact.FactWarehouseOperations existing
        WHERE existing.OperationID = ext.external_id
    );
    
    COMMIT TRANSACTION;
    
    -- Update last load timestamp
    UPDATE admin.ETLControl
    SET LastLoadTimestamp = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
GO
```

### Real-Time Telemetry via REST API

```sql
-- External table setup for streaming data
CREATE EXTERNAL DATA SOURCE TelemetryAPI
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = '${TELEMETRY_API_ENDPOINT}',
    CREDENTIAL = TelemetryCredential
);

-- Scheduled job to poll and insert
CREATE PROCEDURE sp_PollFleetTelemetry
AS
BEGIN
    DECLARE @JsonResponse NVARCHAR(MAX);
    
    -- Call REST API (requires CLR or external script)
    EXEC sp_InvokeRESTAPI 
        @Endpoint = '${TELEMETRY_API_ENDPOINT}/vehicles/active',
        @Method = 'GET',
        @Headers = 'Authorization: Bearer ${TELEMETRY_API_TOKEN}',
        @Response = @JsonResponse OUTPUT;
    
    -- Parse and insert JSON
    INSERT INTO fact.FactFleetTrips (
        TimeKey,
        VehicleKey,
        RouteKey,
        OriginGeographyKey,
        DestinationGeographyKey,
        FuelConsumedLiters,
        IdleTimeMinutes,
        LoadWeightKg,
        AverageSpeedKmh,
        MaintenanceScore
    )
    SELECT 
        t.TimeKey,
        v.VehicleKey,
        r.RouteKey,
        og.GeographyKey,
        dg.GeographyKey,
        JSON_VALUE(trip, '$.fuel_consumed'),
        JSON_VALUE(trip, '$.idle_minutes'),
        JSON_VALUE(trip, '$.load_weight'),
        JSON_VALUE(trip, '$.avg_speed'),
        JSON_VALUE(trip, '$.maintenance_score')
    FROM OPENJSON(@JsonResponse, '$.trips') trip
    CROSS APPLY (SELECT GETDATE() AS CurrentTime) ct
    INNER JOIN dim.DimTime t ON DATEPART(MINUTE, ct.CurrentTime) / 15 * 15 = t.QuarterHour
    INNER JOIN dim.DimVehicle v ON JSON_VALUE(trip, '$.vehicle_id') = v.VehicleExternalID
    INNER JOIN dim.DimRoute r ON JSON_VALUE(trip, '$.route_id') = r.RouteExternalID
    INNER JOIN dim.DimGeography og ON JSON_VALUE(trip, '$.origin') = og.LocationCode
    INNER JOIN dim.DimGeography dg ON JSON_VALUE(trip, '$.destination') = dg.LocationCode;
END;
GO
```

## Alert Configuration

### SQL Server Agent Job for Proactive Alerts

```sql
CREATE PROCEDURE sp_CheckCriticalAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Alert 1: High idle time threshold
    IF EXISTS (
        SELECT 1 
        FROM fact.FactFleetTrips ft
        INNER JOIN dim.DimTime t ON ft.TimeKey = t.TimeKey
        WHERE t.Date = CAST(GETDATE() AS DATE)
        GROUP BY ft.VehicleKey
        HAVING AVG(ft.IdleTimeMinutes) > 30
    )
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = '${FLEET_MANAGER_EMAIL}',
            @subject = 'ALERT: Fleet Idle Time Exceeded',
            @body = 'One or more vehicles exceeded 30 minutes average idle time today.',
            @importance = 'High';
    END;
    
    -- Alert 2: Warehouse dwell time anomaly
    DECLARE @P90Dwell DECIMAL(10,2);
    SELECT @P90Dwell = PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY DwellTimeMinutes)
    FROM fact.FactWarehouseOperations
    WHERE TimeKey IN (SELECT TimeKey FROM dim.DimTime WHERE Date >= DATEADD(DAY, -30, GETDATE()));
    
    IF EXISTS (
        SELECT 1
        FROM fact.FactWarehouseOperations wo
        INNER JOIN dim.DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.Date = CAST(GETDATE() AS DATE)
            AND wo.DwellTimeMinutes > @P90Dwell * 1.5
    )
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = '${WAREHOUSE_MANAGER_EMAIL}',
            @subject = 'ALERT: Abnormal Dwell Time Detected',
            @body = 'Dwell time exceeded 150% of 90th percentile baseline.',
            @importance = 'High';
    END;
END;
GO

-- Schedule job to run every 15 minutes
EXEC msdb.dbo.sp_add_job @job_name = 'LogiFleet_CriticalAlerts';
EXEC msdb.dbo.sp_add_jobstep 
    @job_name = 'LogiFleet_CriticalAlerts',
    @step_name = 'Check Alerts',
    @subsystem = 'TSQL',
    @command = 'EXEC sp_CheckCriticalAlerts';
EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = 'Every15Minutes',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 15;
EXEC msdb.dbo.sp_attach_schedule 
    @job_name = 'LogiFleet_CriticalAlerts',
    @schedule_name = 'Every15Minutes';
```

## Power BI Deployment

### Publish to Power BI Service

```powershell
# Install Power BI module
Install-Module -Name MicrosoftPowerBIMgmt

# Authenticate
Connect-PowerBIServiceAccount

# Publish report
Publish-PowerBIFile `
    -Path ".\LogiFleet_Pulse_Master.pbix" `
    -WorkspaceId "${POWERBI_WORKSPACE_ID}" `
    -ConflictAction CreateOrOverwrite

# Configure scheduled refresh
Set-PowerBIDatasetRefresh `
    -DatasetId "${DATASET_ID}" `
    -RefreshSchedule @{
        days = @("Monday", "Tuesday", "Wednesday", "Thursday", "Friday")
        times = @("06:00", "12:00", "18:00")
        enabled = $true
    }
```

## Common Troubleshooting

### Query Performance Issues

**Symptom**: Dashboards load slowly or timeout

**Solution**: Check columnstore index fragmentation

```sql
-- Analyze columnstore health
SELECT 
    OBJECT_NAME(i.object_id) AS TableName,
    i.name AS IndexName,
    100.0 * (ISNULL(deleted_rows, 0)) / NULLIF(total_rows, 0) AS FragmentationPercent
FROM sys.dm_db_column_store_row_group_physical_stats rg
INNER JOIN sys.indexes i ON rg.object_id = i.object_id AND rg.index_id = i.index_id
WHERE OBJECT_SCHEMA_NAME(i.object_id) = 'fact'
ORDER BY FragmentationPercent DESC;

-- Rebuild if fragmentation > 20%
ALTER INDEX NCCI_FactWarehouseOperations ON fact.FactWarehouseOperations REBUILD;
ALTER INDEX NCCI_FactFleetTrips ON fact.FactFleetTrips REBUILD;
```

### Data Refresh Failures

**Symptom**: Power BI shows "Unable to refresh dataset"

**Solution**: Verify gateway and credentials

```powershell
# Test gateway connection
Test-PowerBIGatewayDataSource `
    -GatewayId "${GATEWAY_ID}" `
    -DataSourceId "${DATASOURCE_ID}"

# Update credentials
Update-PowerBIDatasetDatasource `
    -DatasetId "${DATASET_ID}" `
    -DatasourceId "${DATASOURCE_ID}" `
    -UpdateDetails @{
        credentialType = "Windows"
        credentials = "${SQL_USERNAME}:${SQL_PASSWORD}"
    }
```

### Missing Time Dimension Records

**Symptom**: Fact table inserts fail with foreign key violations

**Solution**: Extend time dimension

```sql
-- Check time dimension coverage
SELECT MIN(Date) AS MinDate, MAX(Date) AS MaxDate
FROM dim.DimTime;

-- Extend if needed
EXEC sp_PopulateTimeDimension 
    @StartDate = '2027-01-01', 
    @EndDate = '2028-12-31';
```

## Best Practices

### 1. Incremental vs. Full Refresh

Use incremental refresh for fact tables with partitioning:

```sql
-- Create partition function for monthly partitions
CREATE PARTITION FUNCTION PF_Monthly (DATE)
AS RANGE RIGHT FOR VALUES (
    '2024-02-01', '2024-03-01', '2024-04-01' -- ... extend as needed
);

-- Apply to fact table
CREATE PARTITION SCHEME PS_Monthly
AS PARTITION PF_Monthly
ALL TO ([PRIMARY]);

-- Rebuild table on partition scheme
CREATE TABLE fact.FactWarehouseOperations_Partitioned (
    OperationID BIGINT IDENTITY(1,1),
    OperationDate DATE NOT NULL,
    -- ... other columns
    CONSTRAINT PK_WO_Partitioned PRIMARY KEY (OperationID, OperationDate)
) ON PS_Monthly(OperationDate);
```

### 2. Row-Level Security

Implement RLS for multi-tenant scenarios:

```sql
CREATE FUNCTION security.fn_WarehouseSecurityPredicate(@WarehouseKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS result
WHERE @WarehouseKey IN (
    SELECT WarehouseKey 
    FROM security.UserWarehouseAccess
    WHERE Username = USER_NAME()
);
GO

CREATE SECURITY POLICY security.WarehouseSecurityPolicy
ADD FILTER PREDICATE security.fn_WarehouseSecurityPredicate(WarehouseKey)
ON fact.FactWarehouseOperations,
ADD FILTER PREDICATE security.fn_WarehouseSecurityPredicate(WarehouseKey)
ON dim.DimWarehouse
WITH (STATE = ON);
```

### 3. Change Data Capture

Enable CDC for audit trails:

```sql
-- Enable CDC on database
EXEC sys.sp_cdc_enable_db;

-- Enable CDC on fact table
EXEC sys.sp_cdc_enable_table
    @source_schema = 'fact',
    @source_name = 'FactWarehouseOperations',
    @role_name = NULL,
    @supports_net_changes = 1;

-- Query changes
SELECT *
FROM cdc.fn_cdc_get_net_changes_fact_FactWarehouseOperations(
    sys.fn_cdc_get_min_lsn('fact_FactWarehouseOperations'),
    sys.fn_cdc_get_max_lsn(),
    'all'
);
```

## Environment Variables Reference

```bash
# SQL Server
SQL_SERVER_HOST=your-server.database.windows.net
SQL_DATABASE=LogiFleetPulse
SQL_USERNAME=your-username
SQL_PASSWORD=your-password

# External Data Sources
WMS_CONNECTION_STRING=Server=wms-server;Database=WMS;Trusted_Connection=True;
TELEMETRY_API_ENDPOINT=https://api.telemetry-provider.com/v2
TELEMETRY_API_TOKEN=your-api-token
WEATHER_API_KEY=your-weather-api-key

# Power BI
POWERBI_WORKSPACE_ID=your-workspace-guid
POWERBI_DATASET_ID=your-dataset-guid
GATEWAY_ID=your-gateway-guid
DATASOURCE_ID=your-datasource-guid

# Alerts
FLEET_MANAGER_EMAIL=fleet@company.com
WAREHOUSE_MANAGER_EMAIL=warehouse@company.com
```

## Additional Resources

- Original repository: https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
- SQL Server performance tuning: https://docs.microsoft.com/sql/relational-databases/performance/
- Power BI best practices: https://docs.microsoft.com/power-bi/guidance/
- DAX formula reference: https://dax.guide/
