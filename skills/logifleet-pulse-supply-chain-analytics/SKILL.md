---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing template for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - set up logistics analytics dashboard
  - configure supply chain data warehouse
  - implement fleet management reporting
  - build warehouse operations KPI dashboard
  - create multi-fact star schema for logistics
  - deploy LogiFleet Pulse analytics platform
  - integrate warehouse and fleet data models
  - setup Power BI logistics visualization
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and analytics template for logistics and supply chain management. It provides:

- **Multi-fact star schema** combining warehouse operations, fleet telemetry, and inventory data
- **MS SQL Server database scripts** for creating dimension and fact tables
- **Power BI templates** for real-time logistics dashboards
- **Cross-fact KPI harmonization** linking warehouse metrics with fleet performance
- **Time-phased dimensional modeling** with 15-minute granularity
- **Predictive analytics** for bottleneck detection and fleet optimization

The platform integrates warehouse management systems (WMS), telematics feeds, supplier data, and external APIs into a unified semantic layer for logistics intelligence.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems: WMS, TMS, telematics, ERP

### Database Deployment

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**

```sql
-- Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run schema creation script
:r schema/create_tables.sql
:r schema/create_views.sql
:r schema/create_stored_procedures.sql
```

3. **Configure data source connections:**

Edit `config.json` with your connection strings:

```json
{
  "connections": {
    "sql_server": "Server=${SQL_SERVER_HOST};Database=LogiFleetPulse;User Id=${SQL_USER};Password=${SQL_PASSWORD};",
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_api": "${TELEMATICS_API_ENDPOINT}",
    "weather_api": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15,
  "alert_thresholds": {
    "fleet_idle_percent": 15,
    "dwell_time_hours": 72,
    "freezer_temp_deviation_celsius": 2
  }
}
```

### Power BI Configuration

1. **Open the template:**

```
File > Open > LogiFleet_Pulse_Master.pbit
```

2. **Configure data source parameters:**

- Server: Your SQL Server instance
- Database: LogiFleetPulse
- Authentication: Windows or SQL Server

3. **Set up row-level security:**

```dax
-- Create security role in Power BI
[UserEmail] = USERPRINCIPALNAME()
```

## Core Database Schema

### Dimension Tables

**DimTime** - Time intelligence with 15-minute granularity:

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    Date DATE NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    DayOfMonth INT NOT NULL,
    DayOfWeek INT NOT NULL,
    HourOfDay INT NOT NULL,
    MinuteOfHour INT NOT NULL,
    FiscalYear INT NOT NULL,
    FiscalQuarter INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
);

-- Insert time records
DECLARE @StartDate DATETIME = '2024-01-01 00:00:00';
DECLARE @EndDate DATETIME = '2027-12-31 23:45:00';

WITH TimeCTE AS (
    SELECT @StartDate AS TimeValue
    UNION ALL
    SELECT DATEADD(MINUTE, 15, TimeValue)
    FROM TimeCTE
    WHERE TimeValue < @EndDate
)
INSERT INTO DimTime (TimeKey, FullDateTime, Date, Year, Quarter, Month, 
                     DayOfMonth, DayOfWeek, HourOfDay, MinuteOfHour,
                     FiscalYear, FiscalQuarter, IsWeekend, IsHoliday)
SELECT 
    CAST(FORMAT(TimeValue, 'yyyyMMddHHmm') AS INT) AS TimeKey,
    TimeValue,
    CAST(TimeValue AS DATE) AS Date,
    YEAR(TimeValue),
    DATEPART(QUARTER, TimeValue),
    MONTH(TimeValue),
    DAY(TimeValue),
    DATEPART(WEEKDAY, TimeValue),
    DATEPART(HOUR, TimeValue),
    DATEPART(MINUTE, TimeValue),
    CASE WHEN MONTH(TimeValue) >= 7 THEN YEAR(TimeValue) + 1 ELSE YEAR(TimeValue) END,
    CASE WHEN MONTH(TimeValue) >= 7 THEN DATEPART(QUARTER, TimeValue) - 2 ELSE DATEPART(QUARTER, TimeValue) + 2 END,
    CASE WHEN DATEPART(WEEKDAY, TimeValue) IN (1, 7) THEN 1 ELSE 0 END,
    0  -- Update with holiday logic
