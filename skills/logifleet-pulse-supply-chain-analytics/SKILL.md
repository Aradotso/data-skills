---
name: logifleet-pulse-supply-chain-analytics
description: Advanced MS SQL Server & Power BI data warehousing and fleet logistics analytics engine with multi-fact star schema
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy SQL schema for logistics data warehouse"
  - "configure Power BI logistics dashboard"
  - "implement warehouse gravity zones"
  - "create fleet analytics data model"
  - "build cross-modal supply chain KPIs"
  - "integrate warehouse and fleet telemetry data"
  - "setup logistics intelligence platform"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to deploy and configure LogiFleet Pulse, a multi-modal logistics intelligence engine that unifies warehouse operations, fleet telemetry, and supply chain data into a semantic data warehouse using MS SQL Server and Power BI.

## What LogiFleet Pulse Does

LogiFleet Pulse is a comprehensive data warehousing solution that:

- **Unifies logistics data streams**: Warehouse operations, fleet GPS/telemetry, supplier data, weather/traffic APIs
- **Multi-fact star schema**: Cross-fact KPI harmonization with time-phased dimensions
- **Real-time dashboards**: Power BI visualizations refreshed every 15 minutes
- **Predictive analytics**: Bottleneck detection, fleet triage engine, warehouse gravity zones
- **Semantic layer**: Single source of truth across warehouse, fleet, and supply chain metrics

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Access to data sources: WMS, fleet telemetry, ERP systems

### Step 1: Deploy SQL Schema

Clone the repository and run the schema deployment script:

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

Connect to your SQL Server and execute the schema script:

```sql
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the provided schema script
-- This creates fact tables, dimensions, views, and stored procedures
:r schema_deployment.sql
GO
```

### Step 2: Configure Data Sources

Create a configuration file for your data connections:

```json
{
  "connections": {
    "sql_server": {
      "server": "${SQL_SERVER_HOST}",
      "database": "LogiFleetPulse",
      "username": "${SQL_USERNAME}",
      "password": "${SQL_PASSWORD}"
    },
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}"
    },
    "fleet_telemetry": {
      "endpoint": "${FLEET_API_ENDPOINT}",
      "api_key": "${FLEET_API_KEY}"
    }
  },
  "refresh_interval_minutes": 15
}
```

### Step 3: Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI Template
3. Select `LogiFleet_Pulse_Master.pbit`
4. Enter SQL Server connection details when prompted

## Core Data Model Structure

### Fact Tables

**FactWarehouseOperations** - Warehouse activity metrics:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    WarehouseKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OperationType VARCHAR(50), -- 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    DwellTimeMinutes INT,
    PickRate DECIMAL(10,2),
    PackingTimeSeconds INT,
    QuantityProcessed INT,
    LoadTimestamp DATETIME DEFAULT GETDATE()
);

-- Clustered columnstore for analytical queries
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOps 
ON FactWarehouseOperations;
```

**FactFleetTrips** - Fleet telemetry and route data:

```sql
CREATE TABLE FactFleetTrips (
    TripID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    RouteKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DriverKey INT FOREIGN KEY REFERENCES DimDriver(DriverKey),
    DistanceMiles DECIMAL(10,2),
    FuelConsumptionGallons DECIMAL(10,3),
    IdleTimeMinutes INT,
    LoadingUnloadingMinutes INT,
    AverageSpeedMPH DECIMAL(5,2),
    DelayMinutes INT,
    DelayReason VARCHAR(100),
    LoadTimestamp DATETIME DEFAULT GETDATE()
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips 
ON FactFleetTrips;
```

### Dimension Tables

**DimTime** - Time-phased dimension (15-minute granularity):

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME,
    Date DATE,
    TimeOfDay TIME,
    Hour INT,
    Minute INT,
    QuarterHour INT, -- 0, 15, 30, 45
    DayOfWeek INT,
    DayName VARCHAR(10),
    WeekOfYear INT,
    Month INT,
    MonthName VARCHAR(10),
    Quarter INT,
    Year INT,
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT,
    IsHoliday BIT
);

-- Populate DimTime with 15-minute intervals
CREATE PROCEDURE PopulateDimTime
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    DECLARE @CurrentDateTime DATETIME = @StartDate;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        INSERT INTO DimTime (
            TimeKey, FullDateTime, Date, TimeOfDay, 
            Hour, Minute, QuarterHour, DayOfWeek, 
            DayName, WeekOfYear, Month, MonthName, 
            Quarter, Year, IsWeekend
        )
        VALUES (
            CONVERT(INT, FORMAT(@CurrentDateTime, 'yyyyMMddHHmm')),
            @CurrentDateTime,
            CAST(@CurrentDateTime AS DATE),
            CAST(@CurrentDateTime AS TIME),
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(MINUTE, @CurrentDateTime),
            (DATEPART(MINUTE, @CurrentDateTime) / 15) * 15,
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            DATEPART(MONTH, @CurrentDateTime),
            DATENAME(MONTH, @CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            DATEPART(YEAR, @CurrentDateTime),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1,7) THEN 1 ELSE 0 END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
```

**DimProductGravity** - Product hierarchy with gravity scoring:

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    PickFrequency INT, -- Picks per day
    ItemValue DECIMAL(10,2),
    FragilityScore INT, -- 1-10
    GravityScore DECIMAL(5,2), -- Calculated composite score
    OptimalZone VARCHAR(50), -- 'HIGH_GRAVITY', 'MEDIUM_GRAVITY', 'LOW_GRAVITY'
    LastRecalculated DATETIME
);

