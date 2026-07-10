---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse for fleet, warehouse, and supply chain KPI analytics
triggers:
  - "set up LogiFleet Pulse analytics"
  - "deploy supply chain data warehouse"
  - "configure Power BI logistics dashboard"
  - "implement fleet and warehouse KPI tracking"
  - "create multi-fact star schema for logistics"
  - "integrate warehouse and fleet data sources"
  - "build real-time supply chain analytics"
  - "setup logistics intelligence platform"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform that unifies warehouse operations, fleet telemetry, and supply chain metrics into a single semantic data model. Built on MS SQL Server with Power BI visualization, it implements a multi-fact star schema with time-phased dimensions for cross-modal supply chain analytics.

**Core capabilities:**
- Multi-fact data warehouse (warehouse ops, fleet trips, cross-dock)
- Real-time KPI harmonization across logistics domains
- Predictive bottleneck detection and fleet triage
- Power BI dashboards with 15-minute refresh cycles
- Role-based access control and row-level security

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (or Azure SQL Database)
- Power BI Desktop (latest version)
- Data sources: WMS, TMS, telematics APIs, or compatible exports

### Step 1: Deploy SQL Schema

```sql
-- Execute the schema deployment script
-- This creates fact tables, dimensions, views, and stored procedures

-- Core fact tables created:
-- FactWarehouseOperations, FactFleetTrips, FactCrossDock

-- Core dimensions created:
-- DimTime, DimGeography, DimProductGravity, DimSupplierReliability

-- Example: Create FactWarehouseOperations
CREATE TABLE dbo.FactWarehouseOperations (
    OperationID INT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes INT,
    PickRate DECIMAL(10,2),
    PackingTimeSeconds INT,
    OperationTimestamp DATETIME2,
    CreatedDate DATETIME2 DEFAULT GETDATE(),
    CONSTRAINT FK_WHOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WHOps_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    CONSTRAINT FK_WHOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_WHOps_Time ON dbo.FactWarehouseOperations(TimeKey) INCLUDE (DwellTimeMinutes, PickRate);
CREATE NONCLUSTERED INDEX IX_WHOps_Product ON dbo.FactWarehouseOperations(ProductKey, OperationType);
```

### Step 2: Configure Time Dimension

```sql
-- Generate DimTime with 15-minute granularity
CREATE TABLE dbo.DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2,
    Hour INT,
    QuarterHour INT, -- 0, 15, 30, 45
    DayOfWeek VARCHAR(20),
    DayOfMonth INT,
    Month INT,
    Quarter INT,
    FiscalYear INT,
    IsBusinessHour BIT,
    IsWeekend BIT
);

-- Populate time dimension (example for one day)
DECLARE @StartDate DATETIME2 = '2026-01-01';
DECLARE @EndDate DATETIME2 = '2026-12-31';
DECLARE @CurrentTime DATETIME2 = @StartDate;

WHILE @CurrentTime <= @EndDate
BEGIN
    INSERT INTO dbo.DimTime (
        TimeKey,
        FullDateTime,
        Hour,
        QuarterHour,
        DayOfWeek,
        DayOfMonth,
        Month,
        Quarter,
        FiscalYear,
        IsBusinessHour,
        IsWeekend
    )
    VALUES (
        CONVERT(INT, FORMAT(@CurrentTime, 'yyyyMMddHHmm')),
        @CurrentTime,
        DATEPART(HOUR, @CurrentTime),
        (DATEPART(MINUTE, @CurrentTime) / 15) * 15,
        DATENAME(WEEKDAY, @CurrentTime),
        DAY(@CurrentTime),
        MONTH(@CurrentTime),
        DATEPART(QUARTER, @CurrentTime),
        YEAR(@CurrentTime),
        CASE WHEN DATEPART(HOUR, @CurrentTime) BETWEEN 8 AND 17 THEN 1 ELSE 0 END,
        CASE WHEN DATEPART(WEEKDAY, @CurrentTime) IN (1, 7) THEN 1 ELSE 0 END
    );
    
    SET @CurrentTime = DATEADD(MINUTE, 15, @CurrentTime);
END;
```

### Step 3: Create Fleet Fact Table

```sql
CREATE TABLE dbo.FactFleetTrips (
    TripID INT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingUnloadingTimeMinutes INT,
    AverageSpeedKMH DECIMAL(6,2),
    DelayMinutes INT,
    DelayReasonKey INT,
    CargoWeightKG DECIMAL(10,2),
    CreatedDate DATETIME2 DEFAULT GETDATE(),
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_Fleet_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED INDEX IX_Fleet_TimeVehicle ON dbo.FactFleetTrips(TimeKey, VehicleKey) INCLUDE (FuelConsumedLiters, IdleTimeMinutes);
```

