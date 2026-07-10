---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI multi-fact star schema for logistics intelligence, warehouse operations, and fleet telemetry analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy logicore analytics SQL schema"
  - "create warehouse and fleet logistics dashboard"
  - "implement multi-fact star schema for logistics"
  - "configure Power BI for supply chain intelligence"
  - "build cross-modal logistics data warehouse"
  - "integrate warehouse operations with fleet telemetry"
  - "query logistics KPIs across warehouse and fleet"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform built on MS SQL Server and Power BI. It implements a multi-fact star schema that unifies:

- **Warehouse operations** (receiving, putaway, picking, packing, shipping, dwell time)
- **Fleet telemetry** (GPS tracking, fuel consumption, idle time, maintenance alerts)
- **Inventory management** (aging curves, turnover rates, gravity zone optimization)
- **Supply chain metrics** (lead times, supplier reliability, cross-dock operations)

The platform uses time-phased dimensions, bridge tables for many-to-many relationships, and composite aggregation to enable cross-fact KPI analysis (e.g., correlating warehouse dwell time with fleet fuel costs).

## Installation and Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SSMS (SQL Server Management Studio) or Azure Data Studio
- Appropriate permissions to create databases, tables, and stored procedures

### Step 1: Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance using SSMS or Azure Data Studio
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the schema creation script (provided in the repository)
-- This creates all dimension and fact tables
```

### Step 3: Configure Data Sources

Create a `config.json` file (copy from `config_sample.json`):

```json
{
  "sql_server": {
    "server": "YOUR_SERVER_NAME",
    "database": "LogiFleetPulse",
    "authentication": "windows",
    "connection_string": "Server=${SQL_SERVER};Database=${SQL_DATABASE};Trusted_Connection=yes;"
  },
  "data_sources": {
    "wms_api_endpoint": "${WMS_API_ENDPOINT}",
    "fleet_telemetry_endpoint": "${FLEET_TELEMETRY_ENDPOINT}",
    "weather_api_key": "${WEATHER_API_KEY}"
  },
  "refresh_intervals": {
    "real_time_minutes": 15,
    "daily_snapshot_hour": 2
  }
}
```

### Step 4: Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI Template
3. Select `LogiFleet_Pulse_Master.pbit`
4. Enter your SQL Server connection details
5. The data model will auto-configure relationships

## Core SQL Schema Structure

### Dimension Tables

```sql
-- Time dimension (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay INT NOT NULL,
    DayOfWeek NVARCHAR(10) NOT NULL,
    DayOfMonth INT NOT NULL,
    WeekOfYear INT NOT NULL,
    MonthName NVARCHAR(10) NOT NULL,
    Quarter INT NOT NULL,
    FiscalYear INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
);

-- Geography dimension (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID NVARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50) NOT NULL, -- Warehouse, Hub, Route_Node
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone NVARCHAR(50)
);

-- Product dimension with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100),
    Subcategory NVARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    TemperatureMin DECIMAL(5,2),
    TemperatureMax DECIMAL(5,2),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * fragility
    OptimalStorageZone NVARCHAR(50)
);

-- Supplier reliability dimension
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID NVARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200) NOT NULL,
    LeadTimeDaysAvg DECIMAL(5,2),
    LeadTimeDaysStdDev DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,2),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2),
    LastReviewDate DATE
);
```

### Fact Tables

```sql
-- Warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType NVARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OrderID NVARCHAR(50),
    Quantity INT NOT NULL,
    DurationMinutes INT,
    DwellTimeHours DECIMAL(10,2),
    StorageZone NVARCHAR(50),
    OperatorID NVARCHAR(50),
    EquipmentID NVARCHAR(50),
    IsError BIT DEFAULT 0,
    ErrorType NVARCHAR(100)
);

-- Fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50) NOT NULL,
    TripDistanceKM DECIMAL(10,2),
    TripDurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKG DECIMAL(10,2),
    AverageSpeedKMH DECIMAL(5,2),
    HarshBrakingEvents INT DEFAULT 0,
    OverspeedingMinutes INT DEFAULT 0,
    WeatherCondition NVARCHAR(50),
    TrafficDelayMinutes INT DEFAULT 0
);

-- Cross-dock operations fact table
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT NOT NULL FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    Quantity INT NOT NULL,
    DockDwellMinutes INT,
    IsDirectTransfer BIT DEFAULT 1
);
```

### Bridge Tables for Many-to-Many Relationships

```sql
-- Bridge table linking fleet trips to multiple products
CREATE TABLE BridgeTripProduct (
    TripKey BIGINT NOT NULL FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    QuantityLoaded INT NOT NULL,
    PRIMARY KEY (TripKey, ProductKey)
);

