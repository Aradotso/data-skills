---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence platform with MS SQL Server data warehousing and Power BI dashboards for supply chain optimization
triggers:
  - set up logistics analytics dashboard
  - implement supply chain data warehouse
  - create power bi logistics reports
  - configure warehouse and fleet tracking
  - build multi-fact star schema for logistics
  - deploy logifleet pulse analytics
  - integrate supply chain KPI tracking
  - analyze warehouse and fleet operations data
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is an advanced logistics intelligence platform that combines:

- **Multi-fact star schema data warehouse** (MS SQL Server) linking warehouse operations, fleet telemetry, inventory, and external signals
- **Power BI dashboards** for real-time supply chain visualization
- **Cross-fact KPI harmonization** connecting warehouse metrics with fleet performance
- **Predictive analytics** for bottleneck detection and maintenance triage
- **Warehouse Gravity Zones™** for spatial optimization based on velocity and value

The platform ingests data from WMS, telematics/GPS, supplier portals, weather APIs, and order history to provide unified logistics intelligence.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, TMS/telematics, ERP systems

### Database Schema Deployment

```sql
-- 1. Create the LogiFleet database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- 2. Deploy dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTime DATETIME2 NOT NULL,
    MinuteBucket INT, -- 0-95 (15-min buckets per day)
    HourOfDay INT,
    DayOfWeek NVARCHAR(10),
    FiscalPeriod NVARCHAR(20),
    IsBusinessHour BIT,
    INDEX IX_DimTime_DateTime (DateTime)
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    NodeID NVARCHAR(50) UNIQUE NOT NULL,
    NodeName NVARCHAR(200),
    NodeType NVARCHAR(50), -- Warehouse, RoutePoint, CrossDock
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    INDEX IX_DimGeography_NodeID (NodeID)
);

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(100) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value / fragility
    VelocityClass NVARCHAR(20), -- FastMover, SlowMover, Dead
    ValueTier NVARCHAR(20), -- High, Medium, Low
    FragilityIndex DECIMAL(3,2),
    OptimalZone NVARCHAR(50),
    INDEX IX_DimProduct_SKU (SKU),
    INDEX IX_DimProduct_Gravity (GravityScore DESC)
);

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID NVARCHAR(50) UNIQUE NOT NULL,
    SupplierName NVARCHAR(200),
    LeadTimeAvg INT, -- days
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    ComplianceScore DECIMAL(3,2),
    RiskTier NVARCHAR(20), -- Low, Medium, High
    INDEX IX_DimSupplier_ID (SupplierID)
);

-- 3. Deploy fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType NVARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    OperationID NVARCHAR(100),
    Quantity INT,
    DwellTimeMinutes INT,
    CycleTimeMinutes DECIMAL(8,2),
    PickRate DECIMAL(8,2), -- units per hour
    ErrorFlag BIT,
    StorageZone NVARCHAR(50),
    INDEX IX_FactWH_Time (TimeKey),
    INDEX IX_FactWH_Product (ProductKey),
    INDEX IX_FactWH_DwellTime (DwellTimeMinutes DESC)
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID NVARCHAR(50),
    DriverID NVARCHAR(50),
    TripID NVARCHAR(100),
    DistanceKm DECIMAL(8,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(8,2),
    LoadWeightKg DECIMAL(10,2),
    OnTimeDelivery BIT,
    WeatherCondition NVARCHAR(50),
    MaintenanceFlag BIT,
    INDEX IX_FactFleet_Time (TimeKey),
    INDEX IX_FactFleet_Vehicle (VehicleID),
    INDEX IX_FactFleet_IdleTime (IdleTimeMinutes DESC)
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    TransferTimeMinutes INT,
    Quantity INT,
    BypassStorage BIT,
    INDEX IX_FactCD_Time (TimeKey),
    INDEX IX_FactCD_Geography (GeographyKey)
);
```

### Stored Procedures for Data Loading

