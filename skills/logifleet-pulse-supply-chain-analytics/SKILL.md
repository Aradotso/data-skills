---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server data warehouse and Power BI dashboard template for real-time logistics, fleet management, and supply chain analytics
triggers:
  - set up logifleet pulse supply chain analytics
  - configure logistics intelligence dashboard
  - deploy warehouse and fleet data warehouse
  - create supply chain power bi dashboard
  - implement multi-fact star schema for logistics
  - build real-time fleet optimization analytics
  - set up cross-modal supply chain reporting
  - configure logifleet pulse sql schema
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What It Does

LogiFleet Pulse is a data warehousing and analytics template for logistics and supply chain operations. It provides:

- **Multi-fact star schema** for MS SQL Server combining warehouse operations, fleet telemetry, and inventory data
- **Power BI dashboard templates** for real-time logistics KPIs
- **Cross-fact analytics** linking warehouse dwell time, fleet utilization, and demand patterns
- **Predictive bottleneck detection** and maintenance prioritization
- **Time-phased dimensions** for granular temporal analysis (15-minute buckets)

The project is a template/framework rather than an installable package — you deploy the SQL schema and customize the Power BI reports.

## Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems: WMS, TMS, telematics APIs, or CSV/Excel exports

## Installation & Deployment

### 1. Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### 2. Deploy SQL Schema

Connect to your SQL Server instance and execute the schema creation scripts:

```sql
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute dimension table scripts
:r DimTime.sql
:r DimGeography.sql
:r DimProductGravity.sql
:r DimSupplierReliability.sql
:r DimVehicle.sql
:r DimWarehouseZone.sql

-- Execute fact table scripts
:r FactWarehouseOperations.sql
:r FactFleetTrips.sql
:r FactCrossDock.sql
:r FactInventorySnapshot.sql

-- Execute bridge tables for many-to-many relationships
:r BridgeRouteZone.sql

-- Execute views and stored procedures
:r ViewUnifiedKPI.sql
:r SpIncrementalLoad.sql
:r SpAlertMonitoring.sql
```

### 3. Configure Data Source Connections

Update the configuration file with your connection strings:

```json
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}",
    "port": 1433
  },
  "dataSources": {
    "wms": {
      "type": "API",
      "endpoint": "${WMS_API_ENDPOINT}",
      "apiKey": "${WMS_API_KEY}"
    },
    "telematics": {
      "type": "API",
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "apiKey": "${TELEMATICS_API_KEY}"
    },
    "weather": {
      "type": "API",
      "endpoint": "https://api.weatherapi.com/v1",
      "apiKey": "${WEATHER_API_KEY}"
    }
  },
  "refreshInterval": 900
}
```

### 4. Open Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. When prompted, enter your SQL Server connection details
3. Configure row-level security based on your organizational structure
4. Publish to Power BI Service for sharing

## Key SQL Schema Components

### Fact Tables

**FactWarehouseOperations** — Core warehouse activity tracking:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseZoneKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20), -- 'RECEIVING', 'PUTAWAY', 'PICKING', 'PACKING', 'SHIPPING'
    Quantity DECIMAL(18,2),
    DwellTimeMinutes INT,
    EmployeeKey INT,
    BatchNumber VARCHAR(50),
    StartTimestamp DATETIME2,
    EndTimestamp DATETIME2,
    CONSTRAINT FK_WO_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WO_Zone FOREIGN KEY (WarehouseZoneKey) REFERENCES DimWarehouseZone(ZoneKey),
    CONSTRAINT FK_WO_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

-- Recommended indexes
CREATE NONCLUSTERED INDEX IX_WO_Time ON FactWarehouseOperations(TimeKey) INCLUDE (DwellTimeMinutes, Quantity);
CREATE NONCLUSTERED INDEX IX_WO_Product ON FactWarehouseOperations(ProductKey, OperationType);
CREATE COLUMNSTORE INDEX CIX_WO_Analytics ON FactWarehouseOperations;
```

**FactFleetTrips** — Vehicle telemetry and route tracking:

```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(12,2),
    AverageSpeedKmh DECIMAL(6,2),
    EngineHealthScore INT, -- 0-100
    TirePressurePSI INT,
    WeatherCondition VARCHAR(50),
    DelayMinutes INT,
    StartTimestamp DATETIME2,
    EndTimestamp DATETIME2,
    CONSTRAINT FK_FT_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FT_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_FT_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE NONCLUSTERED INDEX IX_FT_Time ON FactFleetTrips(TimeKey) INCLUDE (IdleTimeMinutes, FuelConsumedLiters);
