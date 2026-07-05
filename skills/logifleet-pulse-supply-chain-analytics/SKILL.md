---
name: logifleet-pulse-supply-chain-analytics
description: Advanced MS SQL Server & Power BI data warehousing solution for logistics intelligence, fleet optimization, and warehouse management analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "implement warehouse gravity zones analytics"
  - "create multi-fact star schema for logistics"
  - "deploy fleet optimization SQL database"
  - "build real-time supply chain KPI dashboard"
  - "integrate warehouse and fleet telemetry data"
  - "configure logistics data warehouse"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What It Does

LogiFleet Pulse is a multi-modal logistics intelligence engine that combines warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic data model. It provides:

- **Multi-fact star schema** for cross-domain logistics analytics
- **Power BI dashboards** with 15-minute refresh cycles
- **Warehouse Gravity Zones™** for spatial optimization
- **Adaptive Fleet Triage Engine** for proactive maintenance
- **Predictive bottleneck detection** using historical patterns
- **Time-phased dimension tables** for temporal analysis
- **Role-based access control** with row-level security

The platform uses MS SQL Server for the data warehouse and Power BI for visualization, enabling cross-functional KPI harmonization (e.g., linking inventory dwell time with fleet fuel consumption).

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telemetry APIs
- Network connectivity for external APIs (weather, traffic)

### Database Deployment

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**

```sql
-- Connect to your SQL Server instance
-- Execute the schema creation script
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the complete schema script
-- (typically in SQL/schema_complete.sql)
```

3. **Configure connection strings:**

Create `config.json` (based on `config_sample.json`):

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}",
    "trusted_connection": false
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "fleet_telemetry": "${TELEMETRY_API_ENDPOINT}",
    "weather_api_key": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

### Power BI Template Setup

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter connection credentials when prompted
3. Verify data refresh completes successfully
4. Publish to Power BI Service (optional)

## Core Data Model

### Fact Tables

The multi-fact architecture includes these core tables:

**FactWarehouseOperations** - Warehouse activity metrics

```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Putaway', 'Pick', 'Pack', 'Ship'
    QuantityHandled INT,
    DwellTimeMinutes INT,
    PickRateUnitsPerHour DECIMAL(10,2),
    OperatorKey INT,
    GravityZoneKey INT,
    CONSTRAINT FK_WO_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WO_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    CONSTRAINT FK_WO_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

CREATE NONCLUSTERED INDEX IX_FactWO_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWO_Product ON FactWarehouseOperations(ProductKey);
```

**FactFleetTrips** - Fleet and route performance

```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    OriginLocationKey INT,
    DestinationLocationKey INT,
    DistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightKG INT,
    OnTimeFlag BIT,
    DelayMinutes INT,
    CONSTRAINT FK_FT_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FT_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey)
);

CREATE NONCLUSTERED INDEX IX_FactFT_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFT_Route ON FactFleetTrips(RouteKey);
```

### Key Dimension Tables

**DimTime** - 15-minute granularity time dimension

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    Year INT,
    Quarter INT,
    Month INT,
    Day INT,
    Hour INT,
    Minute15Bucket INT, -- 0, 15, 30, 45
    DayOfWeek INT,
    IsWeekend BIT,
    FiscalPeriod VARCHAR(20),
    CONSTRAINT UQ_DimTime_DateTime UNIQUE(FullDateTime)
);

-- Populate time dimension
CREATE PROCEDURE PopulateDimTime
    @StartDate DATETIME,
    @EndDate DATETIME
AS
BEGIN
    DECLARE @CurrentTime DATETIME = @StartDate;
    
    WHILE @CurrentTime <= @EndDate
    BEGIN
        INSERT INTO DimTime (TimeKey, FullDateTime, Year, Quarter, Month, Day, Hour, Minute15Bucket, DayOfWeek, IsWeekend)
        VALUES (
            CONVERT(INT, FORMAT(@CurrentTime, 'yyyyMMddHHmm')),
            @CurrentTime,
            YEAR(@CurrentTime),
            DATEPART(QUARTER, @CurrentTime),
            MONTH(@CurrentTime),
            DAY(@CurrentTime),
            DATEPART(HOUR, @CurrentTime),
            (DATEPART(MINUTE, @CurrentTime) / 15) * 15,
            DATEPART(WEEKDAY, @CurrentTime),
            CASE WHEN DATEPART(WEEKDAY, @CurrentTime) IN (1, 7) THEN 1 ELSE 0 END
        );
        
        SET @CurrentTime = DATEADD(MINUTE, 15, @CurrentTime);
    END