```sql
-- Incremental time dimension population
CREATE PROCEDURE sp_PopulateTimeDimension
    @StartDate DATETIME2,
    @EndDate DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    
    WHILE @CurrentDateTime < @EndDate
    BEGIN
        INSERT INTO DimTime (DateTime, MinuteBucket, HourOfDay, DayOfWeek, FiscalPeriod, IsBusinessHour)
        VALUES (
            @CurrentDateTime,
            (DATEPART(HOUR, @CurrentDateTime) * 4) + (DATEPART(MINUTE, @CurrentDateTime) / 15),
            DATEPART(HOUR, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            CONCAT('FY', YEAR(@CurrentDateTime), 'Q', DATEPART(QUARTER, @CurrentDateTime)),
            CASE 
                WHEN DATEPART(HOUR, @CurrentDateTime) BETWEEN 8 AND 17 
                     AND DATEPART(WEEKDAY, @CurrentDateTime) NOT IN (1, 7)
                THEN 1 ELSE 0 
            END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

-- Calculate product gravity scores
CREATE PROCEDURE sp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    UPDATE DimProductGravity
    SET GravityScore = (
        -- Velocity weight: picks per day
        (SELECT COUNT(*) / NULLIF(DATEDIFF(DAY, MIN(t.DateTime), MAX(t.DateTime)), 0)
         FROM FactWarehouseOperations f
         JOIN DimTime t ON f.TimeKey = t.TimeKey
         WHERE f.ProductKey = DimProductGravity.ProductKey
           AND f.OperationType = 'Picking'
           AND t.DateTime >= DATEADD(DAY, -90, GETDATE())
        ) * 10.0 -- Velocity multiplier
        +
        -- Value tier multiplier
        CASE ValueTier
            WHEN 'High' THEN 50.0
            WHEN 'Medium' THEN 25.0
            ELSE 10.0
        END
    ) / NULLIF(FragilityIndex, 0);
    
    UPDATE DimProductGravity
    SET VelocityClass = CASE
        WHEN GravityScore >= 100 THEN 'FastMover'
        WHEN GravityScore >= 20 THEN 'MediumMover'
        WHEN GravityScore >= 5 THEN 'SlowMover'
        ELSE 'Dead'
    END;
END;
GO
```

### Configuration File Structure

```json
{
  "connections": {
    "sqlServer": {
      "server": "${SQL_SERVER_HOST}",
      "database": "LogiFleetPulse",
      "authentication": "integrated",
      "trustedConnection": true
    },
    "dataSources": {
      "wms": {
        "type": "odbc",
        "connectionString": "${WMS_CONNECTION_STRING}",
        "refreshIntervalMinutes": 15
      },
      "telematics": {
        "type": "rest",
        "endpoint": "${TELEMATICS_API_ENDPOINT}",
        "authHeader": "Bearer ${TELEMATICS_API_KEY}",
        "refreshIntervalMinutes": 5
      },
      "weatherApi": {
        "type": "rest",
        "endpoint": "https://api.weather.service/v2/current",
        "authHeader": "ApiKey ${WEATHER_API_KEY}"
      }
    }
  },
  "alerts": {
    "dwellTimeThresholdHours": 72,
    "fleetIdleThresholdPercent": 15,
    "deliveryAlertEmail": "${ALERT_EMAIL_ADDRESS}",
    "smtpServer": "${SMTP_SERVER}",
    "smtpPort": 587
  },
  "powerBi": {
    "workspaceId": "${POWERBI_WORKSPACE_ID}",
    "datasetRefreshSchedule": "0 */15 * * *"
  }
}
```

## Power BI Template Configuration

### Import and Connect

```powershell
# 1. Open Power BI Desktop
# 2. File > Import > Power BI Template (.pbit)
# 3. Navigate to LogiFleet_Pulse_Master.pbit

# 4. Enter connection parameters when prompted:
#    - SQL Server: your-server.database.windows.net
#    - Database: LogiFleetPulse
#    - Data Connectivity mode: DirectQuery (for real-time) or Import (for performance)
```

### Key DAX Measures