FROM TimeCTE
OPTION (MAXRECURSION 0);
```

**DimGeography** - Hierarchical location dimension:

```sql
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID NVARCHAR(50) NOT NULL UNIQUE,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50) NOT NULL, -- Warehouse, Distribution Center, Route Node
    Address NVARCHAR(500),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    PostalCode NVARCHAR(20),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone NVARCHAR(50),
    ParentLocationID NVARCHAR(50),
    EffectiveDate DATE NOT NULL,
    EndDate DATE,
    IsCurrent BIT DEFAULT 1
);

CREATE INDEX IX_DimGeography_LocationID ON DimGeography(LocationID);
CREATE INDEX IX_DimGeography_LocationType ON DimGeography(LocationType);
```

**DimProductGravity** - Product classification with gravity scoring:

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU NVARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100),
    Subcategory NVARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsPerishable BIT DEFAULT 0,
    RequiresRefrigeration BIT DEFAULT 0,
    FragilityScore INT CHECK (FragilityScore BETWEEN 1 AND 10),
    VelocityClass NVARCHAR(20), -- Fast, Medium, Slow
    GravityScore DECIMAL(5,2), -- Calculated composite score
    RecommendedZoneType NVARCHAR(50),
    EffectiveDate DATE NOT NULL,
    EndDate DATE,
    IsCurrent BIT DEFAULT 1
);

-- Calculate gravity score
UPDATE DimProductGravity
SET GravityScore = (
    CASE VelocityClass 
        WHEN 'Fast' THEN 10.0 
        WHEN 'Medium' THEN 5.0 
        WHEN 'Slow' THEN 1.0 
    END * 0.4 +
    FragilityScore * 0.3 +
    CASE WHEN IsPerishable = 1 THEN 10.0 ELSE 0.0 END * 0.3
);
```

### Fact Tables

