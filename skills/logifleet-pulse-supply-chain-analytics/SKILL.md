---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI logistics data warehouse with multi-fact star schema for warehouse, fleet, and cross-dock analytics
triggers:
  - "set up logistics data warehouse"
  - "create supply chain analytics dashboard"
  - "implement warehouse gravity zones"
  - "build fleet optimization reporting"
  - "deploy logifleet pulse analytics"
  - "configure multi-fact star schema for logistics"
  - "integrate warehouse and fleet data"
  - "create cross-dock analytics system"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is a multi-modal logistics intelligence engine combining warehouse operations, fleet telemetry, and inventory management into a unified Power BI analytical platform. Built on MS SQL Server with a custom multi-fact star schema, it harmonizes cross-functional KPIs through time-phased dimensions and bridge tables.

## What It Does

- **Unified Semantic Layer**: Links warehouse operations, fleet trips, and cross-dock activities through shared dimensions
- **Multi-Fact Star Schema**: Time-phased dimensions with 15-minute granularity for real-time operational awareness
- **Warehouse Gravity Zones**: Spatial optimization based on pick frequency, item value, and replenishment lead time
- **Fleet Triage Engine**: Proactive maintenance prioritization weighted by revenue impact
- **Predictive Bottleneck Detection**: ML-powered heatmaps identifying probable congestion points
- **Cross-Fact KPI Harmonization**: Mathematical linking of inventory turnover with fuel burn rates, dwell time with route optimization

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to WMS, TMS, or telemetry data sources

### Deploy SQL Schema

1. Clone the repository:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. Execute the schema creation script in SSMS:
```sql
-- Connect to your SQL Server instance
-- Run the schema deployment script
:r "sql\01_create_schema.sql"
:r "sql\02_create_dimensions.sql"
:r "sql\03_create_facts.sql"
:r "sql\04_create_views.sql"
:r "sql\05_create_procedures.sql"
```

3. Configure data source connections:
```json
{
  "wms_connection": "Server=${WMS_SERVER};Database=${WMS_DB};Integrated Security=true;",
  "telemetry_api": "${TELEMETRY_API_ENDPOINT}",
  "erp_connection": "Server=${ERP_SERVER};Database=${ERP_DB};User Id=${ERP_USER};Password=${ERP_PASSWORD};",
  "refresh_interval_minutes": 15
}
```

### Deploy Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter SQL Server connection parameters when prompted
3. Configure row-level security roles through Modeling > Manage Roles
4. Publish to Power BI Service for scheduled refreshes

## Core Data Model

### Fact Tables

**FactWarehouseOperations**
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- 'Putaway', 'Pick', 'Pack', 'Ship'
    DurationMinutes DECIMAL(10,2),
    DwellHours DECIMAL(10,2),
    QuantityHandled INT,
    GravityZoneID INT,
    OperatorID VARCHAR(50),
    RecordedDateTime DATETIME2 DEFAULT GETDATE(),
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

-- Indexing for performance
CREATE NONCLUSTERED INDEX IX_WH_Time_Warehouse ON FactWarehouseOperations(TimeKey, WarehouseKey) INCLUDE (DurationMinutes, DwellHours);
CREATE NONCLUSTERED INDEX IX_WH_Product_Gravity ON FactWarehouseOperations(ProductKey, GravityZoneID) INCLUDE (QuantityHandled);
```

**FactFleetTrips**
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripDistanceKM DECIMAL(10,2),
    FuelLiters DECIMAL(10,2),
    IdleMinutes DECIMAL(10,2),
    LoadingMinutes DECIMAL(10,2),
    UnloadingMinutes DECIMAL(10,2),
    DelayMinutes DECIMAL(10,2),
    DelayReasonCode VARCHAR(20), -- 'Weather', 'Traffic', 'Mechanical', 'Customer'
    TripStartDateTime DATETIME2,
    TripEndDateTime DATETIME2,
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_Fleet_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE NONCLUSTERED INDEX IX_Fleet_Time_Vehicle ON FactFleetTrips(TimeKey, VehicleKey) INCLUDE (FuelLiters, IdleMinutes);
CREATE NONCLUSTERED INDEX IX_Fleet_Route_Delay ON FactFleetTrips(RouteKey, DelayReasonCode) INCLUDE (DelayMinutes);
```

