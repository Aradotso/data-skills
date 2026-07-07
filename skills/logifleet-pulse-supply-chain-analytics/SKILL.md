---
name: logifleet-pulse-supply-chain-analytics
description: Multi-fact SQL data warehouse and Power BI dashboard system for logistics intelligence, fleet optimization, and warehouse operations analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure warehouse and fleet data warehouse"
  - "implement logistics KPI dashboards in Power BI"
  - "build multi-fact star schema for supply chain"
  - "create fleet telemetry and warehouse operations analytics"
  - "deploy LogiFleet Pulse SQL schema"
  - "optimize logistics data model with gravity zones"
  - "implement cross-fact supply chain reporting"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is an advanced MS SQL Server data warehousing solution with Power BI visualization for unified logistics intelligence. It combines warehouse operations, fleet telemetry, inventory management, and external data sources into a multi-fact star schema that enables cross-modal supply chain analytics and predictive bottleneck detection.

## What It Does

- **Multi-Fact Star Schema**: Links warehouse operations, fleet trips, cross-dock transfers, and supplier data through time-aware dimensions
- **Warehouse Gravity Zones**: Spatial optimization based on pick frequency, item value, and replenishment lead time
- **Fleet Triage Engine**: Predictive maintenance prioritization weighted by revenue impact
- **Cross-Fact KPI Harmonization**: Query relationships between inventory turnover and fuel consumption, dwell time and route efficiency
- **Real-Time Dashboards**: Power BI reports with 15-minute refresh intervals and role-based security

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SSMS (SQL Server Management Studio) or Azure Data Studio

### Database Setup

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema**:
```sql
-- Connect to your SQL Server instance via SSMS
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the schema creation script
-- (Assumes schema.sql exists in repository)
:r schema.sql
GO
```

3. **Configure data connections** (update `config_sample.json`):
```json
{
  "sql_server": {
    "server": "localhost\\SQLEXPRESS",
    "database": "LogiFleetPulse",
    "authentication": "Windows"
  },
  "data_sources": {
    "wms_connection_string": "USE_ENV_VAR:WMS_CONN_STRING",
    "telemetry_api_endpoint": "USE_ENV_VAR:TELEMETRY_API_URL",
    "telemetry_api_key": "USE_ENV_VAR:TELEMETRY_API_KEY"
  },
  "refresh_interval_minutes": 15
}
```

4. **Set environment variables**:
```bash
# Windows PowerShell
$env:WMS_CONN_STRING="Server=wms.company.local;Database=WMS;Trusted_Connection=True;"
$env:TELEMETRY_API_URL="https://fleet-telemetry.company.com/api/v1"
$env:TELEMETRY_API_KEY="your-api-key-here"

# Linux/macOS
export WMS_CONN_STRING="Server=wms.company.local;Database=WMS;Trusted_Connection=True;"
export TELEMETRY_API_URL="https://fleet-telemetry.company.com/api/v1"
export TELEMETRY_API_KEY="your-api-key-here"
```

## Core Data Model

### Fact Tables

The system uses multiple fact tables for different operational domains:

```sql
-- FactWarehouseOperations: Core warehouse activity metrics
CREATE TABLE FactWarehouseOperations (
    OperationID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProduct(ProductKey),
    LocationKey INT FOREIGN KEY REFERENCES DimLocation(LocationKey),
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity INT,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes DECIMAL(10,2),
    OperatorID INT,
    CostUSD DECIMAL(10,2)
);

-- FactFleetTrips: Fleet telemetry and route performance
CREATE TABLE FactFleetTrips (
    TripID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    RouteKey INT FOREIGN KEY REFERENCES DimRoute(RouteKey),
    OriginLocationKey INT FOREIGN KEY REFERENCES DimLocation(LocationKey),
    DestinationLocationKey INT FOREIGN KEY REFERENCES DimLocation(LocationKey),
    DistanceMiles DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedGallons DECIMAL(10,3),
    LoadWeightLbs INT,
    OnTimeFlag BIT
);

-- FactCrossDock: Transfer operations between inbound/outbound
CREATE TABLE FactCrossDock (
    CrossDockID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProduct(ProductKey),
    InboundTripKey INT,
    OutboundTripKey INT,
    TransferTimeMinutes INT,
    Quantity INT
);
```

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME,
    Date DATE,
    TimeOfDay TIME,
    Hour INT,
    Minute INT,
    DayOfWeek VARCHAR(20),
    DayOfMonth INT,
    Month INT,
    Quarter INT,
    Year INT,
    FiscalPeriod VARCHAR(20),
    IsWeekend BIT,
    IsHoliday BIT
);

