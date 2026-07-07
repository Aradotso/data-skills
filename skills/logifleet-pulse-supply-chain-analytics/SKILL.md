---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for logistics intelligence with multi-fact star schema, fleet optimization, and warehouse gravity zones
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure warehouse and fleet data model"
  - "create logistics KPI dashboard with Power BI"
  - "implement multi-fact star schema for supply chain"
  - "build warehouse gravity zone optimization"
  - "deploy fleet telemetry analytics"
  - "connect WMS data to SQL Server warehouse"
  - "create cross-modal supply chain reports"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced logistics intelligence platform built on MS SQL Server and Power BI. It provides:

- **Multi-fact star schema** for warehouse operations, fleet trips, and cross-dock activities
- **Warehouse Gravity Zones™** - spatial optimization based on pick frequency, value, and fragility
- **Cross-fact KPI harmonization** - linking inventory metrics with fleet performance
- **Temporal elasticity modeling** - time-phased simulation scenarios
- **Adaptive fleet triage** - predictive maintenance prioritization by revenue impact

The system ingests data from WMS, telematics, supplier portals, weather/traffic APIs, and order history into a unified semantic layer.

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Database Setup

1. **Deploy the SQL schema:**

```sql
-- Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Enable snapshot isolation for consistent reads
ALTER DATABASE LogiFleetPulse SET ALLOW_SNAPSHOT_ISOLATION ON;
ALTER DATABASE LogiFleetPulse SET READ_COMMITTED_SNAPSHOT ON;
GO
```

2. **Create dimension tables:**

```sql
-- DimTime (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2(0) NOT NULL,
    [Date] DATE NOT NULL,
    [Hour] TINYINT NOT NULL,
    [Minute] TINYINT NOT NULL,
    QuarterHour TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(20) NOT NULL,
    FiscalPeriod VARCHAR(10) NOT NULL,
    IsBusinessHours BIT NOT NULL,
    INDEX IX_DimTime_Date ([Date]),
    INDEX IX_DimTime_FiscalPeriod (FiscalPeriod)
);

-- DimGeography (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    Continent VARCHAR(50) NOT NULL,
    Country VARCHAR(100) NOT NULL,
    Region VARCHAR(100) NOT NULL,
    City VARCHAR(100) NOT NULL,
    LocationCode VARCHAR(50) NOT NULL,
    LocationType VARCHAR(20) NOT NULL, -- 'Warehouse', 'RouteNode', 'CustomerSite'
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    INDEX IX_DimGeography_LocationCode (LocationCode),
    INDEX IX_DimGeography_Type (LocationType)
);

-- DimProductGravity (warehouse spatial optimization)
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200) NOT NULL,
    Category VARCHAR(100) NOT NULL,
    GravityScore DECIMAL(5,2) NOT NULL, -- Calculated: velocity * value / fragility
    PickFrequency DECIMAL(8,2) NOT NULL, -- Picks per day
    UnitValue DECIMAL(10,2) NOT NULL,
    FragilityIndex DECIMAL(3,2) NOT NULL, -- 0.0 (durable) to 1.0 (fragile)
    OptimalZone VARCHAR(20) NOT NULL, -- 'HighGravity', 'MediumGravity', 'LowGravity'
    CurrentZone VARCHAR(20) NOT NULL,
    LastUpdated DATETIME2(0) NOT NULL,
    INDEX IX_ProductGravity_Score (GravityScore DESC),
    INDEX IX_ProductGravity_Zone (CurrentZone)
);

-- DimSupplierReliability
CREATE TABLE DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierCode VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200) NOT NULL,
    LeadTimeMean DECIMAL(6,2) NOT NULL, -- Days
    LeadTimeVariance DECIMAL(6,2) NOT NULL,
    DefectRate DECIMAL(5,4) NOT NULL, -- Percentage
    ComplianceScore DECIMAL(5,2) NOT NULL, -- 0-100
    ReliabilityTier VARCHAR(20) NOT NULL, -- 'Premium', 'Standard', 'Unreliable'
    LastEvaluated DATE NOT NULL,
    INDEX IX_Supplier_Tier (ReliabilityTier)
);
```

