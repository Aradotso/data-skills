---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI data warehousing platform for logistics, fleet management, and supply chain analytics with multi-fact star schema modeling
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure warehouse and fleet data model"
  - "create logistics star schema in SQL Server"
  - "build Power BI logistics dashboard"
  - "implement supply chain KPI tracking"
  - "deploy fleet and warehouse analytics"
  - "configure cross-modal logistics reporting"
  - "set up real-time supply chain monitoring"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is an advanced data warehousing and analytics platform for logistics and supply chain management. It provides:

- **Multi-fact star schema** for warehouse operations, fleet trips, and cross-dock transfers
- **Unified semantic layer** connecting inventory, fleet telemetry, and external data sources
- **Power BI dashboards** with 15-minute refresh cycles for real-time operational awareness
- **Predictive analytics** for bottleneck detection and maintenance prioritization
- **Cross-fact KPI harmonization** linking warehouse metrics with fleet performance

The platform uses MS SQL Server (2019+) for data warehousing and Power BI for visualization, designed for 3PL operators, retail chains, and distribution centers.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS)
- Access to source systems: WMS, TMS, telematics APIs

### Step 1: Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance via SSMS
-- Execute the main schema creation script
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the schema deployment script
-- (Assuming script is named schema_deployment.sql)
:r schema_deployment.sql
```

### Step 3: Configure Data Sources

Create a configuration file for connection strings:

```json
{
  "connections": {
    "sqlServer": "Server=${SQL_SERVER_HOST};Database=LogiFleetPulse;User Id=${SQL_USER};Password=${SQL_PASSWORD};",
    "wmsApi": {
      "baseUrl": "${WMS_API_URL}",
      "apiKey": "${WMS_API_KEY}"
    },
    "telematicsApi": {
      "baseUrl": "${TELEMATICS_API_URL}",
      "apiKey": "${TELEMATICS_API_KEY}"
    },
    "weatherApi": {
      "baseUrl": "${WEATHER_API_URL}",
      "apiKey": "${WEATHER_API_KEY}"
    }
  },
  "refreshInterval": 900
}
```

### Step 4: Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI template (`.pbit`)
3. Select `LogiFleet_Pulse_Master.pbit`
4. Enter SQL Server connection details when prompted

## Core Data Model Structure

### Fact Tables

#### FactWarehouseOperations

```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity INT NOT NULL,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes DECIMAL(10,2),
    ZoneGravityScore DECIMAL(5,2),
    OperatorID INT,
    BatchNumber VARCHAR(50),
    RecordedTimestamp DATETIME2 NOT NULL,
    CONSTRAINT FK_WarehouseOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WarehouseOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_WarehouseOps_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
);

-- Optimized indexing for common queries
CREATE NONCLUSTERED INDEX IX_WarehouseOps_Time_Product 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (Quantity, DwellTimeMinutes);

CREATE NONCLUSTERED INDEX IX_WarehouseOps_Type_Time 
ON FactWarehouseOperations(OperationType, TimeKey) 
INCLUDE (ProcessingTimeMinutes);
```

#### FactFleetTrips

```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    TripStartTime DATETIME2 NOT NULL,
    TripEndTime DATETIME2,
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    SpeedViolations INT DEFAULT 0,
    HarshBrakingEvents INT DEFAULT 0,
    TotalRevenue DECIMAL(12,2),
    WeatherCondition VARCHAR(50),
    TrafficDelayMinutes INT DEFAULT 0,
    CONSTRAINT FK_FleetTrips_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrips_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_FleetTrips_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey),
    CONSTRAINT FK_FleetTrips_Driver FOREIGN KEY (DriverKey) REFERENCES DimDriver(DriverKey),
    CONSTRAINT FK_FleetTrips_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE NONCLUSTERED INDEX IX_FleetTrips_Vehicle_Time 