-- Calculate gravity score
CREATE FUNCTION CalculateGravityScore(
    @PickFrequency INT,
    @ItemValue DECIMAL(10,2),
    @FragilityScore INT
)
RETURNS DECIMAL(5,2)
AS
BEGIN
    DECLARE @Score DECIMAL(5,2);
    
    -- Weighted formula: frequency (40%) + value (40%) + fragility (20%)
    SET @Score = 
        (@PickFrequency * 0.4) + 
        ((@ItemValue / 100) * 0.4) + 
        (@FragilityScore * 0.2);
    
    RETURN @Score;
END;
```

## Key Stored Procedures

### Incremental Data Loading

```sql
CREATE PROCEDURE LoadWarehouseOperations
    @LastLoadTimestamp DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last hour if no timestamp provided
    IF @LastLoadTimestamp IS NULL
        SET @LastLoadTimestamp = DATEADD(HOUR, -1, GETDATE());
    
    -- Insert new operations from staging
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, 
        OperationType, DwellTimeMinutes, PickRate, 
        PackingTimeSeconds, QuantityProcessed
    )
    SELECT 
        CONVERT(INT, FORMAT(s.OperationDateTime, 'yyyyMMddHHmm')) AS TimeKey,
        p.ProductKey,
        g.GeographyKey,
        s.OperationType,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
        s.PickRate,
        s.PackingTimeSeconds,
        s.QuantityProcessed
    FROM StagingWarehouseOps s
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    INNER JOIN DimGeography g ON s.WarehouseCode = g.LocationCode
    WHERE s.OperationDateTime > @LastLoadTimestamp
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f
            WHERE f.TimeKey = CONVERT(INT, FORMAT(s.OperationDateTime, 'yyyyMMddHHmm'))
                AND f.ProductKey = p.ProductKey
        );
    
    PRINT 'Loaded ' + CAST(@@ROWCOUNT AS VARCHAR) + ' warehouse operations.';
END;
```

### Cross-Fact KPI Query

```sql
CREATE PROCEDURE GetCrossModalKPIs
    @StartDate DATE,
    @EndDate DATE,
    @WarehouseCode VARCHAR(50) = NULL
AS
BEGIN
    -- Unified KPI showing warehouse dwell impact on fleet performance
    SELECT 
        t.Date,
        g.WarehouseName,
        p.Category,
        
        -- Warehouse metrics
        AVG(w.DwellTimeMinutes) AS AvgDwellMinutes,
        SUM(w.QuantityProcessed) AS TotalUnitsProcessed,
        
        -- Fleet metrics
        AVG(f.IdleTimeMinutes) AS AvgFleetIdleMinutes,
        AVG(f.FuelConsumptionGallons) AS AvgFuelConsumption,
        SUM(f.DelayMinutes) AS TotalDelayMinutes,
        
        -- Cross-fact composite KPI
        (AVG(w.DwellTimeMinutes) * AVG(f.IdleTimeMinutes)) / 60.0 AS EfficiencyImpactHours
        
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
    LEFT JOIN FactFleetTrips f ON t.TimeKey = f.TimeKey 
        AND g.GeographyKey = f.RouteKey
    
    WHERE t.Date BETWEEN @StartDate AND @EndDate
        AND (@WarehouseCode IS NULL OR g.LocationCode = @WarehouseCode)
    
    GROUP BY t.Date, g.WarehouseName, p.Category
    ORDER BY EfficiencyImpactHours DESC;
