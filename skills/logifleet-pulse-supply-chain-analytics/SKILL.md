---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for logistics, fleet management, and supply chain KPI analytics with multi-fact star schema
triggers:
  - set up logistics data warehouse
  - create supply chain analytics dashboard
  - implement fleet management reporting
  - build warehouse operations star schema
  - configure power bi logistics dashboard
  - analyze cross-modal supply chain metrics
  - deploy logifleet pulse analytics
  - implement warehouse gravity zones
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

LogiFleet Pulse is an advanced logistics intelligence platform combining MS SQL Server data warehousing with Power BI visualization. It implements a multi-fact star schema architecture to unify warehouse operations, fleet telemetry, inventory management, and supply chain KPIs into a single semantic layer.

**Core Capabilities:**
- Multi-fact star schema with time-phased dimensions
- Cross-fact KPI harmonization (warehouse + fleet + inventory)
- Real-time logistics dashboards (15-minute refresh)
- Warehouse Gravity Zones™ spatial optimization
- Predictive bottleneck detection
- Fleet maintenance triage engine
- Role-based access control with row-level security

**Primary Language:** SQL (T-SQL for MS SQL Server)
**Visualization:** Power BI Desktop (.pbit templates)

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- SQL Server Management Studio (SSMS) 18+
- Power BI Desktop (latest version)
- Appropriate permissions: `CREATE DATABASE`, `CREATE TABLE`, `CREATE PROCEDURE`

### Deployment Steps

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**
```sql
-- Connect to your SQL Server instance via SSMS
-- Open and execute: schema/01_create_database.sql

USE master;
GO

CREATE DATABASE LogiFleetPulse
ON PRIMARY 
(
    NAME = LogiFleetPulse_Data,
    FILENAME = 'C:\SQLData\LogiFleetPulse.mdf',
    SIZE = 1GB,
    MAXSIZE = 100GB,
    FILEGROWTH = 512MB
)
LOG ON 
(
    NAME = LogiFleetPulse_Log,
    FILENAME = 'C:\SQLData\LogiFleetPulse_log.ldf',
    SIZE = 256MB,
    MAXSIZE = 10GB,
    FILEGROWTH = 128MB
);
GO

ALTER DATABASE LogiFleetPulse SET RECOVERY SIMPLE;
GO
```

3. **Create core schema objects:**
```sql
-- Execute in sequence:
-- schema/02_create_dimensions.sql
-- schema/03_create_facts.sql
-- schema/04_create_views.sql
-- schema/05_create_stored_procedures.sql
```

4. **Configure data connections:**
```bash
# Copy and customize configuration
cp config_sample.json config.json

# Edit config.json with your connection strings
# Set environment variables for sensitive data:
export SQL_SERVER_HOST="your-server.database.windows.net"
export SQL_SERVER_USER="logifleet_admin"
export SQL_SERVER_PASSWORD="${SQL_SERVER_PASSWORD}"
export WMS_API_KEY="${WMS_API_KEY}"
export FLEET_API_KEY="${FLEET_API_KEY}"
```

5. **Import Power BI template:**
```
- Open Power BI Desktop
- File → Import → Power BI Template
- Select: powerbi/LogiFleet_Pulse_Master.pbit
- Enter SQL Server connection details when prompted
```

## Core Schema Architecture

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE dbo.DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    IsWeekend BIT,
    FiscalPeriod VARCHAR(10),
    INDEX IX_FullDateTime NONCLUSTERED (FullDateTime)
);

-- DimGeography: Hierarchical location dimension
CREATE TABLE dbo.DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200),
    LocationType VARCHAR(20), -- 'Warehouse', 'Route Node', 'Customer Site'
    ParentLocationID VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    INDEX IX_LocationID NONCLUSTERED (LocationID)
);

