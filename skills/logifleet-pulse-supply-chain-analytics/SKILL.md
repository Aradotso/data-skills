---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing engine for logistics, fleet management, and supply chain intelligence with multi-fact star schema
triggers:
  - "set up logifleet pulse supply chain analytics"
  - "configure power bi logistics dashboard"
  - "deploy sql server warehouse schema for fleet management"
  - "implement multi-fact star schema for logistics"
  - "create supply chain kpi dashboard"
  - "integrate warehouse and fleet telemetry data"
  - "build cross-modal logistics intelligence"
  - "configure logifleet pulse data model"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

LogiFleet Pulse is a multi-modal logistics intelligence engine combining MS SQL Server data warehousing with Power BI visualization. It unifies warehouse operations, fleet telemetry, inventory management, and external data (weather, traffic) into a single semantic layer using a custom multi-fact star schema.

**Key capabilities:**
- Multi-fact star schema with time-phased dimensions
- Real-time warehouse and fleet KPI harmonization
- Predictive bottleneck detection
- Warehouse gravity zone optimization
- Cross-fact queries linking inventory turnover to fleet metrics
- Role-based access control and row-level security

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- .NET Framework 4.8+ (for stored procedures)

### Step 1: Clone Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance in SSMS
-- Execute the main schema deployment script
:r .\sql\01_create_database.sql
:r .\sql\02_create_dimensions.sql
:r .\sql\03_create_facts.sql
:r .\sql\04_create_views.sql
:r .\sql\05_create_stored_procedures.sql
:r .\sql\06_create_indexes.sql
```

Or use PowerShell for automated deployment:

```powershell
$ServerInstance = $env:SQL_SERVER_INSTANCE
$Database = "LogiFleetPulse"

Invoke-Sqlcmd -ServerInstance $ServerInstance -InputFile ".\sql\deploy_all.sql" -Variable "DatabaseName=$Database"
```

### Step 3: Configure Data Sources

Create a configuration file `config.json`:

```json
{
  "connections": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}"
    },
    "fleet_telemetry": {
      "endpoint": "${FLEET_TELEMETRY_ENDPOINT}",
      "api_key": "${FLEET_API_KEY}"
    },
    "weather_api": {
      "endpoint": "https://api.weatherapi.com/v1",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "refresh_intervals": {
    "warehouse_operations": 15,
    "fleet_trips": 10,
    "external_data": 60
  }
}
```

### Step 4: Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI Template
3. Select `LogiFleet_Pulse_Master.pbit`
4. Enter connection string: `Server=${SQL_SERVER_INSTANCE};Database=LogiFleetPulse;`
5. Configure authentication (Windows or SQL Server)

## Core Data Model

### Dimension Tables

**DimTime** - 15-minute granularity time dimension

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME NOT NULL,
    HourOfDay INT,
    DayOfWeek INT,
    WeekOfYear INT,
    Month INT,
    Quarter INT,
    FiscalPeriod INT,
    IsWeekend BIT,
    IsHoliday BIT
);

-- Create time dimension for 2 years
EXEC sp_GenerateTimeDimension 
    @StartDate = '2026-01-01', 
    @EndDate = '2027-12-31', 
    @IntervalMinutes = 15;
```

**DimGeography** - Hierarchical location dimension

```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID NVARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200),
    LocationType NVARCHAR(50), -- 'Warehouse', 'RouteNode', 'Customer'
    Continent NVARCHAR(50),
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    ParentLocationID NVARCHAR(50)
);
```

**DimProductGravity** - Product classification with velocity scoring

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- 0-100, higher = faster moving
    IsFragile BIT,
    RequiresRefrigeration BIT,
    AverageLeadTimeDays INT,
    UnitValue DECIMAL(10,2)
);

-- Calculate gravity score
UPDATE DimProductGravity
SET GravityScore = (
    (VelocityScore * 0.5) + 
    (ValueScore * 0.3) + 
    (FragilityPenalty * 0.2)
);
```

### Fact Tables

**FactWarehouseOperations** - Warehouse activity tracking

```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OperationType NVARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityUnits INT,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes INT,
    ZoneID NVARCHAR(50),
    GravityZone NVARCHAR(50), -- 'High', 'Medium', 'Low'
    EmployeeID NVARCHAR(50),
    BatchID NVARCHAR(100)
);