-- DimProductGravity: Products with calculated gravity scores
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    PickFrequencyScore DECIMAL(5,2), -- 0-100
    ValueScore DECIMAL(5,2), -- 0-100
    FragilityScore DECIMAL(5,2), -- 0-100
    GravityScore AS (PickFrequencyScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2),
    RecommendedZone VARCHAR(50) -- 'High', 'Medium', 'Low'
);

-- DimLocation: Hierarchical warehouse/route locations
CREATE TABLE DimLocation (
    LocationKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) UNIQUE,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'Distribution Center', 'Route Node'
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);
```

## Key SQL Procedures

### Incremental Data Load

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations from WMS source
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, LocationKey, OperationType, 
        Quantity, DwellTimeMinutes, ProcessingTimeMinutes, 
        OperatorID, CostUSD
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        l.LocationKey,
        wms.OperationType,
        wms.Quantity,
        DATEDIFF(MINUTE, wms.StartTime, wms.EndTime) AS DwellTimeMinutes,
        wms.ProcessingTimeMinutes,
        wms.OperatorID,
        wms.CostUSD
    FROM ExternalWMS.dbo.Operations wms
    INNER JOIN DimTime t ON t.FullDateTime = DATEADD(MINUTE, 
        (DATEDIFF(MINUTE, 0, wms.StartTime) / 15) * 15, 0)
    INNER JOIN DimProductGravity p ON p.SKU = wms.SKU
    INNER JOIN DimLocation l ON l.LocationID = wms.LocationID
    WHERE wms.LastModified > @LastLoadDateTime;
    
    -- Update metadata table with last load time
    UPDATE LoadMetadata 
    SET LastLoadDateTime = GETDATE() 
    WHERE TableName = 'FactWarehouseOperations';
END;
GO
```

### Calculate Warehouse Gravity Scores

```sql
-- Procedure to recalculate product gravity scores based on recent activity
CREATE PROCEDURE usp_RecalculateProductGravity
    @DaysToAnalyze INT = 90
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            COUNT(*) AS PickCount,
            AVG(wo.ProcessingTimeMinutes) AS AvgProcessingTime,
            SUM(wo.Quantity * p.ValueScore) AS TotalValue
        FROM FactWarehouseOperations wo
        INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.Date >= DATEADD(DAY, -@DaysToAnalyze, GETDATE())
            AND wo.OperationType = 'Picking'
        GROUP BY p.ProductKey
    ),
    ScoredProducts AS (
        SELECT 
            ProductKey,
            -- Normalize to 0-100 scale
            (PickCount * 1.0 / MAX(PickCount) OVER ()) * 100 AS PickFrequencyScore,
            (TotalValue * 1.0 / MAX(TotalValue) OVER ()) * 100 AS ValueScore
        FROM ProductMetrics
    )
    UPDATE p
    SET 
        p.PickFrequencyScore = sp.PickFrequencyScore,
        p.ValueScore = sp.ValueScore,
        p.RecommendedZone = CASE 
            WHEN (sp.PickFrequencyScore * 0.5 + sp.ValueScore * 0.3) > 70 THEN 'High'
            WHEN (sp.PickFrequencyScore * 0.5 + sp.ValueScore * 0.3) > 40 THEN 'Medium'
            ELSE 'Low'
        END
    FROM DimProductGravity p
    INNER JOIN ScoredProducts sp ON p.ProductKey = sp.ProductKey;
END;
GO
```