3. **Create fact tables:**

```sql
-- FactWarehouseOperations
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(20) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity INT NOT NULL,
    DurationMinutes DECIMAL(8,2) NOT NULL,
    DwellTimeHours DECIMAL(10,2), -- Time in warehouse before shipping
    ZoneCode VARCHAR(20) NOT NULL,
    OperatorID VARCHAR(50) NOT NULL,
    BatchID VARCHAR(50),
    INDEX IX_FactWarehouse_Time (TimeKey),
    INDEX IX_FactWarehouse_Product (ProductKey),
    INDEX IX_FactWarehouse_Type (OperationType)
) WITH (DATA_COMPRESSION = PAGE);

-- FactFleetTrips
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    StartTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    EndTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50) NOT NULL,
    DistanceKM DECIMAL(10,2) NOT NULL,
    FuelConsumedLiters DECIMAL(8,2) NOT NULL,
    IdleMinutes DECIMAL(8,2) NOT NULL,
    LoadingMinutes DECIMAL(8,2) NOT NULL,
    UnloadingMinutes DECIMAL(8,2) NOT NULL,
    LoadWeight DECIMAL(10,2), -- Kilograms
    DelayMinutes DECIMAL(8,2) DEFAULT 0,
    DelayReason VARCHAR(100),
    INDEX IX_FactFleet_StartTime (StartTimeKey),
    INDEX IX_FactFleet_Vehicle (VehicleID),
    INDEX IX_FactFleet_Route (OriginGeographyKey, DestinationGeographyKey)
) WITH (DATA_COMPRESSION = PAGE);

-- FactCrossDock (transfers without long-term storage)
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    Quantity INT NOT NULL,
    TransferMinutes DECIMAL(8,2) NOT NULL,
    INDEX IX_FactCrossDock_Time (TimeKey),
    INDEX IX_FactCrossDock_Product (ProductKey)
) WITH (DATA_COMPRESSION = PAGE);
```

4. **Create bridge table for many-to-many relationships:**

```sql
-- Bridge between routes and storage zones
CREATE TABLE BridgeRouteToPick (
    BridgeKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TripKey BIGINT NOT NULL FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OperationKey BIGINT NOT NULL FOREIGN KEY REFERENCES FactWarehouseOperations(OperationKey),
    SequenceNumber INT NOT NULL,
    INDEX IX_Bridge_Trip (TripKey),
    INDEX IX_Bridge_Operation (OperationKey)
);
```

## Key SQL Procedures

### Data Loading

```sql
-- Incremental load from WMS staging table
CREATE PROCEDURE usp_LoadWarehouseOperations
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        Quantity, DurationMinutes, DwellTimeHours, ZoneCode, OperatorID, BatchID
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        stg.OperationType,
        stg.Quantity,
        stg.DurationMinutes,
        DATEDIFF(HOUR, stg.ReceiveTime, stg.ShipTime) AS DwellTimeHours,
        stg.ZoneCode,
        stg.OperatorID,
        stg.BatchID
    FROM StagingWarehouseOps stg
    INNER JOIN DimTime t ON stg.OperationTime = t.FullDateTime
    INNER JOIN DimGeography g ON stg.LocationCode = g.LocationCode
    INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
    WHERE NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations f
        WHERE f.BatchID = stg.BatchID AND f.OperatorID = stg.OperatorID
    );
    
    -- Update gravity scores based on recent activity
    UPDATE p
    SET 
        PickFrequency = recent.DailyPicks,
        GravityScore = (recent.DailyPicks * p.UnitValue) / NULLIF(p.FragilityIndex, 0),
        OptimalZone = CASE 
            WHEN (recent.DailyPicks * p.UnitValue) / NULLIF(p.FragilityIndex, 0) > 1000 THEN 'HighGravity'
            WHEN (recent.DailyPicks * p.UnitValue) / NULLIF(p.FragilityIndex, 0) > 500 THEN 'MediumGravity'
            ELSE 'LowGravity'
        END,
        LastUpdated = GETDATE()
    FROM DimProductGravity p
    INNER JOIN (
        SELECT 
            ProductKey,
            COUNT(*) / 30.0 AS DailyPicks
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
          AND TimeKey >= DATEPART(YEAR, DATEADD(MONTH, -1, GETDATE())) * 1000000 
                        + DATEPART(MONTH, DATEADD(MONTH, -1, GETDATE())) * 10000
        GROUP BY ProductKey
    ) recent ON p.ProductKey = recent.ProductKey;
END;
GO
```

