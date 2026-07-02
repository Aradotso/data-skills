---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing framework for logistics, fleet management, and supply chain KPI analytics with multi-fact star schema
triggers:
  - set up logistics data warehouse
  - configure supply chain analytics dashboard
  - implement fleet management KPI tracking
  - create warehouse operations star schema
  - build Power BI logistics visualization
  - integrate supply chain intelligence system
  - deploy multi-fact logistics data model
  - analyze cross-modal fleet performance
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

LogiFleet Pulse is a comprehensive data warehousing and business intelligence framework for supply chain analytics. It provides a multi-fact star schema architecture combining warehouse operations, fleet telemetry, inventory management, and cross-dock logistics into a unified semantic layer. Built on MS SQL Server with Power BI dashboards, it enables real-time KPI tracking, predictive bottleneck detection, and cross-functional logistics optimization.

**Core Components:**
- Multi-fact star schema (warehouse ops, fleet trips, cross-dock transfers)
- Time-phased dimensions (15-minute granularity)
- Power BI dashboard templates
- Automated alerting system
- Role-based access control with row-level security

**Key Features:**
- Cross-fact KPI harmonization (linking warehouse metrics with fleet performance)
- Warehouse Gravity Zones™ for spatial optimization
- Adaptive fleet triage engine with predictive maintenance
- Temporal elasticity modeling for scenario simulation
- Natural language query support via Power BI Q&A

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, TMS, telematics APIs)

### Initial Setup

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**
```sql
-- Connect to your SQL Server instance
-- Create new database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Execute the schema creation script
-- (run the provided schema.sql file from the repository)
```

3. **Configure data source connections:**
```json
// config.json (create from config_sample.json)
{
  "sql_server": {
    "server": "YOUR_SERVER_NAME",
    "database": "LogiFleetPulse",
    "authentication": "windows" // or "sql"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_api": "${TELEMATICS_API_ENDPOINT}",
    "erp_connection": "${ERP_CONNECTION_STRING}"
  },
  "refresh_interval_minutes": 15,
  "alert_email": "${ALERT_EMAIL_ADDRESS}"
}
```

4. **Import Power BI template:**
- Open `LogiFleet_Pulse_Master.pbit`
- Enter SQL Server connection details
- Configure refresh schedule

## Data Model Architecture

### Core Fact Tables

```sql
-- FactWarehouseOperations
-- Tracks all warehouse micro-operations
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType NVARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity INT,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes INT,
    GravityZoneID INT,
    UserID INT,
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_WH_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
)

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWH_Time ON FactWarehouseOperations(TimeKey) INCLUDE (Quantity, DwellTimeMinutes)
CREATE NONCLUSTERED INDEX IX_FactWH_Product ON FactWarehouseOperations(ProductKey, TimeKey)
CREATE COLUMNSTORE INDEX CIX_FactWH_Analytics ON FactWarehouseOperations
```

```sql
-- FactFleetTrips
-- Tracks vehicle routes, fuel consumption, and telemetry
CREATE TABLE FactFleetTrips (
    TripID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKeyStart INT NOT NULL,
    TimeKeyEnd INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    OriginWarehouseKey INT,
    DestinationKey INT,
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DelayReasonKey INT,
    AvgSpeedMPH DECIMAL(5,2),
    MaxSpeedMPH DECIMAL(5,2),
    CONSTRAINT FK_Fleet_TimeStart FOREIGN KEY (TimeKeyStart) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey)
)

CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKeyStart, TimeKeyEnd)
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleKey) INCLUDE (FuelConsumedGallons, IdleTimeMinutes)
```

```sql
-- FactCrossDock
-- Transfers between inbound and outbound without storage
CREATE TABLE FactCrossDock (
    CrossDockID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKeyReceived INT NOT NULL,
    TimeKeyShipped INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT NOT NULL,
    CustomerKey INT NOT NULL,
    Quantity INT,
    TransitTimeMinutes INT,
    TemperatureCompliant BIT,
    QualityCheckPassed BIT
)
```

### Dimension Tables