-- Indexed for common queries
CREATE NONCLUSTERED INDEX IX_FactWarehouse_TimeProduct 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (DwellTimeMinutes, ProcessingTimeMinutes);
```

**FactFleetTrips** - Fleet and route telemetry

```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleID NVARCHAR(50),
    DriverID NVARCHAR(50),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    WeightKG DECIMAL(10,2),
    TripStatus NVARCHAR(50), -- 'Completed', 'InProgress', 'Delayed', 'Cancelled'
    DelayReasonCode NVARCHAR(50)
);
```

**FactCrossDock** - Cross-dock operations bridge table

```sql
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    TransferTimeMinutes INT,
    QuantityUnits INT
);
```

## Key Queries and Patterns

### Cross-Fact KPI: Dwell Time vs Fleet Idle Time

```sql
-- Find correlation between warehouse dwell time and fleet idling
WITH WarehouseDwell AS (
    SELECT 
        t.DateTime,
        g.Region,
        p.Category,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE t.DateTime >= DATEADD(day, -30, GETDATE())
    GROUP BY t.DateTime, g.Region, p.Category
),
FleetIdle AS (
    SELECT 
        t.DateTime,
        g.Region,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        COUNT(*) AS TripCount
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON f.OriginGeographyKey = g.GeographyKey
    WHERE t.DateTime >= DATEADD(day, -30, GETDATE())
    GROUP BY t.DateTime, g.Region
)
SELECT 
    wd.DateTime,
    wd.Region,
    wd.Category,
    wd.AvgDwellTime,
    fi.AvgIdleTime,
    (wd.AvgDwellTime * fi.AvgIdleTime) AS CompositeDelayScore
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.DateTime = fi.DateTime AND wd.Region = fi.Region
ORDER BY CompositeDelayScore DESC;
```

### Warehouse Gravity Zone Analysis

```sql
-- Identify products that should be relocated to different gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    w.GravityZone AS CurrentZone,
    CASE 
        WHEN p.GravityScore >= 80 THEN 'High'
        WHEN p.GravityScore >= 50 THEN 'Medium'
        ELSE 'Low'
    END AS RecommendedZone,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    AVG(w.ProcessingTimeMinutes) AS AvgProcessingTime,
    COUNT(*) AS MovementCount
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE t.DateTime >= DATEADD(day, -90, GETDATE())
GROUP BY p.SKU, p.ProductName, p.GravityScore, w.GravityZone
HAVING p.GravityScore >= 60 AND w.GravityZone = 'Low' -- Misalignment
ORDER BY p.GravityScore DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify potential bottlenecks by time window
CREATE OR ALTER PROCEDURE sp_DetectBottlenecks
    @ThresholdDwellMinutes INT = 120,
    @ThresholdIdlePercent DECIMAL(5,2) = 0.15
AS
BEGIN
    SELECT 
        t.DateTime,
        t.HourOfDay,
        t.DayOfWeek,
        g.LocationName,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        AVG(CAST(f.IdleTimeMinutes AS FLOAT) / NULLIF((f.IdleTimeMinutes + (f.DistanceKM / 60)), 0)) AS IdlePercent,
        COUNT(DISTINCT w.OperationKey) AS WarehouseOps,
        COUNT(DISTINCT f.TripKey) AS FleetTrips,
        CASE 
            WHEN AVG(w.DwellTimeMinutes) > @ThresholdDwellMinutes THEN 'Warehouse Congestion'
            WHEN AVG(CAST(f.IdleTimeMinutes AS FLOAT) / NULLIF((f.IdleTimeMinutes + (f.DistanceKM / 60)), 0)) > @ThresholdIdlePercent THEN 'Fleet Idling'
            ELSE 'Normal'
        END AS BottleneckType
    FROM DimTime t
    LEFT JOIN FactWarehouseOperations w ON t.TimeKey = w.TimeKey
    LEFT JOIN FactFleetTrips f ON t.TimeKey = f.TimeKey
    LEFT JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    WHERE t.DateTime >= DATEADD(day, -7, GETDATE())
    GROUP BY t.DateTime, t.HourOfDay, t.DayOfWeek, g.LocationName
    HAVING AVG(w.DwellTimeMinutes) > @ThresholdDwellMinutes 
        OR AVG(CAST(f.IdleTimeMinutes AS FLOAT) / NULLIF((f.IdleTimeMinutes + (f.DistanceKM / 60)), 0)) > @ThresholdIdlePercent
    ORDER BY t.DateTime DESC;