-- DimProductGravity: Product classification with gravity score
CREATE TABLE dbo.DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200),
    Category VARCHAR(100),
    VelocityScore DECIMAL(5,2), -- Pick frequency weight
    ValueScore DECIMAL(5,2), -- Monetary value weight
    FragilityScore DECIMAL(5,2), -- Handling care weight
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    OptimalZone VARCHAR(20), -- 'High', 'Medium', 'Low' gravity
    INDEX IX_SKU NONCLUSTERED (SKU),
    INDEX IX_GravityScore NONCLUSTERED (GravityScore DESC)
);

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE dbo.DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200),
    LeadTimeMean INT, -- Days
    LeadTimeStdDev DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,2),
    ComplianceScore DECIMAL(3,2), -- 0-1 scale
    INDEX IX_SupplierID NONCLUSTERED (SupplierID)
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity tracking
CREATE TABLE dbo.FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity INT,
    DwellTimeMinutes INT, -- Time since last movement
    CycleTimeSeconds INT, -- Operation completion time
    OperatorID VARCHAR(50),
    BatchNumber VARCHAR(50),
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES dbo.DimTime(TimeKey),
    CONSTRAINT FK_WH_Geography FOREIGN KEY (GeographyKey) REFERENCES dbo.DimGeography(GeographyKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES dbo.DimProductGravity(ProductKey),
    INDEX IX_TimeGeo NONCLUSTERED (TimeKey, GeographyKey),
    INDEX IX_Product NONCLUSTERED (ProductKey)
) WITH (DATA_COMPRESSION = PAGE);

-- FactFleetTrips: Vehicle telemetry and route performance
CREATE TABLE dbo.FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DistanceKM DECIMAL(8,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelLiters DECIMAL(6,2),
    LoadWeightKG DECIMAL(8,2),
    DriverID VARCHAR(50),
    WeatherCondition VARCHAR(50),
    DelayMinutes INT DEFAULT 0,
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES dbo.DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES dbo.DimGeography(GeographyKey),
    CONSTRAINT FK_Fleet_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES dbo.DimGeography(GeographyKey),
    INDEX IX_TimeVehicle NONCLUSTERED (TimeKey, VehicleID),
    INDEX IX_Route NONCLUSTERED (OriginGeographyKey, DestinationGeographyKey)
) WITH (DATA_COMPRESSION = PAGE);

-- FactCrossDock: Direct transfer operations
CREATE TABLE dbo.FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    TransferTimeMinutes INT,
    Quantity INT,
    CONSTRAINT FK_CD_Time FOREIGN KEY (TimeKey) REFERENCES dbo.DimTime(TimeKey),
    CONSTRAINT FK_CD_Geography FOREIGN KEY (GeographyKey) REFERENCES dbo.DimGeography(GeographyKey),
    CONSTRAINT FK_CD_Product FOREIGN KEY (ProductKey) REFERENCES dbo.DimProductGravity(ProductKey),
    INDEX IX_TimeProd NONCLUSTERED (TimeKey, ProductKey)
) WITH (DATA_COMPRESSION = PAGE);
```

## Key Stored Procedures

### Data Loading

```sql
-- Incremental warehouse operations load
CREATE PROCEDURE dbo.usp_LoadWarehouseOperations
    @StartDateTime DATETIME2,
    @EndDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO dbo.FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        Quantity, DwellTimeMinutes, CycleTimeSeconds, OperatorID, BatchNumber
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        wms.operation_type,
        wms.quantity,
        DATEDIFF(MINUTE, wms.last_movement_time, wms.operation_time),
        wms.cycle_time_seconds,
        wms.operator_id,
        wms.batch_number
    FROM ExternalWarehouseSystem.dbo.Operations wms
    INNER JOIN dbo.DimTime t ON wms.operation_time = t.FullDateTime
    INNER JOIN dbo.DimGeography g ON wms.location_id = g.LocationID
    INNER JOIN dbo.DimProductGravity p ON wms.sku = p.SKU
    WHERE wms.operation_time BETWEEN @StartDateTime AND @EndDateTime
        AND NOT EXISTS (
            SELECT 1 FROM dbo.FactWarehouseOperations f
            WHERE f.TimeKey = t.TimeKey 
                AND f.ProductKey = p.ProductKey
                AND f.BatchNumber = wms.batch_number
        );
    
    RETURN @@ROWCOUNT;