## Data Ingestion Patterns

### Incremental ETL with Stored Procedure

```sql
CREATE PROCEDURE dbo.usp_LoadWarehouseOperations
    @LastLoadDate DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations from staging
    INSERT INTO dbo.FactWarehouseOperations (
        TimeKey,
        WarehouseKey,
        ProductKey,
        OperationType,
        DwellTimeMinutes,
        PickRate,
        PackingTimeSeconds,
        OperationTimestamp
    )
    SELECT 
        CONVERT(INT, FORMAT(stg.OperationTimestamp, 'yyyyMMddHH') + 
            RIGHT('00' + CAST((DATEPART(MINUTE, stg.OperationTimestamp) / 15) * 15 AS VARCHAR), 2)),
        dw.WarehouseKey,
        dp.ProductKey,
        stg.OperationType,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DwellTimeMinutes,
        stg.PickRate,
        DATEDIFF(SECOND, stg.PackStartTime, stg.PackEndTime) AS PackingTimeSeconds,
        stg.OperationTimestamp
    FROM StagingWarehouseOps stg
    INNER JOIN DimWarehouse dw ON stg.WarehouseCode = dw.WarehouseCode
    INNER JOIN DimProduct dp ON stg.SKU = dp.SKU
    WHERE stg.OperationTimestamp > @LastLoadDate
        AND NOT EXISTS (
            SELECT 1 FROM dbo.FactWarehouseOperations f
            WHERE f.OperationTimestamp = stg.OperationTimestamp
                AND f.WarehouseKey = dw.WarehouseKey
                AND f.ProductKey = dp.ProductKey
        );
        
    -- Log execution
    INSERT INTO ETLLog (ProcedureName, ExecutionTime, RowsAffected)
    VALUES ('usp_LoadWarehouseOperations', GETDATE(), @@ROWCOUNT);
END;
```

### External API Integration (Polybase Example)

```sql
-- Create external data source for telemetry API
CREATE EXTERNAL DATA SOURCE TelemetryAPI
WITH (
    TYPE = HADOOP,
    LOCATION = 'https://telemetry-api.example.com/data',
    CREDENTIAL = TelemetryAPICredential
);

-- Create external table for fleet data
CREATE EXTERNAL TABLE ExtFleetTelemetry (
    VehicleID VARCHAR(50),
    Timestamp DATETIME2,
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    SpeedKMH DECIMAL(6,2),
    FuelLevelPercent DECIMAL(5,2),
    EngineTemp DECIMAL(5,2)
)
WITH (
    LOCATION = '/fleet-telemetry/',
    DATA_SOURCE = TelemetryAPI,
    FILE_FORMAT = JSONFormat
);
```

## Cross-Fact KPI Queries

### Dwell Time vs. Fleet Idling Cost

```sql
-- Calculate correlation between warehouse dwell time and fleet idle costs
WITH WarehouseDwell AS (
    SELECT 
        t.DayOfMonth,
        t.Month,
        AVG(f.DwellTimeMinutes) AS AvgDwellTime
    FROM FactWarehouseOperations f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.FiscalYear = 2026
    GROUP BY t.DayOfMonth, t.Month
),
FleetIdleCost AS (
    SELECT 
        t.DayOfMonth,
        t.Month,
        SUM(f.IdleTimeMinutes * v.HourlyOperatingCost / 60.0) AS TotalIdleCost
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    INNER JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
    WHERE t.FiscalYear = 2026
    GROUP BY t.DayOfMonth, t.Month
)
SELECT 
    wd.Month,
    wd.DayOfMonth,
    wd.AvgDwellTime,
    fc.TotalIdleCost,
    CASE 
        WHEN wd.AvgDwellTime > 72 AND fc.TotalIdleCost > 5000 THEN 'High Risk'
        WHEN wd.AvgDwellTime > 48 OR fc.TotalIdleCost > 3000 THEN 'Medium Risk'
        ELSE 'Normal'
    END AS RiskCategory
FROM WarehouseDwell wd
INNER JOIN FleetIdleCost fc ON wd.DayOfMonth = fc.DayOfMonth AND wd.Month = fc.Month
ORDER BY wd.Month, wd.DayOfMonth;
```

