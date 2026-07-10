---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing template for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "deploy multi-fact star schema for warehouse data"
  - "create fleet management data warehouse"
  - "implement supply chain KPI tracking"
  - "build logistics intelligence dashboard"
  - "optimize warehouse gravity zones"
  - "analyze cross-modal supply chain metrics"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is an advanced data warehousing and analytics template for supply chain, logistics, and fleet management. It provides a complete MS SQL Server database schema with multi-fact star schema architecture, Power BI dashboard templates, and integration patterns for warehouse management systems (WMS), telematics, and external data sources.

## What It Does

- **Multi-fact star schema** linking warehouse operations, fleet trips, cross-dock activities, and supplier data
- **Time-phased dimensions** with 15-minute granularity for real-time analytics
- **Cross-fact KPI harmonization** (e.g., inventory turnover vs. fuel consumption)
- **Power BI dashboard templates** with role-based access and natural language queries
- **Predictive analytics** for bottleneck detection and fleet maintenance prioritization
- **Warehouse Gravity Zones™** for spatial optimization based on pick frequency and item value

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Access to source systems: WMS, TMS, telematics APIs, ERP

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance and create a new database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the schema creation scripts (from repository SQL files)
-- Execute in this order:
-- 1. CreateDimensions.sql
-- 2. CreateFactTables.sql
-- 3. CreateBridgeTables.sql
-- 4. CreateViews.sql
-- 5. CreateStoredProcedures.sql
```

### Step 2: Configure Data Connections

Create a configuration file for connection strings:

```json
{
  "connections": {
    "sql_server": "Server=${SQL_SERVER_HOST};Database=LogiFleetPulse;User Id=${SQL_USER};Password=${SQL_PASSWORD};",
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_api": "${TELEMATICS_API_ENDPOINT}",
    "weather_api": "${WEATHER_API_ENDPOINT}"
  },
  "refresh_intervals": {
    "warehouse_operations": 15,
    "fleet_trips": 15,
    "external_feeds": 60
  }
}
```

### Step 3: Load Power BI Template

```powershell
# Open Power BI Desktop
# File > Import > Power BI Template (.pbit)
# Navigate to LogiFleet_Pulse_Master.pbit
# Enter SQL Server connection details when prompted
```

## Core Database Schema

### Dimension Tables

```sql
-- DimTime: Time-phased dimension with 15-minute granularity
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeSlot TIME NOT NULL,
    HourOfDay INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayName VARCHAR(20),
    WeekOfYear INT NOT NULL,
    MonthNumber INT NOT NULL,
    MonthName VARCHAR(20),
    Quarter INT NOT NULL,
    FiscalYear INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
);

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50), -- 'Warehouse', 'Distribution Center', 'Route Node'
    AddressLine1 VARCHAR(200),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    Country VARCHAR(100),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(10,8),
    Longitude DECIMAL(11,8),
    ParentGeographyKey INT,
    HierarchyLevel INT
);

-- DimProductGravity: Product classification with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(200) NOT NULL,
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,2),
    IsPerishable BIT,
    IsFragile BIT,
    PickFrequencyScore INT, -- 1-100
    ValueScore INT, -- 1-100
    GravityScore AS (PickFrequencyScore * 0.6 + ValueScore * 0.4) PERSISTED,
    OptimalStorageZone VARCHAR(50)
);

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200) NOT NULL,
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectPercentage DECIMAL(5,2),
    OnTimeDeliveryRate DECIMAL(5,2),
    ComplianceScore INT, -- 1-100
    LastEvaluationDate DATE
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Warehouse activity tracking
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationStartTime DATETIME NOT NULL,
    OperationEndTime DATETIME,
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime) PERSISTED,
    QuantityHandled INT,
    EmployeeID VARCHAR(50),
    EquipmentID VARCHAR(50),
    StorageZone VARCHAR(50),
    DwellTimeHours DECIMAL(10,2), -- Time spent in storage before next operation
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

-- FactFleetTrips: Fleet telemetry and trip data
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripStartTime DATETIME NOT NULL,
    TripEndTime DATETIME,
    TripDurationMinutes AS DATEDIFF(MINUTE, TripStartTime, TripEndTime) PERSISTED,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightKG DECIMAL(10,2),
    SpeedingEventsCount INT,
    HarshBrakingEventsCount INT,
    MaintenanceScore INT, -- Calculated from telemetry
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

