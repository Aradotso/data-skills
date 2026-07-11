---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing template for logistics, fleet management, and supply chain intelligence with multi-fact star schema
triggers:
  - set up logistics analytics dashboard
  - deploy supply chain data warehouse
  - configure fleet management Power BI reports
  - implement warehouse operations star schema
  - create cross-modal logistics KPIs
  - integrate fleet telemetry with warehouse data
  - build predictive supply chain analytics
  - optimize logistics data model in SQL Server
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and analytics template for supply chain and logistics operations. It provides:

- **Multi-fact star schema** for warehouse operations, fleet trips, and cross-dock activities
- **Power BI dashboard templates** for real-time logistics intelligence
- **SQL Server database schema** with time-phased dimensions and bridge tables
- **Cross-fact KPI harmonization** linking warehouse metrics with fleet performance
- **Predictive analytics** for bottleneck detection and maintenance prioritization

The system integrates data from warehouse management systems (WMS), fleet telematics, supplier portals, and external APIs (weather, traffic) into a unified semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version recommended)
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Step 1: Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance via SSMS
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the main schema script (assumed to be in /sql directory)
-- Execute schema_deployment.sql from the repository
```

### Step 3: Configure Data Sources

Create a configuration file for your data connections:

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "SQL",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_api": "${TELEMATICS_API_ENDPOINT}",
    "weather_api": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

### Step 4: Import Power BI Template

1. Open Power BI Desktop
2. File → Open → Select `LogiFleet_Pulse_Master.pbit`
3. Enter connection parameters when prompted
4. Refresh data to validate connections

## Core Database Schema

### Fact Tables

**FactWarehouseOperations** - Tracks all warehouse activities

```sql
CREATE TABLE FactWarehouseOperations (
    OperationID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(20), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled INT,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes DECIMAL(10,2),
    GravityZoneID INT,
    BatchNumber VARCHAR(50),
    RecordedTimestamp DATETIME DEFAULT GETDATE(),
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_WH_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
);

-- Recommended indexes
CREATE NONCLUSTERED INDEX IX_FactWH_Time ON FactWarehouseOperations(TimeKey) INCLUDE (DwellTimeMinutes);
CREATE NONCLUSTERED INDEX IX_FactWH_Product ON FactWarehouseOperations(ProductKey, OperationType);
CREATE NONCLUSTERED INDEX IX_FactWH_GravityZone ON FactWarehouseOperations(GravityZoneID, TimeKey);
```

**FactFleetTrips** - Captures fleet and vehicle telemetry

```sql
CREATE TABLE FactFleetTrips (
    TripID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    TripStartTime DATETIME,
    TripEndTime DATETIME,
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    WeightLbs INT,
    DelayMinutes INT,
    DelayReason VARCHAR(100),
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_Fleet_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleKey, TripStartTime);
CREATE NONCLUSTERED INDEX IX_FactFleet_Delay ON FactFleetTrips(DelayMinutes) WHERE DelayMinutes > 0;
```

### Dimension Tables

**DimTime** - 15-minute granularity time dimension

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    [Year] INT,
    Quarter INT,
    [Month] INT,
    [Week] INT,
    [Day] INT,
    DayOfWeek INT,
    [Hour] INT,
    MinuteBucket INT, -- 0, 15, 30, 45
    IsWeekend BIT,
    FiscalPeriod VARCHAR(10)
);

-- Sample population script
CREATE PROCEDURE PopulateDimTime
    @StartDate DATETIME,
    @EndDate DATETIME
AS
BEGIN
    DECLARE @CurrentDateTime DATETIME = @StartDate;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        INSERT INTO DimTime (
            TimeKey, FullDateTime, [Year], Quarter, [Month], [Week], 
            [Day], DayOfWeek, [Hour], MinuteBucket, IsWeekend
        )
        VALUES (
            CONVERT(INT, FORMAT(@CurrentDateTime, 'yyyyMMddHHmm')),
            @CurrentDateTime,
            YEAR(@CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            MONTH(@CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            DAY(@CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATEPART(HOUR, @CurrentDateTime),
            (DATEPART(MINUTE, @CurrentDateTime) / 15) * 15,
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
```

**DimProductGravity** - Product classification with gravity scoring

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(50),
    VelocityScore DECIMAL(5,2), -- Pick frequency
    ValueScore DECIMAL(5,2), -- Unit value
    FragilityScore DECIMAL(5,2),
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2),
    RecommendedZone VARCHAR(20),
    LastUpdated DATETIME DEFAULT GETDATE()
);