### Product Gravity Score Calculation

```sql
-- Calculate and update product gravity scores
CREATE PROCEDURE dbo.usp_CalculateProductGravity
AS
BEGIN
    -- Gravity = (PickFrequency * 0.4) + (UnitValue * 0.3) + (FragilityIndex * 0.3)
    UPDATE dp
    SET dp.GravityScore = 
        (picks.PickFrequency * 0.4) + 
        (dp.UnitValue / 100.0 * 0.3) + 
        (dp.FragilityIndex * 0.3)
    FROM DimProduct dp
    CROSS APPLY (
        SELECT 
            COUNT(*) AS PickFrequency
        FROM FactWarehouseOperations f
        WHERE f.ProductKey = dp.ProductKey
            AND f.OperationType = 'Picking'
            AND f.OperationTimestamp >= DATEADD(DAY, -30, GETDATE())
    ) picks;
    
    -- Recommend zone reassignments for high-gravity items far from shipping
    SELECT 
        p.SKU,
        p.ProductName,
        p.GravityScore,
        p.CurrentZone,
        'Move to Zone A (Closer to Shipping)' AS Recommendation
    FROM DimProduct p
    WHERE p.GravityScore > 0.7
        AND p.CurrentZone NOT IN ('Zone A', 'Zone B')
    ORDER BY p.GravityScore DESC;
END;
```

## Power BI Configuration

### Connection String Setup

```powerquery
// In Power BI, use environment variable for connection
let
    ServerName = Environment.Variable("SQL_SERVER_NAME"),
    DatabaseName = Environment.Variable("SQL_DATABASE_NAME"),
    Source = Sql.Database(ServerName, DatabaseName, [
        Query = null,
        CommandTimeout = #duration(0, 0, 5, 0),
        ConnectionTimeout = #duration(0, 0, 1, 30)
    ])
in
    Source
```

### Row-Level Security Implementation

```sql
-- Create security table
CREATE TABLE dbo.SecurityUserRegion (
    UserEmail VARCHAR(255) PRIMARY KEY,
    RegionCode VARCHAR(10)
);

-- Create RLS function
CREATE FUNCTION dbo.fn_SecurityPredicate(@RegionCode VARCHAR(10))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN 
    SELECT 1 AS result
    WHERE @RegionCode IN (
        SELECT RegionCode 
        FROM dbo.SecurityUserRegion 
        WHERE UserEmail = USER_NAME()
    )
    OR USER_NAME() = 'admin@company.com';

-- Apply security policy
CREATE SECURITY POLICY RegionSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(RegionCode) ON DimWarehouse,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(RegionCode) ON DimRoute
WITH (STATE = ON);
```

### DAX Measures for Power BI

```dax
// Fleet Utilization Rate
Fleet Utilization % = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKM]),
    SUM(DimVehicle[MaxDailyKM]) * DISTINCTCOUNT(FactFleetTrips[TimeKey]) / 96,
    0
) * 100

// Warehouse Dwell Time Average
Avg Dwell Time = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    FactWarehouseOperations[OperationType] IN {"Receiving", "Putaway"}
)

// Predictive Bottleneck Score
Bottleneck Score = 
VAR CurrentDwell = [Avg Dwell Time]
VAR HistoricalAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATESINPERIOD(DimTime[FullDateTime], MAX(DimTime[FullDateTime]), -90, DAY)
    )
VAR IdlePercent = DIVIDE(SUM(FactFleetTrips[IdleTimeMinutes]), SUM(FactFleetTrips[IdleTimeMinutes]) + SUM(FactFleetTrips[LoadingUnloadingTimeMinutes]), 0)
RETURN
    IF(
        CurrentDwell > HistoricalAvg * 1.2 && IdlePercent > 0.15,
        100 * (CurrentDwell / HistoricalAvg) * IdlePercent,
        0
    )
```

## Automated Alerting

### SQL Server Agent Job for Threshold Monitoring