```sql
-- DimTime (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    DayOfYear INT,
    DayOfMonth INT,
    DayOfWeek INT,
    DayName NVARCHAR(10),
    Hour INT,
    Minute INT,
    TimeSlot15Min INT, -- 0-95 (96 slots per day)
    IsWeekend BIT,
    IsHoliday BIT,
    FiscalPeriod NVARCHAR(10)
)

-- Populate time dimension
DECLARE @StartDate DATETIME = '2024-01-01'
DECLARE @EndDate DATETIME = '2028-12-31'
DECLARE @CurrentDate DATETIME = @StartDate

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO DimTime (TimeKey, FullDateTime, Year, Quarter, Month, Week, DayOfYear, 
                         DayOfMonth, DayOfWeek, DayName, Hour, Minute, TimeSlot15Min, 
                         IsWeekend, IsHoliday)
    SELECT 
        CONVERT(INT, FORMAT(@CurrentDate, 'yyyyMMddHHmm')),
        @CurrentDate,
        YEAR(@CurrentDate),
        DATEPART(QUARTER, @CurrentDate),
        MONTH(@CurrentDate),
        DATEPART(WEEK, @CurrentDate),
        DATEPART(DAYOFYEAR, @CurrentDate),
        DAY(@CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        DATENAME(WEEKDAY, @CurrentDate),
        DATEPART(HOUR, @CurrentDate),
        DATEPART(MINUTE, @CurrentDate),
        (DATEPART(HOUR, @CurrentDate) * 4) + (DATEPART(MINUTE, @CurrentDate) / 15),
        CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1,7) THEN 1 ELSE 0 END,
        0 -- Update separately with holiday calendar
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate)
END
```

```sql
-- DimProductGravity (with velocity scoring)
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,2),
    IsFragile BIT,
    IsPerishable BIT,
    ShelfLifeDays INT,
    GravityScore DECIMAL(5,2), -- Calculated: frequency * value / lead_time
    GravityZone NVARCHAR(20), -- 'High', 'Medium', 'Low'
    OptimalBinLocation NVARCHAR(50),
    LastGravityRecalc DATETIME
)

-- Stored procedure to recalculate gravity scores
CREATE PROCEDURE sp_RecalculateGravityScores
AS
BEGIN
    UPDATE DimProductGravity
    SET GravityScore = (
        SELECT 
            (COUNT(*) * AVG(p.UnitPrice) / NULLIF(AVG(s.LeadTimeDays), 0))
        FROM FactWarehouseOperations f
        JOIN DimProduct p ON f.ProductKey = p.ProductKey
        JOIN DimSupplier s ON p.SupplierKey = s.SupplierKey
        WHERE f.ProductKey = DimProductGravity.ProductKey
          AND f.TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMdd'))
    ),
    GravityZone = CASE 
        WHEN GravityScore > 75 THEN 'High'
        WHEN GravityScore > 40 THEN 'Medium'
        ELSE 'Low'
    END,
    LastGravityRecalc = GETDATE()
END
```

## Key Queries and Patterns

### Cross-Fact KPI Analysis

```sql
-- Analyze relationship between warehouse dwell time and fleet idle time
SELECT 
    t.DayName,
    t.Hour,
    w.WarehouseName,
    AVG(fw.DwellTimeMinutes) AS AvgWarehouseDwell,
    AVG(ff.IdleTimeMinutes) AS AvgFleetIdle,
    COUNT(DISTINCT ff.TripID) AS TripCount,
    SUM(ff.FuelConsumedGallons) AS TotalFuelConsumed,
    -- Correlation indicator
    CASE 
        WHEN AVG(fw.DwellTimeMinutes) > 120 AND AVG(ff.IdleTimeMinutes) > 45 
        THEN 'High Congestion'
        ELSE 'Normal'
    END AS CongestionStatus
FROM FactWarehouseOperations fw
JOIN DimTime t ON fw.TimeKey = t.TimeKey
JOIN DimWarehouse w ON fw.WarehouseKey = w.WarehouseKey
LEFT JOIN FactFleetTrips ff ON ff.OriginWarehouseKey = w.WarehouseKey
    AND ff.TimeKeyStart BETWEEN t.TimeKey AND DATEADD(MINUTE, 240, t.TimeKey)
WHERE t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY t.DayName, t.Hour, w.WarehouseName
HAVING COUNT(DISTINCT ff.TripID) > 5
ORDER BY AvgWarehouseDwell DESC
```

### Predictive Bottleneck Detection