END;
```

### Temporal Elasticity Simulation

```sql
-- Simulate impact of increased warehouse capacity
WITH CurrentState AS (
    SELECT 
        AVG(DwellTimeMinutes) AS BaselineDwell,
        AVG(ProcessingTimeMinutes) AS BaselineProcessing
    FROM FactWarehouseOperations
    WHERE TimeKey IN (SELECT TimeKey FROM DimTime WHERE DateTime >= DATEADD(day, -30, GETDATE()))
),
SimulatedState AS (
    SELECT 
        -- Assume 95% capacity reduces dwell by 30%
        AVG(DwellTimeMinutes) * 0.70 AS SimulatedDwell,
        AVG(ProcessingTimeMinutes) * 0.85 AS SimulatedProcessing
    FROM FactWarehouseOperations
    WHERE TimeKey IN (SELECT TimeKey FROM DimTime WHERE DateTime >= DATEADD(day, -30, GETDATE()))
)
SELECT 
    cs.BaselineDwell,
    ss.SimulatedDwell,
    (cs.BaselineDwell - ss.SimulatedDwell) AS DwellReduction,
    ((cs.BaselineDwell - ss.SimulatedDwell) / cs.BaselineDwell) * 100 AS PercentImprovement
FROM CurrentState cs, SimulatedState ss;
```

## Data Loading and ETL

### Incremental Load Pattern

```sql
CREATE OR ALTER PROCEDURE sp_IncrementalLoadWarehouseOps
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET @LastLoadDateTime = ISNULL(@LastLoadDateTime, DATEADD(hour, -1, GETDATE()));
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, GeographyKey, OperationType,
        QuantityUnits, DwellTimeMinutes, ProcessingTimeMinutes,
        ZoneID, GravityZone, EmployeeID, BatchID
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        src.OperationType,
        src.QuantityUnits,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime) AS DwellTimeMinutes,
        src.ProcessingTimeMinutes,
        src.ZoneID,
        src.GravityZone,
        src.EmployeeID,
        src.BatchID
    FROM ExternalDataSource.WarehouseOperationsStaging src
    INNER JOIN DimTime t ON DATEADD(MINUTE, (DATEDIFF(MINUTE, 0, src.StartTime) / 15) * 15, 0) = t.DateTime
    INNER JOIN DimProductGravity p ON src.SKU = p.SKU
    INNER JOIN DimGeography g ON src.LocationID = g.LocationID
    WHERE src.LastModified > @LastLoadDateTime;
    
    -- Update control table
    UPDATE ETLControlTable
    SET LastLoadDateTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
```

### External Data Integration (Weather API)

```sql
-- Create external table for weather data
CREATE EXTERNAL TABLE ExternalWeatherData (
    LocationID NVARCHAR(50),
    DateTime DATETIME,
    Temperature DECIMAL(5,2),
    Precipitation DECIMAL(5,2),
    WindSpeed DECIMAL(5,2),
    Visibility DECIMAL(5,2)
)
WITH (
    LOCATION = '/weather',
    DATA_SOURCE = WeatherAPIDataSource,
    FILE_FORMAT = JSONFileFormat
);

-- Correlate weather with delays
SELECT 
    f.TripKey,
    g.LocationName,
    t.DateTime,
    w.Temperature,
    w.Precipitation,
    f.IdleTimeMinutes,
    f.DelayReasonCode