CREATE NONCLUSTERED INDEX IX_FT_Vehicle ON FactFleetTrips(VehicleKey, EngineHealthScore);
```

### Dimension Tables

**DimTime** — Time-phased dimension with 15-minute granularity:

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
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
    FiscalYear INT,
    FiscalQuarter INT,
    ShiftCode VARCHAR(10) -- 'DAY', 'NIGHT', 'SWING'
);

-- Example population for 15-minute buckets
DECLARE @StartDate DATETIME2 = '2026-01-01';
DECLARE @EndDate DATETIME2 = '2027-12-31';

WITH TimeSeriesCTE AS (
    SELECT @StartDate AS TimeValue
    UNION ALL
    SELECT DATEADD(MINUTE, 15, TimeValue)
    FROM TimeSeriesCTE
    WHERE DATEADD(MINUTE, 15, TimeValue) <= @EndDate
)
INSERT INTO DimTime (TimeKey, FullDateTime, Year, Quarter, Month, Week, DayOfMonth, DayOfWeek, DayName, Hour, Minute, QuarterHour, IsWeekend, FiscalYear)
SELECT 
    CONVERT(INT, FORMAT(TimeValue, 'yyyyMMddHHmm')),
    TimeValue,
    YEAR(TimeValue),
    DATEPART(QUARTER, TimeValue),
    MONTH(TimeValue),
    DATEPART(WEEK, TimeValue),
    DAY(TimeValue),
    DATEPART(WEEKDAY, TimeValue),
    DATENAME(WEEKDAY, TimeValue),
    DATEPART(HOUR, TimeValue),
    DATEPART(MINUTE, TimeValue),
    DATEPART(MINUTE, TimeValue),
    CASE WHEN DATEPART(WEEKDAY, TimeValue) IN (1, 7) THEN 1 ELSE 0 END,
    CASE WHEN MONTH(TimeValue) >= 4 THEN YEAR(TimeValue) ELSE YEAR(TimeValue) - 1 END
FROM TimeSeriesCTE
OPTION (MAXRECURSION 0);
```

**DimProductGravity** — Product dimension with velocity-based gravity scoring:

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    IsFragile BIT,
    RequiresColdStorage BIT,
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value / fragility
    VelocityClass VARCHAR(20), -- 'FAST', 'MEDIUM', 'SLOW', 'OBSOLETE'
    LastUpdated DATETIME2 DEFAULT GETDATE()
);

-- Stored procedure to recalculate gravity scores
CREATE PROCEDURE SpUpdateProductGravity
AS
BEGIN
    UPDATE DimProductGravity
    SET GravityScore = (
        SELECT 
            (COUNT(*) * AVG(Quantity) * 100.0) / 
            (CASE WHEN IsFragile = 1 THEN 2.0 ELSE 1.0 END)
        FROM FactWarehouseOperations
        WHERE ProductKey = DimProductGravity.ProductKey
        AND TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMddHHmm'))
    ),
    VelocityClass = CASE 
        WHEN GravityScore > 75 THEN 'FAST'
        WHEN GravityScore > 40 THEN 'MEDIUM'
        WHEN GravityScore > 10 THEN 'SLOW'
        ELSE 'OBSOLETE'
    END,
    LastUpdated = GETDATE();
END;
```

## Common Analytical Queries

### Cross-Fact KPI: Dwell Time vs. Fleet Idling by Product

```sql
-- Identify products with high warehouse dwell correlating to fleet idle time
SELECT 
    p.SKU,
    p.ProductName,
    p.VelocityClass,
    AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwellMin,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMin,
    SUM(wo.Quantity) AS TotalUnitsShipped,
    (AVG(ft.IdleTimeMinutes) * AVG(wo.DwellTimeMinutes)) AS FrictionScore
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN FactFleetTrips ft ON wo.TimeKey = ft.TimeKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE 
    t.FullDateTime >= DATEADD(MONTH, -3, GETDATE())
    AND wo.OperationType = 'SHIPPING'