### Cross-Fact KPI Query

```sql
-- Combined warehouse dwell time vs fleet idle time by product
CREATE VIEW vw_DwellTimeVsFleetIdle AS
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    AVG(wo.DwellTimeHours) AS AvgDwellHours,
    AVG(ft.IdleMinutes) AS AvgFleetIdleMinutes,
    COUNT(DISTINCT wo.OperationKey) AS WarehouseOps,
    COUNT(DISTINCT ft.TripKey) AS FleetTrips,
    -- Composite inefficiency score
    (AVG(wo.DwellTimeHours) * 0.6 + AVG(ft.IdleMinutes) / 60.0 * 0.4) AS CompositeInefficiencyScore
FROM DimProductGravity p
LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
LEFT JOIN BridgeRouteToPick b ON wo.OperationKey = b.OperationKey
LEFT JOIN FactFleetTrips ft ON b.TripKey = ft.TripKey
WHERE wo.TimeKey >= DATEPART(YEAR, DATEADD(MONTH, -3, GETDATE())) * 1000000
GROUP BY p.SKU, p.ProductName, p.GravityScore;
GO
```

### Predictive Bottleneck Detection

```sql
-- Identify potential congestion points
CREATE PROCEDURE usp_PredictBottlenecks
    @ForecastHours INT = 24
AS
BEGIN
    -- Historical pattern: operations per zone per hour
    WITH ZoneActivity AS (
        SELECT 
            wo.ZoneCode,
            t.[Hour],
            AVG(CAST(wo.DurationMinutes AS FLOAT)) AS AvgDurationMinutes,
            COUNT(*) AS OperationCount
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.[Date] >= DATEADD(DAY, -30, CAST(GETDATE() AS DATE))
        GROUP BY wo.ZoneCode, t.[Hour]
    ),
    -- Predict next 24 hours
    FuturePeriods AS (
        SELECT 
            za.ZoneCode,
            za.[Hour],
            za.AvgDurationMinutes,
            za.OperationCount,
            -- Bottleneck score: operations * avg duration / theoretical capacity
            (za.OperationCount * za.AvgDurationMinutes) / 60.0 AS BottleneckScore
        FROM ZoneActivity za
    )
    SELECT 
        ZoneCode,
        [Hour],
        BottleneckScore,
        CASE 
            WHEN BottleneckScore > 0.8 THEN 'CRITICAL'
            WHEN BottleneckScore > 0.6 THEN 'WARNING'
            ELSE 'NORMAL'
        END AS AlertLevel
    FROM FuturePeriods
    WHERE BottleneckScore > 0.5
    ORDER BY BottleneckScore DESC;
END;
GO
```

## Power BI Configuration

### Connection Setup

1. **Open Power BI Desktop** and create new report
2. **Get Data → SQL Server**:
   - Server: `localhost` or your SQL Server instance
   - Database: `LogiFleetPulse`
   - Data Connectivity mode: `DirectQuery` (for real-time) or `Import` (for better performance)

### DAX Measures