FROM FactFleetTrips f
INNER JOIN DimGeography g ON f.OriginGeographyKey = g.GeographyKey
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
LEFT JOIN ExternalWeatherData w ON g.LocationID = w.LocationID AND t.DateTime = w.DateTime
WHERE f.TripStatus = 'Delayed' AND w.Precipitation > 10;
```

## Power BI Configuration

### DAX Measures for Cross-Fact Analysis

**Composite Delay Score**

```dax
CompositeDelayScore = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR AvgIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
RETURN
    (AvgDwell * 0.6) + (AvgIdle * 0.4)
```

**Fleet Utilization Rate**

```dax
FleetUtilizationRate = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKM]),
    SUM(FactFleetTrips[DistanceKM]) + (SUM(FactFleetTrips[IdleTimeMinutes]) * 0.5),
    0
)
```

**Warehouse Efficiency Index**

```dax
WarehouseEfficiencyIndex = 
VAR ExpectedTime = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[ProcessingTimeMinutes]),
        FactWarehouseOperations[GravityZone] = "High"
    )
VAR ActualTime = AVERAGE(FactWarehouseOperations[ProcessingTimeMinutes])
RETURN
    DIVIDE(ExpectedTime, ActualTime, 1) * 100
```

### Row-Level Security Setup

```dax
-- Create role "Regional Manager"
[DimGeography[Region]] = USERNAME()

-- Apply in Power BI: Modeling → Manage Roles → Create Role
-- Add filter: DimGeography[Region] = USERPRINCIPALNAME()
```

## Alerting and Automation

### SQL Server Agent Job for Threshold Alerts

```sql
CREATE OR ALTER PROCEDURE sp_SendThresholdAlerts
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check for high dwell times
    IF EXISTS (
        SELECT 1 FROM FactWarehouseOperations w
        INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
        WHERE t.DateTime >= DATEADD(hour, -1, GETDATE())
        AND w.DwellTimeMinutes > 180
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Warehouse dwell time exceeded 3 hours in the last hour.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_RECIPIENT}',
            @subject = 'LogiFleet Pulse: Warehouse Dwell Time Alert',
            @body = @AlertMessage;
    END;
    
    -- Check for excessive fleet idling
    IF EXISTS (
        SELECT 1 FROM FactFleetTrips f
        INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
        WHERE t.DateTime >= DATEADD(hour, -1, GETDATE())
        AND CAST(f.IdleTimeMinutes AS FLOAT) / NULLIF((f.IdleTimeMinutes + (f.DistanceKM / 60)), 0) > 0.20
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idling exceeded 20% of trip time in the last hour.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_RECIPIENT}',
            @subject = 'LogiFleet Pulse: Fleet Idling Alert',
            @body = @AlertMessage;
    END;
END;

-- Schedule job to run every 15 minutes
EXEC msdb.dbo.sp_add_job @job_name = 'LogiFleet_Threshold_Alerts';
EXEC msdb.dbo.sp_add_jobstep 
    @job_name = 'LogiFleet_Threshold_Alerts',
    @step_name = 'Check Thresholds',
    @command = 'EXEC sp_SendThresholdAlerts';
EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = 'Every15Minutes',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 15;
EXEC msdb.dbo.sp_attach_schedule 
    @job_name = 'LogiFleet_Threshold_Alerts',
    @schedule_name = 'Every15Minutes';
```

## Troubleshooting

### Performance Optimization

**Issue: Slow cross-fact queries**

```sql
-- Add columnstore index for analytical workloads
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouse_Columnstore
ON FactWarehouseOperations (TimeKey, ProductKey, GeographyKey, DwellTimeMinutes, ProcessingTimeMinutes);

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleet_Columnstore
ON FactFleetTrips (TimeKey, VehicleID, OriginGeographyKey, DestinationGeographyKey, IdleTimeMinutes);
```

**Issue: Power BI refresh taking too long**

```sql
-- Implement partitioning by month
CREATE PARTITION FUNCTION PF_Monthly (DATETIME)
AS RANGE RIGHT FOR VALUES (
    '2026-02-01', '2026-03-01', '2026-04-01', '2026-05-01', '2026-06-01'
);