```sql
CREATE PROCEDURE dbo.usp_MonitorKPIThresholds
AS
BEGIN
    DECLARE @AlertRecipients VARCHAR(500) = GETENV('ALERT_EMAIL_RECIPIENTS');
    
    -- Check fleet idle time threshold
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips f
        INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -2, GETDATE())
        GROUP BY f.VehicleKey
        HAVING AVG(CAST(f.IdleTimeMinutes AS FLOAT) / NULLIF(DATEDIFF(MINUTE, f.TripStartTime, f.TripEndTime), 0)) > 0.15
    )
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = @AlertRecipients,
            @subject = 'ALERT: Fleet Idle Time Exceeds 15%',
            @body = 'One or more vehicles have exceeded the idle time threshold in the last 2 hours. Check LogiFleet Pulse dashboard for details.',
            @importance = 'High';
    END;
    
    -- Check warehouse dwell time spike
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations f
        INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -4, GETDATE())
            AND f.DwellTimeMinutes > 72
        GROUP BY f.WarehouseKey
        HAVING COUNT(*) > 10
    )
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = @AlertRecipients,
            @subject = 'ALERT: Warehouse Dwell Time Spike Detected',
            @body = 'Multiple SKUs have exceeded 72-hour dwell time threshold. Review warehouse operations immediately.',
            @importance = 'High';
    END;
END;
```

## Common Troubleshooting

### Issue: Slow Cross-Fact Queries

**Solution:** Implement indexed views for common joins

```sql
CREATE VIEW dbo.vw_FleetWarehouseCorrelation
WITH SCHEMABINDING
AS
SELECT 
    t.TimeKey,
    t.DayOfMonth,
    t.Month,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    COUNT_BIG(*) AS RecordCount
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN dbo.FactFleetTrips ft ON wo.TimeKey = ft.TimeKey
GROUP BY t.TimeKey, t.DayOfMonth, t.Month;

CREATE UNIQUE CLUSTERED INDEX IX_FleetWH_Time ON dbo.vw_FleetWarehouseCorrelation(TimeKey);
```

### Issue: Power BI Refresh Timeout

**Solution:** Partition large fact tables

```sql
-- Create partition scheme
CREATE PARTITION FUNCTION pf_TimeKey (INT)
AS RANGE RIGHT FOR VALUES (202601, 202602, 202603, 202604, 202605, 202606);

CREATE PARTITION SCHEME ps_TimeKey
AS PARTITION pf_TimeKey ALL TO ([PRIMARY]);

-- Rebuild table on partition scheme
CREATE TABLE dbo.FactWarehouseOperations_Partitioned (
    -- columns same as original
) ON ps_TimeKey(TimeKey);
```

### Issue: Missing Data from External APIs

**Solution:** Implement retry logic with exponential backoff

```sql
CREATE PROCEDURE dbo.usp_LoadTelemetryWithRetry
AS
BEGIN
    DECLARE @RetryCount INT = 0;
    DECLARE @MaxRetries INT = 3;
    DECLARE @Success BIT = 0;
    
    WHILE @RetryCount < @MaxRetries AND @Success = 0
    BEGIN
        BEGIN TRY
            INSERT INTO FactFleetTrips (...)
            SELECT * FROM ExtFleetTelemetry
            WHERE Timestamp > (SELECT MAX(TripEndTime) FROM FactFleetTrips);
            
            SET @Success = 1;
        END TRY
        BEGIN CATCH
            SET @RetryCount = @RetryCount + 1;
            WAITFOR DELAY '00:00:05'; -- Wait 5 seconds before retry
            
            IF @RetryCount >= @MaxRetries
            BEGIN
                INSERT INTO ETLErrorLog (ProcedureName, ErrorMessage, ErrorTime)
                VALUES ('usp_LoadTelemetryWithRetry', ERROR_MESSAGE(), GETDATE());
            END;
        END CATCH;
    END;
END;
```

## Best Practices

1. **Always use parameterized queries** to prevent SQL injection when building dynamic filters
2. **Implement incremental refresh** for Power BI datasets over 1GB
3. **Use composite indexes** on fact tables for common filter combinations (TimeKey + ForeignKey)
4. **Set up dedicated service account** with minimal required permissions for Power BI gateway
5. **Archive historical data** beyond 2 years to separate tables for performance
6. **Test RLS policies** before deploying to production
7. **Document custom DAX measures** with business logic comments
8. **Schedule nightly index maintenance** to reorganize fragmented indexes

## Environment Variables Reference

```bash
# SQL Server connection
SQL_SERVER_NAME=your-server.database.windows.net
SQL_DATABASE_NAME=LogiFleetPulse
SQL_USER=logifleet_user
SQL_PASSWORD=your-secure-password

# Alert configuration
ALERT_EMAIL_RECIPIENTS=ops@company.com;manager@company.com

# External API credentials
TELEMETRY_API_KEY=your-api-key
WEATHER_API_ENDPOINT=https://api.weather.com/v1
```