```sql
-- Identify emerging bottlenecks using rolling averages
WITH HourlyMetrics AS (
    SELECT 
        TimeKey,
        WarehouseKey,
        AVG(DwellTimeMinutes) AS AvgDwell,
        AVG(ProcessingTimeMinutes) AS AvgProcessing,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations
    WHERE TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMddHH00'))
    GROUP BY TimeKey, WarehouseKey
),
RollingBaseline AS (
    SELECT 
        TimeKey,
        WarehouseKey,
        AvgDwell,
        AVG(AvgDwell) OVER (
            PARTITION BY WarehouseKey 
            ORDER BY TimeKey 
            ROWS BETWEEN 168 PRECEDING AND 1 PRECEDING
        ) AS BaselineDwell,
        STDEV(AvgDwell) OVER (
            PARTITION BY WarehouseKey 
            ORDER BY TimeKey 
            ROWS BETWEEN 168 PRECEDING AND 1 PRECEDING
        ) AS StdDevDwell
    FROM HourlyMetrics
)
SELECT 
    t.FullDateTime,
    w.WarehouseName,
    w.Region,
    r.AvgDwell AS CurrentDwell,
    r.BaselineDwell,
    (r.AvgDwell - r.BaselineDwell) / NULLIF(r.StdDevDwell, 0) AS ZScore,
    CASE 
        WHEN (r.AvgDwell - r.BaselineDwell) / NULLIF(r.StdDevDwell, 0) > 2 
        THEN 'Critical Alert'
        WHEN (r.AvgDwell - r.BaselineDwell) / NULLIF(r.StdDevDwell, 0) > 1.5 
        THEN 'Warning'
        ELSE 'Normal'
    END AS BottleneckRisk
FROM RollingBaseline r
JOIN DimTime t ON r.TimeKey = t.TimeKey
JOIN DimWarehouse w ON r.WarehouseKey = w.WarehouseKey
WHERE (r.AvgDwell - r.BaselineDwell) / NULLIF(r.StdDevDwell, 0) > 1.5
ORDER BY ZScore DESC
```

### Fleet Optimization Analysis

```sql
-- Identify optimal route assignments based on gravity zones
SELECT 
    v.VehicleID,
    v.VehicleType,
    v.CapacityWeight,
    r.RouteCode,
    r.OriginWarehouse,
    r.DestinationCity,
    SUM(p.GravityScore * wo.Quantity) AS WeightedGravityScore,
    COUNT(DISTINCT wo.ProductKey) AS UniqueProducts,
    SUM(wo.Quantity * p.UnitWeight) AS TotalWeight,
    AVG(ft.FuelConsumedGallons) AS AvgFuelPerTrip,
    -- Efficiency ratio
    (SUM(p.GravityScore * wo.Quantity) / NULLIF(AVG(ft.FuelConsumedGallons), 0)) AS EfficiencyRatio
FROM FactFleetTrips ft
JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
JOIN DimRoute r ON ft.RouteKey = r.RouteKey
JOIN FactWarehouseOperations wo ON wo.WarehouseKey = r.OriginWarehouseKey
    AND wo.TimeKey BETWEEN ft.TimeKeyStart - 480 AND ft.TimeKeyStart
JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE ft.TimeKeyStart >= CONVERT(INT, FORMAT(DATEADD(DAY, -60, GETDATE()), 'yyyyMMdd0000'))
GROUP BY v.VehicleID, v.VehicleType, v.CapacityWeight, r.RouteCode, 
         r.OriginWarehouse, r.DestinationCity
HAVING SUM(wo.Quantity * p.UnitWeight) / v.CapacityWeight BETWEEN 0.7 AND 0.95
ORDER BY EfficiencyRatio DESC
```

## Data Loading Patterns