**FactWarehouseOperations** - Warehouse activity tracking:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    LocationKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType NVARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OperationID NVARCHAR(100) NOT NULL,
    OrderID NVARCHAR(100),
    QuantityUnits INT NOT NULL,
    DurationMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    ZoneID NVARCHAR(50),
    BinLocation NVARCHAR(100),
    OperatorID NVARCHAR(50),
    EquipmentID NVARCHAR(50),
    BatchNumber NVARCHAR(100),
    TemperatureAtOperation DECIMAL(5,2),
    QualityCheckPassed BIT,
    ExceptionCode NVARCHAR(50),
    CONSTRAINT FK_FactWarehouse_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactWarehouse_Location FOREIGN KEY (LocationKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactWarehouse_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

CREATE INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE INDEX IX_FactWarehouse_OperationType ON FactWarehouseOperations(OperationType);
```

**FactFleetTrips** - Fleet and route performance:

```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT,
    OriginLocationKey INT NOT NULL,
    DestinationLocationKey INT,
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    TripID NVARCHAR(100) NOT NULL UNIQUE,
    OrderIDs NVARCHAR(MAX), -- Comma-separated list
    DistanceKm DECIMAL(10,2),
    DurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    LoadVolumeCubicMeters DECIMAL(10,2),
    AverageSpeedKmh DECIMAL(5,2),
    MaxSpeedKmh DECIMAL(5,2),
    HarshBrakingEvents INT DEFAULT 0,
    HarshAccelerationEvents INT DEFAULT 0,
    RouteDeviationKm DECIMAL(10,2),
    WeatherCondition NVARCHAR(50),
    TrafficDelayMinutes DECIMAL(10,2),
    OnTimeDelivery BIT,
    CustomerSignature BIT,
    CONSTRAINT FK_FactFleet_StartTime FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_EndTime FOREIGN KEY (EndTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_Origin FOREIGN KEY (OriginLocationKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactFleet_Destination FOREIGN KEY (DestinationLocationKey) REFERENCES DimGeography(GeographyKey)
);

CREATE INDEX IX_FactFleet_StartTime ON FactFleetTrips(StartTimeKey);
CREATE INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID);
```

## Key Stored Procedures

### Data Loading Procedure

```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @StartDate DATETIME,
    @EndDate DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Incremental load from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, LocationKey, ProductKey, OperationType, OperationID,
        OrderID, QuantityUnits, DurationMinutes, DwellTimeHours,
        ZoneID, BinLocation, OperatorID
    )
    SELECT 
        CAST(FORMAT(stg.OperationTimestamp, 'yyyyMMddHHmm') AS INT) / 15 * 15 AS TimeKey,
        geo.GeographyKey,
        prd.ProductKey,
        stg.OperationType,
        stg.OperationID,
        stg.OrderID,
        stg.Quantity,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DurationMinutes,
        DATEDIFF(MINUTE, stg.FirstSeenTime, stg.OperationTimestamp) / 60.0 AS DwellTimeHours,
        stg.ZoneID,
        stg.BinLocation,
        stg.OperatorID
    FROM StagingWarehouseOperations stg
    INNER JOIN DimGeography geo ON stg.LocationID = geo.LocationID AND geo.IsCurrent = 1
    INNER JOIN DimProductGravity prd ON stg.SKU = prd.SKU AND prd.IsCurrent = 1
    WHERE stg.OperationTimestamp BETWEEN @StartDate AND @EndDate
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f 
            WHERE f.OperationID = stg.OperationID
        );
    
    -- Log execution
    INSERT INTO ETLLog (ProcedureName, ExecutionTime, RowsAffected)
    VALUES ('sp_LoadWarehouseOperations', GETDATE(), @@ROWCOUNT);
END;
GO
```

### Alert Generation Procedure

```sql
CREATE PROCEDURE sp_GenerateFleetAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Detect fleet idle time exceeding threshold
    INSERT INTO AlertQueue (AlertType, Severity, EntityID, Message, DetectedTime)
    SELECT 
        'FleetIdleExceeded',
        'High',
        VehicleID,
        'Vehicle ' + VehicleID + ' idle time ' + 
        CAST(CAST(AVG(IdleTimeMinutes) AS INT) AS NVARCHAR) + 
        ' min exceeds threshold in last 24 hours',
        GETDATE()
    FROM FactFleetTrips
    WHERE StartTimeKey >= CAST(FORMAT(DATEADD(DAY, -1, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY VehicleID
    HAVING AVG(IdleTimeMinutes) / NULLIF(AVG(DurationMinutes), 0) > 0.15
    
    -- Detect temperature deviations
    INSERT INTO AlertQueue (AlertType, Severity, EntityID, Message, DetectedTime)
    SELECT 
        'TemperatureDeviation',
        'Critical',
        OperationID,
        'Temperature deviation detected for order ' + OrderID + 
        ' in zone ' + ZoneID,
        GETDATE()
    FROM FactWarehouseOperations
    WHERE TimeKey >= CAST(FORMAT(DATEADD(HOUR, -1, GETDATE()), 'yyyyMMddHHmm') AS INT)
        AND TemperatureAtOperation NOT BETWEEN -2 AND 2
        AND OperationType IN ('Receiving', 'Shipping');
END;
GO
```

## Power BI DAX Measures

### Cross-Fact KPIs

**Warehouse Efficiency Score:**

```dax
Warehouse Efficiency = 
VAR AvgPickTime = AVERAGE(FactWarehouseOperations[DurationMinutes])
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR BenchmarkPickTime = 3.5
VAR BenchmarkDwellTime = 24
RETURN
    (BenchmarkPickTime / DIVIDE(AvgPickTime, 1, 1) * 0.5 +
     BenchmarkDwellTime / DIVIDE(AvgDwellTime, 1, 1) * 0.5) * 100
```

**Fleet Utilization Rate:**

```dax
Fleet Utilization % = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100
```

**Fuel Efficiency (km per liter):**

```dax
Fuel Efficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKm]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)
```

**Dwell Time by Gravity Zone:**

```dax
Avg Dwell Time by Gravity = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeHours]),
    ALLEXCEPT(DimProductGravity, DimProductGravity[VelocityClass])
)
```

### Time Intelligence

**YTD Shipments:**

```dax
YTD Shipments = 
CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    DATESYTD(DimTime[Date]),
    FactWarehouseOperations[OperationType] = "Shipping"
)
```

**Moving Average - Fleet Distance:**

```dax
Fleet Distance 7-Day MA = 
CALCULATE(
    AVERAGE(FactFleetTrips[DistanceKm]),
    DATESINPERIOD(
        DimTime[Date],
        LASTDATE(DimTime[Date]),
        -7,
        DAY
    )
)
```

## Configuration Patterns

### Row-Level Security Setup

```sql
-- Create security table
CREATE TABLE UserSecurity (
    UserEmail NVARCHAR(255) PRIMARY KEY,
    LocationAccess NVARCHAR(MAX), -- Comma-separated location IDs
    RoleType NVARCHAR(50) -- User, Supervisor, Executive
);