-- FactCrossDock: Cross-docking transfer events
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    ReceiveTime DATETIME NOT NULL,
    ShipTime DATETIME,
    DwellMinutes AS DATEDIFF(MINUTE, ReceiveTime, ShipTime) PERSISTED,
    QuantityTransferred INT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    FOREIGN KEY (InboundTripKey) REFERENCES FactFleetTrips(TripKey),
    FOREIGN KEY (OutboundTripKey) REFERENCES FactFleetTrips(TripKey)
);
```

## Common ETL Patterns

### Incremental Data Loading

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey,
        GeographyKey,
        ProductKey,
        OperationType,
        OperationStartTime,
        OperationEndTime,
        QuantityHandled,
        EmployeeID,
        EquipmentID,
        StorageZone,
        DwellTimeHours
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        s.OperationStartTime,
        s.OperationEndTime,
        s.QuantityHandled,
        s.EmployeeID,
        s.EquipmentID,
        s.StorageZone,
        DATEDIFF(HOUR, s.OperationStartTime, COALESCE(s.NextOperationTime, GETDATE()))
    FROM Staging_WarehouseOperations s
    INNER JOIN DimTime t ON 
        DATEPART(YEAR, s.OperationStartTime) = t.FiscalYear AND
        DATEPART(MONTH, s.OperationStartTime) = t.MonthNumber AND
        DATEPART(DAY, s.OperationStartTime) = DATEPART(DAY, t.DateTime) AND
        DATEPART(HOUR, s.OperationStartTime) = t.HourOfDay
    INNER JOIN DimGeography g ON s.LocationID = g.LocationID
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    WHERE s.LoadedDateTime > @LastLoadDateTime;
    
    -- Update last load timestamp
    UPDATE ETL_Control
    SET LastLoadDateTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
GO
```

### Cross-Fact KPI Calculation

```sql
-- View for unified logistics KPIs
CREATE VIEW vw_UnifiedLogisticsKPIs AS
SELECT 
    t.DateKey,
    t.MonthName,
    g.LocationName AS Warehouse,
    
    -- Warehouse metrics
    AVG(wo.DurationMinutes) AS AvgOperationDuration,
    SUM(wo.QuantityHandled) AS TotalUnitsProcessed,
    AVG(wo.DwellTimeHours) AS AvgDwellTime,
    
    -- Fleet metrics
    SUM(ft.DistanceKM) AS TotalDistanceTraveled,
    SUM(ft.FuelConsumedLiters) AS TotalFuelConsumed,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    
    -- Cross-fact KPIs
    (SUM(ft.FuelConsumedLiters) / NULLIF(SUM(wo.QuantityHandled), 0)) AS FuelPerUnit,
    (AVG(ft.IdleTimeMinutes) / NULLIF(AVG(wo.DurationMinutes), 0)) AS FleetIdleToWarehouseRatio
    
FROM DimTime t
INNER JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
LEFT JOIN FactFleetTrips ft ON 
    t.TimeKey = ft.TimeKey AND 
    (g.GeographyKey = ft.OriginGeographyKey OR g.GeographyKey = ft.DestinationGeographyKey)
GROUP BY t.DateKey, t.MonthName, g.LocationName;
GO
```

## Power BI Integration

### DAX Measures for Advanced Analytics

```dax
// Warehouse Gravity Score Weighted Average
GravityScore_Weighted = 
CALCULATE(
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[QuantityHandled] * 
        RELATED(DimProductGravity[GravityScore])
    ) / 
    SUM(FactWarehouseOperations[QuantityHandled])
)

// Predictive Bottleneck Index
BottleneckIndex = 
VAR CurrentDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR HistoricalAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeHours]),
        DATESINPERIOD(DimTime[Date], LASTDATE(DimTime[Date]), -90, DAY)
    )
VAR CurrentIdleTime = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR HistoricalIdleAvg = 
    CALCULATE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes]),
        DATESINPERIOD(DimTime[Date], LASTDATE(DimTime[Date]), -90, DAY)
    )
RETURN
    (CurrentDwell / HistoricalAvg) * 0.6 + 
    (CurrentIdleTime / HistoricalIdleAvg) * 0.4

// Fleet Triage Priority Score
FleetTriagePriority = 
SUMX(
    FactFleetTrips,
    (100 - FactFleetTrips[MaintenanceScore]) * 
    RELATED(DimProductGravity[GravityScore]) * 0.01 *
    FactFleetTrips[LoadWeightKG]
)
```

### Power BI Report Connection (M Query)

```powerquery
let
    Source = Sql.Database(
        #"SQL_SERVER_HOST" meta [IsParameterQuery=true],
        "LogiFleetPulse"
    ),
    
    // Load dimension tables
    DimTime = Source{[Schema="dbo",Item="DimTime"]}[Data],
    DimGeography = Source{[Schema="dbo",Item="DimGeography"]}[Data],
    DimProductGravity = Source{[Schema="dbo",Item="DimProductGravity"]}[Data],
    
    // Load fact tables with incremental refresh
    FactWarehouseOperations = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
    FactFleetTrips = Source{[Schema="dbo",Item="FactFleetTrips"]}[Data],
    
    // Filter for incremental refresh (last 30 days)
    FilteredOperations = Table.SelectRows(
        FactWarehouseOperations,
        each [OperationStartTime] >= DateTime.LocalNow() - #duration(30,0,0,0)
    ),
    FilteredTrips = Table.SelectRows(
        FactFleetTrips,
        each [TripStartTime] >= DateTime.LocalNow() - #duration(30,0,0,0)
    )
in
    FilteredOperations
```