END;
```

### Warehouse Gravity Zone Optimizer

```sql
CREATE PROCEDURE OptimizeWarehouseGravityZones
AS
BEGIN
    -- Recalculate gravity scores based on recent activity
    UPDATE p
    SET 
        GravityScore = dbo.CalculateGravityScore(
            recent.AvgPickFrequency,
            p.ItemValue,
            p.FragilityScore
        ),
        OptimalZone = CASE
            WHEN dbo.CalculateGravityScore(recent.AvgPickFrequency, p.ItemValue, p.FragilityScore) >= 8 
                THEN 'HIGH_GRAVITY'
            WHEN dbo.CalculateGravityScore(recent.AvgPickFrequency, p.ItemValue, p.FragilityScore) >= 5 
                THEN 'MEDIUM_GRAVITY'
            ELSE 'LOW_GRAVITY'
        END,
        LastRecalculated = GETDATE()
    FROM DimProductGravity p
    INNER JOIN (
        SELECT 
            ProductKey,
            AVG(CAST(QuantityProcessed AS FLOAT) / 
                NULLIF(DwellTimeMinutes, 0)) AS AvgPickFrequency
        FROM FactWarehouseOperations
        WHERE TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm'))
        GROUP BY ProductKey
    ) recent ON p.ProductKey = recent.ProductKey;
    
    -- Generate zone reassignment recommendations
    SELECT 
        p.SKU,
        p.ProductName,
        p.OptimalZone AS RecommendedZone,
        wz.CurrentZone,
        p.GravityScore,
        CASE 
            WHEN p.OptimalZone <> wz.CurrentZone THEN 'REASSIGN'
            ELSE 'OK'
        END AS Action
    FROM DimProductGravity p
    LEFT JOIN WarehouseZoneAssignments wz ON p.ProductKey = wz.ProductKey
    WHERE p.OptimalZone <> wz.CurrentZone OR wz.CurrentZone IS NULL
    ORDER BY p.GravityScore DESC;
END;
```

## Power BI Integration

### DAX Measures for Cross-Fact Analysis

```dax
// Total Efficiency Score (composite KPI)
Total Efficiency Score = 
VAR WarehouseDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR FleetIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR WarehouseUtil = DIVIDE(
    SUM(FactWarehouseOperations[QuantityProcessed]),
    COUNTROWS(FactWarehouseOperations)
)
RETURN
    100 - ((WarehouseDwell * 0.3) + (FleetIdle * 0.3) + ((100 - WarehouseUtil) * 0.4))

// Predictive Bottleneck Index
Bottleneck Risk Index = 
VAR CurrentDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR HistoricalDwell = CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATEADD(DimTime[Date], -30, DAY)
)
VAR DwellVariance = DIVIDE(CurrentDwell - HistoricalDwell, HistoricalDwell)

VAR CurrentDelay = AVERAGE(FactFleetTrips[DelayMinutes])
VAR HistoricalDelay = CALCULATE(
    AVERAGE(FactFleetTrips[DelayMinutes]),
    DATEADD(DimTime[Date], -30, DAY)
)
VAR DelayVariance = DIVIDE(CurrentDelay - HistoricalDelay, HistoricalDelay)

RETURN
    (DwellVariance * 50) + (DelayVariance * 50)