END;
```

**DimProductGravity** - Warehouse Gravity Zones™

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(100),
    ProductName VARCHAR(255),
    VelocityScore DECIMAL(5,2), -- Movement frequency (0-100)
    ValueScore DECIMAL(5,2), -- Unit value tier (0-100)
    FragilityScore DECIMAL(5,2), -- Handling sensitivity (0-100)
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    RecommendedZone VARCHAR(50), -- 'High-Gravity', 'Medium-Gravity', 'Low-Gravity'
    LastRecalculatedDate DATETIME
);

-- Update gravity scores based on historical data
CREATE PROCEDURE UpdateProductGravityScores
AS
BEGIN
    UPDATE DimProductGravity
    SET 
        VelocityScore = (
            SELECT TOP 1 
                CASE 
                    WHEN AVG(PickFrequency) > 50 THEN 90
                    WHEN AVG(PickFrequency) > 20 THEN 60
                    ELSE 30
                END
            FROM (
                SELECT ProductKey, COUNT(*) as PickFrequency
                FROM FactWarehouseOperations
                WHERE OperationType = 'Pick' 
                  AND TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm'))
                GROUP BY ProductKey, TimeKey
            ) freq
            WHERE freq.ProductKey = DimProductGravity.ProductKey
        ),
        LastRecalculatedDate = GETDATE();
END;
```

## Common Analytical Queries

### Cross-Fact KPI: Dwell Time vs Fleet Idle Time

```sql
-- Identify products with high dwell time that correlate with fleet delays
SELECT 
    p.SKU,
    p.ProductName,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    COUNT(DISTINCT wo.OperationKey) AS WarehouseOps,
    COUNT(DISTINCT ft.TripKey) AS FleetTrips
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN FactFleetTrips ft ON wo.TimeKey = ft.TimeKey
WHERE wo.OperationType = 'Pick'
  AND ft.OnTimeFlag = 0
  AND wo.TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMddHHmm'))
GROUP BY p.SKU, p.ProductName, p.GravityScore
HAVING AVG(wo.DwellTimeMinutes) > 60
ORDER BY AVG(wo.DwellTimeMinutes) DESC;
```

### Predictive Bottleneck Index

```sql
-- Calculate bottleneck probability based on historical patterns
WITH HourlyLoad AS (
    SELECT 
        t.Hour,
        t.DayOfWeek,
        COUNT(*) AS OperationCount,
        AVG(wo.DwellTimeMinutes) AS AvgDwell,
        STDEV(wo.DwellTimeMinutes) AS StdDevDwell
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -90, GETDATE())
    GROUP BY t.Hour, t.DayOfWeek
)
SELECT 
    Hour,
    DayOfWeek,
    OperationCount,
    AvgDwell,
    CASE 
        WHEN AvgDwell > 90 AND OperationCount > 100 THEN 'HIGH'
        WHEN AvgDwell > 60 OR OperationCount > 80 THEN 'MEDIUM'
        ELSE 'LOW'
    END AS BottleneckRisk
FROM HourlyLoad
ORDER BY BottleneckRisk DESC, OperationCount DESC;
```

### Fleet Triage Priority Score

```sql
-- Rank vehicles for maintenance based on revenue impact
WITH VehicleHealth AS (
    SELECT 
        v.VehicleKey,
        v.VehicleID,
        v.MaintenanceStatus,
        AVG(ft.LoadWeight) AS AvgCargoWeight,
        AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0)) AS FuelEfficiency,
        SUM(CASE WHEN ft.OnTimeFlag = 0 THEN 1 ELSE 0 END) AS DelayCount,
        COUNT(*) AS TotalTrips
    FROM DimVehicle v
    INNER JOIN FactFleetTrips ft ON v.VehicleKey = ft.VehicleKey
    WHERE ft.TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm'))
    GROUP BY v.VehicleKey, v.VehicleID, v.MaintenanceStatus
)
SELECT 
    VehicleID,
    MaintenanceStatus,
    AvgCargoWeight,
    FuelEfficiency,
    DelayCount,
    -- Weighted score: delay impact (40%) + cargo value (30%) + efficiency (30%)
    (DelayCount * 0.4 + (AvgCargoWeight / 1000) * 0.3 + (1 / NULLIF(FuelEfficiency, 0)) * 0.3) AS TriagePriorityScore
FROM VehicleHealth
WHERE MaintenanceStatus IN ('Warning', 'Critical')
ORDER BY TriagePriorityScore DESC;
```