```dax
-- Cross-fact KPI: Dwell Time Impact on Fuel Cost
DwellTimeVsFuelCost = 
VAR AvgDwellTime = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        FactWarehouseOperations[OperationType] = "Picking"
    )
VAR AvgFuelPerTrip = 
    AVERAGE(FactFleetTrips[FuelConsumedLiters])
VAR Correlation = 
    -- Products with high dwell correlate with rushed shipments
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FactFleetTrips[IdleTimeMinutes] > 30,
        RELATEDTABLE(FactWarehouseOperations),
        FactWarehouseOperations[DwellTimeMinutes] > 4320 -- 72 hours
    )
RETURN
    IF(AvgDwellTime > 4320, AvgFuelPerTrip * 1.15, AvgFuelPerTrip)

-- Fleet Maintenance Triage Score
MaintenanceTriageScore = 
VAR TripValue = 
    SUMX(
        RELATEDTABLE(FactWarehouseOperations),
        FactWarehouseOperations[Quantity] * 
        RELATED(DimProductGravity[GravityScore])
    )
VAR VehicleIssues = 
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FactFleetTrips[MaintenanceFlag] = TRUE(),
        FactFleetTrips[VehicleID] = EARLIER(FactFleetTrips[VehicleID])
    )
RETURN
    (TripValue / 1000) * VehicleIssues * 10

-- Warehouse Gravity Zone Efficiency
GravityZoneEfficiency = 
DIVIDE(
    CALCULATE(
        SUM(FactWarehouseOperations[Quantity]),
        FILTER(
            FactWarehouseOperations,
            FactWarehouseOperations[StorageZone] = 
            RELATED(DimProductGravity[OptimalZone])
        )
    ),
    SUM(FactWarehouseOperations[Quantity]),
    0
) * 100

-- Predictive Bottleneck Index (simplified heuristic)
BottleneckIndex = 
VAR CurrentCapacity = 
    CALCULATE(
        DISTINCTCOUNT(FactWarehouseOperations[OperationID]),
        DimTime[IsBusinessHour] = TRUE()
    )
VAR HistoricalPeak = 
    CALCULATE(
        MAXX(
            SUMMARIZE(
                FactWarehouseOperations,
                DimTime[DateTime],
                "OpCount", COUNTROWS(FactWarehouseOperations)
            ),
            [OpCount]
        ),
        DATESINPERIOD(DimTime[DateTime], MAX(DimTime[DateTime]), -30, DAY)
    )
RETURN
    IF(CurrentCapacity > HistoricalPeak * 0.85, 1, 0) -- Flag if approaching capacity
```

## Key API Patterns & Usage

### Querying Cross-Fact Metrics

```sql
-- Find shipments delayed by weather originating from high-dwell zones
SELECT 
    ft.TripID,
    ft.VehicleID,
    geo_origin.NodeName AS OriginWarehouse,
    pg.SKU,
    pg.ProductName,
    fwo.DwellTimeMinutes / 60.0 AS DwellTimeHours,
    ft.WeatherCondition,
    ft.DurationMinutes - ft.ExpectedDurationMinutes AS DelayMinutes
FROM FactFleetTrips ft
JOIN DimGeography geo_origin ON ft.OriginKey = geo_origin.GeographyKey
JOIN FactWarehouseOperations fwo ON 
    fwo.GeographyKey = geo_origin.GeographyKey
    AND ABS(DATEDIFF(MINUTE, fwo.TimeKey, ft.TimeKey)) < 120 -- Within 2 hours
JOIN DimProductGravity pg ON fwo.ProductKey = pg.ProductKey
WHERE ft.WeatherCondition IN ('Rain', 'Snow', 'Fog')
  AND fwo.DwellTimeMinutes > 4320 -- 72 hours
  AND ft.OnTimeDelivery = 0
ORDER BY DelayMinutes DESC;

-- Warehouse gravity zone misalignment report
SELECT 
    pg.SKU,
    pg.ProductName,
    pg.VelocityClass,
    pg.OptimalZone,
    fwo.StorageZone AS CurrentZone,
    COUNT(*) AS PickCount,
    AVG(fwo.CycleTimeMinutes) AS AvgCycleTime
FROM FactWarehouseOperations fwo
JOIN DimProductGravity pg ON fwo.ProductKey = pg.ProductKey
WHERE fwo.OperationType = 'Picking'
  AND fwo.StorageZone <> pg.OptimalZone
  AND fwo.TimeKey IN (
      SELECT TimeKey FROM DimTime 
      WHERE DateTime >= DATEADD(DAY, -30, GETDATE())
  )
GROUP BY pg.SKU, pg.ProductName, pg.VelocityClass, pg.OptimalZone, fwo.StorageZone
HAVING AVG(fwo.CycleTimeMinutes) > 5.0
ORDER BY PickCount DESC;

-- Fleet idle time correlation with warehouse operations
SELECT 
    ft.VehicleID,
    geo_origin.NodeName AS WarehouseOrigin,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    AVG(fwo.CycleTimeMinutes) AS AvgWarehouseCycleTime,
    COUNT(DISTINCT ft.TripID) AS TripCount
FROM FactFleetTrips ft
JOIN DimGeography geo_origin ON ft.OriginKey = geo_origin.GeographyKey
JOIN FactWarehouseOperations fwo ON 
    fwo.GeographyKey = geo_origin.GeographyKey
    AND fwo.OperationType = 'Shipping'
    AND ABS(DATEDIFF(MINUTE, 
        (SELECT DateTime FROM DimTime WHERE TimeKey = fwo.TimeKey),
        (SELECT DateTime FROM DimTime WHERE TimeKey = ft.TimeKey)
    )) < 60 -- Trips starting within 1 hour of shipping operation
WHERE ft.IdleTimeMinutes > 0
GROUP BY ft.VehicleID, geo_origin.NodeName
HAVING AVG(ft.IdleTimeMinutes) > 20
ORDER BY AvgIdleTime DESC;
```