### Cross-Fact KPI Query

```sql
-- Query linking warehouse dwell time with fleet idle time
CREATE VIEW vw_DwellTimeFleetImpact AS
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    l.LocationName AS WarehouseLocation,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleTimeMinutes,
    AVG(ft.FuelConsumedGallons) AS AvgFuelConsumed,
    COUNT(DISTINCT ft.TripID) AS TotalTrips,
    -- Cross-fact efficiency metric
    (AVG(wo.DwellTimeMinutes) * AVG(ft.IdleTimeMinutes)) / 
        NULLIF(AVG(ft.DurationMinutes), 0) AS EfficiencyPenaltyScore
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN DimLocation l ON wo.LocationKey = l.LocationKey
INNER JOIN FactFleetTrips ft ON ft.OriginLocationKey = l.LocationKey
    AND ft.TimeKey BETWEEN wo.TimeKey AND wo.TimeKey + 96 -- 24 hours window
WHERE wo.OperationType = 'Shipping'
GROUP BY p.SKU, p.ProductName, p.GravityScore, l.LocationName;
GO
```

## Power BI Integration

### Connect Power BI to SQL Server

1. Open `LogiFleet_Pulse_Master.pbit` template
2. When prompted, enter SQL Server connection details:
   - **Server**: `localhost\SQLEXPRESS` or your server name
   - **Database**: `LogiFleetPulse`
   - **Data Connectivity Mode**: `Import` (for faster dashboards) or `DirectQuery` (for real-time)

### Key DAX Measures

```dax
// Total Dwell Time with Time Intelligence
Total Dwell Time = 
SUM(FactWarehouseOperations[DwellTimeMinutes])

Dwell Time vs PY = 
VAR CurrentDwell = [Total Dwell Time]
VAR PriorYearDwell = CALCULATE(
    [Total Dwell Time],
    DATEADD(DimTime[Date], -1, YEAR)
)
RETURN
DIVIDE(CurrentDwell - PriorYearDwell, PriorYearDwell, 0)

// Fleet Efficiency Score
Fleet Efficiency % = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

// Cross-Fact: Cost per Mile by Warehouse Dwell
Cost Per Mile by Dwell = 
VAR TotalFleetCost = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[FuelConsumedGallons] * 3.5 // Assume $3.50/gallon
    )
VAR TotalMiles = SUM(FactFleetTrips[DistanceMiles])
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
RETURN
IF(
    AvgDwell > 120, // High dwell penalty
    DIVIDE(TotalFleetCost, TotalMiles, 0) * 1.15,
    DIVIDE(TotalFleetCost, TotalMiles, 0)
)

// Predictive Bottleneck Index
Bottleneck Risk Score = 
VAR HighGravityProducts = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProductGravity[RecommendedZone] = "High",
        FactWarehouseOperations[DwellTimeMinutes] > 180
    )
VAR FleetUtilization = [Fleet Efficiency %]
RETURN
(HighGravityProducts * 10) + ((100 - FleetUtilization) * 5)
```

### Row-Level Security (RLS)

```dax
// Role: Regional Manager (sees only their region)
[Region] = USERNAME()

// Role: Fleet Supervisor (sees only assigned vehicles)
VAR UserEmail = USERPRINCIPALNAME()
VAR AssignedVehicles = 
    FILTER(
        DimVehicle,
        DimVehicle[SupervisorEmail] = UserEmail
    )
RETURN
DimVehicle[VehicleKey] IN AssignedVehicles
```

## Automated Alerting

### SQL Server Agent Job for Threshold Monitoring