**FactCrossDock**
```sql
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    QuantityTransferred INT,
    TransferDurationMinutes DECIMAL(10,2),
    StorageDwellMinutes DECIMAL(10,2), -- 0 for true cross-dock
    CONSTRAINT FK_CD_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CD_Inbound FOREIGN KEY (InboundTripKey) REFERENCES FactFleetTrips(TripKey),
    CONSTRAINT FK_CD_Outbound FOREIGN KEY (OutboundTripKey) REFERENCES FactFleetTrips(TripKey)
);
```

### Dimension Tables

**DimTime (15-minute granularity)**
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    DayOfMonth INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    Hour INT NOT NULL,
    Minute15Bucket INT NOT NULL, -- 0, 15, 30, 45
    IsWeekend BIT NOT NULL,
    FiscalPeriod INT,
    FiscalYear INT
);

-- Populate DimTime for 5 years with 15-minute intervals
DECLARE @StartDate DATETIME2 = '2024-01-01';
DECLARE @EndDate DATETIME2 = '2029-12-31 23:45:00';

;WITH TimeSeries AS (
    SELECT @StartDate AS FullDateTime
    UNION ALL
    SELECT DATEADD(MINUTE, 15, FullDateTime)
    FROM TimeSeries
    WHERE DATEADD(MINUTE, 15, FullDateTime) <= @EndDate
)
INSERT INTO DimTime (TimeKey, FullDateTime, Year, Quarter, Month, DayOfMonth, DayOfWeek, DayName, Hour, Minute15Bucket, IsWeekend)
SELECT 
    CONVERT(INT, FORMAT(FullDateTime, 'yyyyMMddHHmm')) AS TimeKey,
    FullDateTime,
    YEAR(FullDateTime),
    DATEPART(QUARTER, FullDateTime),
    MONTH(FullDateTime),
    DAY(FullDateTime),
    DATEPART(WEEKDAY, FullDateTime),
    DATENAME(WEEKDAY, FullDateTime),
    DATEPART(HOUR, FullDateTime),
    (DATEPART(MINUTE, FullDateTime) / 15) * 15,
    CASE WHEN DATEPART(WEEKDAY, FullDateTime) IN (1,7) THEN 1 ELSE 0 END
FROM TimeSeries
OPTION (MAXRECURSION 0);
```

**DimProductGravity (Warehouse Gravity Zones)**
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200),
    CategoryLevel1 VARCHAR(50),
    CategoryLevel2 VARCHAR(50),
    CategoryLevel3 VARCHAR(50),
    GravityScore DECIMAL(5,2), -- Calculated: (Velocity * Value) / (Fragility + ReplenishmentLeadTime)
    VelocityRank INT, -- 1=Fastest moving
    ValueTier VARCHAR(10), -- 'High', 'Medium', 'Low'
    FragilityIndex DECIMAL(3,2), -- 0.0 to 1.0
    ReplenishmentLeadTimeDays INT,
    OptimalGravityZone INT, -- 1=Closest to shipping, 5=Farthest
    LastGravityRecalc DATETIME2 DEFAULT GETDATE()
);

-- Stored procedure to recalculate gravity scores
CREATE PROCEDURE sp_RecalculateProductGravity
AS
BEGIN
    UPDATE DimProductGravity
    SET GravityScore = (
        CAST(VelocityRank AS DECIMAL(10,2)) * 
        CASE ValueTier WHEN 'High' THEN 3 WHEN 'Medium' THEN 2 ELSE 1 END
    ) / (FragilityIndex + (ReplenishmentLeadTimeDays / 10.0)),
    OptimalGravityZone = CASE
        WHEN GravityScore > 50 THEN 1
        WHEN GravityScore > 30 THEN 2
        WHEN GravityScore > 15 THEN 3
        WHEN GravityScore > 5 THEN 4
        ELSE 5
    END,
    LastGravityRecalc = GETDATE();
END;
```