ON FactFleetTrips(VehicleKey, TimeKey) 
INCLUDE (DistanceMiles, FuelConsumedGallons, IdleTimeMinutes);
```

### Dimension Tables

#### DimTime (Time-phased at 15-minute granularity)

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    Week INT NOT NULL,
    Day INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayName VARCHAR(20),
    Hour INT NOT NULL,
    Minute INT NOT NULL,
    QuarterHour INT NOT NULL, -- 0, 15, 30, 45
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL,
    FiscalYear INT,
    FiscalQuarter INT,
    ShiftType VARCHAR(20) -- 'Morning', 'Afternoon', 'Night', 'Weekend'
);

-- Populate time dimension (example for one day)
DECLARE @StartDate DATETIME2 = '2026-01-01 00:00:00';
DECLARE @EndDate DATETIME2 = '2026-12-31 23:45:00';
DECLARE @CurrentDate DATETIME2 = @StartDate;

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO DimTime (TimeKey, FullDateTime, Year, Quarter, Month, Week, Day, DayOfWeek, DayName, Hour, Minute, QuarterHour, IsWeekend, IsHoliday)
    VALUES (
        CAST(FORMAT(@CurrentDate, 'yyyyMMddHHmm') AS INT),
        @CurrentDate,
        YEAR(@CurrentDate),
        DATEPART(QUARTER, @CurrentDate),
        MONTH(@CurrentDate),
        DATEPART(WEEK, @CurrentDate),
        DAY(@CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        DATENAME(WEEKDAY, @CurrentDate),
        DATEPART(HOUR, @CurrentDate),
        DATEPART(MINUTE, @CurrentDate),
        (DATEPART(MINUTE, @CurrentDate) / 15) * 15,
        CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1, 7) THEN 1 ELSE 0 END,
        0 -- Update separately with holiday calendar
    );
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate);
END;
```

#### DimProductGravity (Warehouse Gravity Zones)

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200) NOT NULL,
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,2),
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    StorageTemperature VARCHAR(50), -- 'Ambient', 'Refrigerated', 'Frozen'
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value / (dwell_time * fragility_factor)
    VelocityClass VARCHAR(20), -- 'Fast-Mover', 'Medium-Mover', 'Slow-Mover', 'Dead-Stock'
    OptimalZone VARCHAR(50), -- 'High-Gravity', 'Mid-Gravity', 'Low-Gravity', 'Archive'
    LastGravityUpdate DATETIME2 DEFAULT GETDATE()
);

-- Stored procedure to recalculate gravity scores
CREATE PROCEDURE sp_RecalculateProductGravity
AS
BEGIN
    UPDATE DimProductGravity
    SET GravityScore = (
        SELECT 
            (COUNT(DISTINCT wo.OperationID) * AVG(p.UnitWeight * 100)) / 
            NULLIF(AVG(wo.DwellTimeMinutes) * CASE WHEN p.IsFragile = 1 THEN 2 ELSE 1 END, 0)
        FROM FactWarehouseOperations wo
        WHERE wo.ProductKey = DimProductGravity.ProductKey
            AND wo.RecordedTimestamp >= DATEADD(DAY, -30, GETDATE())
    ),
    VelocityClass = CASE
        WHEN GravityScore >= 75 THEN 'Fast-Mover'
        WHEN GravityScore >= 40 THEN 'Medium-Mover'
        WHEN GravityScore >= 10 THEN 'Slow-Mover'
        ELSE 'Dead-Stock'
    END,
    OptimalZone = CASE
        WHEN GravityScore >= 75 THEN 'High-Gravity'
        WHEN GravityScore >= 40 THEN 'Mid-Gravity'
        WHEN GravityScore >= 10 THEN 'Low-Gravity'
        ELSE 'Archive'
    END,
    LastGravityUpdate = GETDATE();
END;
```

## Key Stored Procedures

### Incremental Data Loading

```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new warehouse operations from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType, 
        Quantity, DwellTimeMinutes, ProcessingTimeMinutes, 
        ZoneGravityScore, OperatorID, BatchNumber, RecordedTimestamp
    )
    SELECT 
        CAST(FORMAT(stg.OperationTimestamp, 'yyyyMMddHHmm') AS INT) / 100 * 100 AS TimeKey, -- Round to nearest 15-min
        p.ProductKey,
        w.WarehouseKey,
        stg.OperationType,
        stg.Quantity,
        DATEDIFF(MINUTE, stg.ItemReceivedTime, stg.OperationTimestamp) AS DwellTimeMinutes,
        stg.ProcessingTimeMinutes,
        p.GravityScore AS ZoneGravityScore,
        stg.OperatorID,
        stg.BatchNumber,
        stg.OperationTimestamp
    FROM StagingWarehouseOperations stg
    INNER JOIN DimProduct p ON stg.SKU = p.SKU
    INNER JOIN DimWarehouse w ON stg.WarehouseCode = w.WarehouseCode
    WHERE stg.OperationTimestamp > @LastLoadTimestamp
        AND stg.IsProcessed = 0;
    
    -- Mark staging records as processed
    UPDATE StagingWarehouseOperations
    SET IsProcessed = 1, ProcessedTimestamp = GETDATE()
    WHERE OperationTimestamp > @LastLoadTimestamp
        AND IsProcessed = 0;