END;
GO
```

### Cross-Fact KPI Calculation

```sql
-- Calculate warehouse efficiency vs fleet utilization correlation
CREATE PROCEDURE dbo.usp_CalculateCrossFactEfficiency
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH WarehouseMetrics AS (
        SELECT 
            t.FullDateTime,
            g.LocationID,
            AVG(CAST(DwellTimeMinutes AS FLOAT)) AS AvgDwellTime,
            SUM(Quantity) AS TotalThroughput,
            COUNT(DISTINCT OperatorID) AS ActiveOperators
        FROM dbo.FactWarehouseOperations f
        INNER JOIN dbo.DimTime t ON f.TimeKey = t.TimeKey
        INNER JOIN dbo.DimGeography g ON f.GeographyKey = g.GeographyKey
        WHERE CAST(t.FullDateTime AS DATE) BETWEEN @StartDate AND @EndDate
        GROUP BY t.FullDateTime, g.LocationID
    ),
    FleetMetrics AS (
        SELECT 
            t.FullDateTime,
            og.LocationID AS OriginLocation,
            AVG(CAST(IdleTimeMinutes AS FLOAT) / NULLIF(DurationMinutes, 0)) AS IdlePercent,
            COUNT(DISTINCT VehicleID) AS ActiveVehicles,
            SUM(LoadWeightKG) AS TotalLoad
        FROM dbo.FactFleetTrips f
        INNER JOIN dbo.DimTime t ON f.TimeKey = t.TimeKey
        INNER JOIN dbo.DimGeography og ON f.OriginGeographyKey = og.GeographyKey
        WHERE CAST(t.FullDateTime AS DATE) BETWEEN @StartDate AND @EndDate
        GROUP BY t.FullDateTime, og.LocationID
    )
    SELECT 
        wm.FullDateTime,
        wm.LocationID,
        wm.AvgDwellTime,
        wm.TotalThroughput,
        fm.IdlePercent * 100 AS FleetIdlePercent,
        fm.ActiveVehicles,
        -- Composite efficiency score (lower is better)
        (wm.AvgDwellTime / 60.0) * (fm.IdlePercent + 0.1) AS EfficiencyPenalty
    FROM WarehouseMetrics wm
    LEFT JOIN FleetMetrics fm 
        ON wm.FullDateTime = fm.FullDateTime 
        AND wm.LocationID = fm.OriginLocation
    ORDER BY EfficiencyPenalty DESC;
END;
GO
```

### Predictive Bottleneck Detection

```sql
-- Identify probable congestion points using historical patterns
CREATE PROCEDURE dbo.usp_PredictBottlenecks
    @ForecastHours INT = 24
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH HistoricalPatterns AS (
        SELECT 
            g.LocationID,
            t.HourOfDay,
            t.DayOfWeek,
            AVG(CAST(DwellTimeMinutes AS FLOAT)) AS AvgHistoricalDwell,
            STDEV(CAST(DwellTimeMinutes AS FLOAT)) AS StdDevDwell,
            COUNT(*) AS SampleSize
        FROM dbo.FactWarehouseOperations f
        INNER JOIN dbo.DimTime t ON f.TimeKey = t.TimeKey
        INNER JOIN dbo.DimGeography g ON f.GeographyKey = g.GeographyKey
        WHERE t.FullDateTime >= DATEADD(DAY, -90, GETDATE())
        GROUP BY g.LocationID, t.HourOfDay, t.DayOfWeek
        HAVING COUNT(*) >= 10 -- Minimum sample size
    ),
    ForecastPeriods AS (
        SELECT 
            TimeKey,
            FullDateTime,
            HourOfDay,
            DayOfWeek
        FROM dbo.DimTime
        WHERE FullDateTime BETWEEN GETDATE() AND DATEADD(HOUR, @ForecastHours, GETDATE())
    )
    SELECT 
        fp.FullDateTime AS ForecastTime,
        hp.LocationID,
        hp.AvgHistoricalDwell,
        hp.StdDevDwell,
        -- Bottleneck risk score: higher = more likely
        CASE 
            WHEN hp.AvgHistoricalDwell > 120 AND hp.StdDevDwell > 30 THEN 'HIGH'
            WHEN hp.AvgHistoricalDwell > 60 THEN 'MEDIUM'
            ELSE 'LOW'
        END AS BottleneckRisk,
        hp.SampleSize AS HistoricalDataPoints
    FROM ForecastPeriods fp
    INNER JOIN HistoricalPatterns hp 
        ON fp.HourOfDay = hp.HourOfDay 
        AND fp.DayOfWeek = hp.DayOfWeek
    WHERE hp.AvgHistoricalDwell > 60 -- Only flag potential issues
    ORDER BY hp.AvgHistoricalDwell DESC, fp.FullDateTime;