## Data Loading Patterns

### Incremental Load with Change Data Capture

```sql
-- Incremental warehouse operations load
CREATE PROCEDURE LoadIncrementalWarehouseOps
    @LastLoadTimestamp DATETIME
AS
BEGIN
    -- Assuming external staging table from WMS
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType, 
        QuantityHandled, DwellTimeMinutes, PickRateUnitsPerHour
    )
    SELECT 
        CONVERT(INT, FORMAT(stg.OperationTimestamp, 'yyyyMMddHHmm')) AS TimeKey,
        dw.WarehouseKey,
        dp.ProductKey,
        stg.OperationType,
        stg.Quantity,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DwellTimeMinutes,
        CASE 
            WHEN DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) > 0 
            THEN stg.Quantity * 60.0 / DATEDIFF(MINUTE, stg.StartTime, stg.EndTime)
            ELSE NULL 
        END AS PickRateUnitsPerHour
    FROM ExternalWarehouseOpsStaging stg
    INNER JOIN DimWarehouse dw ON stg.WarehouseCode = dw.WarehouseCode
    INNER JOIN DimProductGravity dp ON stg.SKU = dp.SKU
    WHERE stg.OperationTimestamp > @LastLoadTimestamp
      AND stg.IsProcessed = 0;
    
    -- Mark as processed
    UPDATE ExternalWarehouseOpsStaging
    SET IsProcessed = 1, ProcessedDate = GETDATE()
    WHERE OperationTimestamp > @LastLoadTimestamp;
END;
```

### Real-Time Telemetry Ingestion

```sql
-- Stored procedure for streaming fleet telemetry
CREATE PROCEDURE IngestFleetTelemetry
    @VehicleID VARCHAR(50),
    @Timestamp DATETIME,
    @Latitude DECIMAL(9,6),
    @Longitude DECIMAL(9,6),
    @Speed DECIMAL(5,2),
    @FuelLevel DECIMAL(5,2),
    @EngineStatus VARCHAR(50)
AS
BEGIN
    DECLARE @VehicleKey INT, @TimeKey INT;
    
    -- Resolve vehicle key
    SELECT @VehicleKey = VehicleKey FROM DimVehicle WHERE VehicleID = @VehicleID;
    
    -- Resolve time key (round to nearest 15-minute bucket)
    DECLARE @Rounded15Min DATETIME = DATEADD(
        MINUTE, 
        (DATEDIFF(MINUTE, 0, @Timestamp) / 15) * 15, 
        0
    );
    SELECT @TimeKey = TimeKey FROM DimTime WHERE FullDateTime = @Rounded15Min;
    
    -- Insert or update telemetry snapshot
    MERGE FactFleetTelemetry AS target
    USING (SELECT @VehicleKey AS VehicleKey, @TimeKey AS TimeKey) AS source
    ON target.VehicleKey = source.VehicleKey AND target.TimeKey = source.TimeKey
    WHEN MATCHED THEN
        UPDATE SET Latitude = @Latitude, Longitude = @Longitude, Speed = @Speed, FuelLevel = @FuelLevel
    WHEN NOT MATCHED THEN
        INSERT (VehicleKey, TimeKey, Latitude, Longitude, Speed, FuelLevel, EngineStatus)
        VALUES (@VehicleKey, @TimeKey, @Latitude, @Longitude, @Speed, @FuelLevel, @EngineStatus);
END;
```

## Power BI Integration

### DAX Measures for Cross-Fact Analysis

**Fleet Utilization Rate**

```dax
Fleet Utilization % = 
VAR TotalActiveTime = SUM(FactFleetTrips[DistanceKm]) / AVERAGE(FactFleetTrips[Speed])
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes]) / 60
RETURN 
DIVIDE(
    TotalActiveTime,
    TotalActiveTime + TotalIdleTime,
    0
) * 100
```

**Warehouse Throughput Efficiency**

```dax
Throughput Efficiency = 
VAR ActualPickRate = AVERAGE(FactWarehouseOperations[PickRateUnitsPerHour])
VAR BenchmarkPickRate = 150 -- units per hour
RETURN 
DIVIDE(ActualPickRate, BenchmarkPickRate, 0) * 100
```

**Gravity Zone Optimization Score**

```dax
Gravity Optimization Score = 
VAR HighGravityInCorrectZone = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProductGravity[RecommendedZone] = DimProductGravity[CurrentZone],
        DimProductGravity[GravityScore] > 70
    )
VAR TotalHighGravity = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProductGravity[GravityScore] > 70
    )
RETURN 
DIVIDE(HighGravityInCorrectZone, TotalHighGravity, 0) * 100
```