END;
```

### Cross-Fact KPI Query

```sql
CREATE PROCEDURE sp_GetCrossFactKPI
    @StartDate DATETIME2,
    @EndDate DATETIME2
AS
BEGIN
    -- Query combining warehouse dwell time with fleet idle time
    SELECT 
        t.Year,
        t.Month,
        t.Week,
        p.Category,
        g.Region,
        -- Warehouse metrics
        AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
        SUM(wo.Quantity) AS TotalUnitsProcessed,
        -- Fleet metrics
        AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
        SUM(ft.FuelConsumedGallons) AS TotalFuelConsumed,
        -- Cross-fact KPI: Efficiency ratio
        CASE 
            WHEN AVG(wo.DwellTimeMinutes) > 0 
            THEN (SUM(wo.Quantity) / AVG(wo.DwellTimeMinutes)) / NULLIF(AVG(ft.IdleTimeMinutes), 0)
            ELSE 0 
        END AS EfficiencyRatio
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    INNER JOIN FactFleetTrips ft ON ft.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON ft.GeographyKey = g.GeographyKey
    WHERE t.FullDateTime BETWEEN @StartDate AND @EndDate
    GROUP BY t.Year, t.Month, t.Week, p.Category, g.Region
    ORDER BY t.Year, t.Month, t.Week;
END;
```

### Predictive Bottleneck Detection

```sql
CREATE PROCEDURE sp_PredictBottlenecks
    @ForecastHours INT = 4
AS
BEGIN
    -- Identify probable congestion points using historical patterns
    WITH RecentPatterns AS (
        SELECT 
            wo.WarehouseKey,
            wo.OperationType,
            t.Hour,
            t.DayOfWeek,
            AVG(wo.ProcessingTimeMinutes) AS AvgProcessingTime,
            STDEV(wo.ProcessingTimeMinutes) AS StdDevProcessingTime,
            COUNT(*) AS OperationCount
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE wo.RecordedTimestamp >= DATEADD(DAY, -30, GETDATE())
        GROUP BY wo.WarehouseKey, wo.OperationType, t.Hour, t.DayOfWeek
    ),
    CurrentLoad AS (
        SELECT 
            wo.WarehouseKey,
            COUNT(*) AS ActiveOperations,
            AVG(wo.ProcessingTimeMinutes) AS CurrentAvgProcessing
        FROM FactWarehouseOperations wo
        WHERE wo.RecordedTimestamp >= DATEADD(HOUR, -1, GETDATE())
        GROUP BY wo.WarehouseKey
    )
    SELECT 
        w.WarehouseName,
        rp.OperationType,
        DATEADD(HOUR, @ForecastHours, GETDATE()) AS ForecastTime,
        rp.AvgProcessingTime,
        cl.CurrentAvgProcessing,
        CASE 
            WHEN cl.CurrentAvgProcessing > (rp.AvgProcessingTime + rp.StdDevProcessingTime) THEN 'High Risk'
            WHEN cl.CurrentAvgProcessing > rp.AvgProcessingTime THEN 'Medium Risk'
            ELSE 'Low Risk'
        END AS BottleneckRisk,
        rp.OperationCount AS HistoricalVolume,
        cl.ActiveOperations AS CurrentVolume
    FROM RecentPatterns rp
    INNER JOIN CurrentLoad cl ON rp.WarehouseKey = cl.WarehouseKey
    INNER JOIN DimWarehouse w ON rp.WarehouseKey = w.WarehouseKey
    WHERE DATEPART(HOUR, DATEADD(HOUR, @ForecastHours, GETDATE())) = rp.Hour
        AND DATEPART(WEEKDAY, DATEADD(HOUR, @ForecastHours, GETDATE())) = rp.DayOfWeek
    ORDER BY BottleneckRisk DESC, CurrentVolume DESC;