-- Gravity score calculation stored procedure
CREATE PROCEDURE CalculateProductGravity
AS
BEGIN
    UPDATE DimProductGravity
    SET VelocityScore = (
        SELECT COUNT(*) * 1.0 / NULLIF(DATEDIFF(DAY, MIN(f.RecordedTimestamp), MAX(f.RecordedTimestamp)), 0)
        FROM FactWarehouseOperations f
        WHERE f.ProductKey = DimProductGravity.ProductKey
          AND f.OperationType = 'Picking'
          AND f.RecordedTimestamp >= DATEADD(DAY, -90, GETDATE())
    ),
    LastUpdated = GETDATE();
    
    -- Update recommended zones based on gravity score
    UPDATE DimProductGravity
    SET RecommendedZone = CASE
        WHEN GravityScore >= 8.0 THEN 'Zone-A-HighGravity'
        WHEN GravityScore >= 5.0 THEN 'Zone-B-MediumGravity'
        ELSE 'Zone-C-LowGravity'
    END;
END;
```

## Common Data Loading Patterns

### Incremental Load from WMS

```sql
CREATE PROCEDURE LoadWarehouseOperations
    @LastLoadTimestamp DATETIME
AS
BEGIN
    -- Assuming external table or linked server to WMS
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType,
        QuantityHandled, DwellTimeMinutes, ProcessingTimeMinutes,
        BatchNumber, RecordedTimestamp
    )
    SELECT 
        CONVERT(INT, FORMAT(w.EventTimestamp, 'yyyyMMddHHmm')) / 15 * 15 AS TimeKey,
        p.ProductKey,
        wh.WarehouseKey,
        w.OperationType,
        w.Quantity,
        DATEDIFF(MINUTE, w.StartTime, w.EndTime) AS DwellTimeMinutes,
        w.ProcessingMinutes,
        w.BatchNumber,
        w.EventTimestamp
    FROM ExternalWMS.dbo.WarehouseEvents w
    INNER JOIN DimProduct p ON w.SKU = p.SKU
    INNER JOIN DimWarehouse wh ON w.WarehouseCode = wh.WarehouseCode
    WHERE w.EventTimestamp > @LastLoadTimestamp;
    
    -- Update last load timestamp in control table
    UPDATE DataLoadControl
    SET LastLoadTimestamp = GETDATE()
    WHERE SourceSystem = 'WMS';
END;
```

### Fleet Telemetry Streaming

```sql
CREATE PROCEDURE LoadFleetTripData
    @TripJSON NVARCHAR(MAX)
AS
BEGIN
    -- Parse JSON from telematics API
    INSERT INTO FactFleetTrips (
        TimeKey, VehicleKey, RouteKey, DriverKey,
        TripStartTime, TripEndTime, DistanceMiles,
        FuelConsumedGallons, IdleTimeMinutes
    )
    SELECT 
        CONVERT(INT, FORMAT(JSON_VALUE(value, '$.start_time'), 'yyyyMMddHHmm')),
        v.VehicleKey,
        r.RouteKey,
        d.DriverKey,
        JSON_VALUE(value, '$.start_time'),
        JSON_VALUE(value, '$.end_time'),
        JSON_VALUE(value, '$.distance'),
        JSON_VALUE(value, '$.fuel_consumed'),
        JSON_VALUE(value, '$.idle_minutes')
    FROM OPENJSON(@TripJSON) 
    INNER JOIN DimVehicle v ON JSON_VALUE(value, '$.vehicle_id') = v.VehicleID
    INNER JOIN DimRoute r ON JSON_VALUE(value, '$.route_id') = r.RouteID
    INNER JOIN DimDriver d ON JSON_VALUE(value, '$.driver_id') = d.DriverID;
END;
```

## Cross-Fact KPI Queries

### Dwell Time vs Fleet Idling Cost

```sql
-- Correlate warehouse dwell time with fleet idle costs
SELECT 
    p.Category,
    AVG(w.DwellTimeMinutes) AS AvgDwellMinutes,
    AVG(f.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    SUM(f.FuelConsumedGallons * 3.50) AS TotalFuelCost, -- $3.50/gallon
    COUNT(DISTINCT w.OperationID) AS WarehouseOps,
    COUNT(DISTINCT f.TripID) AS FleetTrips
FROM FactWarehouseOperations w
INNER JOIN DimProduct p ON w.ProductKey = p.ProductKey
INNER JOIN FactFleetTrips f ON w.TimeKey = f.TimeKey
WHERE w.OperationType = 'Shipping'
  AND w.RecordedTimestamp >= DATEADD(DAY, -30, GETDATE())
GROUP BY p.Category
HAVING AVG(w.DwellTimeMinutes) > 60
ORDER BY AvgDwellMinutes DESC;
```

### Gravity Zone Optimization

```sql
-- Find products in wrong gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.RecommendedZone,
    w.GravityZoneID AS CurrentZone,
    COUNT(*) AS PickOperations,
    AVG(w.ProcessingTimeMinutes) AS AvgProcessingTime
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE w.OperationType = 'Picking'
  AND w.GravityZoneID != p.RecommendedZone
  AND w.RecordedTimestamp >= DATEADD(DAY, -7, GETDATE())
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.RecommendedZone, w.GravityZoneID
ORDER BY COUNT(*) DESC;
```

## Power BI DAX Measures

### Fleet Utilization Rate

```dax
Fleet Utilization % = 
VAR TotalAvailableHours = 
    SUMX(
        VALUES(DimVehicle[VehicleKey]),
        DATEDIFF(MIN(DimTime[FullDateTime]), MAX(DimTime[FullDateTime]), HOUR)
    )
VAR ActiveHours = 
    SUMX(
        FactFleetTrips,
        DATEDIFF(FactFleetTrips[TripStartTime], FactFleetTrips[TripEndTime], HOUR)
    )
RETURN
    DIVIDE(ActiveHours, TotalAvailableHours, 0) * 100
```

### Predictive Bottleneck Index

```dax
Bottleneck Risk Index = 
VAR CurrentDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR HistoricalAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATEADD(DimTime[FullDateTime], -30, DAY)
    )