## Automated Alerting

### Alert Configuration Stored Procedure

```sql
-- Configure automated alerts for KPI breaches
CREATE PROCEDURE usp_ConfigureAlert
    @AlertName VARCHAR(100),
    @MetricQuery NVARCHAR(MAX),
    @ThresholdValue DECIMAL(10,2),
    @ComparisonOperator VARCHAR(10), -- '>', '<', '>=', '<=', '='
    @AlertRecipients VARCHAR(500), -- Comma-separated email list
    @CheckFrequencyMinutes INT = 15
AS
BEGIN
    INSERT INTO AlertConfiguration (
        AlertName,
        MetricQuery,
        ThresholdValue,
        ComparisonOperator,
        AlertRecipients,
        CheckFrequencyMinutes,
        IsActive,
        CreatedDate
    )
    VALUES (
        @AlertName,
        @MetricQuery,
        @ThresholdValue,
        @ComparisonOperator,
        @AlertRecipients,
        @CheckFrequencyMinutes,
        1,
        GETDATE()
    );
END;
GO

-- Example: Configure alert for excessive fleet idle time
EXEC usp_ConfigureAlert
    @AlertName = 'Fleet Idle Time Exceeded',
    @MetricQuery = 'SELECT AVG(IdleTimeMinutes) FROM FactFleetTrips WHERE TimeKey >= (SELECT MAX(TimeKey) FROM DimTime)',
    @ThresholdValue = 45.0,
    @ComparisonOperator = '>',
    @AlertRecipients = '${ALERT_EMAIL_RECIPIENTS}',
    @CheckFrequencyMinutes = 15;
```

### Alert Execution Job

```sql
-- SQL Agent Job to check alerts
CREATE PROCEDURE usp_ExecuteAlerts
AS
BEGIN
    DECLARE @AlertID INT, @AlertName VARCHAR(100), @MetricQuery NVARCHAR(MAX);
    DECLARE @ThresholdValue DECIMAL(10,2), @ComparisonOperator VARCHAR(10);
    DECLARE @Recipients VARCHAR(500), @CurrentValue DECIMAL(10,2);
    
    DECLARE alert_cursor CURSOR FOR
    SELECT AlertID, AlertName, MetricQuery, ThresholdValue, ComparisonOperator, AlertRecipients
    FROM AlertConfiguration
    WHERE IsActive = 1;
    
    OPEN alert_cursor;
    FETCH NEXT FROM alert_cursor INTO @AlertID, @AlertName, @MetricQuery, @ThresholdValue, @ComparisonOperator, @Recipients;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Execute metric query
        EXEC sp_executesql @MetricQuery, N'@Result DECIMAL(10,2) OUTPUT', @Result = @CurrentValue OUTPUT;
        
        -- Check threshold breach
        IF (@ComparisonOperator = '>' AND @CurrentValue > @ThresholdValue) OR
           (@ComparisonOperator = '<' AND @CurrentValue < @ThresholdValue) OR
           (@ComparisonOperator = '>=' AND @CurrentValue >= @ThresholdValue) OR
           (@ComparisonOperator = '<=' AND @CurrentValue <= @ThresholdValue) OR
           (@ComparisonOperator = '=' AND @CurrentValue = @ThresholdValue)
        BEGIN
            -- Send alert email
            EXEC msdb.dbo.sp_send_dbmail
                @profile_name = 'LogiFleetAlerts',
                @recipients = @Recipients,
                @subject = @AlertName,
                @body = CONCAT('Alert: ', @AlertName, CHAR(13), CHAR(10),
                              'Current Value: ', @CurrentValue, CHAR(13), CHAR(10),
                              'Threshold: ', @ComparisonOperator, ' ', @ThresholdValue);
            
            -- Log alert
            INSERT INTO AlertHistory (AlertID, AlertName, CurrentValue, ThresholdValue, AlertDateTime)
            VALUES (@AlertID, @AlertName, @CurrentValue, @ThresholdValue, GETDATE());
        END;
        
        FETCH NEXT FROM alert_cursor INTO @AlertID, @AlertName, @MetricQuery, @ThresholdValue, @ComparisonOperator, @Recipients;
    END;
    
    CLOSE alert_cursor;
    DEALLOCATE alert_cursor;
END;
GO
```

## Indexing Strategy