```sql
-- Stored procedure for KPI threshold alerts
CREATE PROCEDURE usp_MonitorKPIThresholds
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check for excessive fleet idle time
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips ft
        INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
        WHERE t.Date = CAST(GETDATE() AS DATE)
        GROUP BY ft.VehicleKey
        HAVING AVG(CAST(ft.IdleTimeMinutes AS FLOAT) / ft.DurationMinutes) > 0.15
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeded 15% threshold today.';
        
        -- Send email via Database Mail
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'USE_ENV_VAR:ALERT_EMAIL_RECIPIENTS',
            @subject = 'LogiFleet Pulse: Fleet Idle Time Alert',
            @body = @AlertMessage;
    END;
    
    -- Check for warehouse dwell time spikes
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
        WHERE t.Date = CAST(GETDATE() AS DATE)
            AND p.RecommendedZone = 'High'
            AND wo.DwellTimeMinutes > 180
    )
    BEGIN
        SET @AlertMessage = 'ALERT: High-gravity products exceeding 3-hour dwell time.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'USE_ENV_VAR:ALERT_EMAIL_RECIPIENTS',
            @subject = 'LogiFleet Pulse: Warehouse Dwell Time Alert',
            @body = @AlertMessage;
    END;
END;
GO

-- Schedule the job to run every hour
USE msdb;
GO

EXEC sp_add_job @job_name = 'LogiFleet_KPI_Monitoring';

EXEC sp_add_jobstep
    @job_name = 'LogiFleet_KPI_Monitoring',
    @step_name = 'Check Thresholds',
    @subsystem = 'TSQL',
    @database_name = 'LogiFleetPulse',
    @command = 'EXEC usp_MonitorKPIThresholds;';

EXEC sp_add_schedule
    @schedule_name = 'Hourly',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 8, -- Hours
    @freq_subday_interval = 1;

EXEC sp_attach_schedule
    @job_name = 'LogiFleet_KPI_Monitoring',
    @schedule_name = 'Hourly';

EXEC sp_add_jobserver
    @job_name = 'LogiFleet_KPI_Monitoring';
GO
```

## Common Patterns

### Pattern 1: Initial Data Load from WMS

```sql
-- One-time bulk load from external WMS system
BULK INSERT FactWarehouseOperations
FROM 'USE_ENV_VAR:WMS_EXPORT_PATH\operations.csv'
WITH (
    FIRSTROW = 2,
    FIELDTERMINATOR = ',',
    ROWTERMINATOR = '\n',
    TABLOCK
);

-- Rebuild indexes after bulk insert
ALTER INDEX ALL ON FactWarehouseOperations REBUILD;
```

### Pattern 2: Incremental Refresh Setup

```sql
-- Create staging table for incremental loads
CREATE TABLE Staging_WarehouseOperations (
    OperationID INT,
    Timestamp DATETIME,
    SKU VARCHAR(50),
    LocationID VARCHAR(50),
    OperationType VARCHAR(50),
    Quantity INT,
    DwellTimeMinutes INT
);

-- Merge from staging to production
MERGE FactWarehouseOperations AS target
USING Staging_WarehouseOperations AS source
ON target.OperationID = source.OperationID
WHEN MATCHED AND target.DwellTimeMinutes <> source.DwellTimeMinutes THEN
    UPDATE SET DwellTimeMinutes = source.DwellTimeMinutes
WHEN NOT MATCHED BY TARGET THEN
    INSERT (TimeKey, ProductKey, LocationKey, OperationType, Quantity, DwellTimeMinutes)
    VALUES (
        (SELECT TimeKey FROM DimTime WHERE FullDateTime = source.Timestamp),
        (SELECT ProductKey FROM DimProductGravity WHERE SKU = source.SKU),
        (SELECT LocationKey FROM DimLocation WHERE LocationID = source.LocationID),
        source.OperationType,
        source.Quantity,
        source.DwellTimeMinutes
    );
```

### Pattern 3: Time-Phased Dimension Snapshots