// Warehouse Gravity Zone Efficiency
Gravity Zone Efficiency = 
CALCULATE(
    DIVIDE(
        SUM(FactWarehouseOperations[QuantityProcessed]),
        SUM(FactWarehouseOperations[DwellTimeMinutes])
    ),
    FILTER(
        DimProductGravity,
        DimProductGravity[OptimalZone] = "HIGH_GRAVITY"
    )
)
```

### Power BI Dashboard Setup

1. **Create Data Model Relationships**:
   - FactWarehouseOperations[TimeKey] → DimTime[TimeKey]
   - FactWarehouseOperations[ProductKey] → DimProductGravity[ProductKey]
   - FactFleetTrips[TimeKey] → DimTime[TimeKey]

2. **Configure Row-Level Security**:

```dax
// RLS Rule: Users only see their assigned warehouses
[WarehouseCode] IN { USERNAME_WAREHOUSES() }

// Helper function
USERNAME_WAREHOUSES = 
LOOKUPVALUE(
    UserWarehouseAccess[WarehouseCode],
    UserWarehouseAccess[Username],
    USERNAME()
)
```

## Automated Alerting

### SQL Server Agent Job for KPI Monitoring

```sql
CREATE PROCEDURE MonitorKPIThresholds
AS
BEGIN
    DECLARE @AlertTable TABLE (
        AlertType VARCHAR(50),
        EntityName VARCHAR(100),
        MetricValue DECIMAL(10,2),
        Threshold DECIMAL(10,2),
        AlertMessage VARCHAR(500)
    );
    
    -- Check warehouse dwell time threshold
    INSERT INTO @AlertTable
    SELECT 
        'WAREHOUSE_DWELL',
        g.WarehouseName,
        AVG(w.DwellTimeMinutes) AS AvgDwell,
        120.0 AS Threshold,
        'Dwell time exceeds 2 hours at ' + g.WarehouseName
    FROM FactWarehouseOperations w
    INNER JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.Date = CAST(GETDATE() AS DATE)
    GROUP BY g.WarehouseName
    HAVING AVG(w.DwellTimeMinutes) > 120;
    
    -- Check fleet idle time threshold
    INSERT INTO @AlertTable
    SELECT 
        'FLEET_IDLE',
        v.VehicleID,
        AVG(f.IdleTimeMinutes) AS AvgIdle,
        30.0 AS Threshold,
        'Excessive idle time for vehicle ' + v.VehicleID
    FROM FactFleetTrips f
    INNER JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.Date = CAST(GETDATE() AS DATE)
    GROUP BY v.VehicleID
    HAVING AVG(f.IdleTimeMinutes) > 30;
    
    -- Send alerts if any thresholds breached
    IF EXISTS (SELECT 1 FROM @AlertTable)
    BEGIN
        DECLARE @EmailBody VARCHAR(MAX);
        
        SELECT @EmailBody = STRING_AGG(AlertMessage, CHAR(13) + CHAR(10))
        FROM @AlertTable;
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'LogiFleet Pulse - KPI Threshold Alert',
            @body = @EmailBody;
    END
END;
```

## Common Usage Patterns

### Pattern 1: Daily Operations Dashboard Refresh

```sql
-- Execute daily at 6 AM
EXEC LoadWarehouseOperations @LastLoadTimestamp = '2026-07-07 00:00:00';
EXEC LoadFleetTrips @LastLoadTimestamp = '2026-07-07 00:00:00';
EXEC OptimizeWarehouseGravityZones;
EXEC MonitorKPIThresholds;
```

### Pattern 2: Ad-Hoc Analysis Query

```sql
-- Find correlation between warehouse dwell and fleet delays
SELECT 
    t.Date,
    w.OperationType,
    AVG(w.DwellTimeMinutes) AS AvgWarehouseDwell,
    AVG(f.DelayMinutes) AS AvgFleetDelay,
    CORR(w.DwellTimeMinutes, f.DelayMinutes) AS Correlation
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
LEFT JOIN FactFleetTrips f ON t.TimeKey = f.TimeKey
WHERE t.Date >= DATEADD(DAY, -90, GETDATE())
GROUP BY t.Date, w.OperationType
ORDER BY Correlation DESC;
```

### Pattern 3: Predictive Scenario Simulation

```sql
-- Simulate impact of increasing warehouse capacity to 95%
WITH CurrentCapacity AS (
    SELECT 
        AVG(DwellTimeMinutes) AS BaselineDwell,
        AVG(QuantityProcessed) AS BaselineVolume
    FROM FactWarehouseOperations
    WHERE TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm'))
),
ScenarioImpact AS (
    SELECT 
        BaselineDwell * 0.85 AS ProjectedDwell, -- 15% reduction expected
        BaselineVolume * 1.19 AS ProjectedVolume -- 19% increase (from 80% to 95%)
    FROM CurrentCapacity
)
SELECT 
    'Current' AS Scenario,
    BaselineDwell AS DwellMinutes,
    BaselineVolume AS DailyVolume