END;
GO
```

## Power BI Integration

### DAX Measures

Create these measures in Power BI for cross-fact analysis:

```dax
// Warehouse Throughput Rate (items per hour)
Throughput Rate = 
DIVIDE(
    SUM(FactWarehouseOperations[Quantity]),
    SUM(FactWarehouseOperations[CycleTimeSeconds]) / 3600,
    0
)

// Fleet Efficiency Score (0-100, higher is better)
Fleet Efficiency = 
VAR IdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
VAR TotalTime = SUM(FactFleetTrips[DurationMinutes])
VAR IdlePercent = DIVIDE(IdleTime, TotalTime, 0)
RETURN 
    (1 - IdlePercent) * 100

// Warehouse Gravity Alignment Score
Gravity Alignment = 
VAR HighGravityInOptimalZone = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProductGravity[OptimalZone] = "High",
        DimGeography[LocationType] = "Warehouse",
        FactWarehouseOperations[DwellTimeMinutes] < 60
    )
VAR TotalHighGravity = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProductGravity[OptimalZone] = "High"
    )
RETURN 
    DIVIDE(HighGravityInOptimalZone, TotalHighGravity, 0) * 100

// Cross-Fact Cost per Delivery
Cost per Delivery = 
VAR FuelCost = SUMX(FactFleetTrips, [FuelLiters] * 1.5) // $1.50/liter
VAR WarehouseLaborCost = 
    SUMX(
        FactWarehouseOperations,
        [CycleTimeSeconds] / 3600 * 25 // $25/hour labor
    )
VAR TotalDeliveries = DISTINCTCOUNT(FactFleetTrips[TripKey])
RETURN 
    DIVIDE(FuelCost + WarehouseLaborCost, TotalDeliveries, 0)
```

### Report Configuration

```json
// powerbi/datasource_config.json
{
  "server": "${SQL_SERVER_HOST}",
  "database": "LogiFleetPulse",
  "authentication": "SQL",
  "username": "${SQL_SERVER_USER}",
  "refresh_schedule": {
    "interval_minutes": 15,
    "full_refresh_hour": 2
  },
  "row_level_security": {
    "enabled": true,
    "role_table": "dbo.UserRoles",
    "filter_column": "LocationID"
  }
}
```

## Common Patterns

### Pattern 1: Daily Operational Dashboard Refresh

```sql
-- Schedule this via SQL Server Agent (daily at 6 AM)
EXEC dbo.usp_LoadWarehouseOperations 
    @StartDateTime = CAST(CAST(GETDATE() - 1 AS DATE) AS DATETIME2),
    @EndDateTime = CAST(CAST(GETDATE() AS DATE) AS DATETIME2);

EXEC dbo.usp_LoadFleetTelemetry 
    @StartDateTime = CAST(CAST(GETDATE() - 1 AS DATE) AS DATETIME2),
    @EndDateTime = CAST(CAST(GETDATE() AS DATE) AS DATETIME2);

-- Refresh materialized summary tables
EXEC dbo.usp_RefreshDailySummaries;
```

### Pattern 2: Warehouse Gravity Zone Optimization

```sql
-- Recalculate product gravity scores based on last 30 days
UPDATE p
SET 
    VelocityScore = velocity.Score,
    ValueScore = value.Score,
    FragilityScore = fragility.Score,
    OptimalZone = CASE 
        WHEN (velocity.Score * 0.5 + value.Score * 0.3 + fragility.Score * 0.2) > 8 THEN 'High'
        WHEN (velocity.Score * 0.5 + value.Score * 0.3 + fragility.Score * 0.2) > 5 THEN 'Medium'
        ELSE 'Low'
    END