### Automated Alert Configuration

```sql
-- Create alert monitoring stored procedure
CREATE PROCEDURE sp_CheckAlertThresholds
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Alert 1: High dwell time
    INSERT INTO AlertLog (AlertType, Severity, Message, DetectedAt)
    SELECT 
        'HighDwellTime' AS AlertType,
        'High' AS Severity,
        CONCAT('SKU ', pg.SKU, ' at ', geo.NodeName, 
               ' has dwell time of ', fwo.DwellTimeMinutes / 60.0, ' hours') AS Message,
        GETDATE() AS DetectedAt
    FROM FactWarehouseOperations fwo
    JOIN DimProductGravity pg ON fwo.ProductKey = pg.ProductKey
    JOIN DimGeography geo ON fwo.GeographyKey = geo.GeographyKey
    JOIN DimTime t ON fwo.TimeKey = t.TimeKey
    WHERE fwo.DwellTimeMinutes > 4320
      AND t.DateTime >= DATEADD(HOUR, -1, GETDATE())
      AND NOT EXISTS (
          SELECT 1 FROM AlertLog al 
          WHERE al.AlertType = 'HighDwellTime' 
            AND al.Message LIKE CONCAT('%', pg.SKU, '%')
            AND al.DetectedAt >= DATEADD(HOUR, -24, GETDATE())
      );
    
    -- Alert 2: Fleet excessive idle time
    INSERT INTO AlertLog (AlertType, Severity, Message, DetectedAt)
    SELECT 
        'ExcessiveIdleTime' AS AlertType,
        'Medium' AS Severity,
        CONCAT('Vehicle ', ft.VehicleID, ' idle time ', 
               ft.IdleTimeMinutes, ' min (', 
               CAST((ft.IdleTimeMinutes * 100.0 / NULLIF(ft.DurationMinutes, 0)) AS DECIMAL(5,1)),
               '% of trip)') AS Message,
        GETDATE() AS DetectedAt
    FROM FactFleetTrips ft
    JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE (ft.IdleTimeMinutes * 100.0 / NULLIF(ft.DurationMinutes, 0)) > 15
      AND t.DateTime >= DATEADD(HOUR, -1, GETDATE())
      AND NOT EXISTS (
          SELECT 1 FROM AlertLog al 
          WHERE al.AlertType = 'ExcessiveIdleTime' 
            AND al.Message LIKE CONCAT('%', ft.VehicleID, '%')
            AND al.DetectedAt >= DATEADD(HOUR, -6, GETDATE())
      );
END;
GO

-- Schedule via SQL Agent Job
EXEC msdb.dbo.sp_add_job 
    @job_name = N'LogiFleetPulse_AlertMonitoring';

EXEC msdb.dbo.sp_add_jobstep 
    @job_name = N'LogiFleetPulse_AlertMonitoring',
    @step_name = N'CheckThresholds',
    @subsystem = N'TSQL',
    @command = N'EXEC sp_CheckAlertThresholds;',
    @database_name = N'LogiFleetPulse';

EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule 
    @job_name = N'LogiFleetPulse_AlertMonitoring',
    @schedule_name = N'Every15Minutes';

EXEC msdb.dbo.sp_add_jobserver 
    @job_name = N'LogiFleetPulse_AlertMonitoring';
```