```sql
-- Generate 15-minute time dimension for a year
DECLARE @StartDate DATETIME = '2026-01-01 00:00:00';
DECLARE @EndDate DATETIME = '2026-12-31 23:45:00';
DECLARE @CurrentTime DATETIME = @StartDate;

WHILE @CurrentTime <= @EndDate
BEGIN
    INSERT INTO DimTime (
        TimeKey, FullDateTime, Date, TimeOfDay, Hour, Minute,
        DayOfWeek, DayOfMonth, Month, Quarter, Year, IsWeekend
    )
    VALUES (
        CONVERT(INT, FORMAT(@CurrentTime, 'yyyyMMddHHmm')),
        @CurrentTime,
        CAST(@CurrentTime AS DATE),
        CAST(@CurrentTime AS TIME),
        DATEPART(HOUR, @CurrentTime),
        DATEPART(MINUTE, @CurrentTime),
        DATENAME(WEEKDAY, @CurrentTime),
        DAY(@CurrentTime),
        MONTH(@CurrentTime),
        DATEPART(QUARTER, @CurrentTime),
        YEAR(@CurrentTime),
        CASE WHEN DATEPART(WEEKDAY, @CurrentTime) IN (1, 7) THEN 1 ELSE 0 END
    );
    
    SET @CurrentTime = DATEADD(MINUTE, 15, @CurrentTime);
END;
```

### Pattern 4: External API Integration (Fleet Telemetry)

```sql
-- Enable external scripts (requires SQL Server 2017+)
EXEC sp_configure 'external scripts enabled', 1;
RECONFIGURE;
GO

-- Python script to fetch telemetry and load into SQL
CREATE PROCEDURE usp_LoadFleetTelemetry
AS
BEGIN
    EXEC sp_execute_external_script
        @language = N'Python',
        @script = N'
import os
import requests
import pandas as pd
from sqlalchemy import create_engine

api_url = os.environ["TELEMETRY_API_URL"]
api_key = os.environ["TELEMETRY_API_KEY"]

headers = {"Authorization": f"Bearer {api_key}"}
response = requests.get(f"{api_url}/trips/latest", headers=headers)
data = response.json()

df = pd.DataFrame(data)
engine = create_engine(os.environ["SQL_CONNECTION_STRING"])
df.to_sql("Staging_FleetTrips", engine, if_exists="append", index=False)
';
END;
GO
```

## Troubleshooting

### Issue: Power BI Refresh Fails

**Symptoms**: "Data source error" or "Unable to connect"

**Solutions**:
```sql
-- Check SQL Server is allowing remote connections
EXEC sp_configure 'remote access', 1;
RECONFIGURE;

-- Verify firewall allows port 1433
-- Windows: netsh advfirewall firewall add rule name="SQL Server" dir=in action=allow protocol=TCP localport=1433

-- Test connection from Power BI machine
sqlcmd -S YourServerName -U YourUser -P YourPassword -d LogiFleetPulse -Q "SELECT TOP 1 * FROM FactWarehouseOperations"
```

### Issue: Slow Query Performance

**Symptoms**: Dashboards take >30 seconds to load

**Solutions**:
```sql
-- Create columnstore indexes on fact tables (SQL Server 2016+)
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouse_Columnstore
ON FactWarehouseOperations (TimeKey, ProductKey, LocationKey, Quantity, DwellTimeMinutes);

-- Partition large fact tables by date
CREATE PARTITION FUNCTION pf_MonthlyPartition (INT)
AS RANGE RIGHT FOR VALUES (202601, 202602, 202603, ..., 202612);

CREATE PARTITION SCHEME ps_MonthlyPartition
AS PARTITION pf_MonthlyPartition ALL TO ([PRIMARY]);

-- Rebuild table on partition scheme
CREATE TABLE FactWarehouseOperations_New (
    OperationID INT IDENTITY(1,1),
    TimeKey INT,
    -- ... other columns
) ON ps_MonthlyPartition(TimeKey);

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
```

### Issue: Gravity Score Not Updating

**Symptoms**: `RecommendedZone` stays outdated despite running `usp_RecalculateProductGravity`