-- Bridge table linking warehouse operations to multiple storage zones
CREATE TABLE BridgeOperationZone (
    OperationKey BIGINT NOT NULL FOREIGN KEY REFERENCES FactWarehouseOperations(OperationKey),
    StorageZone NVARCHAR(50) NOT NULL,
    TimeSpentMinutes INT NOT NULL,
    PRIMARY KEY (OperationKey, StorageZone)
);
```

## Key SQL Queries and Patterns

### Cross-Fact KPI: Warehouse Dwell Time vs Fleet Idle Time

```sql
-- Correlate warehouse dwell time with fleet idle time for specific products
SELECT 
    dp.SKU,
    dp.ProductName,
    AVG(wo.DwellTimeHours) AS AvgWarehouseDwellHours,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    CORR(wo.DwellTimeHours, ft.IdleTimeMinutes) OVER (PARTITION BY dp.ProductKey) AS Correlation
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity dp ON wo.ProductKey = dp.ProductKey
INNER JOIN BridgeTripProduct btp ON dp.ProductKey = btp.ProductKey
INNER JOIN FactFleetTrips ft ON btp.TripKey = ft.TripKey
INNER JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
WHERE dt.FullDateTime >= DATEADD(MONTH, -3, GETDATE())
GROUP BY dp.SKU, dp.ProductName, dp.ProductKey;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products in suboptimal storage zones based on velocity
WITH ProductVelocity AS (
    SELECT 
        ProductKey,
        COUNT(*) AS PickFrequency,
        AVG(DurationMinutes) AS AvgPickTime
    FROM FactWarehouseOperations
    WHERE OperationType = 'Picking'
        AND TimeKey IN (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()))
    GROUP BY ProductKey
)
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.OptimalStorageZone,
    wo.StorageZone AS CurrentStorageZone,
    pv.PickFrequency,
    pv.AvgPickTime,
    CASE 
        WHEN dp.OptimalStorageZone <> wo.StorageZone THEN 'RELOCATE'
        ELSE 'OK'
    END AS RecommendedAction
FROM ProductVelocity pv
INNER JOIN DimProductGravity dp ON pv.ProductKey = dp.ProductKey
INNER JOIN FactWarehouseOperations wo ON pv.ProductKey = wo.ProductKey
WHERE dp.OptimalStorageZone <> wo.StorageZone
    AND pv.PickFrequency > 100 -- High-velocity items
ORDER BY pv.PickFrequency DESC;
```

### Fleet Fuel Efficiency by Route and Load Weight

```sql
SELECT 
    og.LocationName AS Origin,
    dg.LocationName AS Destination,
    AVG(ft.LoadWeightKG) AS AvgLoadWeight,
    AVG(ft.FuelConsumedLiters) AS AvgFuelConsumed,
    AVG(ft.TripDistanceKM) AS AvgDistanceKM,
    (AVG(ft.FuelConsumedLiters) / NULLIF(AVG(ft.TripDistanceKM), 0)) AS LitersPer100KM,
    COUNT(*) AS TripCount
FROM FactFleetTrips ft
INNER JOIN DimGeography og ON ft.OriginGeographyKey = og.GeographyKey
INNER JOIN DimGeography dg ON ft.DestinationGeographyKey = dg.GeographyKey
INNER JOIN DimTime dt ON ft.TimeKey = dt.TimeKey
WHERE dt.FullDateTime >= DATEADD(MONTH, -1, GETDATE())
GROUP BY og.LocationName, dg.LocationName
HAVING COUNT(*) >= 10
ORDER BY (AVG(ft.FuelConsumedLiters) / NULLIF(AVG(ft.TripDistanceKM), 0)) DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify warehouse operations with increasing dwell time trends
WITH DwellTrends AS (
    SELECT 
        ProductKey,
        StorageZone,
        DATEPART(WEEK, dt.FullDateTime) AS WeekNumber,
        AVG(DwellTimeHours) AS AvgDwellTime
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
    WHERE dt.FullDateTime >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY ProductKey, StorageZone, DATEPART(WEEK, dt.FullDateTime)
),
TrendAnalysis AS (
    SELECT 
        ProductKey,
        StorageZone,
        AVG(AvgDwellTime) AS OverallAvgDwell,
        STDEV(AvgDwellTime) AS DwellStdDev,
        (MAX(AvgDwellTime) - MIN(AvgDwellTime)) AS DwellRange
    FROM DwellTrends
    GROUP BY ProductKey, StorageZone
)
SELECT 
    dp.SKU,
    dp.ProductName,
    ta.StorageZone,
    ta.OverallAvgDwell,
    ta.DwellStdDev,
    ta.DwellRange,
    CASE 
        WHEN ta.DwellRange > ta.OverallAvgDwell * 0.5 THEN 'HIGH_RISK'
        WHEN ta.DwellRange > ta.OverallAvgDwell * 0.3 THEN 'MEDIUM_RISK'
        ELSE 'LOW_RISK'
    END AS BottleneckRisk