**DimGeography (Hierarchical)**
```sql
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID VARCHAR(50) NOT NULL UNIQUE,
    LocationName NVARCHAR(200),
    LocationType VARCHAR(20), -- 'Warehouse', 'RouteNode', 'CustomerSite'
    AddressLine1 NVARCHAR(200),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50),
    WarehouseCapacityM3 DECIMAL(12,2), -- NULL for non-warehouse locations
    ActiveFrom DATETIME2 DEFAULT GETDATE(),
    ActiveTo DATETIME2
);
```

## Key Stored Procedures

### Incremental ETL for Warehouse Operations
```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Extract from WMS source (assumes linked server or external table)
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType,
        DurationMinutes, DwellHours, QuantityHandled, GravityZoneID, OperatorID
    )
    SELECT 
        CONVERT(INT, FORMAT(wms.OperationDateTime, 'yyyyMMddHHmm')) AS TimeKey,
        geo.GeographyKey,
        prod.ProductKey,
        wms.OperationType,
        wms.DurationMinutes,
        wms.DwellHours,
        wms.Quantity,
        prod.OptimalGravityZone,
        wms.OperatorID
    FROM WMS_LINKED_SERVER.WMS_DB.dbo.Operations wms
    INNER JOIN DimGeography geo ON wms.WarehouseID = geo.LocationID
    INNER JOIN DimProductGravity prod ON wms.SKU = prod.SKU
    WHERE wms.OperationDateTime > @LastLoadDateTime
        AND wms.OperationDateTime <= GETDATE();
    
    -- Update last load timestamp
    UPDATE ETL_Control
    SET LastLoadDateTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
```

### Cross-Fact KPI Query
```sql
CREATE VIEW vw_InventoryTurnoverVsFuelEfficiency AS
SELECT 
    t.Year,
    t.Quarter,
    geo.Country,
    prod.CategoryLevel1,
    -- Warehouse metrics
    SUM(wh.QuantityHandled) AS TotalUnitsHandled,
    AVG(wh.DwellHours) AS AvgDwellHours,
    -- Fleet metrics
    SUM(fleet.TripDistanceKM) AS TotalDistanceKM,
    SUM(fleet.FuelLiters) AS TotalFuelLiters,
    SUM(fleet.FuelLiters) / NULLIF(SUM(fleet.TripDistanceKM), 0) AS FuelEfficiencyLiterPerKM,
    SUM(fleet.IdleMinutes) / NULLIF(SUM(fleet.TripDistanceKM), 0) * 100 AS IdlePercentage,
    -- Cross-fact KPI: Fuel cost attributed to warehouse dwell
    SUM(fleet.FuelLiters) * AVG(wh.DwellHours) / 24.0 AS EstimatedDwellRelatedFuelLiters
FROM FactWarehouseOperations wh
INNER JOIN DimTime t ON wh.TimeKey = t.TimeKey
INNER JOIN DimGeography geo ON wh.WarehouseKey = geo.GeographyKey
INNER JOIN DimProductGravity prod ON wh.ProductKey = prod.ProductKey
LEFT JOIN FactFleetTrips fleet ON 
    fleet.TimeKey = t.TimeKey 
    AND fleet.OriginGeographyKey = geo.GeographyKey
GROUP BY t.Year, t.Quarter, geo.Country, prod.CategoryLevel1;
```