END;
```

## Power BI DAX Measures

### Core KPIs

```dax
// Total Dwell Time
Total Dwell Time = 
SUM(FactWarehouseOperations[DwellTimeMinutes])

// Average Processing Efficiency
Avg Processing Efficiency = 
DIVIDE(
    SUM(FactWarehouseOperations[Quantity]),
    SUM(FactWarehouseOperations[ProcessingTimeMinutes]),
    0
)

// Fleet Idle Percentage
Fleet Idle % = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[IdleTimeMinutes]) + 
    SUMX(FactFleetTrips, DATEDIFF(FactFleetTrips[TripStartTime], FactFleetTrips[TripEndTime], MINUTE)),
    0
) * 100

// Fuel Efficiency (Miles per Gallon)
Fuel Efficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceMiles]),
    SUM(FactFleetTrips[FuelConsumedGallons]),
    0
)

// Cross-Fact: Revenue per Warehouse Hour
Revenue per Warehouse Hour = 
DIVIDE(
    SUM(FactFleetTrips[TotalRevenue]),
    SUM(FactWarehouseOperations[ProcessingTimeMinutes]) / 60,
    0
)
```

### Time Intelligence Measures

```dax
// Previous Period Comparison
Dwell Time LM = 
CALCULATE(
    [Total Dwell Time],
    DATEADD(DimTime[FullDateTime], -1, MONTH)
)

Dwell Time Change % = 
DIVIDE(
    [Total Dwell Time] - [Dwell Time LM],
    [Dwell Time LM],
    0
) * 100

// Moving Average (7-day)
Dwell Time MA7 = 
CALCULATE(
    [Total Dwell Time],
    DATESINPERIOD(DimTime[FullDateTime], LASTDATE(DimTime[FullDateTime]), -7, DAY)
) / 7
```

### Gravity Zone Optimization

```dax
// Products in Wrong Zone
Products Misplaced = 
CALCULATE(
    DISTINCTCOUNT(FactWarehouseOperations[ProductKey]),
    FILTER(
        DimProductGravity,
        DimProductGravity[OptimalZone] <> RELATED(FactWarehouseOperations[ZoneGravityScore])
    )
)

// Potential Time Savings
Potential Time Savings = 
SUMX(
    FILTER(
        FactWarehouseOperations,
        RELATED(DimProductGravity[OptimalZone]) <> FactWarehouseOperations[ZoneGravityScore]
    ),
    FactWarehouseOperations[DwellTimeMinutes] * 0.15  // Assumes 15% improvement
)
```

## Automated Alerting System

### Alert Configuration Table

```sql
CREATE TABLE AlertRules (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName NVARCHAR(100) NOT NULL,
    MetricType VARCHAR(50) NOT NULL, -- 'DwellTime', 'IdleTime', 'FuelEfficiency', etc.
    ThresholdValue DECIMAL(10,2) NOT NULL,
    ComparisonOperator VARCHAR(10) NOT NULL, -- '>', '<', '>=', '<=', '='
    EvaluationWindowMinutes INT NOT NULL,
    AlertSeverity VARCHAR(20) NOT NULL, -- 'Critical', 'Warning', 'Info'
    NotificationChannels VARCHAR(200), -- 'Email,SMS,Teams'
    RecipientList NVARCHAR(500),
    IsActive BIT DEFAULT 1,
    CreatedDate DATETIME2 DEFAULT GETDATE()
);

-- Example alert rules
INSERT INTO AlertRules (AlertName, MetricType, ThresholdValue, ComparisonOperator, EvaluationWindowMinutes, AlertSeverity, NotificationChannels, RecipientList)
VALUES 
    ('High Fleet Idle Time', 'IdleTime', 15.0, '>', 60, 'Warning', 'Email,Teams', '${ALERT_EMAIL_LOGISTICS}'),
    ('Critical Dwell Time', 'DwellTime', 240, '>', 30, 'Critical', 'Email,SMS,Teams', '${ALERT_EMAIL_WAREHOUSE_MGR}'),
    ('Low Fuel Efficiency', 'FuelEfficiency', 6.5, '<', 120, 'Warning', 'Email', '${ALERT_EMAIL_FLEET_MGR}');