FROM TrendAnalysis ta
INNER JOIN DimProductGravity dp ON ta.ProductKey = dp.ProductKey
WHERE ta.DwellStdDev > 5 -- High variability
ORDER BY ta.DwellRange DESC;
```

## Stored Procedures for Data Loading

### Incremental Load for Warehouse Operations

```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last 24 hours if no parameter provided
    IF @LastLoadDateTime IS NULL
        SET @LastLoadDateTime = DATEADD(DAY, -1, GETDATE());
    
    -- Insert new records from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OrderID, Quantity, DurationMinutes, DwellTimeHours,
        StorageZone, OperatorID, EquipmentID, IsError, ErrorType
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        stg.OperationType,
        stg.OrderID,
        stg.Quantity,
        stg.DurationMinutes,
        stg.DwellTimeHours,
        stg.StorageZone,
        stg.OperatorID,
        stg.EquipmentID,
        stg.IsError,
        stg.ErrorType
    FROM StagingWarehouseOperations stg
    INNER JOIN DimTime dt ON stg.OperationDateTime = dt.FullDateTime
    INNER JOIN DimGeography dg ON stg.WarehouseID = dg.LocationID
    INNER JOIN DimProductGravity dp ON stg.SKU = dp.SKU
    WHERE stg.OperationDateTime > @LastLoadDateTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations wo
            WHERE wo.OrderID = stg.OrderID 
                AND wo.OperationType = stg.OperationType
        );
    
    -- Log load statistics
    DECLARE @RowsInserted INT = @@ROWCOUNT;
    INSERT INTO LoadLog (LoadDateTime, TargetTable, RowsInserted)
    VALUES (GETDATE(), 'FactWarehouseOperations', @RowsInserted);
END;
GO
```

### Gravity Score Calculation

```sql
CREATE PROCEDURE usp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate velocity (picks per day)
    WITH ProductVelocity AS (
        SELECT 
            ProductKey,
            COUNT(*) / 30.0 AS PicksPerDay -- Last 30 days
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
            AND TimeKey IN (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()))
        GROUP BY ProductKey
    ),
    -- Calculate value score (normalized)
    ProductValue AS (
        SELECT 
            ProductKey,
            AVG(UnitWeight * 100) AS ValueScore -- Placeholder: replace with actual price
        FROM DimProductGravity
        GROUP BY ProductKey
    )
    -- Update gravity scores
    UPDATE dp
    SET GravityScore = 
        COALESCE(pv.PicksPerDay, 0) * 0.5 +
        COALESCE(pvl.ValueScore, 0) * 0.3 +
        CASE WHEN dp.IsFragile = 1 THEN 20 ELSE 0 END,
        OptimalStorageZone = CASE 
            WHEN COALESCE(pv.PicksPerDay, 0) > 50 THEN 'A_HighVelocity'
            WHEN COALESCE(pv.PicksPerDay, 0) > 20 THEN 'B_MediumVelocity'
            ELSE 'C_LowVelocity'
        END
    FROM DimProductGravity dp
    LEFT JOIN ProductVelocity pv ON dp.ProductKey = pv.ProductKey
    LEFT JOIN ProductValue pvl ON dp.ProductKey = pvl.ProductKey;
END;
GO
```

## Power BI DAX Measures

### Cross-Fact KPI: Dwell-to-Idle Ratio

```dax
Dwell_to_Idle_Ratio = 
VAR AvgDwellHours = 
    AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR AvgIdleMinutes = 
    AVERAGE(FactFleetTrips[IdleTimeMinutes])
RETURN
    DIVIDE(AvgDwellHours * 60, AvgIdleMinutes, 0)
```

### Predictive Bottleneck Score

```dax
Bottleneck_Risk_Score = 
VAR CurrentDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeHours]),
        DATESINPERIOD(DimTime[FullDateTime], MAX(DimTime[FullDateTime]), -7, DAY)
    )