### Automated Alert Procedure
```sql
CREATE PROCEDURE sp_CheckKPIAlerts
AS
BEGIN
    -- Fleet idle time threshold
    INSERT INTO AlertLog (AlertType, Severity, Message, DetectedDateTime)
    SELECT 
        'FleetIdle',
        'High',
        'Vehicle ' + CAST(VehicleKey AS VARCHAR) + ' idle time exceeded 15% on route ' + CAST(RouteKey AS VARCHAR),
        GETDATE()
    FROM (
        SELECT 
            VehicleKey,
            RouteKey,
            SUM(IdleMinutes) * 100.0 / NULLIF(SUM(DATEDIFF(MINUTE, TripStartDateTime, TripEndDateTime)), 0) AS IdlePercentage
        FROM FactFleetTrips
        WHERE TripStartDateTime >= DATEADD(HOUR, -24, GETDATE())
        GROUP BY VehicleKey, RouteKey
    ) metrics
    WHERE IdlePercentage > 15;
    
    -- Warehouse dwell time threshold
    INSERT INTO AlertLog (AlertType, Severity, Message, DetectedDateTime)
    SELECT 
        'WarehouseDwell',
        'Medium',
        'SKU ' + prod.SKU + ' dwell time exceeded 72 hours at warehouse ' + geo.LocationName,
        GETDATE()
    FROM FactWarehouseOperations wh
    INNER JOIN DimProductGravity prod ON wh.ProductKey = prod.ProductKey
    INNER JOIN DimGeography geo ON wh.WarehouseKey = geo.GeographyKey
    WHERE wh.DwellHours > 72
        AND wh.RecordedDateTime >= DATEADD(HOUR, -24, GETDATE());
    
    -- Send alerts via email/SMS (requires Database Mail configuration)
    EXEC msdb.dbo.sp_send_dbmail
        @profile_name = 'LogiFleetAlerts',
        @recipients = '${ALERT_EMAIL}',
        @subject = 'LogiFleet Pulse - KPI Threshold Breached',
        @body = 'Critical alerts detected. Check AlertLog table for details.';
END;
```

## Power BI DAX Measures

### Warehouse Gravity Zone Effectiveness
```dax
GravityZoneAccuracy = 
VAR CurrentPlacements = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[GravityZoneID] = DimProductGravity[OptimalGravityZone]
    )
VAR TotalPlacements = COUNTROWS(FactWarehouseOperations)
RETURN
    DIVIDE(CurrentPlacements, TotalPlacements, 0)
```

### Fleet Triage Priority Score
```dax
FleetTriagePriority = 
SUMX(
    FactFleetTrips,
    (FactFleetTrips[DelayMinutes] * 2) +  // Delay weight
    (FactFleetTrips[IdleMinutes] * 1.5) +  // Idle weight
    (FactFleetTrips[FuelLiters] / FactFleetTrips[TripDistanceKM] * 100) // Inefficiency weight
)
```

### Cross-Dock Efficiency
```dax
CrossDockEfficiency = 
VAR TrueCrossDock = 
    CALCULATE(
        COUNTROWS(FactCrossDock),
        FactCrossDock[StorageDwellMinutes] = 0
    )
VAR TotalCrossDock = COUNTROWS(FactCrossDock)
RETURN
    DIVIDE(TrueCrossDock, TotalCrossDock, 0) * 100
```

## Common Patterns

### Daily Incremental Load Pattern
```sql
-- Job scheduled via SQL Server Agent (daily at 2 AM)
EXEC sp_LoadWarehouseOperations @LastLoadDateTime = (
    SELECT LastLoadDateTime 
    FROM ETL_Control 
    WHERE TableName = 'FactWarehouseOperations'
);

EXEC sp_LoadFleetTrips @LastLoadDateTime = (
    SELECT LastLoadDateTime 
    FROM ETL_Control 
    WHERE TableName = 'FactFleetTrips'
);

EXEC sp_RecalculateProductGravity;

EXEC sp_CheckKPIAlerts;
```

### Role-Level Security Implementation
```sql
-- Create security predicate function
CREATE FUNCTION fn_SecurityPredicate(@UserRole VARCHAR(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN 
    SELECT 1 AS AccessAllowed
    WHERE 
        @UserRole = 'Executive' -- Full access
        OR (@UserRole = 'Supervisor' AND GEOGRAPHY.Country = USER_NAME()) -- Country-level
        OR (@UserRole = 'User' AND GEOGRAPHY.LocationID = USER_NAME()); -- Location-level

-- Apply to fact tables
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(USER_NAME())
ON dbo.FactWarehouseOperations;
```

### External Data Integration (Weather API)
```sql
-- Create external table for weather delays
CREATE EXTERNAL TABLE ExternalWeatherData (
    LocationID VARCHAR(50),
    EventDateTime DATETIME2,
    WeatherCondition VARCHAR(100),
    DelayImpactMinutes INT
)
WITH (
    LOCATION = '/weather-feed/',
    DATA_SOURCE = WeatherAPISource,
    FILE_FORMAT = JSONFormat
);

-- Join with fleet data
SELECT 
    fleet.*,
    weather.WeatherCondition,
    weather.DelayImpactMinutes
FROM FactFleetTrips fleet
LEFT JOIN ExternalWeatherData weather ON
    fleet.OriginGeographyKey = weather.LocationID
    AND ABS(DATEDIFF(MINUTE, fleet.TripStartDateTime, weather.EventDateTime)) < 60;
```