### Dynamic Row-Level Security

```dax
-- Create role for regional managers
[RegionalAccess] = 
VAR UserRegion = 
    LOOKUPVALUE(
        DimUser[Region],
        DimUser[Email], USERPRINCIPALNAME()
    )
RETURN 
DimWarehouse[Region] = UserRegion || 
DimWarehouse[Region] = "Global" -- Allow global view for executives
```

## Alerts & Automation

### Automated Alerting Stored Procedure

```sql
CREATE PROCEDURE CheckKPIThresholdsAndAlert
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check for excessive fleet idle time
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips
        WHERE TimeKey >= CONVERT(INT, FORMAT(DATEADD(HOUR, -4, GETDATE()), 'yyyyMMddHHmm'))
        GROUP BY VehicleKey
        HAVING AVG(IdleTimeMinutes) > 30
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeded 30 minutes average in last 4 hours';
        -- Send via email (configure Database Mail)
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'LogiFleet Pulse: Fleet Idle Time Alert',
            @body = @AlertMessage;
    END
    
    -- Check for warehouse dwell time anomalies
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations
        WHERE TimeKey >= CONVERT(INT, FORMAT(DATEADD(HOUR, -2, GETDATE()), 'yyyyMMddHHmm'))
          AND OperationType = 'Pick'
        GROUP BY ProductKey
        HAVING AVG(DwellTimeMinutes) > 120
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Warehouse dwell time exceeded 2 hours for some products';
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'LogiFleet Pulse: Dwell Time Alert',
            @body = @AlertMessage;
    END
END;

-- Schedule via SQL Agent Job to run every 15 minutes
```

## Configuration Best Practices

### Indexing Strategy

```sql
-- Composite indexes for common query patterns
CREATE NONCLUSTERED INDEX IX_FactWO_Time_Product_Type 
ON FactWarehouseOperations(TimeKey, ProductKey, OperationType)
INCLUDE (QuantityHandled, DwellTimeMinutes);

CREATE NONCLUSTERED INDEX IX_FactFT_Time_Vehicle_OnTime
ON FactFleetTrips(TimeKey, VehicleKey, OnTimeFlag)
INCLUDE (IdleTimeMinutes, FuelConsumedLiters);

-- Columnstore index for analytical queries on large fact tables
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWO_Columnstore
ON FactWarehouseOperations(TimeKey, ProductKey, QuantityHandled, DwellTimeMinutes);
```

### Partition Strategy for Time-Phased Data

```sql
-- Partition function for monthly partitions
CREATE PARTITION FUNCTION PF_Monthly (INT)
AS RANGE RIGHT FOR VALUES (
    202601, 202602, 202603, 202604, 202605, 202606, 
    202607, 202608, 202609, 202610, 202611, 202612
);

-- Partition scheme
CREATE PARTITION SCHEME PS_Monthly
AS PARTITION PF_Monthly
ALL TO ([PRIMARY]);

-- Apply to fact table (modify CREATE TABLE)
CREATE TABLE FactWarehouseOperations (
    -- columns...
) ON PS_Monthly(TimeKey);
```

## Troubleshooting

### Data Refresh Failures

**Issue:** Power BI reports show "Data refresh failed"

**Solution:**
1. Check SQL Server connection string in Power BI
2. Verify service account has db_datareader permissions
3. Test query timeout settings:

```sql
-- Identify long-running queries
SELECT 
    req.session_id,
    req.start_time,
    req.status,
    req.command,
    SUBSTRING(txt.text, (req.statement_start_offset/2)+1, 
        ((CASE req.statement_end_offset 
            WHEN -1 THEN DATALENGTH(txt.text)
            ELSE req.statement_end_offset 
        END - req.statement_start_offset)/2) + 1) AS query_text,
    req.wait_type,
    req.total_elapsed_time / 1000 AS elapsed_seconds
FROM sys.dm_exec_requests req
CROSS APPLY sys.dm_exec_sql_text(req.sql_handle) txt
WHERE req.database_id = DB_ID('LogiFleetPulse')
ORDER BY req.total_elapsed_time DESC;
```

### Missing Time Dimension Records

**Issue:** Queries fail with "TimeKey not found in DimTime"

**Solution:**
```sql
-- Check current time dimension coverage
SELECT MIN(FullDateTime) AS MinDate, MAX(FullDateTime) AS MaxDate
FROM DimTime;

-- Extend time dimension if needed
EXEC PopulateDimTime 
    @StartDate = '2026-01-01 00:00:00',
    @EndDate = '2027-12-31 23:45:00';
```