-- Insert sample users
INSERT INTO UserSecurity (UserEmail, LocationAccess, RoleType)
VALUES 
    ('${USER_EMAIL}', 'WH001,WH002', 'Supervisor'),
    ('${MANAGER_EMAIL}', 'ALL', 'Executive');
```

**Power BI RLS DAX:**

```dax
-- Create role: Regional Supervisor
[LocationID] IN 
    SELECTCOLUMNS(
        FILTER(
            UserSecurity,
            UserSecurity[UserEmail] = USERPRINCIPALNAME()
        ),
        "Locations", UserSecurity[LocationAccess]
    )
```

### Incremental Refresh Configuration

In Power BI Desktop:

1. Create parameters:
   - `RangeStart` (Date/Time)
   - `RangeEnd` (Date/Time)

2. Filter fact table query:

```powerquery
= Table.SelectRows(
    Source, 
    each [OperationTimestamp] >= RangeStart and [OperationTimestamp] < RangeEnd
)
```

3. Enable incremental refresh:
   - Refresh data from last: 7 days
   - Store data for: 3 years
   - Refresh complete days only: Yes

## Common Usage Patterns

### Analyzing Cross-Dock Efficiency

```sql
-- Find products with high cross-dock vs. storage ratios
SELECT 
    p.SKU,
    p.ProductName,
    p.VelocityClass,
    COUNT(CASE WHEN w.OperationType = 'Receiving' THEN 1 END) AS TotalReceived,
    AVG(CASE WHEN w.OperationType = 'Shipping' THEN w.DwellTimeHours END) AS AvgDwellHours,
    COUNT(CASE WHEN w.DwellTimeHours < 4 THEN 1 END) * 100.0 / 
        COUNT(*) AS CrossDockPercentage
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE w.TimeKey >= CAST(FORMAT(DATEADD(MONTH, -1, GETDATE()), 'yyyyMMddHHmm') AS INT)
GROUP BY p.SKU, p.ProductName, p.VelocityClass
HAVING COUNT(*) > 100
ORDER BY CrossDockPercentage DESC;
```

### Route Optimization Analysis

```sql
-- Identify routes with high idle time for optimization
SELECT 
    origin.LocationName AS Origin,
    dest.LocationName AS Destination,
    COUNT(*) AS TripCount,
    AVG(f.DistanceKm) AS AvgDistanceKm,
    AVG(f.DurationMinutes) AS AvgDurationMin,
    AVG(f.IdleTimeMinutes) AS AvgIdleMin,
    AVG(f.IdleTimeMinutes) * 100.0 / NULLIF(AVG(f.DurationMinutes), 0) AS IdlePercentage,
    SUM(f.FuelConsumedLiters) AS TotalFuelLiters