## Common Patterns

### Incremental Data Loading

```sql
-- ETL pattern for warehouse operations (idempotent)
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRANSACTION;
    
    -- Merge new records from staging table
    MERGE FactWarehouseOperations AS target
    USING (
        SELECT 
            t.TimeKey,
            g.GeographyKey,
            p.ProductKey,
            s.SupplierKey,
            stg.OperationType,
            stg.OperationID,
            stg.Quantity,
            stg.DwellTimeMinutes,
            stg.CycleTimeMinutes,
            stg.PickRate,
            stg.ErrorFlag,
            stg.StorageZone
        FROM StagingWarehouseOps stg
        JOIN DimTime t ON stg.OperationDateTime = t.DateTime
        JOIN DimGeography g ON stg.WarehouseNodeID = g.NodeID
        JOIN DimProductGravity p ON stg.SKU = p.SKU
        LEFT JOIN DimSupplierReliability s ON stg.SupplierID = s.SupplierID
        WHERE stg.LoadedAt > @LastLoadDateTime
    ) AS source
    ON target.OperationID = source.OperationID
    WHEN MATCHED THEN
        UPDATE SET 
            target.Quantity = source.Quantity,
            target.DwellTimeMinutes = source.DwellTimeMinutes,
            target.CycleTimeMinutes = source.CycleTimeMinutes,
            target.PickRate = source.PickRate,
            target.ErrorFlag = source.ErrorFlag
    WHEN NOT MATCHED BY TARGET THEN
        INSERT (TimeKey, GeographyKey, ProductKey, SupplierKey, OperationType, 
                OperationID, Quantity, DwellTimeMinutes, CycleTimeMinutes, 
                PickRate, ErrorFlag, StorageZone)
        VALUES (source.TimeKey, source.GeographyKey, source.ProductKey, 
                source.SupplierKey, source.OperationType, source.OperationID, 
                source.Quantity, source.DwellTimeMinutes, source.CycleTimeMinutes, 
                source.PickRate, source.ErrorFlag, source.StorageZone);
    
    COMMIT TRANSACTION;
END;
GO
```

### Row-Level Security Setup

```sql
-- Enable RLS for geography-based access
CREATE SCHEMA Security;
GO

CREATE FUNCTION Security.fn_GeographyAccessPredicate(@GeographyKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS AccessResult
    WHERE @GeographyKey IN (
        SELECT g.GeographyKey
        FROM dbo.DimGeography g
        JOIN dbo.UserGeographyAccess uga ON g.GeographyKey = uga.GeographyKey
        WHERE uga.UserName = USER_NAME()
    )
    OR IS_MEMBER('LogisticsAdmin') = 1
);
GO

CREATE SECURITY POLICY GeographyAccessPolicy
ADD FILTER PREDICATE Security.fn_GeographyAccessPredicate(GeographyKey)
ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE Security.fn_GeographyAccessPredicate(OriginKey)
ON dbo.FactFleetTrips
WITH (STATE = ON);
GO
```

### Power BI Refresh via PowerShell

```powershell
# Install module if needed
# Install-Module -Name MicrosoftPowerBIMgmt

# Authenticate
$credential = Get-Credential
Connect-PowerBIServiceAccount -Credential $credential

# Trigger dataset refresh
$workspaceId = $env:POWERBI_WORKSPACE_ID
$datasetId = "your-dataset-id"

Invoke-PowerBIRestMethod `
    -Url "groups/$workspaceId/datasets/$datasetId/refreshes" `
    -Method Post `
    -Body "{}"

# Monitor refresh status
$refreshes = Invoke-PowerBIRestMethod `
    -Url "groups/$workspaceId/datasets/$datasetId/refreshes" `
    -Method Get | ConvertFrom-Json