VAR HistoricalAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeHours]),
        DATESINPERIOD(DimTime[FullDateTime], MAX(DimTime[FullDateTime]), -90, DAY)
    )
VAR DwellVariance = 
    DIVIDE(CurrentDwell - HistoricalAvg, HistoricalAvg, 0)
RETURN
    IF(DwellVariance > 0.3, 100 * DwellVariance, 0)
```

### Fleet Efficiency Score

```dax
Fleet_Efficiency_Score = 
VAR ActualFuelPerKM = 
    DIVIDE(
        SUM(FactFleetTrips[FuelConsumedLiters]),
        SUM(FactFleetTrips[TripDistanceKM]),
        0
    )
VAR BenchmarkFuelPerKM = 0.25 -- Industry benchmark: 25 liters per 100 km = 0.25
VAR IdleTimePct = 
    DIVIDE(
        SUM(FactFleetTrips[IdleTimeMinutes]),
        SUM(FactFleetTrips[TripDurationMinutes]),
        0
    )
RETURN
    (1 - DIVIDE(ActualFuelPerKM, BenchmarkFuelPerKM, 0)) * 0.6 +
    (1 - IdleTimePct) * 0.4
```

## Automated Alerting with SQL Agent

### Create Alert Threshold Table

```sql
CREATE TABLE AlertThresholds (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName NVARCHAR(100) NOT NULL,
    MetricType NVARCHAR(50) NOT NULL, -- DwellTime, IdleTime, FuelConsumption
    ThresholdValue DECIMAL(10,2) NOT NULL,
    ComparisonOperator NVARCHAR(10) NOT NULL, -- GreaterThan, LessThan
    NotificationEmails NVARCHAR(500),
    IsActive BIT DEFAULT 1
);

-- Insert sample thresholds
INSERT INTO AlertThresholds (AlertName, MetricType, ThresholdValue, ComparisonOperator, NotificationEmails)
VALUES 
    ('High Warehouse Dwell', 'DwellTime', 72.0, 'GreaterThan', '${ALERT_EMAIL_LOGISTICS}'),
    ('Excessive Fleet Idle', 'IdleTime', 15.0, 'GreaterThan', '${ALERT_EMAIL_FLEET}'),
    ('Low Fuel Efficiency', 'FuelConsumption', 0.30, 'GreaterThan', '${ALERT_EMAIL_FLEET}');
```

### Alert Execution Stored Procedure

```sql
CREATE PROCEDURE usp_ExecuteAlertChecks
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Check warehouse dwell time alerts
    DECLARE @DwellThreshold DECIMAL(10,2);
    SELECT @DwellThreshold = ThresholdValue 
    FROM AlertThresholds 
    WHERE AlertName = 'High Warehouse Dwell' AND IsActive = 1;
    
    IF EXISTS (
        SELECT 1 
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
        WHERE dt.FullDateTime >= DATEADD(HOUR, -4, GETDATE())
            AND wo.DwellTimeHours > @DwellThreshold
    )
    BEGIN
        -- Log alert and send notification
        INSERT INTO AlertLog (AlertDateTime, AlertName, MetricValue, Recipients)
        SELECT 
            GETDATE(),
            'High Warehouse Dwell',
            MAX(wo.DwellTimeHours),
            (SELECT NotificationEmails FROM AlertThresholds WHERE AlertName = 'High Warehouse Dwell')
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
        WHERE dt.FullDateTime >= DATEADD(HOUR, -4, GETDATE())
            AND wo.DwellTimeHours > @DwellThreshold;
        
        -- Send email via sp_send_dbmail
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleet_Alerts',
            @recipients = '${ALERT_EMAIL_LOGISTICS}',
            @subject = 'LogiFleet ALERT: High Warehouse Dwell Time Detected',
            @body = 'One or more products have exceeded the dwell time threshold. Check the dashboard for details.';
    END;
END;
GO
```

## Configuration and Environment Variables

Always reference environment variables for sensitive configuration:

```sql
-- Connection string example
DECLARE @ConnectionString NVARCHAR(500) = 
    'Server=' + '${SQL_SERVER}' + ';' +
    'Database=' + '${SQL_DATABASE}' + ';' +
    'Trusted_Connection=yes;';