### Incremental ETL Process

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Staging table for incoming data
    CREATE TABLE #StagingWarehouseOps (
        OperationDateTime DATETIME,
        SKU NVARCHAR(50),
        WarehouseCode NVARCHAR(20),
        OperationType NVARCHAR(50),
        Quantity INT,
        ProcessingTimeMinutes INT,
        UserID INT
    )
    
    -- Insert from source system (replace with actual source query)
    INSERT INTO #StagingWarehouseOps
    SELECT 
        OperationDateTime,
        SKU,
        WarehouseCode,
        OperationType,
        Quantity,
        ProcessingTimeMinutes,
        UserID
    FROM OPENQUERY(WMS_LINKED_SERVER, 
        'SELECT * FROM operations WHERE modified_date > ''' + 
        CONVERT(VARCHAR(23), @LastLoadDateTime, 121) + '''')
    
    -- Transform and load into fact table
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType, 
        Quantity, ProcessingTimeMinutes, UserID, GravityZoneID
    )
    SELECT 
        CONVERT(INT, FORMAT(s.OperationDateTime, 'yyyyMMddHHmm')),
        p.ProductKey,
        w.WarehouseKey,
        s.OperationType,
        s.Quantity,
        s.ProcessingTimeMinutes,
        s.UserID,
        p.GravityZone
    FROM #StagingWarehouseOps s
    JOIN DimProductGravity p ON s.SKU = p.SKU
    JOIN DimWarehouse w ON s.WarehouseCode = w.WarehouseCode
    WHERE NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations f
        WHERE f.TimeKey = CONVERT(INT, FORMAT(s.OperationDateTime, 'yyyyMMddHHmm'))
          AND f.ProductKey = p.ProductKey
          AND f.WarehouseKey = w.WarehouseKey
          AND f.UserID = s.UserID
    )
    
    DROP TABLE #StagingWarehouseOps
    
    -- Update last load timestamp
    UPDATE ETL_Control
    SET LastLoadDateTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations'
END
```

### External API Integration

```sql
-- Example: Loading fleet telemetry from REST API
CREATE PROCEDURE sp_LoadFleetTelemetry
AS
BEGIN
    -- Create staging table for JSON response
    DECLARE @JSON NVARCHAR(MAX)
    DECLARE @URL NVARCHAR(500) = '${TELEMATICS_API_ENDPOINT}/trips/recent'
    DECLARE @AuthToken NVARCHAR(500) = '${TELEMATICS_API_KEY}'
    
    -- Make HTTP request (requires SQL Server 2022+ or CLR integration)
    EXEC sp_invoke_external_rest_endpoint
        @url = @URL,
        @method = 'GET',
        @headers = '{"Authorization": "Bearer ' + @AuthToken + '"}',
        @response = @JSON OUTPUT
    
    -- Parse JSON and insert
    INSERT INTO FactFleetTrips (
        TimeKeyStart, TimeKeyEnd, VehicleKey, RouteKey, DriverKey,
        DistanceMiles, FuelConsumedGallons, IdleTimeMinutes, AvgSpeedMPH
    )
    SELECT 
        CONVERT(INT, FORMAT(JSON_VALUE(value, '$.start_time'), 'yyyyMMddHHmm')),
        CONVERT(INT, FORMAT(JSON_VALUE(value, '$.end_time'), 'yyyyMMddHHmm')),
        v.VehicleKey,
        r.RouteKey,
        d.DriverKey,
        CAST(JSON_VALUE(value, '$.distance') AS DECIMAL(10,2)),
        CAST(JSON_VALUE(value, '$.fuel_consumed') AS DECIMAL(8,2)),
        CAST(JSON_VALUE(value, '$.idle_minutes') AS INT),
        CAST(JSON_VALUE(value, '$.avg_speed') AS DECIMAL(5,2))
    FROM OPENJSON(@JSON, '$.trips') 
    JOIN DimVehicle v ON JSON_VALUE(value, '$.vehicle_id') = v.VehicleID
    JOIN DimRoute r ON JSON_VALUE(value, '$.route_code') = r.RouteCode
    JOIN DimDriver d ON JSON_VALUE(value, '$.driver_id') = d.DriverID
END
```

## Automated Alerting System

```sql
-- Alert configuration table
CREATE TABLE AlertConfig (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName NVARCHAR(100),
    MetricType NVARCHAR(50), -- 'DwellTime', 'FuelConsumption', 'IdleTime', etc.
    ThresholdValue DECIMAL(10,2),
    ThresholdOperator NVARCHAR(10), -- '>', '<', '>=', '<=', '='
    EvaluationQuery NVARCHAR(MAX),
    RecipientEmail NVARCHAR(500),
    IsActive BIT,
    LastTriggered DATETIME
)

-- Alert execution procedure
CREATE PROCEDURE sp_EvaluateAlerts
AS
BEGIN
    DECLARE @AlertID INT, @MetricType NVARCHAR(50), @Threshold DECIMAL(10,2)
    DECLARE @Operator NVARCHAR(10), @Query NVARCHAR(MAX), @Recipient NVARCHAR(500)
    DECLARE @CurrentValue DECIMAL(10,2), @AlertBody NVARCHAR(MAX)
    
    DECLARE alert_cursor CURSOR FOR
    SELECT AlertID, MetricType, ThresholdValue, ThresholdOperator, 
           EvaluationQuery, RecipientEmail
    FROM AlertConfig
    WHERE IsActive = 1
    
    OPEN alert_cursor
    FETCH NEXT FROM alert_cursor INTO @AlertID, @MetricType, @Threshold, 
                                       @Operator, @Query, @Recipient
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Execute evaluation query
        DECLARE @ParamDef NVARCHAR(100) = '@Value DECIMAL(10,2) OUTPUT'
        EXEC sp_executesql @Query, @ParamDef, @Value = @CurrentValue OUTPUT
        
        -- Check threshold
        DECLARE @BreachDetected BIT = 0
        IF @Operator = '>' AND @CurrentValue > @Threshold SET @BreachDetected = 1
        IF @Operator = '<' AND @CurrentValue < @Threshold SET @BreachDetected = 1
        IF @Operator = '>=' AND @CurrentValue >= @Threshold SET @BreachDetected = 1
        IF @Operator = '<=' AND @CurrentValue <= @Threshold SET @BreachDetected = 1
        
        IF @BreachDetected = 1
        BEGIN
            SET @AlertBody = 'Alert: ' + @MetricType + ' threshold breached. ' +
                           'Current value: ' + CAST(@CurrentValue AS NVARCHAR(20)) + 
                           ', Threshold: ' + CAST(@Threshold AS NVARCHAR(20))
            
            -- Send email (requires Database Mail configuration)
            EXEC msdb.dbo.sp_send_dbmail
                @profile_name = 'LogiFleetAlerts',
                @recipients = @Recipient,
                @subject = 'LogiFleet Pulse Alert',
                @body = @AlertBody
            
            -- Log alert
            UPDATE AlertConfig
            SET LastTriggered = GETDATE()
            WHERE AlertID = @AlertID
        END
        
        FETCH NEXT FROM alert_cursor INTO @AlertID, @MetricType, @Threshold, 
                                           @Operator, @Query, @Recipient
    END
    
    CLOSE alert_cursor
    DEALLOCATE alert_cursor
END

-- Schedule this procedure to run every 15 minutes via SQL Agent
```

### Example Alert Configurations

```sql
-- Alert for excessive warehouse dwell time
INSERT INTO AlertConfig (AlertName, MetricType, ThresholdValue, ThresholdOperator, 
                        EvaluationQuery, RecipientEmail, IsActive)
VALUES (
    'High Dwell Time Alert',
    'DwellTime',
    180, -- 3 hours
    '>',
    'SELECT @Value = AVG(DwellTimeMinutes) FROM FactWarehouseOperations 
     WHERE TimeKey >= CONVERT(INT, FORMAT(DATEADD(HOUR, -1, GETDATE()), ''yyyyMMddHH00''))',
    '${OPERATIONS_EMAIL}',
    1
)

-- Alert for fleet idle time exceeding threshold
INSERT INTO AlertConfig (AlertName, MetricType, ThresholdValue, ThresholdOperator, 
                        EvaluationQuery, RecipientEmail, IsActive)
VALUES (
    'Fleet Idle Time Warning',
    'IdleTime',
    20, -- 20% of trip time
    '>',
    'SELECT @Value = AVG(CAST(IdleTimeMinutes AS DECIMAL) / 
                         NULLIF(DATEDIFF(MINUTE, TimeKeyStart, TimeKeyEnd), 0) * 100)
     FROM FactFleetTrips 
     WHERE TimeKeyStart >= CONVERT(INT, FORMAT(DATEADD(HOUR, -4, GETDATE()), ''yyyyMMddHH00''))',
    '${FLEET_MANAGER_EMAIL}',
    1
)
```

## Power BI Configuration

### DAX Measures for Cross-Fact Analysis

```dax
// Average Warehouse Dwell Time
AvgDwellTime = 
AVERAGE(FactWarehouseOperations[DwellTimeMinutes])

// Fleet Utilization Rate
FleetUtilization = 
DIVIDE(
    CALCULATE(
        SUM(FactFleetTrips[DistanceMiles]),
        FactFleetTrips[IdleTimeMinutes] < 30
    ),
    CALCULATE(SUM(FactFleetTrips[DistanceMiles]))
)

// Gravity-Weighted Throughput
GravityThroughput = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[Quantity] * 
    RELATED(DimProductGravity[GravityScore])
)

// Cross-Fact Efficiency Ratio
LogisticsEfficiency = 
DIVIDE(
    [GravityThroughput],
    [TotalFuelConsumed] + ([AvgDwellTime] * 0.5)
)

// Predictive Bottleneck Index (uses rolling 7-day baseline)
BottleneckIndex = 
VAR CurrentDwell = [AvgDwellTime]
VAR BaselineDwell = 
    CALCULATE(
        [AvgDwellTime],
        DATESINPERIOD(
            DimTime[FullDateTime],
            MAX(DimTime[FullDateTime]) - 7,
            -7,
            DAY
        )
    )
VAR StdDev = 
    STDEVX.P(
        CALCULATETABLE(
            FactWarehouseOperations,
            DATESINPERIOD(
                DimTime[FullDateTime],
                MAX(DimTime[FullDateTime]) - 7,
                -7,
                DAY
            )
        ),
        FactWarehouseOperations[DwellTimeMinutes]
    )
RETURN
DIVIDE(CurrentDwell - BaselineDwell, StdDev)
```

### Row-Level Security Setup

```dax
// In Power BI, create role "Regional Manager"
// Add filter to DimWarehouse table:
[Region] = USERPRINCIPALNAME()

// Or for more granular control:
[WarehouseCode] IN (
    LOOKUPVALUE(
        UserWarehouseAccess[WarehouseCode],
        UserWarehouseAccess[UserEmail],
        USERPRINCIPALNAME()
    )
)
```

## Troubleshooting

### Performance Optimization

**Issue: Slow cross-fact queries**
```sql
-- Add missing indexes
CREATE NONCLUSTERED INDEX IX_FactWH_TimeProduct 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (Quantity, DwellTimeMinutes, ProcessingTimeMinutes)

-- Consider partitioning by TimeKey
CREATE PARTITION FUNCTION pf_TimeKey (INT)
AS RANGE RIGHT FOR VALUES (20240101, 20240201, 20240301, ..., 20271201)

CREATE PARTITION SCHEME ps_TimeKey
AS PARTITION pf_TimeKey ALL TO ([PRIMARY])

-- Rebuild fact table on partitioned scheme
```

**Issue: Power BI dashboard slow refresh**
```dax
// Use aggregation tables for large fact tables
// Create summary table in SQL:
CREATE VIEW vw_FactWarehouseDaily AS
SELECT 
    CONVERT(INT, FORMAT(t.FullDateTime, 'yyyyMMdd')) AS DateKey,
    f.WarehouseKey,
    f.ProductKey,
    SUM(f.Quantity) AS TotalQuantity,
    AVG(f.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(*) AS OperationCount
FROM FactWarehouseOperations f
JOIN DimTime t ON f.TimeKey = t.TimeKey
GROUP BY CONVERT(INT, FORMAT(t.FullDateTime, 'yyyyMMdd')), 
         f.WarehouseKey, f.ProductKey
```

### Data Quality Issues

**Issue: Missing dimension key lookups**
```sql
-- Create unknown member for each dimension
INSERT INTO DimProduct (SKU, ProductName, GravityScore)
VALUES ('UNKNOWN', 'Unknown Product', 0)

-- Use ISNULL in ETL
INSERT INTO FactWarehouseOperations (ProductKey, ...)
SELECT 
    ISNULL(p.ProductKey, (SELECT ProductKey FROM DimProduct WHERE SKU = 'UNKNOWN')),
    ...
FROM StagingTable s
LEFT JOIN DimProduct p ON s.SKU = p.SKU
```

**Issue: Time dimension granularity mismatch**
```sql
-- Round timestamps to nearest 15-minute interval
DECLARE @SourceTime DATETIME = '2026-07-02 14:23:00'
DECLARE @RoundedTime DATETIME = 
    DATEADD(MINUTE, 
            (DATEDIFF(MINUTE, 0, @SourceTime) / 15) * 15, 
            0)
SELECT @RoundedTime -- Returns 2026-07-02 14:15:00