FROM dbo.DimProductGravity p
INNER JOIN (
    SELECT 
        ProductKey,
        (COUNT(*) * 1.0 / 30) AS Score -- Picks per day normalized to 0-10
    FROM dbo.FactWarehouseOperations
    WHERE OperationType = 'Picking'
        AND TimeKey >= (SELECT MIN(TimeKey) FROM dbo.DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()))
    GROUP BY ProductKey
) velocity ON p.ProductKey = velocity.ProductKey
INNER JOIN (
    SELECT 
        ProductKey,
        CASE 
            WHEN AVG(UnitPrice) > 100 THEN 10
            WHEN AVG(UnitPrice) > 50 THEN 7
            WHEN AVG(UnitPrice) > 20 THEN 5
            ELSE 3
        END AS Score
    FROM dbo.ProductPricing
    GROUP BY ProductKey
) value ON p.ProductKey = value.ProductKey
INNER JOIN (
    SELECT 
        ProductKey,
        FragilityScore AS Score
    FROM dbo.ProductAttributes
) fragility ON p.ProductKey = fragility.ProductKey;
```

### Pattern 3: Automated Alerting

```sql
-- Alert when fleet idle time exceeds threshold
CREATE PROCEDURE dbo.usp_CheckFleetIdleAlert
AS
BEGIN
    DECLARE @AlertThresholdPercent DECIMAL(5,2) = 15.0;
    
    DECLARE @Alerts TABLE (
        VehicleID VARCHAR(50),
        IdlePercent DECIMAL(5,2),
        TripCount INT
    );
    
    INSERT INTO @Alerts
    SELECT 
        VehicleID,
        AVG(CAST(IdleTimeMinutes AS FLOAT) / NULLIF(DurationMinutes, 0)) * 100 AS IdlePercent,
        COUNT(*) AS TripCount
    FROM dbo.FactFleetTrips
    WHERE TimeKey >= (SELECT MIN(TimeKey) FROM dbo.DimTime WHERE FullDateTime >= DATEADD(HOUR, -4, GETDATE()))
    GROUP BY VehicleID
    HAVING AVG(CAST(IdleTimeMinutes AS FLOAT) / NULLIF(DurationMinutes, 0)) * 100 > @AlertThresholdPercent;
    
    IF EXISTS (SELECT 1 FROM @Alerts)
    BEGIN
        -- Send email via Database Mail
        DECLARE @Subject NVARCHAR(255) = 'Fleet Idle Time Alert';
        DECLARE @Body NVARCHAR(MAX);
        
        SELECT @Body = STRING_AGG(
            'Vehicle: ' + VehicleID + 
            ' | Idle: ' + CAST(IdlePercent AS VARCHAR(10)) + '%' +
            ' | Trips: ' + CAST(TripCount AS VARCHAR(10)),
            CHAR(13) + CHAR(10)
        )
        FROM @Alerts;
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = @Subject,
            @body = @Body;
    END;
END;
GO
```

## Troubleshooting

### Issue: Power BI Report Slow to Load

**Symptoms:** Dashboard refresh takes >2 minutes

**Solutions:**
```sql
-- 1. Check missing indexes
SELECT 
    OBJECT_NAME(ips.object_id) AS TableName,
    ips.avg_total_user_cost * ips.avg_user_impact * (ips.user_seeks + ips.user_scans) AS ImprovementScore,
    'CREATE INDEX IX_' + OBJECT_NAME(ips.object_id) + '_' + 
        REPLACE(REPLACE(id.equality_columns, '[', ''), ']', '') + 
        ' ON ' + OBJECT_NAME(ips.object_id) + ' (' + id.equality_columns + ')' AS CreateIndexStatement
FROM sys.dm_db_missing_index_group_stats ips
INNER JOIN sys.dm_db_missing_index_groups ig ON ips.group_handle = ig.index_group_handle
INNER JOIN sys.dm_db_missing_index_details id ON ig.index_handle = id.index_handle
WHERE id.database_id = DB_ID('LogiFleetPulse')
ORDER BY ImprovementScore DESC;

-- 2. Enable query store for performance tracking
ALTER DATABASE LogiFleetPulse SET QUERY_STORE = ON;
ALTER DATABASE LogiFleetPulse SET QUERY_STORE (OPERATION_MODE = READ_WRITE);