```sql
-- Critical indexes for performance optimization

-- FactWarehouseOperations indexes
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time_Geography 
ON FactWarehouseOperations(TimeKey, GeographyKey) 
INCLUDE (ProductKey, QuantityHandled, DurationMinutes, DwellTimeHours);

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product_Time 
ON FactWarehouseOperations(ProductKey, TimeKey) 
INCLUDE (GeographyKey, OperationType, DwellTimeHours);

-- FactFleetTrips indexes
CREATE NONCLUSTERED INDEX IX_FactFleet_Time_Origin 
ON FactFleetTrips(TimeKey, OriginGeographyKey) 
INCLUDE (DestinationGeographyKey, DistanceKM, FuelConsumedLiters, IdleTimeMinutes);

CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle_Time 
ON FactFleetTrips(VehicleID, TimeKey) 
INCLUDE (MaintenanceScore, TripDurationMinutes);

-- DimProductGravity index
CREATE NONCLUSTERED INDEX IX_Product_GravityScore 
ON DimProductGravity(GravityScore DESC) 
INCLUDE (SKU, ProductName, OptimalStorageZone);

-- DimGeography spatial index (if using geography data type)
CREATE SPATIAL INDEX SI_Geography_Location 
ON DimGeography(Latitude, Longitude)
USING GEOGRAPHY_GRID;
```

## Configuration Best Practices

### Row-Level Security Setup

```sql
-- Create security roles
CREATE ROLE WarehouseManager;
CREATE ROLE FleetSupervisor;
CREATE ROLE Executive;

-- Create security predicate function
CREATE FUNCTION dbo.fn_SecurityPredicate(@GeographyKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS fn_SecurityPredicate_Result
    WHERE 
        @GeographyKey IN (
            SELECT g.GeographyKey
            FROM dbo.DimGeography g
            INNER JOIN dbo.UserGeographyAccess u ON g.GeographyKey = u.GeographyKey
            WHERE u.UserID = USER_NAME()
        )
        OR IS_MEMBER('Executive') = 1; -- Executives see all data
GO

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(GeographyKey) 
ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(OriginGeographyKey) 
ON dbo.FactFleetTrips
WITH (STATE = ON);
GO
```

## Troubleshooting

### Performance Issues

**Problem**: Slow query performance on cross-fact reports

**Solution**: Ensure partition alignment and proper indexing
```sql
-- Check index usage statistics
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    s.user_seeks,
    s.user_scans,
    s.user_lookups,
    s.user_updates
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE OBJECT_NAME(s.object_id) LIKE 'Fact%'
ORDER BY s.user_seeks + s.user_scans + s.user_lookups DESC;
```

**Problem**: Power BI refresh timeout

**Solution**: Implement incremental refresh and query folding
```powerquery
// Enable incremental refresh parameters in Power BI
// Add RangeStart and RangeEnd parameters
let
    FilteredData = Table.SelectRows(
        Source,
        each [OperationStartTime] >= RangeStart and [OperationStartTime] < RangeEnd
    )
in
    FilteredData
```

### Data Quality Issues

**Problem**: Missing dimension keys causing fact table inserts to fail

**Solution**: Implement staging validation
```sql
-- Validate staging data before loading
CREATE PROCEDURE usp_ValidateStagingData
AS
BEGIN
    -- Check for missing geography references
    SELECT DISTINCT s.LocationID
    FROM Staging_WarehouseOperations s
    LEFT JOIN DimGeography g ON s.LocationID = g.LocationID
    WHERE g.GeographyKey IS NULL;
    
    -- Check for missing product references
    SELECT DISTINCT s.SKU
    FROM Staging_WarehouseOperations s
    LEFT JOIN DimProductGravity p ON s.SKU = p.SKU
    WHERE p.ProductKey IS NULL;
    
    -- Return validation status
    IF EXISTS (
        SELECT 1 FROM Staging_WarehouseOperations s
        LEFT JOIN DimGeography g ON s.LocationID = g.LocationID
        WHERE g.GeographyKey IS NULL
    )
    BEGIN
        RAISERROR('Staging validation failed: Missing geography references', 16, 1);
        RETURN;
    END;
END;
GO
```

### Environment Variables Reference

```bash
# SQL Server connection
SQL_SERVER_HOST=your-server.database.windows.net
SQL_USER=logifleet_admin
SQL_PASSWORD=${SQL_SERVER_PASSWORD}

# External API endpoints
WMS_API_ENDPOINT=https://your-wms-api.com/v1
TELEMATICS_API_ENDPOINT=https://telematics-provider.com/api
WEATHER_API_ENDPOINT=https://api.weather.com/v3

# Alert configuration
ALERT_EMAIL_RECIPIENTS=ops-team@company.com,managers@company.com

# Power BI service
POWERBI_WORKSPACE_ID=${WORKSPACE_ID}
POWERBI_DATASET_ID=${DATASET_ID}
```