**Solutions**:
```sql
-- Check if recent data exists
SELECT TOP 10 * 
FROM FactWarehouseOperations 
ORDER BY TimeKey DESC;

-- Verify time dimension is populated
SELECT COUNT(*) FROM DimTime WHERE Date >= DATEADD(DAY, -90, GETDATE());

-- Run procedure with debug output
ALTER PROCEDURE usp_RecalculateProductGravity
    @DaysToAnalyze INT = 90
AS
BEGIN
    SET NOCOUNT OFF; -- Show affected rows
    
    -- Add debug SELECT before UPDATE
    SELECT 
        p.SKU,
        p.PickFrequencyScore AS OldScore,
        sp.PickFrequencyScore AS NewScore
    FROM DimProductGravity p
    INNER JOIN (
        SELECT 
            ProductKey,
            (PickCount * 1.0 / MAX(PickCount) OVER ()) * 100 AS PickFrequencyScore
        FROM (
            SELECT ProductKey, COUNT(*) AS PickCount
            FROM FactWarehouseOperations
            WHERE OperationType = 'Picking'
            GROUP BY ProductKey
        ) metrics
    ) sp ON p.ProductKey = sp.ProductKey;
    
    -- Existing UPDATE logic...
END;
```

### Issue: Row-Level Security Not Working

**Symptoms**: Users see all data despite RLS filters

**Solutions**:
```sql
-- Verify RLS is enabled on table
SELECT 
    OBJECT_NAME(t.object_id) AS TableName,
    p.name AS PolicyName,
    pp.type_desc AS PredicateType
FROM sys.security_policies p
INNER JOIN sys.security_predicates pp ON p.object_id = pp.object_id
INNER JOIN sys.tables t ON pp.target_object_id = t.object_id;

-- Create and apply RLS policy
CREATE FUNCTION fn_SecurityPredicate_Region(@Region VARCHAR(100))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS AccessPredicate
WHERE @Region = USER_NAME();
GO

CREATE SECURITY POLICY RegionalAccessPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate_Region(Region)
ON dbo.DimLocation
WITH (STATE = ON);
GO

-- Test RLS
EXECUTE AS USER = 'NorthAmerica';
SELECT * FROM DimLocation; -- Should only see North America locations
REVERT;
```

## Configuration Reference

### Required Environment Variables

```bash
# SQL Server Connection
SQL_SERVER=localhost\SQLEXPRESS
SQL_DATABASE=LogiFleetPulse
SQL_AUTH_MODE=Windows  # or 'SQL'
SQL_USERNAME=  # Only if SQL auth
SQL_PASSWORD=  # Only if SQL auth

# External Data Sources
WMS_CONN_STRING=Server=wms.local;Database=WMS;Trusted_Connection=True;
TELEMETRY_API_URL=https://api.fleet-telemetry.com/v1
TELEMETRY_API_KEY=your-api-key
WMS_EXPORT_PATH=\\fileserver\wms-exports

# Alerting
ALERT_EMAIL_RECIPIENTS=logistics@company.com;ops@company.com
DBMAIL_PROFILE_NAME=LogiFleetAlerts

# Thresholds
IDLE_TIME_THRESHOLD_PCT=0.15
DWELL_TIME_THRESHOLD_MINUTES=180
HIGH_GRAVITY_THRESHOLD=70
```

### Power BI Refresh Schedule

Configure in Power BI Service (app.powerbi.com):
1. Navigate to workspace → Dataset → Settings
2. **Scheduled refresh**: Every 15 minutes (Premium) or 8x daily (Pro)
3. **Gateway connection**: Install on-premises data gateway if SQL Server is not cloud-accessible
4. **Credentials**: Service account with read-only access to `LogiFleetPulse` database

## Best Practices

1. **Index Strategy**: Always create covering indexes on frequently joined columns (TimeKey, ProductKey, LocationKey)
2. **Partitioning**: Partition fact tables by month once they exceed 50M rows
3. **Incremental Loads**: Use watermark tables to track last successful load timestamp
4. **DAX Optimization**: Use variables to avoid recalculating measures multiple times
5. **RLS Testing**: Always test security roles with `EXECUTE AS USER` before deploying
6. **Backup**: Schedule daily full backups of SQL database before overnight ETL runs