-- 3. Update statistics on large fact tables
UPDATE STATISTICS dbo.FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS dbo.FactFleetTrips WITH FULLSCAN;
```

### Issue: Cross-Fact Queries Timing Out

**Symptoms:** `usp_CalculateCrossFactEfficiency` exceeds 30 seconds

**Solutions:**
```sql
-- Create indexed views for common aggregations
CREATE VIEW dbo.vw_DailyWarehouseMetrics
WITH SCHEMABINDING
AS
SELECT 
    CAST(t.FullDateTime AS DATE) AS OperationDate,
    f.GeographyKey,
    f.ProductKey,
    COUNT_BIG(*) AS OperationCount,
    SUM(f.Quantity) AS TotalQuantity,
    AVG(CAST(f.DwellTimeMinutes AS FLOAT)) AS AvgDwellTime
FROM dbo.FactWarehouseOperations f
INNER JOIN dbo.DimTime t ON f.TimeKey = t.TimeKey
GROUP BY CAST(t.FullDateTime AS DATE), f.GeographyKey, f.ProductKey;
GO

CREATE UNIQUE CLUSTERED INDEX IX_DailyWarehouseMetrics 
ON dbo.vw_DailyWarehouseMetrics (OperationDate, GeographyKey, ProductKey);
GO
```

### Issue: Data Source Connection Failures

**Symptoms:** External table queries fail with "connection timeout"

**Solutions:**
```sql
-- Test external data source connectivity
SELECT * FROM sys.external_data_sources;

-- Recreate data source with increased timeout
DROP EXTERNAL DATA SOURCE IF EXISTS WMSDataSource;

CREATE EXTERNAL DATA SOURCE WMSDataSource
WITH (
    TYPE = RDBMS,
    LOCATION = '${WMS_DB_HOST}',
    DATABASE_NAME = 'WarehouseManagement',
    CREDENTIAL = WMSCredential,
    CONNECTION_OPTIONS = 'ConnectTimeout=60;CommandTimeout=120'
);
```

### Issue: Row-Level Security Not Working

**Symptoms:** Users see all locations despite RLS configuration

**Solutions:**
```sql
-- Verify RLS predicate function
CREATE FUNCTION dbo.fn_SecurityPredicate(@LocationID VARCHAR(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS access_result
WHERE @LocationID IN (
    SELECT LocationID 
    FROM dbo.UserRoles 
    WHERE UserName = USER_NAME()
);
GO

-- Apply security policy to all fact tables
CREATE SECURITY POLICY LocationSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(LocationID) 
    ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(LocationID) 
    ON dbo.FactFleetTrips
WITH (STATE = ON);
GO

-- Test as specific user
EXECUTE AS USER = 'regional_manager';
SELECT COUNT(*) FROM dbo.FactWarehouseOperations;
REVERT;
```

## Configuration Reference

### Environment Variables

```bash
# Database connection
export SQL_SERVER_HOST="logifleet-prod.database.windows.net"
export SQL_SERVER_USER="analytics_service"
export SQL_SERVER_PASSWORD="${SQL_SERVER_PASSWORD}" # Set securely

# External data sources
export WMS_API_KEY="${WMS_API_KEY}"
export WMS_DB_HOST="wms-db.internal.company.com"
export FLEET_API_KEY="${FLEET_API_KEY}"
export FLEET_TELEMETRY_URL="https://api.telematics-provider.com/v2"

# Alert configuration
export ALERT_EMAIL="logistics-ops@company.com"
export SMTP_SERVER="smtp.office365.com"
export SMTP_PORT="587"

# Power BI service
export POWERBI_WORKSPACE_ID="${POWERBI_WORKSPACE_ID}"
export POWERBI_DATASET_ID="${POWERBI_DATASET_ID}"
```

### Key Configuration Files

- `config.json` - Main configuration (connection strings, refresh schedules)
- `powerbi/datasource_config.json` - Power BI connection parameters
- `schema/partition_config.sql` - Table partitioning strategy
- `alerts/threshold_config.sql` - Alert rule definitions

## Best Practices

1. **Partition large fact tables by date** - Use monthly partitions for tables with >10M rows
2. **Implement incremental loading** - Only load new/changed records, track with watermark columns
3. **Use columnstore indexes** - Apply to aggregation tables and historical fact tables
4. **Enable query result caching** - Configure Power BI to cache common queries
5. **Schedule off-peak maintenance** - Run statistics updates and index rebuilds at 2 AM
6. **Monitor query performance** - Review Query Store weekly for regression patterns
7. **Document DAX calculations** - Add comments explaining business logic in all measures
8. **Test RLS thoroughly** - Validate security policies with actual user accounts before production