### Performance Degradation

**Issue:** Dashboard load time increases over time

**Solution:**
1. Update statistics on fact tables:

```sql
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

2. Rebuild fragmented indexes:

```sql
-- Find fragmented indexes
SELECT 
    OBJECT_NAME(ips.object_id) AS TableName,
    i.name AS IndexName,
    ips.avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 30
ORDER BY ips.avg_fragmentation_in_percent DESC;

-- Rebuild indexes
ALTER INDEX ALL ON FactWarehouseOperations REBUILD;
```

3. Archive historical data (older than 2 years):

```sql
-- Move to archive table
SELECT * INTO FactWarehouseOperations_Archive
FROM FactWarehouseOperations
WHERE TimeKey < CONVERT(INT, FORMAT(DATEADD(YEAR, -2, GETDATE()), 'yyyyMMddHHmm'));

DELETE FROM FactWarehouseOperations
WHERE TimeKey < CONVERT(INT, FORMAT(DATEADD(YEAR, -2, GETDATE()), 'yyyyMMddHHmm'));
```

## External API Integration

### Weather Correlation for Fleet Delays

```sql
-- Stored procedure to fetch and store weather data
CREATE PROCEDURE IngestWeatherData
    @LocationKey INT,
    @Timestamp DATETIME,
    @WeatherCondition VARCHAR(100),
    @Temperature DECIMAL(5,2),
    @Precipitation DECIMAL(5,2)
AS
BEGIN
    DECLARE @TimeKey INT;
    
    -- Round to 15-minute bucket
    DECLARE @Rounded15Min DATETIME = DATEADD(
        MINUTE, 
        (DATEDIFF(MINUTE, 0, @Timestamp) / 15) * 15, 
        0
    );
    SELECT @TimeKey = TimeKey FROM DimTime WHERE FullDateTime = @Rounded15Min;
    
    INSERT INTO FactWeatherConditions (TimeKey, LocationKey, Condition, Temperature, Precipitation)
    VALUES (@TimeKey, @LocationKey, @WeatherCondition, @Temperature, @Precipitation);
END;

-- Query to correlate weather with fleet delays
SELECT 
    w.Condition,
    COUNT(DISTINCT ft.TripKey) AS TotalTrips,
    SUM(CASE WHEN ft.OnTimeFlag = 0 THEN 1 ELSE 0 END) AS DelayedTrips,
    CAST(SUM(CASE WHEN ft.OnTimeFlag = 0 THEN 1 ELSE 0 END) AS FLOAT) / COUNT(DISTINCT ft.TripKey) * 100 AS DelayPercentage
FROM FactFleetTrips ft
INNER JOIN FactWeatherConditions w ON ft.TimeKey = w.TimeKey AND ft.OriginLocationKey = w.LocationKey
WHERE ft.TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm'))
GROUP BY w.Condition
ORDER BY DelayPercentage DESC;
```

## Advanced Scenarios

### Temporal Elasticity Simulation

```sql
-- Simulate warehouse capacity impact on fleet utilization
WITH CapacityScenarios AS (
    SELECT 0.80 AS CapacityLevel UNION ALL
    SELECT 0.85 UNION ALL
    SELECT 0.90 UNION ALL
    SELECT 0.95
),
HistoricalBaseline AS (
    SELECT 
        AVG(wo.DwellTimeMinutes) AS AvgDwell,
        AVG(ft.IdleTimeMinutes) AS AvgIdle
    FROM FactWarehouseOperations wo
    INNER JOIN FactFleetTrips ft ON wo.TimeKey = ft.TimeKey
    WHERE wo.TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMddHHmm'))
)
SELECT 
    cs.CapacityLevel,
    hb.AvgDwell * (1 + (cs.CapacityLevel - 0.80) * 2) AS ProjectedDwellTime,
    hb.AvgIdle * (1 + (cs.CapacityLevel - 0.80) * 1.5) AS ProjectedIdleTime,
    100 - ((hb.AvgDwell * (1 + (cs.CapacityLevel - 0.80) * 2)) / 120 * 100) AS EfficiencyScore
FROM CapacityScenarios cs
CROSS JOIN HistoricalBaseline hb
ORDER BY cs.CapacityLevel;
```

This skill provides comprehensive guidance for deploying and using LogiFleet Pulse as an advanced logistics analytics platform with MS SQL Server and Power BI.