VAR FleetDelayFactor = 
    DIVIDE(
        CALCULATE(SUM(FactFleetTrips[DelayMinutes])),
        CALCULATE(COUNT(FactFleetTrips[TripID])),
        0
    )
RETURN
    (CurrentDwellTime / HistoricalAvg - 1) * 50 + FleetDelayFactor * 0.5
```

### Warehouse Gravity Score

```dax
Warehouse Efficiency Score = 
VAR ZoneAOperations = 
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationID]),
        DimProductGravity[RecommendedZone] = "Zone-A-HighGravity"
    )
VAR TotalOperations = COUNT(FactWarehouseOperations[OperationID])
VAR AvgProcessingTime = AVERAGE(FactWarehouseOperations[ProcessingTimeMinutes])
RETURN
    (DIVIDE(ZoneAOperations, TotalOperations, 0) * 40) + 
    (1 / (AvgProcessingTime + 1)) * 60
```

## Automated Alerting System

### Create Alert Configuration Table

```sql
CREATE TABLE AlertConfiguration (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName VARCHAR(100),
    MetricQuery NVARCHAR(MAX), -- SQL query returning single value
    ThresholdValue DECIMAL(18,2),
    ThresholdOperator VARCHAR(10), -- '>', '<', '=', '>=', '<='
    NotificationEmail VARCHAR(255),
    IsActive BIT DEFAULT 1,
    CheckFrequencyMinutes INT DEFAULT 15
);

-- Sample alert configuration
INSERT INTO AlertConfiguration (AlertName, MetricQuery, ThresholdValue, ThresholdOperator, NotificationEmail)
VALUES (
    'Fleet Idle Time Excessive',
    'SELECT AVG(IdleTimeMinutes) FROM FactFleetTrips WHERE TripStartTime >= DATEADD(HOUR, -1, GETDATE())',
    15.0,
    '>',
    '${ALERT_EMAIL}'
);
```

### Alert Processing Procedure

```sql
CREATE PROCEDURE ProcessAlerts
AS
BEGIN
    DECLARE @AlertID INT, @MetricValue DECIMAL(18,2), @Threshold DECIMAL(18,2);
    DECLARE @Operator VARCHAR(10), @Email VARCHAR(255), @AlertName VARCHAR(100);
    
    DECLARE alert_cursor CURSOR FOR
    SELECT AlertID, AlertName, MetricQuery, ThresholdValue, ThresholdOperator, NotificationEmail
    FROM AlertConfiguration
    WHERE IsActive = 1;
    
    OPEN alert_cursor;
    FETCH NEXT FROM alert_cursor INTO @AlertID, @AlertName, @MetricQuery, @Threshold, @Operator, @Email;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Execute metric query
        EXEC sp_executesql @MetricQuery, N'@Result DECIMAL(18,2) OUTPUT', @MetricValue OUTPUT;
        
        -- Check threshold
        IF (@Operator = '>' AND @MetricValue > @Threshold) OR
           (@Operator = '<' AND @MetricValue < @Threshold) OR
           (@Operator = '>=' AND @MetricValue >= @Threshold) OR
           (@Operator = '<=' AND @MetricValue <= @Threshold)
        BEGIN
            -- Log alert
            INSERT INTO AlertLog (AlertID, MetricValue, TriggeredTime)
            VALUES (@AlertID, @MetricValue, GETDATE());
            
            -- Send notification (use Database Mail)
            EXEC msdb.dbo.sp_send_dbmail
                @profile_name = 'LogiFleet',
                @recipients = @Email,
                @subject = @AlertName,
                @body = CONCAT('Alert triggered: ', @AlertName, ' | Value: ', @MetricValue, ' | Threshold: ', @Threshold);
        END;
        
        FETCH NEXT FROM alert_cursor INTO @AlertID, @AlertName, @MetricQuery, @Threshold, @Operator, @Email;
    END;
    
    CLOSE alert_cursor;
    DEALLOCATE alert_cursor;