```

### Alert Evaluation Procedure

```sql
CREATE PROCEDURE sp_EvaluateAlerts
AS
BEGIN
    DECLARE @AlertID INT, @MetricType VARCHAR(50), @ThresholdValue DECIMAL(10,2);
    DECLARE @ComparisonOperator VARCHAR(10), @EvaluationWindowMinutes INT;
    DECLARE @AlertSeverity VARCHAR(20), @NotificationChannels VARCHAR(200), @RecipientList NVARCHAR(500);
    
    DECLARE alert_cursor CURSOR FOR
    SELECT AlertID, MetricType, ThresholdValue, ComparisonOperator, 
           EvaluationWindowMinutes, AlertSeverity, NotificationChannels, RecipientList
    FROM AlertRules
    WHERE IsActive = 1;
    
    OPEN alert_cursor;
    FETCH NEXT FROM alert_cursor INTO @AlertID, @MetricType, @ThresholdValue, 
        @ComparisonOperator, @EvaluationWindowMinutes, @AlertSeverity, @NotificationChannels, @RecipientList;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        DECLARE @CurrentValue DECIMAL(10,2);
        DECLARE @AlertTriggered BIT = 0;
        
        -- Evaluate based on metric type
        IF @MetricType = 'IdleTime'
        BEGIN
            SELECT @CurrentValue = AVG(IdleTimeMinutes)
            FROM FactFleetTrips
            WHERE TripStartTime >= DATEADD(MINUTE, -@EvaluationWindowMinutes, GETDATE());
            
            IF (@ComparisonOperator = '>' AND @CurrentValue > @ThresholdValue) OR
               (@ComparisonOperator = '<' AND @CurrentValue < @ThresholdValue)
                SET @AlertTriggered = 1;
        END
        ELSE IF @MetricType = 'DwellTime'
        BEGIN
            SELECT @CurrentValue = AVG(DwellTimeMinutes)
            FROM FactWarehouseOperations
            WHERE RecordedTimestamp >= DATEADD(MINUTE, -@EvaluationWindowMinutes, GETDATE());
            
            IF (@ComparisonOperator = '>' AND @CurrentValue > @ThresholdValue) OR
               (@ComparisonOperator = '<' AND @CurrentValue < @ThresholdValue)
                SET @AlertTriggered = 1;
        END
        ELSE IF @MetricType = 'FuelEfficiency'
        BEGIN
            SELECT @CurrentValue = SUM(DistanceMiles) / NULLIF(SUM(FuelConsumedGallons), 0)
            FROM FactFleetTrips
            WHERE TripStartTime >= DATEADD(MINUTE, -@EvaluationWindowMinutes, GETDATE());
            
            IF (@ComparisonOperator = '<' AND @CurrentValue < @ThresholdValue)
                SET @AlertTriggered = 1;
        END
        
        -- Log alert if triggered
        IF @AlertTriggered = 1
        BEGIN
            INSERT INTO AlertLog (AlertID, TriggeredTimestamp, MetricValue, Severity, Recipients)
            VALUES (@AlertID, GETDATE(), @CurrentValue, @AlertSeverity, @RecipientList);
            
            -- Send notification (integrate with email/SMS service)
            -- EXEC sp_SendNotification @NotificationChannels, @RecipientList, @MetricType, @CurrentValue, @AlertSeverity;
        END
        
        FETCH NEXT FROM alert_cursor INTO @AlertID, @MetricType, @ThresholdValue, 
            @ComparisonOperator, @EvaluationWindowMinutes, @AlertSeverity, @NotificationChannels, @RecipientList;
    END
    
    CLOSE alert_cursor;
    DEALLOCATE alert_cursor;
END;
```

### Schedule Alert Evaluation (SQL Server Agent Job)

```sql
-- Create SQL Agent Job to run every 15 minutes
USE msdb;
GO