```dax
-- Total Operations
TotalOperations = COUNTROWS(FactWarehouseOperations)

-- Average Dwell Time (hours)
AvgDwellTime = 
AVERAGE(FactWarehouseOperations[DwellTimeHours])

-- Fleet Utilization Rate
FleetUtilization = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKM]),
    COUNTROWS(FactFleetTrips) * 500  -- Assume 500km daily capacity
)

-- Gravity Zone Misalignment Count
GravityMisalignment = 
COUNTROWS(
    FILTER(
        DimProductGravity,
        DimProductGravity[OptimalZone] <> DimProductGravity[CurrentZone]
    )
)

-- Cross-Fact: Dwell Time Impact on Fuel
DwellFuelImpact = 
VAR HighDwellProducts = 
    FILTER(
        VALUES(DimProductGravity[ProductKey]),
        CALCULATE(AVERAGE(FactWarehouseOperations[DwellTimeHours])) > 72
    )
RETURN
    CALCULATE(
        AVERAGE(FactFleetTrips[FuelConsumedLiters]),
        KEEPFILTERS(HighDwellProducts)
    )

-- Predictive: Expected Bottleneck Hours Next Week
ExpectedBottleneckHours = 
VAR HistoricalPattern = 
    CALCULATETABLE(
        SUMMARIZE(
            FactWarehouseOperations,
            DimTime[Hour],
            "AvgOps", COUNTROWS(FactWarehouseOperations)
        ),
        DATESINPERIOD(DimTime[Date], MAX(DimTime[Date]), -30, DAY)
    )
RETURN
    COUNTROWS(FILTER(HistoricalPattern, [AvgOps] > 100))
```

### Row-Level Security

```dax
-- Create role: WarehouseManager (only sees their warehouse)
[GeographyKey] = LOOKUPVALUE(
    DimGeography[GeographyKey],
    DimGeography[LocationCode],
    USERNAME()
)
```

Apply to both `FactWarehouseOperations` and `FactFleetTrips` via Geography dimension.

## Common Patterns

### Pattern 1: Time-Phased Scenario Analysis

```sql
-- Simulate 95% capacity vs current 80%
DECLARE @CurrentCapacity FLOAT = 0.80;
DECLARE @TargetCapacity FLOAT = 0.95;

WITH CurrentState AS (
    SELECT 
        g.LocationCode,
        COUNT(*) AS Operations,
        AVG(DurationMinutes) AS AvgDuration
    FROM FactWarehouseOperations wo
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    WHERE TimeKey >= DATEPART(YEAR, GETDATE()) * 1000000 + DATEPART(MONTH, GETDATE()) * 10000
    GROUP BY g.LocationCode
)
SELECT 
    LocationCode,
    Operations AS CurrentOperations,
    Operations * (@TargetCapacity / @CurrentCapacity) AS ProjectedOperations,
    AvgDuration AS CurrentAvgDuration,
    AvgDuration * (@TargetCapacity / @CurrentCapacity) AS ProjectedAvgDuration,
    -- Estimated fleet impact
    (Operations * (@TargetCapacity / @CurrentCapacity) - Operations) * 0.5 AS AdditionalFleetTripsNeeded
FROM CurrentState;
```

### Pattern 2: Warehouse Gravity Zone Optimization

```sql
-- Recommend zone reassignments
SELECT 
    p.SKU,
    p.ProductName,
    p.CurrentZone,
    p.OptimalZone,
    p.GravityScore,
    COUNT(wo.OperationKey) AS RecentPicks,
    AVG(wo.DurationMinutes) AS AvgPickTime
FROM DimProductGravity p
LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
WHERE p.CurrentZone <> p.OptimalZone
  AND wo.OperationType = 'Picking'
  AND wo.TimeKey >= DATEPART(YEAR, DATEADD(MONTH, -1, GETDATE())) * 1000000
GROUP BY p.SKU, p.ProductName, p.CurrentZone, p.OptimalZone, p.GravityScore
HAVING COUNT(wo.OperationKey) > 10
ORDER BY p.GravityScore DESC;
```

### Pattern 3: Fleet Maintenance Prioritization

```sql
-- Adaptive fleet triage based on telemetry and cargo value
CREATE VIEW vw_FleetTriageQueue AS
SELECT 
    ft.VehicleID,
    SUM(p.UnitValue * wo.Quantity) AS TotalCargoValue,
    AVG(ft.IdleMinutes) AS AvgIdleTime,
    MAX(ft.DelayMinutes) AS MaxDelay,
    COUNT(DISTINCT ft.TripKey) AS TripsLast30Days,
    -- Triage score: cargo value weight + delay penalty
    (SUM(p.UnitValue * wo.Quantity) * 0.7 + AVG(ft.IdleMinutes) * 0.3) AS TriageScore
FROM FactFleetTrips ft
INNER JOIN BridgeRouteToPick b ON ft.TripKey = b.TripKey
INNER JOIN FactWarehouseOperations wo ON b.OperationKey = wo.OperationKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE ft.StartTimeKey >= DATEPART(YEAR, DATEADD(DAY, -30, GETDATE())) * 1000000
GROUP BY ft.VehicleID;
GO
```