-- External API endpoints
DECLARE @WMS_API NVARCHAR(200) = '${WMS_API_ENDPOINT}';
DECLARE @FleetAPI NVARCHAR(200) = '${FLEET_TELEMETRY_ENDPOINT}';
DECLARE @WeatherAPIKey NVARCHAR(100) = '${WEATHER_API_KEY}';
```

## Troubleshooting

### Issue: Power BI Template Fails to Load

**Cause**: Missing SQL Server native client or incorrect connection string.

**Solution**:
1. Install SQL Server Native Client 11.0 or higher
2. Verify connection string in Power BI Desktop: Home → Transform Data → Data Source Settings
3. Ensure SQL Server allows remote connections (TCP/IP enabled)

### Issue: Slow Cross-Fact Queries

**Cause**: Missing indexes on foreign key columns.

**Solution**:
```sql
-- Create indexes on fact table foreign keys
CREATE NONCLUSTERED INDEX IX_FactWarehouseOperations_TimeKey 
ON FactWarehouseOperations(TimeKey);

CREATE NONCLUSTERED INDEX IX_FactWarehouseOperations_ProductKey 
ON FactWarehouseOperations(ProductKey);

CREATE NONCLUSTERED INDEX IX_FactFleetTrips_OriginGeographyKey 
ON FactFleetTrips(OriginGeographyKey);

-- Create composite indexes for common query patterns
CREATE NONCLUSTERED INDEX IX_FactWarehouseOperations_TimeProduct 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (DwellTimeHours, Quantity);
```

### Issue: Data Refresh Fails with Timeout

**Cause**: Large fact tables with full refresh mode.

**Solution**:
1. Switch to incremental refresh in Power BI:
   - Table Tools → Incremental Refresh → Define Policy
   - Archive data: Older than 2 years
   - Refresh data: Last 90 days
2. Use partitioning in SQL Server:
```sql
-- Partition fact table by month
CREATE PARTITION FUNCTION PF_MonthlyPartition (DATETIME)
AS RANGE RIGHT FOR VALUES (
    '2025-01-01', '2025-02-01', '2025-03-01', '2025-04-01',
    '2025-05-01', '2025-06-01', '2025-07-01', '2025-08-01',
    '2025-09-01', '2025-10-01', '2025-11-01', '2025-12-01'
);

CREATE PARTITION SCHEME PS_MonthlyPartition
AS PARTITION PF_MonthlyPartition ALL TO ([PRIMARY]);

-- Apply to fact table (requires table rebuild)
```

### Issue: Gravity Scores Not Updating

**Cause**: Stored procedure not scheduled or missing data in last 30 days.

**Solution**:
1. Schedule `usp_UpdateProductGravityScores` via SQL Server Agent:
```sql
EXEC msdb.dbo.sp_add_job @job_name = 'Update_Gravity_Scores';

EXEC msdb.dbo.sp_add_jobstep 
    @job_name = 'Update_Gravity_Scores',
    @step_name = 'Execute_Gravity_Update',
    @command = 'EXEC usp_UpdateProductGravityScores;',
    @database_name = 'LogiFleetPulse';

EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = 'Daily_2AM',
    @freq_type = 4, -- Daily
    @active_start_time = 020000; -- 2:00 AM

EXEC msdb.dbo.sp_attach_schedule 
    @job_name = 'Update_Gravity_Scores',
    @schedule_name = 'Daily_2AM';

EXEC msdb.dbo.sp_add_jobserver 
    @job_name = 'Update_Gravity_Scores';
```

2. Verify data availability:
```sql
SELECT COUNT(*) AS RecentOperations
FROM FactWarehouseOperations wo
INNER JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
WHERE dt.FullDateTime >= DATEADD(DAY, -30, GETDATE());
```

## Best Practices

1. **Use Row-Level Security** in Power BI for multi-tenant access:
```dax
-- Create role "Warehouse_Manager" with filter:
[DimGeography][LocationType] = "Warehouse" 
&& [DimGeography][LocationName] IN {'Location1', 'Location2'}
```

2. **Implement Change Data Capture** for large fact tables:
```sql
-- Enable CDC on database
EXEC sys.sp_cdc_enable_db;

-- Enable CDC on fact table
EXEC sys.sp_cdc_enable_table
    @source_schema = 'dbo',
    @source_name = 'FactWarehouseOperations',
    @role_name = NULL;
```

3. **Archive historical data** to Azure Blob Storage or separate archive database after 2 years.

4. **Monitor query performance** using SQL Server Extended Events:
```sql
CREATE EVENT SESSION [LogiFleet_Query_Monitor]
ON SERVER
ADD EVENT sqlserver.sql_statement_completed
(WHERE duration > 5000000); -- Queries longer than 5 seconds
```

5. **Use Power BI aggregations** for frequently queried summarizations to improve dashboard performance.