END;
```

## Row-Level Security Implementation

```sql
-- Create security table
CREATE TABLE UserSecurity (
    UserID INT PRIMARY KEY,
    Username VARCHAR(100),
    Email VARCHAR(255),
    AllowedWarehouses VARCHAR(MAX), -- Comma-separated warehouse keys
    AllowedRegions VARCHAR(MAX),
    RoleLevel VARCHAR(20) -- 'User', 'Supervisor', 'Executive'
);

-- Create security function
CREATE FUNCTION fn_SecurityFilter(@Username VARCHAR(100))
RETURNS TABLE
AS
RETURN
(
    SELECT WarehouseKey
    FROM STRING_SPLIT(
        (SELECT AllowedWarehouses FROM UserSecurity WHERE Username = @Username),
        ','
    )
);

-- Apply to fact table view
CREATE VIEW vw_SecureWarehouseOperations
AS
SELECT f.*
FROM FactWarehouseOperations f
WHERE f.WarehouseKey IN (SELECT CAST(value AS INT) FROM fn_SecurityFilter(SUSER_SNAME()))
   OR (SELECT RoleLevel FROM UserSecurity WHERE Username = SUSER_SNAME()) = 'Executive';
```

## Troubleshooting

### Performance Issues

**Problem**: Slow dashboard refresh times

**Solution**: Implement incremental refresh in Power BI

```sql
-- Create view for incremental refresh
CREATE VIEW vw_IncrementalWarehouseOps
AS
SELECT *
FROM FactWarehouseOperations
WHERE RecordedTimestamp >= DATEADD(DAY, -90, GETDATE());
```

In Power BI, configure incremental refresh policy:
- Archive data older than 2 years
- Refresh data in the last 90 days

**Problem**: Query timeout on cross-fact queries

**Solution**: Create indexed views for common joins

```sql
CREATE VIEW vw_WarehouseFleetCrossMetrics
WITH SCHEMABINDING
AS
SELECT 
    w.TimeKey,
    w.ProductKey,
    w.DwellTimeMinutes,
    f.IdleTimeMinutes,
    f.FuelConsumedGallons,
    COUNT_BIG(*) AS RecordCount
FROM dbo.FactWarehouseOperations w
INNER JOIN dbo.FactFleetTrips f ON w.TimeKey = f.TimeKey
GROUP BY w.TimeKey, w.ProductKey, w.DwellTimeMinutes, f.IdleTimeMinutes, f.FuelConsumedGallons;

CREATE UNIQUE CLUSTERED INDEX IX_CrossMetrics 
ON vw_WarehouseFleetCrossMetrics(TimeKey, ProductKey);
```

### Data Quality Issues

**Problem**: Missing time dimension records

**Solution**: Add validation check before data load

```sql
CREATE PROCEDURE ValidateTimeDimension
    @TargetDate DATETIME
AS
BEGIN
    IF NOT EXISTS (SELECT 1 FROM DimTime WHERE CAST(FullDateTime AS DATE) = CAST(@TargetDate AS DATE))
    BEGIN
        RAISERROR('Time dimension missing for date %s', 16, 1, @TargetDate);
        RETURN;
    END;
END;
```

**Problem**: Duplicate records in fact tables

**Solution**: Add unique constraint on business key

```sql
ALTER TABLE FactFleetTrips
ADD CONSTRAINT UQ_FleetTrip UNIQUE (VehicleKey, TripStartTime);
```

### Power BI Connection Issues

**Problem**: Cannot connect to SQL Server

**Solution**: Verify connection string and firewall rules

```powershell
# Test SQL Server connectivity
Test-NetConnection -ComputerName ${SQL_SERVER_HOST} -Port 1433

# Verify SQL authentication
sqlcmd -S ${SQL_SERVER_HOST} -U ${SQL_USERNAME} -P ${SQL_PASSWORD} -Q "SELECT @@VERSION"
```

## Best Practices

1. **Partition large fact tables** by time range for better query performance
2. **Use columnstore indexes** for analytical queries on fact tables with > 1M rows
3. **Schedule alerts during off-peak hours** to avoid resource contention
4. **Version control Power BI templates** using Git LFS for `.pbit` files
5. **Document custom DAX measures** with comments explaining business logic
6. **Implement data retention policies** to archive historical data beyond 2 years
7. **Use parameterized queries** in Power BI for dynamic filtering
8. **Enable Power BI incremental refresh** for fact tables with daily growth
9. **Create separate data marts** for different business units using views
10. **Monitor SQL Server performance** with Extended Events for slow queries