GROUP BY p.SKU, p.ProductName, p.VelocityClass
HAVING AVG(wo.DwellTimeMinutes) > 72 -- More than 3 days dwell
ORDER BY FrictionScore DESC;
```

### Predictive Bottleneck Index by Warehouse Zone

```sql
-- Calculate bottleneck risk score based on capacity, velocity, and incoming volume
WITH ZoneMetrics AS (
    SELECT 
        wz.ZoneName,
        wz.CapacityUnits,
        COUNT(*) AS OperationCount,
        SUM(wo.Quantity) AS CurrentLoad,
        AVG(wo.DwellTimeMinutes) AS AvgDwell,
        MAX(t.FullDateTime) AS LastActivity
    FROM FactWarehouseOperations wo
    INNER JOIN DimWarehouseZone wz ON wo.WarehouseZoneKey = wz.ZoneKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY wz.ZoneName, wz.CapacityUnits
)
SELECT 
    ZoneName,
    CapacityUnits,
    CurrentLoad,
    (CurrentLoad * 100.0 / CapacityUnits) AS UtilizationPct,
    AvgDwell,
    -- Bottleneck Index: (utilization * dwell) / ideal_dwell_threshold
    ((CurrentLoad * 100.0 / CapacityUnits) * AvgDwell / 24.0) AS BottleneckIndex,
    CASE 
        WHEN (CurrentLoad * 100.0 / CapacityUnits) > 95 THEN 'CRITICAL'
        WHEN (CurrentLoad * 100.0 / CapacityUnits) > 80 THEN 'WARNING'
        ELSE 'NORMAL'
    END AS RiskLevel
FROM ZoneMetrics
ORDER BY BottleneckIndex DESC;
```

### Fleet Maintenance Priority Queue

```sql
-- Prioritize vehicle maintenance by combining health score with revenue impact
WITH RevenueImpact AS (
    SELECT 
        v.VehicleID,
        v.LicensePlate,
        AVG(ft.EngineHealthScore) AS AvgHealthScore,
        AVG(ft.TirePressurePSI) AS AvgTirePressure,
        SUM(ft.DistanceKm) AS TotalDistanceKm,
        COUNT(DISTINCT ft.RouteKey) AS RouteCount,
        -- Estimate revenue impact: distance * avg_value_per_km
        SUM(ft.DistanceKm) * 2.5 AS EstimatedRevenueImpact
    FROM FactFleetTrips ft
    INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY v.VehicleID, v.LicensePlate
)
SELECT 
    VehicleID,
    LicensePlate,
    AvgHealthScore,
    AvgTirePressure,
    TotalDistanceKm,
    EstimatedRevenueImpact,
    -- Priority Score: (100 - health) * revenue_impact / 1000
    ((100 - AvgHealthScore) * EstimatedRevenueImpact / 1000.0) AS MaintenancePriorityScore,
    CASE 
        WHEN AvgHealthScore < 60 OR AvgTirePressure < 28 THEN 'URGENT'
        WHEN AvgHealthScore < 75 THEN 'SCHEDULED'
        ELSE 'MONITOR'
    END AS MaintenanceStatus
FROM RevenueImpact
ORDER BY MaintenancePriorityScore DESC;
```

## Incremental Data Loading

Stored procedure for efficient incremental ETL:

```sql
CREATE PROCEDURE SpIncrementalLoadWarehouseOps
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentTimestamp DATETIME2 = GETDATE();
    
    -- Insert new records from staging
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseZoneKey, ProductKey, OperationType, 
        Quantity, DwellTimeMinutes, EmployeeKey, BatchNumber,
        StartTimestamp, EndTimestamp
    )
    SELECT 
        CONVERT(INT, FORMAT(stg.StartTime, 'yyyyMMddHHmm')) AS TimeKey,
        wz.ZoneKey,
        p.ProductKey,
        stg.OperationType,
        stg.Quantity,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DwellTimeMinutes,
        emp.EmployeeKey,
        stg.BatchNumber,
        stg.StartTime,
        stg.EndTime
    FROM StagingWarehouseOps stg
    LEFT JOIN DimWarehouseZone wz ON stg.ZoneCode = wz.ZoneCode
    LEFT JOIN DimProductGravity p ON stg.SKU = p.SKU
    LEFT JOIN DimEmployee emp ON stg.EmployeeID = emp.EmployeeID
    WHERE stg.StartTime > @LastLoadTimestamp
    AND stg.StartTime <= @CurrentTimestamp;
    
    -- Log the load
    INSERT INTO ETLLog (TableName, RecordsProcessed, LoadTimestamp)
    VALUES ('FactWarehouseOperations', @@ROWCOUNT, @CurrentTimestamp);
    
    SELECT @@ROWCOUNT AS RecordsLoaded;
END;
```

## Power BI DAX Measures

Key calculated measures for the dashboard:

```dax
// Cross-Fact Efficiency Ratio
Logistics Efficiency = 
VAR WarehouseThroughput = 
    CALCULATE(
        SUM(FactWarehouseOperations[Quantity]),
        FactWarehouseOperations[OperationType] = "SHIPPING"
    )
VAR FleetUtilization = 
    DIVIDE(
        SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
        SUM(FactFleetTrips[DurationMinutes])
    )
RETURN
    WarehouseThroughput * FleetUtilization * 100