## Troubleshooting

### Issue: Power BI Refresh Timeout
**Cause**: Large fact tables exceeding 30-minute refresh window
**Solution**: Implement partitioning
```sql
-- Partition by month
CREATE PARTITION FUNCTION pf_MonthPartition (INT)
AS RANGE RIGHT FOR VALUES (202401, 202402, 202403, ...);

CREATE PARTITION SCHEME ps_MonthPartition
AS PARTITION pf_MonthPartition ALL TO ([PRIMARY]);

-- Recreate table on partition scheme
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationKey BIGINT IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ...
) ON ps_MonthPartition(TimeKey);
```

### Issue: Slow Cross-Fact Queries
**Cause**: Missing composite indexes
**Solution**: Add covering indexes
```sql
CREATE NONCLUSTERED INDEX IX_Composite_CrossFact
ON FactWarehouseOperations (TimeKey, WarehouseKey, ProductKey)
INCLUDE (DurationMinutes, DwellHours, QuantityHandled);
```

### Issue: Gravity Score Calculation Errors
**Cause**: NULL values in velocity or fragility columns
**Solution**: Add COALESCE in calculation
```sql
UPDATE DimProductGravity
SET GravityScore = (
    COALESCE(CAST(VelocityRank AS DECIMAL(10,2)), 0) * 
    CASE COALESCE(ValueTier, 'Low') WHEN 'High' THEN 3 WHEN 'Medium' THEN 2 ELSE 1 END
) / (COALESCE(FragilityIndex, 0.1) + (COALESCE(ReplenishmentLeadTimeDays, 1) / 10.0));
```

### Issue: Alert Spam
**Cause**: Alerts firing on temporary spikes
**Solution**: Add threshold persistence check
```sql
-- Only alert if condition persists for 3+ consecutive 15-min intervals
WITH AlertCandidates AS (
    SELECT 
        VehicleKey,
        TimeKey,
        LAG(TimeKey, 1) OVER (PARTITION BY VehicleKey ORDER BY TimeKey) AS PrevTime1,
        LAG(TimeKey, 2) OVER (PARTITION BY VehicleKey ORDER BY TimeKey) AS PrevTime2
    FROM FactFleetTrips
    WHERE IdleMinutes > 30
)
INSERT INTO AlertLog (AlertType, Severity, Message)
SELECT 'FleetIdle', 'High', 'Persistent idle on vehicle ' + CAST(VehicleKey AS VARCHAR)
FROM AlertCandidates
WHERE (TimeKey - PrevTime1 = 15) AND (PrevTime1 - PrevTime2 = 15);
```

## Configuration Best Practices

1. **Connection Strings**: Store in encrypted config file or Azure Key Vault
2. **Refresh Cadence**: 15-minute incremental for operational dashboards, daily for strategic reports
3. **Data Retention**: Partition old data (>2 years) to archive tables
4. **Index Maintenance**: Weekly rebuild on fact tables, monthly on dimensions
5. **Security**: Always use Windows Authentication or Azure AD for SQL connections
6. **Monitoring**: Set up Extended Events to track long-running queries

## Environment Variables

```bash
# SQL Server connections
WMS_SERVER=localhost
WMS_DB=WarehouseManagementSystem
ERP_SERVER=localhost
ERP_DB=EnterpriseResourcePlanning
ERP_USER=erp_reader
ERP_PASSWORD=<use_secure_vault>

# External APIs
TELEMETRY_API_ENDPOINT=https://api.telemetry-provider.com/v2/fleet
WEATHER_API_KEY=<use_secure_vault>

# Alerting
ALERT_EMAIL=logistics-alerts@company.com
SMS_GATEWAY_URL=https://sms.provider.com/send

# Power BI Service
POWERBI_WORKSPACE_ID=<workspace_guid>
POWERBI_DATASET_ID=<dataset_guid>
```