## Environment Configuration

### Config File Structure

Create `config.json` (do not commit with real credentials):

```json
{
  "database": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "auth": {
      "type": "sql",
      "username": "${SQL_SERVER_USER}",
      "password": "${SQL_SERVER_PASSWORD}"
    }
  },
  "dataSources": {
    "wms": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "apiKey": "${WMS_API_KEY}"
    },
    "telematics": {
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "apiKey": "${TELEMATICS_API_KEY}"
    },
    "weather": {
      "endpoint": "https://api.weatherapi.com/v1",
      "apiKey": "${WEATHER_API_KEY}"
    }
  },
  "powerbi": {
    "workspaceId": "${POWERBI_WORKSPACE_ID}",
    "reportId": "${POWERBI_REPORT_ID}",
    "refreshSchedule": "*/15 * * * *"
  },
  "alerts": {
    "smtp": {
      "server": "${SMTP_SERVER}",
      "port": 587,
      "username": "${SMTP_USER}",
      "password": "${SMTP_PASSWORD}"
    },
    "thresholds": {
      "dwellTimeHours": 72,
      "fleetIdlePercent": 15,
      "gravityMisalignmentCount": 50
    }
  }
}
```

### Environment Variables

```bash
export SQL_SERVER_HOST="localhost"
export SQL_SERVER_USER="logifleet_app"
export SQL_SERVER_PASSWORD="your_secure_password"
export WMS_API_ENDPOINT="https://wms.yourcompany.com/api/v1"
export WMS_API_KEY="your_wms_api_key"
export TELEMATICS_API_ENDPOINT="https://fleet.telemetry.com/api"
export TELEMATICS_API_KEY="your_telemetry_key"
export WEATHER_API_KEY="your_weather_api_key"
export POWERBI_WORKSPACE_ID="your_workspace_id"
export POWERBI_REPORT_ID="your_report_id"
export SMTP_SERVER="smtp.office365.com"
export SMTP_USER="alerts@yourcompany.com"
export SMTP_PASSWORD="your_smtp_password"
```

## Automated Alerts

```sql
-- Alert stored procedure
CREATE PROCEDURE usp_SendLogisticsAlerts
AS
BEGIN
    DECLARE @AlertBody NVARCHAR(MAX);
    DECLARE @Subject NVARCHAR(200);
    
    -- Check dwell time threshold
    IF EXISTS (
        SELECT 1 
        FROM FactWarehouseOperations 
        WHERE DwellTimeHours > 72 
          AND TimeKey >= DATEPART(YEAR, CAST(GETDATE() AS DATE)) * 1000000
    )
    BEGIN
        SET @Subject = 'ALERT: High Dwell Time Detected';
        SET @AlertBody = 'Products exceeding 72-hour dwell time: ' + 
            (SELECT STRING_AGG(p.SKU, ', ')
             FROM FactWarehouseOperations wo
             INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
             WHERE wo.DwellTimeHours > 72);
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'logistics-team@yourcompany.com',
            @subject = @Subject,
            @body = @AlertBody;
    END;
    
    -- Check fleet idle time
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips
        WHERE (IdleMinutes / NULLIF(DATEDIFF(MINUTE, StartTimeKey, EndTimeKey), 0)) > 0.15
    )
    BEGIN
        SET @Subject = 'ALERT: High Fleet Idle Time';
        SET @AlertBody = 'Vehicles with >15% idle time detected.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'fleet-managers@yourcompany.com',
            @subject = @Subject,
            @body = @AlertBody;
    END;
END;
GO

-- Schedule the alert (runs every 15 minutes)
EXEC msdb.dbo.sp_add_job
    @job_name = 'LogiFleet_Alert_Check';

EXEC msdb.dbo.sp_add_jobstep
    @job_name = 'LogiFleet_Alert_Check',
    @step_name = 'Execute_Alert_Check',
    @subsystem = 'TSQL',
    @database_name = 'LogiFleetPulse',
    @command = 'EXEC usp_SendLogisticsAlerts';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = 'Every15Minutes',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = 'LogiFleet_Alert_Check',
    @schedule_name = 'Every15Minutes';

EXEC msdb.dbo.sp_add_jobserver
    @job_name = 'LogiFleet_Alert_Check';
```