// Temporal Elasticity Score
Demand Elasticity = 
VAR CurrentWeekOrders = 
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationKey]),
        DATESINPERIOD(DimTime[FullDateTime], MAX(DimTime[FullDateTime]), -7, DAY)
    )
VAR PriorWeekOrders = 
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationKey]),
        DATESINPERIOD(DimTime[FullDateTime], MAX(DimTime[FullDateTime]) - 7, -7, DAY)
    )
RETURN
    DIVIDE(CurrentWeekOrders - PriorWeekOrders, PriorWeekOrders) * 100

// Gravity Zone Optimization Score
Zone Optimization % = 
VAR HighGravityInFastZone = 
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationKey]),
        DimProductGravity[VelocityClass] = "FAST",
        DimWarehouseZone[ZoneType] = "FAST_ACCESS"
    )
VAR TotalHighGravity = 
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationKey]),
        DimProductGravity[VelocityClass] = "FAST"
    )
RETURN
    DIVIDE(HighGravityInFastZone, TotalHighGravity) * 100
```

## Automated Alerting

SQL Server Agent job or Azure Function to monitor thresholds:

```sql
CREATE PROCEDURE SpAlertMonitoring
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Alert: Fleet idle time exceeds 15%
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips ft
        INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -4, GETDATE())
        GROUP BY ft.VehicleKey
        HAVING AVG(CAST(ft.IdleTimeMinutes AS FLOAT) / ft.DurationMinutes) > 0.15
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeds 15% in last 4 hours';
        -- Send email via sp_send_dbmail or log to table
        INSERT INTO AlertLog (Severity, Message, Timestamp)
        VALUES ('WARNING', @AlertMessage, GETDATE());
    END
    
    -- Alert: Warehouse zone utilization over 95%
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations wo
        INNER JOIN DimWarehouseZone wz ON wo.WarehouseZoneKey = wz.ZoneKey
        GROUP BY wz.ZoneKey, wz.CapacityUnits
        HAVING (SUM(wo.Quantity) * 100.0 / MAX(wz.CapacityUnits)) > 95
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Warehouse zone over 95% capacity';
        INSERT INTO AlertLog (Severity, Message, Timestamp)
        VALUES ('CRITICAL', @AlertMessage, GETDATE());
    END
    
    -- Alert: Vehicle health score drops below 60
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips ft
        INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -2, GETDATE())
        AND ft.EngineHealthScore < 60
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Vehicle health score critical';
        INSERT INTO AlertLog (Severity, Message, Timestamp)
        VALUES ('URGENT', @AlertMessage, GETDATE());
    END
END;
```

## Configuration Best Practices

### Indexing Strategy

```sql
-- For large fact tables, use partitioning by time
CREATE PARTITION FUNCTION PF_TimeKey (INT)
AS RANGE RIGHT FOR VALUES (
    202601010000, -- 2026-01-01
    202604010000, -- 2026-04-01
    202607010000, -- 2026-07-01
    202610010000  -- 2026-10-01
);

CREATE PARTITION SCHEME PS_TimeKey
AS PARTITION PF_TimeKey
ALL TO ([PRIMARY]);

-- Apply to fact table
CREATE TABLE FactWarehouseOperations (
    -- columns as before
) ON PS_TimeKey(TimeKey);