FROM CurrentCapacity
UNION ALL
SELECT 
    'Projected (95% Capacity)' AS Scenario,
    ProjectedDwell,
    ProjectedVolume
FROM ScenarioImpact;
```

## Troubleshooting

### Issue: Power BI Refresh Timeout

**Symptom**: Dashboard refresh exceeds gateway timeout (typically 10 minutes)

**Solution**: Implement incremental refresh policy

```powerquery
// Power Query M - Incremental Refresh Function
let
    Source = Sql.Database(SqlServer, Database),
    LastRefresh = DateTime.From(
        Value.NativeQuery(Source, "SELECT MAX(LoadTimestamp) FROM FactWarehouseOperations", null)
    ),
    IncrementalData = Sql.Database(SqlServer, Database, [
        Query = "SELECT * FROM FactWarehouseOperations WHERE LoadTimestamp > '" 
            & DateTime.ToText(LastRefresh, "yyyy-MM-dd HH:mm:ss") & "'"
    ])
in
    IncrementalData
```

### Issue: Cross-Fact Query Performance

**Symptom**: Queries joining multiple fact tables are slow

**Solution**: Create indexed views for common cross-fact patterns

```sql
CREATE VIEW vw_CrossModalEfficiency
WITH SCHEMABINDING
AS
SELECT 
    t.TimeKey,
    w.WarehouseKey,
    AVG(w.DwellTimeMinutes) AS AvgDwell,
    AVG(f.IdleTimeMinutes) AS AvgIdle,
    COUNT_BIG(*) AS RecordCount
FROM dbo.FactWarehouseOperations w
INNER JOIN dbo.DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN dbo.FactFleetTrips f ON t.TimeKey = f.TimeKey
GROUP BY t.TimeKey, w.WarehouseKey;

CREATE UNIQUE CLUSTERED INDEX IX_CrossModal 
ON vw_CrossModalEfficiency (TimeKey, WarehouseKey);
```

### Issue: Gravity Score Calculation Drift

**Symptom**: Gravity zones not reflecting current operations

**Solution**: Schedule weekly recalculation with anomaly detection

```sql
-- Detect significant gravity score changes
WITH GravityChanges AS (
    SELECT 
        ProductKey,
        SKU,
        GravityScore AS CurrentScore,
        LAG(GravityScore) OVER (PARTITION BY ProductKey ORDER BY LastRecalculated) AS PreviousScore
    FROM DimProductGravity
)
SELECT 
    SKU,
    CurrentScore,
    PreviousScore,
    ((CurrentScore - PreviousScore) / PreviousScore) * 100 AS PercentChange
FROM GravityChanges
WHERE ABS((CurrentScore - PreviousScore) / PreviousScore) > 0.20 -- 20% change threshold
ORDER BY ABS(PercentChange) DESC;
```

## Best Practices

1. **Partition fact tables by time** for better query performance:
```sql
ALTER TABLE FactWarehouseOperations 
SWITCH TO PartitionScheme_Time(TimeKey);
```

2. **Use staging tables** for external data ingestion to validate before loading to fact tables

3. **Implement audit logging** for all stored procedure executions:
```sql
CREATE TABLE AuditLog (
    LogID INT PRIMARY KEY IDENTITY,
    ProcedureName VARCHAR(100),
    ExecutionTime DATETIME DEFAULT GETDATE(),
    RowsAffected INT,
    ErrorMessage VARCHAR(MAX)
);
```

4. **Version control your Power BI reports** by exporting PBIX as PBIT templates regularly

5. **Test RLS rules** thoroughly before deploying to production:
```sql
EXECUTE AS USER = 'testuser@company.com';
SELECT * FROM FactWarehouseOperations; -- Should only return authorized rows
REVERT;
```

This skill provides comprehensive guidance for deploying, configuring, and operating the LogiFleet Pulse analytics platform with real-world SQL and DAX examples.