EXEC sp_add_job
    @job_name = N'LogiFleet_Alert_Evaluation',
    @enabled = 1,
    @description = N'Evaluates alert rules for LogiFleet Pulse every 15 minutes';

EXEC sp_add_jobstep
    @job_name = N'LogiFleet_Alert_Evaluation',
    @step_name = N'Run Alert Evaluation',
    @subsystem = N'TSQL',
    @command = N'EXEC sp_EvaluateAlerts',
    @database_name = N'LogiFleetPulse';

EXEC sp_add_schedule
    @schedule_name = N'Every_15_Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC sp_attach_schedule
    @job_name = N'LogiFleet_Alert_Evaluation',
    @schedule_name = N'Every_15_Minutes';

EXEC sp_add_jobserver
    @job_name = N'LogiFleet_Alert_Evaluation';
```

## Common Patterns & Best Practices

### Pattern 1: Incremental ETL with Change Tracking

```sql
-- Enable change tracking on source tables
ALTER DATABASE LogiFleetPulse
SET CHANGE_TRACKING = ON
(CHANGE_RETENTION = 7 DAYS, AUTO_CLEANUP = ON);

ALTER TABLE FactWarehouseOperations
ENABLE CHANGE_TRACKING;

-- Query changes since last load
DECLARE @LastSyncVersion BIGINT = 12345; -- Store this value between loads

SELECT 
    wo.*,
    CT.SYS_CHANGE_OPERATION AS ChangeType
FROM FactWarehouseOperations wo
RIGHT OUTER JOIN CHANGETABLE(CHANGES FactWarehouseOperations, @LastSyncVersion) AS CT
    ON wo.OperationID = CT.OperationID
WHERE CT.SYS_CHANGE_VERSION > @LastSyncVersion;
```

### Pattern 2: Slowly Changing Dimension (Type 2) for Product Gravity

```sql
-- Add SCD columns to DimProductGravity
ALTER TABLE DimProductGravity
ADD EffectiveDate DATETIME2 DEFAULT GETDATE(),
    ExpirationDate DATETIME2,
    IsCurrent BIT DEFAULT 1;

-- Procedure to handle gravity score updates with history
CREATE PROCEDURE sp_UpdateProductGravityWithHistory
    @ProductKey INT,
    @NewGravityScore DECIMAL(5,2),
    @NewVelocityClass VARCHAR(20),
    @NewOptimalZone VARCHAR(50)
AS
BEGIN
    -- Expire current record
    UPDATE DimProductGravity
    SET ExpirationDate = GETDATE(),
        IsCurrent = 0
    WHERE ProductKey = @ProductKey
        AND IsCurrent = 1;
    
    -- Insert new record
    INSERT INTO DimProductGravity (
        SKU, ProductName, Category, Subcategory, UnitWeight, UnitVolume,
        IsFragile, IsPerishable, StorageTemperature,
        GravityScore, VelocityClass, OptimalZone,
        EffectiveDate, IsCurrent
    )
    SELECT 
        SKU, ProductName, Category, Subcategory, UnitWeight, UnitVolume,
        IsFragile, IsPerishable, StorageTemperature,
        @NewGravityScore, @NewVelocityClass, @NewOptimalZone,
        GETDATE(), 1
    FROM DimProductGravity
    WHERE ProductKey = @ProductKey
        AND IsCurrent = 0
        AND ExpirationDate = GETDATE();
END;
```

### Pattern 3: Row-Level Security for Multi-Tenant Access

```sql
-- Create security predicate function
CREATE FUNCTION fn_SecurityPredicate(@WarehouseKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS AccessPermitted
    WHERE @WarehouseKey IN (
        SELECT WarehouseKey 
        FROM dbo.UserWarehouseAccess 
        WHERE UserName = USER_NAME()
    )
    OR IS_MEMBER('db_owner') = 1
);

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(WarehouseKey)
ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(WarehouseKey)
ON dbo.DimWarehouse
WITH (STATE = ON);
```

## Troubleshooting

### Issue: Power BI Refresh Fails with Timeout

**Solution**: Implement incremental refresh in Power BI

1. In Power BI Desktop, create parameters `RangeStart` and `RangeEnd` (DateTime)
2. Filter fact tables using these parameters:
   ```
   = Table.SelectRows(