## Troubleshooting

### Issue: Slow Query Performance

**Solution:** Add columnstore indexes to fact tables:

```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse
ON FactWarehouseOperations (TimeKey, GeographyKey, ProductKey, DwellTimeHours)
WITH (DATA_COMPRESSION = COLUMNSTORE);

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleet
ON FactFleetTrips (StartTimeKey, OriginGeographyKey, DestinationGeographyKey, IdleMinutes, FuelConsumedLiters)
WITH (DATA_COMPRESSION = COLUMNSTORE);
```

### Issue: Power BI DirectQuery Timeout

**Solution:** Switch to Import mode or create aggregated views:

```sql
CREATE VIEW vw_WarehouseDaily AS
SELECT 
    CAST(t.[Date] AS DATE) AS [Date],
    g.LocationCode,
    p.Category,
    COUNT(*) AS Operations,
    AVG(DwellTimeHours) AS AvgDwellTime,
    SUM(Quantity) AS TotalQuantity
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
GROUP BY CAST(t.[Date] AS DATE), g.LocationCode, p.Category;
```

### Issue: Gravity Scores Not Updating

**Solution:** Check if staging tables have recent data:

```sql
SELECT MAX(OperationTime) AS LastStagingRecord
FROM StagingWarehouseOps;

-- If stale, check ETL job status
EXEC msdb.dbo.sp_help_job @job_name = 'LogiFleet_ETL';
```

### Issue: Incorrect Time Zone in Reports

**Solution:** Normalize all timestamps to UTC in dimension table:

```sql
-- Add UTC conversion
ALTER TABLE DimTime ADD UTCDateTime DATETIME2(0);

UPDATE DimTime
SET UTCDateTime = SWITCHOFFSET(TODATETIMEOFFSET(FullDateTime, '+00:00'), '+00:00');
```

## Best Practices

1. **Partition fact tables by month** for faster queries:

```sql
CREATE PARTITION FUNCTION PF_Monthly (INT)
AS RANGE RIGHT FOR VALUES (202601, 202602, 202603); -- Extend monthly

CREATE PARTITION SCHEME PS_Monthly
AS PARTITION PF_Monthly ALL TO ([PRIMARY]);

-- Recreate fact table on partition scheme
CREATE TABLE FactWarehouseOperations_Partitioned (
    -- same columns
) ON PS_Monthly(TimeKey);
```

2. **Use surrogate keys** (INT identity) instead of natural keys for better performance

3. **Implement change data capture** for dimension updates:

```sql
EXEC sys.sp_cdc_enable_db;

EXEC sys.sp_cdc_enable_table
    @source_schema = 'dbo',
    @source_name = 'DimProductGravity',
    @role_name = NULL;
```

4. **Schedule regular index maintenance**:

```sql
CREATE PROCEDURE usp_RebuildIndexes
AS
BEGIN
    ALTER INDEX ALL ON FactWarehouseOperations REBUILD WITH (ONLINE = ON);
    ALTER INDEX ALL ON FactFleetTrips REBUILD WITH (ONLINE = ON);
    UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
    UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
END;
```

5. **Version control your Power BI templates** - save as `.pbit` and commit to source control

## Integration Examples

### Python ETL Script

```python
import pyodbc
import requests
import os
from datetime import datetime

# Connect to SQL Server
conn_str = f"DRIVER={{ODBC Driver 17 for SQL Server}};SERVER={os.getenv('SQL_SERVER_HOST')};DATABASE=LogiFleetPulse;UID={os.getenv('SQL_SERVER_USER')};PWD={os.getenv('SQL_SERVER_PASSWORD')}"
conn = pyodbc.connect(conn_str)
cursor = conn.cursor()

# Fetch from WMS API
wms_response = requests.get(
    f"{os.getenv('WMS_API_ENDPOINT')}/operations",
    headers={"