-- Columnstore for analytical queries
CREATE COLUMNSTORE INDEX CIX_Analytics 
ON FactWarehouseOperations (
    TimeKey, ProductKey, WarehouseZoneKey, Quantity, DwellTimeMinutes
);
```

### Row-Level Security

```sql
-- Create security function
CREATE FUNCTION dbo.fn_securitypredicate(@GeographyKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS fn_securitypredicate_result
WHERE 
    @GeographyKey IN (
        SELECT GeographyKey 
        FROM dbo.UserGeographyAccess 
        WHERE Username = USER_NAME()
    )
    OR IS_MEMBER('db_owner') = 1;

-- Apply to fact table
CREATE SECURITY POLICY GeographySecurityPolicy
ADD FILTER PREDICATE dbo.fn_securitypredicate(GeographyKey)
ON dbo.FactFleetTrips
WITH (STATE = ON);
```

## Troubleshooting

### Slow Dashboard Refresh

**Symptom**: Power BI dashboard takes >5 minutes to refresh

**Solution**:
```sql
-- Check for missing indexes
SELECT 
    t.name AS TableName,
    s.name AS SchemaName,
    i.name AS IndexName,
    ius.user_seeks + ius.user_scans + ius.user_lookups AS TotalReads,
    ius.user_updates AS TotalWrites
FROM sys.dm_db_index_usage_stats ius
INNER JOIN sys.indexes i ON ius.index_id = i.index_id AND ius.object_id = i.object_id
INNER JOIN sys.tables t ON i.object_id = t.object_id
INNER JOIN sys.schemas s ON t.schema_id = s.schema_id
WHERE ius.database_id = DB_ID()
ORDER BY TotalReads DESC;

-- Add missing columnstore index if not present
CREATE COLUMNSTORE INDEX CIX_FactFleetTrips 
ON FactFleetTrips;
```

### Data Freshness Issues

**Symptom**: Dashboard shows stale data

**Solution**: Configure SQL Server Agent job for incremental loads
```sql
EXEC msdb.dbo.sp_add_job 
    @job_name = N'IncrementalLoadWarehouse';

EXEC msdb.dbo.sp_add_jobstep 
    @job_name = N'IncrementalLoadWarehouse',
    @step_name = N'LoadOps',
    @command = N'EXEC SpIncrementalLoadWarehouseOps @LastLoadTimestamp = ?',
    @database_name = N'LogiFleetPulse';

EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;
```

### Cross-Fact Query Performance

**Symptom**: Queries joining multiple facts are slow

**Solution**: Create aggregated views
```sql
CREATE VIEW ViewUnifiedKPI
WITH SCHEMABINDING
AS
SELECT 
    t.TimeKey,
    t.Year,
    t.Month,
    t.DayOfWeek,
    wz.ZoneName,
    p.VelocityClass,
    SUM(wo.Quantity) AS TotalQuantity,
    AVG(wo.DwellTimeMinutes) AS AvgDwell,
    COUNT_BIG(*) AS OperationCount
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN dbo.DimWarehouseZone wz ON wo.WarehouseZoneKey = wz.ZoneKey
INNER JOIN dbo.DimProductGravity p ON wo.ProductKey = p.ProductKey
GROUP BY t.TimeKey, t.Year, t.Month, t.DayOfWeek, wz.ZoneName, p.VelocityClass;

-- Index the view
CREATE UNIQUE CLUSTERED INDEX IX_UnifiedKPI 
ON ViewUnifiedKPI (TimeKey, ZoneName, VelocityClass);
```

## Integration Examples

### Python ETL Script

```python
import pyodbc
import requests
import os
from datetime import datetime

# Connect to SQL Server
conn_str = (
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.getenv('SQL_SERVER_HOST')};"
    f"DATABASE=LogiFleetPulse;"
    f"UID={os.getenv('SQL_USERNAME')};"
    f"PWD={os.getenv('SQL_PASSWORD')}"
)
conn = pyodbc.connect(conn_str)
cursor = conn.cursor()

# Fetch telemetry from fleet API
fleet_api_url = os.getenv('TELEMATICS_API_ENDPOINT')
headers = {'Authorization': f"Bearer {os.getenv('TELEMATICS_API_KEY')}"}
response = requests.get(f"{fleet_api_url}/trips/recent", headers=headers)
trips = response.json()

# Insert into staging table
for trip in trips:
    cursor.execute("""
        INSERT INTO StagingFleetTrips 
        (VehicleID, StartTime, EndTime, DistanceKm, FuelLiters, IdleMinutes, HealthScore)
        VALUES (?, ?, ?, ?, ?, ?, ?)
    """, trip['vehicleId'], trip['startTime'], trip['endTime'], 
         trip['distance'], trip['fuel'], trip['idleTime'], trip['healthScore'])

conn.commit()

# Call incremental load procedure
cursor.execute("EXEC SpIncrementalLoadFleetTrips @LastLoadTimestamp = ?", 
               datetime.now())
conn.commit()
cursor.close()
conn.close()
```

### PowerShell Backup Script

```powershell
# Automated backup with retention
$server = $env:SQL_SERVER_HOST
$database = "LogiFleetPulse"
$backupPath = "C:\SQLBackups\LogiFleetPulse_$(Get-Date -Format 'yyyyMMdd_HHmm').bak"

Invoke-Sqlcmd -ServerInstance $server -Query @"
BACKUP DATABASE [$database] 
TO DISK = '$backupPath'
WITH COMPRESSION, STATS = 10;
"@

# Cleanup backups older than 30 days
Get-ChildItem "C:\SQLBackups" -Filter "LogiFleetPulse_*.bak" | 
Where-Object {$_.LastWriteTime -lt (Get-Date).AddDays(-30)} | 
Remove-Item
```

This skill provides AI agents with comprehensive knowledge to deploy, configure, query, and troubleshoot the LogiFleet Pulse supply chain analytics platform.