FROM FactFleetTrips f
INNER JOIN DimGeography origin ON f.OriginLocationKey = origin.GeographyKey
INNER JOIN DimGeography dest ON f.DestinationLocationKey = dest.GeographyKey
WHERE f.StartTimeKey >= CAST(FORMAT(DATEADD(MONTH, -3, GETDATE()), 'yyyyMMddHHmm') AS INT)
    AND f.IdleTimeMinutes > 0
GROUP BY origin.LocationName, dest.LocationName
HAVING COUNT(*) > 10 AND AVG(f.IdleTimeMinutes) / NULLIF(AVG(f.DurationMinutes), 0) > 0.10
ORDER BY IdlePercentage DESC;
```

### Gravity Zone Recommendations

```sql
-- Suggest products to move to higher gravity zones
WITH ProductVelocity AS (
    SELECT 
        ProductKey,
        COUNT(*) AS PickCount,
        AVG(DurationMinutes) AS AvgPickTime,
        CASE 
            WHEN COUNT(*) > 100 THEN 'Fast'
            WHEN COUNT(*) > 50 THEN 'Medium'
            ELSE 'Slow'
        END AS ActualVelocity
    FROM FactWarehouseOperations
    WHERE OperationType = 'Picking'
        AND TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY ProductKey
)
SELECT 
    p.SKU,
    p.ProductName,
    p.VelocityClass AS ConfiguredVelocity,
    pv.ActualVelocity,
    pv.PickCount,
    pv.AvgPickTime,
    p.GravityScore AS CurrentGravityScore,
    CASE 
        WHEN pv.ActualVelocity = 'Fast' THEN 10.0
        WHEN pv.ActualVelocity = 'Medium' THEN 5.0
        ELSE 1.0
    END * 0.4 + p.FragilityScore * 0.3 AS RecommendedGravityScore
FROM DimProductGravity p
INNER JOIN ProductVelocity pv ON p.ProductKey = pv.ProductKey
WHERE p.VelocityClass <> pv.ActualVelocity
    AND p.IsCurrent = 1
ORDER BY ABS(p.GravityScore - (CASE 
    WHEN pv.ActualVelocity = 'Fast' THEN 10.0
    WHEN pv.ActualVelocity = 'Medium' THEN 5.0
    ELSE 1.0 END * 0.4 + p.FragilityScore * 0.3)) DESC;
```

## Troubleshooting

### Performance Issues

**Slow dashboard refresh:**

```sql
-- Check missing indexes
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    s.user_seeks,
    s.user_scans,
    s.last_user_seek,
    s.last_user_scan
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE database_id = DB_ID()
    AND OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1
ORDER BY s.user_seeks + s.user_scans DESC;

-- Add columnstore index for large fact tables
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouse_CS
ON FactWarehouseOperations (
    TimeKey, LocationKey, ProductKey, OperationType, 
    QuantityUnits, DurationMinutes, DwellTimeHours
);
```

**Partition large tables:**

```sql
-- Partition FactWarehouseOperations by month
CREATE PARTITION FUNCTION PF_Monthly (INT)
AS RANGE RIGHT FOR VALUES (
    202401, 202402, 202403, 202404, 202405, 202406,
    202407, 202408, 202409, 202410, 202411, 202412
);

CREATE PARTITION SCHEME PS_Monthly
AS PARTITION PF_Monthly
ALL TO ([PRIMARY]);

-- Rebuild table on partition scheme (requires recreating table)
```

### Data Quality Issues

**Identify orphaned records:**

```sql
-- Check for missing dimension references
SELECT 'Missing Time Keys' AS Issue, COUNT(*) AS RecordCount
FROM FactWarehouseOperations f
LEFT JOIN DimTime t ON f.TimeKey = t.TimeKey
WHERE t.TimeKey IS NULL

UNION ALL