$refreshes.value | Select-Object -First 1 | Format-List
```

## Troubleshooting

### Issue: Power BI Dashboard Not Refreshing

**Symptoms**: Dashboards show stale data despite SQL database updates.

**Solutions**:

```sql
-- 1. Check last refresh timestamp
SELECT 
    name AS DatasetName,
    refreshed_time AS LastRefresh,
    DATEDIFF(MINUTE, refreshed_time, GETDATE()) AS MinutesSinceRefresh
FROM sys.dm_pdw_nodes_db_partition_stats; -- For Azure SQL

-- 2. Verify DirectQuery connections
SELECT 
    session_id,
    login_name,
    program_name,
    last_request_start_time
FROM sys.dm_exec_sessions
WHERE program_name LIKE '%Power BI%'
ORDER BY last_request_start_time DESC;

-- 3. Check for blocking queries
SELECT 
    blocking_session_id,
    wait_type,
    wait_time,
    wait_resource
FROM sys.dm_exec_requests
WHERE blocking_session_id <> 0;
```

**Power BI Settings**:
- File → Options → Data Load → DirectQuery: Set timeout to 300 seconds
- Transform Data → Data Source Settings: Verify credentials are valid
- Home → Refresh: Use "Refresh Now" to force immediate update

### Issue: Slow Cross-Fact Queries

**Symptoms**: Queries joining multiple fact tables timeout or take >30 seconds.

**Solutions**:

```sql
-- 1. Add covering indexes
CREATE NONCLUSTERED INDEX IX_FactWH_CompositeKey
ON FactWarehouseOperations (TimeKey, GeographyKey, ProductKey)
INCLUDE (OperationType, DwellTimeMinutes, CycleTimeMinutes);

CREATE NONCLUSTERED INDEX IX_FactFleet_CompositeKey
ON FactFleetTrips (TimeKey, OriginKey, DestinationKey)
INCLUDE (VehicleID, IdleTimeMinutes, FuelConsumedLiters);

-- 2. Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;

-- 3. Use query hints for large joins
SELECT /*+ HASH JOIN, MAXDOP 4 */
    fwo.OperationID,
    ft.TripID,
    fwo.DwellTimeMinutes,
    ft.IdleTimeMinutes
FROM FactWarehouseOperations fwo WITH (NOLOCK)
JOIN FactFleetTrips ft WITH (NOLOCK) 
    ON ABS(fwo.TimeKey - ft.TimeKey) < 4 -- 1 hour tolerance
WHERE fwo.TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime); -- Last day only
```

### Issue: Gravity Score Calculation Errors

**Symptoms**: `DimProductGravity.GravityScore` returns NULL or negative values.

**Solutions**:

```sql
-- Validate input data
SELECT 
    SKU,
    VelocityClass,
    ValueTier,
    FragilityIndex,
    GravityScore,
    CASE 
        WHEN FragilityIndex = 0 THEN 'Fragility is zero'
        WHEN ValueTier IS NULL THEN 'Value tier missing'
        WHEN NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations 
            WHERE ProductKey = DimProductGravity.ProductKey
        ) THEN 'No warehouse operations'
        ELSE 'Valid'
    END AS ValidationStatus
FROM DimProductGravity
WHERE GravityScore IS NULL OR GravityScore < 0;

-- Recalculate with safe defaults
UPDATE DimProductGravity
SET FragilityIndex = 1.0
WHERE FragilityIndex = 0 OR FragilityIndex IS NULL;

EXEC sp_UpdateProductGravityScores;
```

### Issue: Alert Spam

**Symptoms**: Receiving duplicate alerts or alerts for resolved issues.

**Solutions**:

```sql
-- Add cooldown period to alert logic (modify sp_CheckAlertThresholds)
-- Already included in example above via NOT EXISTS clause

-- Create alert suppression table
CREATE TABLE AlertSuppressions (
    SuppressionID INT PRIMARY KEY IDENTITY(1,1),
    AlertType NVARCHAR(50),
    EntityID NVARCHAR(100), -- VehicleID, SKU, etc.
    SuppressedUntil DATETIME2,
    Reason NVARCHAR(500)
);

-- Modify alert procedure to check suppressions
ALTER PROCEDURE sp_CheckAlertThresholds
AS
BEGIN
    -- Add to WHERE clauses:
    AND NOT EXISTS (
        SELECT