CREATE PARTITION SCHEME PS_Monthly
AS PARTITION PF_Monthly ALL TO ([PRIMARY]);

-- Recreate fact tables on partition scheme
ALTER TABLE FactWarehouseOperations DROP CONSTRAINT PK_FactWarehouseOperations;
ALTER TABLE FactWarehouseOperations ADD CONSTRAINT PK_FactWarehouseOperations 
PRIMARY KEY (OperationKey, TimeKey) ON PS_Monthly(TimeKey);
```

### Data Quality Checks

```sql
-- Validate dimension integrity
SELECT 
    'Missing Geography Keys' AS Issue,
    COUNT(*) AS RecordCount
FROM FactWarehouseOperations w
LEFT JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
WHERE g.GeographyKey IS NULL

UNION ALL

SELECT 
    'Missing Product Keys',
    COUNT(*)
FROM FactWarehouseOperations w
LEFT JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE p.ProductKey IS NULL

UNION ALL

SELECT 
    'Negative Dwell Times',
    COUNT(*)
FROM FactWarehouseOperations
WHERE DwellTimeMinutes < 0;
```

### Connection Issues

**Power BI can't connect to SQL Server**

1. Verify firewall rules: SQL Server TCP port 1433 must be open
2. Check SQL Server authentication mode:

```sql
-- Enable mixed mode authentication
USE master;
GO
EXEC xp_instance_regwrite 
    N'HKEY_LOCAL_MACHINE', 
    N'Software\Microsoft\MSSQLServer\MSSQLServer',
    N'LoginMode', 
    REG_DWORD, 
    2;
-- Restart SQL Server service
```

3. Test connection string in Power BI:

```
Server=your-server.database.windows.net;Database=LogiFleetPulse;User ID=${SQL_USER};Password=${SQL_PASSWORD};Encrypt=True;
```

## Advanced Use Cases

### Machine Learning Integration (Azure ML)

```sql
-- Export training data for bottleneck prediction model
SELECT 
    t.HourOfDay,
    t.DayOfWeek,
    t.IsWeekend,
    g.Region,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    AVG(f.IdleTimeMinutes) AS AvgIdleTime,
    COUNT(w.OperationKey) AS OperationVolume,
    CASE 
        WHEN AVG(w.DwellTimeMinutes) > 120 OR AVG(f.IdleTimeMinutes) > 30 THEN 1
        ELSE 0
    END AS IsBottleneck
FROM DimTime t
LEFT JOIN FactWarehouseOperations w ON t.TimeKey = w.TimeKey
LEFT JOIN FactFleetTrips f ON t.TimeKey = f.TimeKey
LEFT JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
WHERE t.DateTime >= DATEADD(month, -6, GETDATE())
GROUP BY t.HourOfDay, t.DayOfWeek, t.IsWeekend, g.Region
ORDER BY t.HourOfDay, t.DayOfWeek;
```

### Real-Time Dashboard Auto-Refresh

Configure Power BI Service for automatic refresh:

1. Publish report to Power BI Service
2. Navigate to Dataset Settings
3. Configure scheduled refresh: Every 15 minutes (Premium capacity required)
4. Enable DirectQuery for real-time data:

```powerbi
// In Power BI Desktop, switch to DirectQuery mode
// File → Options → Current File → Data Load
// Select "DirectQuery" as connection mode
```

## Best Practices

1. **Always use parameterized queries** to prevent SQL injection
2. **Partition large fact tables** by month for better query performance
3. **Implement incremental refresh** in Power BI for historical data
4. **Use computed columns sparingly** — prefer measures in Power BI
5. **Regularly rebuild indexes** and update statistics:

```sql
-- Weekly maintenance job
EXEC sp_MSforeachtable 'ALTER INDEX ALL ON ? REBUILD';
EXEC sp_updatestats;
```

6. **Monitor query performance** with SQL Server Profiler or Query Store
7. **Backup Power BI reports** to version control (.pbix files)
8. **Document all custom DAX measures** in Power BI Description fields