SELECT 'Missing Product Keys', COUNT(*)
FROM FactWarehouseOperations f
LEFT JOIN DimProductGravity p ON f.ProductKey = p.ProductKey
WHERE p.ProductKey IS NULL;
```

**Fix data type mismatches:**

```sql
-- Ensure TimeKey format consistency
UPDATE FactWarehouseOperations
SET TimeKey = CAST(FORMAT(
    DATEADD(MINUTE, 
        (CAST(TimeKey AS INT) % 100) / 15 * 15, 
        CAST(CAST(TimeKey / 100 AS VARCHAR) + ':00' AS DATETIME)
    ), 'yyyyMMddHHmm'
) AS INT)
WHERE TimeKey % 15 <> 0;
```

### Power BI Connection Issues

**Refresh fails with timeout:**

1. Increase command timeout in Power Query:

```powerquery
let
    Source = Sql.Database(
        "${SQL_SERVER_HOST}", 
        "LogiFleetPulse",
        [CommandTimeout=#duration(0, 1, 0, 0)] // 1 hour timeout
    )
in
    Source
```

2. Use native query for complex operations:

```powerquery
let
    Source = Sql.Database("${SQL_SERVER_HOST}", "LogiFleetPulse"),
    NativeQuery = Value.NativeQuery(
        Source, 
        "EXEC sp_LoadWarehouseOperations @StartDate, @EndDate",
        [StartDate=#datetime(2026,1,1,0,0,0), EndDate=DateTime.LocalNow()]
    )
in
    NativeQuery
```

## Best Practices

1. **Always use surrogate keys** for dimensions instead of natural keys
2. **Implement Type 2 slowly changing dimensions** for historical tracking (EffectiveDate, EndDate, IsCurrent)
3. **Run incremental loads** every 15 minutes rather than full refreshes
4. **Archive old data** to separate tables after 3 years
5. **Use query folding** in Power BI to push transformations to SQL Server
6. **Test RLS** with different user contexts before deploying to production
7. **Monitor ETL execution** via ETLLog table for performance degradation
8. **Document custom measures** with comments for maintainability

## Integration Examples

### Streaming Data from Telematics API

```sql
-- Create external table for real-time telematics feed
CREATE EXTERNAL DATA SOURCE TelematicsAPI
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = '${TELEMATICS_BLOB_URL}',
    CREDENTIAL = TelematicsCredential
);

CREATE EXTERNAL FILE FORMAT JSONFormat
WITH (FORMAT_TYPE = JSON);

CREATE EXTERNAL TABLE StagingTelemetry (
    VehicleID NVARCHAR(50),
    Timestamp DATETIME2,
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2),
    EngineStatus NVARCHAR(20)
)
WITH (
    LOCATION = '/telemetry/',
    DATA_SOURCE = TelematicsAPI,
    FILE_FORMAT = JSONFormat
);
```

### Exporting Alerts to Azure Service Bus

```sql
-- Use SQL Server Service Broker or external stored procedure
CREATE PROCEDURE sp_SendAlertToServiceBus
    @AlertID INT
AS
BEGIN
    DECLARE @Message NVARCHAR(MAX);
    
    SELECT @Message = (
        SELECT AlertType, Severity, EntityID, Message, DetectedTime
        FROM AlertQueue
        WHERE AlertID = @AlertID
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
    );
    
    -- Call Azure Function or Logic App via HTTP
    EXEC sp_OACreate 'MSXML2.ServerXMLHTTP', @Object OUT;
    EXEC sp_OAMethod @Object, 'open', NULL, 'POST', '${AZURE_FUNCTION_URL}', 'false';
    EXEC sp_OAMethod @Object, 'setRequestHeader', NULL, 'Content-Type', 'application/json';
    EXEC sp_OAMethod @Object, 'send', NULL, @Message;
END;
```

This skill provides the foundation for deploying and customizing LogiFleet Pulse for any logistics operation. Focus on adapting the schema to match your specific source systems and KPI requirements.
