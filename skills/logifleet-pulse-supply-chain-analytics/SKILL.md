---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform with multi-fact star schema for warehouse, fleet, and cross-modal supply chain analytics
triggers:
  - "set up LogiFleet Pulse analytics"
  - "deploy supply chain data warehouse"
  - "create logistics KPI dashboard"
  - "configure warehouse and fleet analytics"
  - "implement multi-fact star schema for logistics"
  - "build Power BI logistics dashboard"
  - "query warehouse operations data"
  - "analyze fleet telemetry with SQL"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI coding agents to work with LogiFleet Pulse, an advanced logistics intelligence platform that combines MS SQL Server data warehousing with Power BI visualization. The system uses a multi-fact star schema to unify warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer for real-time supply chain analytics.

## What LogiFleet Pulse Does

LogiFleet Pulse is a data modeling and analytics template that:

- **Unifies disparate logistics data** from WMS, TMS, telematics, and external APIs
- **Implements multi-fact star schema** with time-phased dimensions for sub-second cross-fact queries
- **Provides Power BI dashboards** for warehouse velocity, fleet optimization, and bottleneck prediction
- **Enables cross-modal analytics** linking warehouse dwell time, fleet routes, and demand patterns
- **Supports predictive maintenance** and proactive bottleneck detection

Core components:
- SQL schema with fact tables (warehouse operations, fleet trips, cross-dock)
- Dimension tables (time, geography, product gravity, supplier reliability)
- Bridge tables for many-to-many relationships
- Power BI template (`.pbit`) with pre-built visualizations
- Stored procedures for incremental loading and alerting

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources (WMS, TMS, telemetry APIs)

### Database Deployment

```sql
-- 1. Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- 2. Create the schema objects
-- Deploy fact tables
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity DECIMAL(18,2),
    DwellTimeMinutes INT,
    PickingTimeSeconds INT,
    PackingTimeSeconds INT,
    OperatorID INT,
    BatchID VARCHAR(50),
    CreatedTimestamp DATETIME2 DEFAULT GETDATE()
)

CREATE TABLE FactFleetTrips (
    TripID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT,
    TripStartTimestamp DATETIME2 NOT NULL,
    TripEndTimestamp DATETIME2,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DelayMinutes INT,
    DelayReasonKey INT,
    TripStatus VARCHAR(20) -- 'Completed', 'InProgress', 'Delayed', 'Cancelled'
)

CREATE TABLE FactCrossDock (
    CrossDockID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripID BIGINT,
    OutboundTripID BIGINT,
    QuantityTransferred DECIMAL(18,2),
    TransferTimeMinutes INT,
    WarehouseKey INT
)

-- 3. Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    Day INT,
    Hour INT,
    Minute INT,
    DayOfWeek VARCHAR(10),
    IsWeekend BIT,
    FiscalPeriod VARCHAR(10),
    TimeSlot VARCHAR(20) -- '00:00-00:15', '00:15-00:30', etc.
)

CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationType VARCHAR(20), -- 'Warehouse', 'RouteNode', 'Customer'
    LocationName VARCHAR(200),
    Address VARCHAR(500),
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    WarehouseCapacityM3 DECIMAL(18,2),
    IsActive BIT DEFAULT 1
)

CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(300),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT,
    RequiresRefrigeration BIT,
    AveragePickFrequency DECIMAL(10,2), -- picks per day
    AverageLeadTimeDays INT,
    UnitValue DECIMAL(18,2),
    GravityScore DECIMAL(5,2), -- Calculated: combines velocity, value, fragility
    RecommendedZone VARCHAR(50) -- 'High-Gravity', 'Medium-Gravity', 'Low-Gravity'
)

CREATE TABLE DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierCode VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200),
    AverageLeadTimeDays DECIMAL(10,2),
    LeadTimeVariance DECIMAL(10,2),
    DefectRatePercentage DECIMAL(5,2),
    OnTimeDeliveryRate DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    LastAuditDate DATE,
    IsActive BIT DEFAULT 1
)

-- 4. Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_TimeProduct 
ON FactWarehouseOperations(TimeKey, ProductKey) INCLUDE (Quantity, DwellTimeMinutes)

CREATE NONCLUSTERED INDEX IX_FactFleetTrips_TimeVehicle 
ON FactFleetTrips(TimeKey, VehicleKey) INCLUDE (DistanceKM, FuelConsumedLiters, IdleTimeMinutes)

CREATE NONCLUSTERED INDEX IX_DimProductGravity_GravityScore 
ON DimProductGravity(GravityScore DESC)

-- 5. Add foreign key constraints
ALTER TABLE FactWarehouseOperations 
ADD CONSTRAINT FK_WarehouseOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey)

ALTER TABLE FactWarehouseOperations 
ADD CONSTRAINT FK_WarehouseOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)

ALTER TABLE FactFleetTrips 
ADD CONSTRAINT FK_FleetTrips_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey)
```

### Populate Dimension Tables

```sql
-- Populate DimTime for 15-minute granularity (example: one day)
DECLARE @StartDate DATETIME2 = '2026-01-01 00:00:00'
DECLARE @EndDate DATETIME2 = '2026-12-31 23:45:00'
DECLARE @CurrentDate DATETIME2 = @StartDate

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO DimTime (
        TimeKey,
        FullDateTime,
        Year,
        Quarter,
        Month,
        Week,
        Day,
        Hour,
        Minute,
        DayOfWeek,
        IsWeekend,
        TimeSlot
    )
    VALUES (
        CONVERT(INT, FORMAT(@CurrentDate, 'yyyyMMddHHmm')),
        @CurrentDate,
        YEAR(@CurrentDate),
        DATEPART(QUARTER, @CurrentDate),
        MONTH(@CurrentDate),
        DATEPART(WEEK, @CurrentDate),
        DAY(@CurrentDate),
        DATEPART(HOUR, @CurrentDate),
        DATEPART(MINUTE, @CurrentDate),
        DATENAME(WEEKDAY, @CurrentDate),
        CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1,7) THEN 1 ELSE 0 END,
        FORMAT(@CurrentDate, 'HH:mm') + '-' + 
        FORMAT(DATEADD(MINUTE, 15, @CurrentDate), 'HH:mm')
    )
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate)
END
```

### Stored Procedure for Incremental Loading

```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load new warehouse operations from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey,
        ProductKey,
        WarehouseKey,
        OperationType,
        Quantity,
        DwellTimeMinutes,
        PickingTimeSeconds,
        PackingTimeSeconds,
        OperatorID,
        BatchID
    )
    SELECT 
        CONVERT(INT, FORMAT(stg.OperationTimestamp, 'yyyyMMddHHmm')),
        p.ProductKey,
        g.GeographyKey,
        stg.OperationType,
        stg.Quantity,
        stg.DwellTimeMinutes,
        stg.PickingTimeSeconds,
        stg.PackingTimeSeconds,
        stg.OperatorID,
        stg.BatchID
    FROM StagingWarehouseOps stg
    INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
    INNER JOIN DimGeography g ON stg.WarehouseCode = g.LocationName
    WHERE stg.OperationTimestamp > @LastLoadTimestamp
        AND stg.IsProcessed = 0
    
    -- Mark as processed
    UPDATE StagingWarehouseOps
    SET IsProcessed = 1, ProcessedTimestamp = GETDATE()
    WHERE OperationTimestamp > @LastLoadTimestamp
        AND IsProcessed = 0
END
GO
```

### Alert Configuration

```sql
CREATE PROCEDURE usp_CheckFleetIdleTimeAlerts
AS
BEGIN
    -- Alert when fleet idle time exceeds 15% of trip duration
    SELECT 
        v.VehicleID,
        v.VehiclePlateNumber,
        t.TripStartTimestamp,
        t.IdleTimeMinutes,
        DATEDIFF(MINUTE, t.TripStartTimestamp, t.TripEndTimestamp) AS TotalTripMinutes,
        CAST(t.IdleTimeMinutes AS FLOAT) / 
        NULLIF(DATEDIFF(MINUTE, t.TripStartTimestamp, t.TripEndTimestamp), 0) * 100 AS IdlePercentage
    FROM FactFleetTrips t
    INNER JOIN DimVehicle v ON t.VehicleKey = v.VehicleKey
    WHERE t.TripStatus = 'Completed'
        AND t.TripEndTimestamp >= DATEADD(DAY, -1, GETDATE())
        AND CAST(t.IdleTimeMinutes AS FLOAT) / 
            NULLIF(DATEDIFF(MINUTE, t.TripStartTimestamp, t.TripEndTimestamp), 0) > 0.15
    ORDER BY IdlePercentage DESC
END
GO
```

## Power BI Configuration

### Connecting to SQL Server

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. When prompted, enter connection details:
   - Server: `your-sql-server.database.windows.net` or `localhost\SQLEXPRESS`
   - Database: `LogiFleetPulse`
   - Data Connectivity mode: **DirectQuery** (for real-time) or **Import** (for faster performance)

### Key DAX Measures

```dax
// Total Warehouse Dwell Time
TotalDwellTime = 
SUM(FactWarehouseOperations[DwellTimeMinutes])

// Average Fleet Idle Percentage
AvgFleetIdlePercentage = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[IdleTimeMinutes]) + 
    SUM(FactFleetTrips[LoadingTimeMinutes]) + 
    SUM(FactFleetTrips[UnloadingTimeMinutes]) + 
    DATEDIFF(
        MIN(FactFleetTrips[TripStartTimestamp]),
        MAX(FactFleetTrips[TripEndTimestamp]),
        MINUTE
    ),
    0
) * 100

// Cross-Fact KPI: Dwell Time per SKU vs Fuel Cost per Route
DwellFuelCorrelation = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR AvgFuel = AVERAGE(FactFleetTrips[FuelConsumedLiters])
VAR ProductFilter = VALUES(DimProductGravity[ProductKey])
RETURN
    CALCULATE(
        AvgFuel,
        FILTER(
            FactFleetTrips,
            FactFleetTrips[TripStartTimestamp] >= 
            CALCULATE(
                MIN(FactWarehouseOperations[CreatedTimestamp]),
                FactWarehouseOperations[ProductKey] IN ProductFilter
            )
        )
    )

// Warehouse Gravity Score
WarehouseGravityIndex = 
SUMX(
    DimProductGravity,
    DimProductGravity[GravityScore] * 
    CALCULATE(SUM(FactWarehouseOperations[Quantity]))
) / SUM(FactWarehouseOperations[Quantity])

// Predictive Bottleneck Index (simplified)
BottleneckIndex = 
VAR CurrentDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR HistoricalAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATEADD(DimTime[FullDateTime], -30, DAY)
    )
VAR DwellVariance = DIVIDE(CurrentDwell - HistoricalAvg, HistoricalAvg, 0)
VAR FleetDelay = AVERAGE(FactFleetTrips[DelayMinutes])
RETURN
    (DwellVariance * 0.6) + (FleetDelay * 0.4) * 100
```

## Common Queries & Analytics Patterns

### Query 1: High-Gravity Products with Excessive Dwell Time

```sql
-- Find high-value products spending too long in warehouse
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.RecommendedZone,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(*) AS OperationCount,
    SUM(w.Quantity) AS TotalQuantity
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE t.FullDateTime >= DATEADD(DAY, -7, GETDATE())
    AND p.GravityScore > 75 -- High gravity products
    AND w.DwellTimeMinutes > 72 * 60 -- More than 72 hours
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.RecommendedZone
ORDER BY AvgDwellTime DESC
```

### Query 2: Fleet Route Efficiency Analysis

```sql
-- Compare fuel efficiency across routes with weather correlation
SELECT 
    r.RouteName,
    g.Region,
    COUNT(DISTINCT f.TripID) AS TripCount,
    AVG(f.DistanceKM) AS AvgDistanceKM,
    AVG(f.FuelConsumedLiters) AS AvgFuelLiters,
    AVG(f.FuelConsumedLiters / NULLIF(f.DistanceKM, 0)) AS FuelPerKM,
    AVG(f.IdleTimeMinutes) AS AvgIdleMinutes,
    AVG(f.DelayMinutes) AS AvgDelayMinutes,
    SUM(CASE WHEN dr.ReasonCategory = 'Weather' THEN 1 ELSE 0 END) AS WeatherDelays
FROM FactFleetTrips f
INNER JOIN DimRoute r ON f.RouteKey = r.RouteKey
INNER JOIN DimGeography g ON r.StartLocationKey = g.GeographyKey
LEFT JOIN DimDelayReason dr ON f.DelayReasonKey = dr.DelayReasonKey
WHERE f.TripStartTimestamp >= DATEADD(MONTH, -1, GETDATE())
    AND f.TripStatus = 'Completed'
GROUP BY r.RouteName, g.Region
HAVING COUNT(DISTINCT f.TripID) > 10
ORDER BY FuelPerKM DESC
```

### Query 3: Cross-Dock Optimization Opportunities

```sql
-- Identify products that could benefit from cross-dock strategy
WITH CrossDockEligible AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        COUNT(DISTINCT w.OperationID) AS TotalOperations,
        AVG(w.DwellTimeMinutes) AS AvgDwellMinutes,
        COUNT(CASE WHEN w.OperationType = 'Receiving' THEN 1 END) AS ReceivingCount,
        COUNT(CASE WHEN w.OperationType = 'Shipping' THEN 1 END) AS ShippingCount
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE w.CreatedTimestamp >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY p.ProductKey, p.SKU, p.ProductName
    HAVING AVG(w.DwellTimeMinutes) < 24 * 60 -- Less than 24 hours average dwell
        AND COUNT(CASE WHEN w.OperationType = 'Receiving' THEN 1 END) > 50
        AND COUNT(CASE WHEN w.OperationType = 'Shipping' THEN 1 END) > 50
)
SELECT 
    ce.*,
    COALESCE(cd.ExistingCrossDockCount, 0) AS CurrentCrossDockOps,
    p.GravityScore
FROM CrossDockEligible ce
LEFT JOIN (
    SELECT ProductKey, COUNT(*) AS ExistingCrossDockCount
    FROM FactCrossDock
    WHERE TimeKey >= CONVERT(INT, FORMAT(DATEADD(MONTH, -3, GETDATE()), 'yyyyMMddHHmm'))
    GROUP BY ProductKey
) cd ON ce.ProductKey = cd.ProductKey
INNER JOIN DimProductGravity p ON ce.ProductKey = p.ProductKey
WHERE COALESCE(cd.ExistingCrossDockCount, 0) < ce.ReceivingCount * 0.3 -- Less than 30% using cross-dock
ORDER BY ce.AvgDwellMinutes ASC, p.GravityScore DESC
```

### Query 4: Supplier Reliability Impact on Warehouse Operations

```sql
-- Correlate supplier delays with warehouse congestion
SELECT 
    s.SupplierName,
    s.OnTimeDeliveryRate,
    s.AverageLeadTimeDays,
    s.LeadTimeVariance,
    COUNT(DISTINCT w.OperationID) AS WarehouseOpsCount,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    SUM(CASE WHEN w.DwellTimeMinutes > 72 * 60 THEN 1 ELSE 0 END) AS ExcessiveDwellCount,
    AVG(p.GravityScore) AS AvgProductGravity
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
INNER JOIN DimSupplierReliability s ON p.PrimarySupplierKey = s.SupplierKey
WHERE w.CreatedTimestamp >= DATEADD(MONTH, -6, GETDATE())
    AND w.OperationType = 'Receiving'
GROUP BY s.SupplierName, s.OnTimeDeliveryRate, s.AverageLeadTimeDays, s.LeadTimeVariance
HAVING COUNT(DISTINCT w.OperationID) > 100
ORDER BY s.OnTimeDeliveryRate ASC, AvgDwellTime DESC
```

## Configuration File Example

Create `config.json` for data source connections:

```json
{
  "database": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "SqlPassword",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}",
    "encrypt": true,
    "trustServerCertificate": false
  },
  "dataSources": {
    "wms": {
      "type": "api",
      "endpoint": "${WMS_API_ENDPOINT}",
      "apiKey": "${WMS_API_KEY}",
      "refreshIntervalMinutes": 15
    },
    "telematics": {
      "type": "api",
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "apiKey": "${TELEMATICS_API_KEY}",
      "refreshIntervalMinutes": 5
    },
    "weather": {
      "type": "api",
      "endpoint": "https://api.weatherapi.com/v1/current.json",
      "apiKey": "${WEATHER_API_KEY}",
      "refreshIntervalMinutes": 60
    }
  },
  "alerting": {
    "enabled": true,
    "smtpServer": "${SMTP_SERVER}",
    "smtpPort": 587,
    "fromEmail": "${ALERT_EMAIL_FROM}",
    "recipients": [
      "${OPERATIONS_MANAGER_EMAIL}",
      "${FLEET_SUPERVISOR_EMAIL}"
    ],
    "thresholds": {
      "fleetIdlePercentage": 15,
      "warehouseDwellHours": 72,
      "fuelEfficiencyVariance": 20
    }
  },
  "powerBI": {
    "workspaceId": "${POWERBI_WORKSPACE_ID}",
    "datasetRefreshSchedule": "*/15 * * * *"
  }
}
```

## Integration Patterns

### Python ETL Script for External API Ingestion

```python
import pyodbc
import requests
import os
from datetime import datetime

# Database connection
conn = pyodbc.connect(
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.getenv('SQL_SERVER_HOST')};"
    f"DATABASE=LogiFleetPulse;"
    f"UID={os.getenv('SQL_USER')};"
    f"PWD={os.getenv('SQL_PASSWORD')}"
)
cursor = conn.cursor()

# Fetch telematics data
telematics_url = os.getenv('TELEMATICS_API_ENDPOINT')
headers = {'Authorization': f"Bearer {os.getenv('TELEMATICS_API_KEY')}"}

response = requests.get(f"{telematics_url}/trips/active", headers=headers)
trips = response.json()

# Insert into staging table
for trip in trips.get('data', []):
    cursor.execute("""
        INSERT INTO StagingFleetTrips (
            VehicleID, RouteID, TripStart, CurrentLatitude, CurrentLongitude,
            FuelLevel, IdleTimeCumulative, DistanceCumulative
        ) VALUES (?, ?, ?, ?, ?, ?, ?, ?)
    """, (
        trip['vehicleId'],
        trip['routeId'],
        datetime.fromisoformat(trip['startTime']),
        trip['location']['lat'],
        trip['location']['lon'],
        trip['fuelLevel'],
        trip['idleTimeMinutes'],
        trip['distanceKM']
    ))

conn.commit()
cursor.close()
conn.close()
```

### Power Automate Flow for Alerting

Create a flow triggered by SQL Server stored procedure:

1. **Trigger**: Recurrence (every 15 minutes)
2. **Action**: Execute SQL query `EXEC usp_CheckFleetIdleTimeAlerts`
3. **Condition**: If result set contains rows
4. **Action**: Send email via Outlook/Teams with formatted alert
5. **Action**: Create task in Planner/Jira for operations team

## Troubleshooting

### Issue: Slow Power BI Dashboard Performance

**Solution**: Switch from Import to DirectQuery mode, or optimize SQL views

```sql
-- Create indexed view for common dashboard query
CREATE VIEW vw_DailyWarehouseMetrics
WITH SCHEMABINDING
AS
SELECT 
    t.Year,
    t.Month,
    t.Day,
    w.WarehouseKey,
    COUNT_BIG(*) AS OperationCount,
    SUM(ISNULL(wo.Quantity, 0)) AS TotalQuantity,
    AVG(ISNULL(wo.DwellTimeMinutes, 0)) AS AvgDwellTime
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN dbo.DimGeography w ON wo.WarehouseKey = w.GeographyKey
GROUP BY t.Year, t.Month, t.Day, w.WarehouseKey
GO

CREATE UNIQUE CLUSTERED INDEX IX_DailyWarehouseMetrics 
ON vw_DailyWarehouseMetrics(Year, Month, Day, WarehouseKey)
```

### Issue: Time Dimension Loading Takes Too Long

**Solution**: Use batch inserts and disable constraints temporarily

```sql
ALTER TABLE DimTime NOCHECK CONSTRAINT ALL

-- Bulk insert with batch size
DECLARE @BatchSize INT = 10000
DECLARE @Counter INT = 0

WHILE @CurrentDate <= @EndDate
BEGIN
    -- Insert logic here
    SET @Counter = @Counter + 1
    
    IF @Counter % @BatchSize = 0
    BEGIN
        CHECKPOINT
    END
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate)
END

ALTER TABLE DimTime CHECK CONSTRAINT ALL
```

### Issue: Foreign Key Violations During Load

**Solution**: Use staging tables with explicit lookup validation

```sql
-- Validate before inserting into fact table
SELECT 
    stg.*,
    CASE WHEN p.ProductKey IS NULL THEN 'Missing Product' ELSE 'OK' END AS ProductStatus,
    CASE WHEN t.TimeKey IS NULL THEN 'Missing Time' ELSE 'OK' END AS TimeStatus
FROM StagingWarehouseOps stg
LEFT JOIN DimProductGravity p ON stg.SKU = p.SKU
LEFT JOIN DimTime t ON CONVERT(INT, FORMAT(stg.OperationTimestamp, 'yyyyMMddHHmm')) = t.TimeKey
WHERE stg.IsProcessed = 0
    AND (p.ProductKey IS NULL OR t.TimeKey IS NULL)
```

### Issue: Power BI Refresh Fails with Timeout

**Solution**: Partition fact tables by date and refresh incrementally

```sql
-- Create partition function
CREATE PARTITION FUNCTION PF_Monthly (INT)
AS RANGE RIGHT FOR VALUES (
    202601, 202602, 202603, 202604, 202605, 202606,
    202607, 202608, 202609, 202610, 202611, 202612
)

CREATE PARTITION SCHEME PS_Monthly
AS PARTITION PF_Monthly ALL TO ([PRIMARY])

-- Create partitioned table
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationID BIGINT IDENTITY(1,1),
    TimeKey INT NOT NULL,
    -- other columns...
    CONSTRAINT PK_FactWarehouseOps_Part PRIMARY KEY (TimeKey, OperationID)
) ON PS_Monthly(TimeKey)
```

## Best Practices

1. **Always use parameterized queries** when building dynamic SQL
2. **Implement row-level security** in Power BI for multi-tenant scenarios
3. **Schedule fact table loads during off-peak hours** (e.g., 2 AM)
4. **Monitor query execution plans** using SQL Server Extended Events
5. **Version control your Power BI reports** using Git with `.pbip` format
6. **Document all calculated columns and measures** in Power BI using descriptions
7. **Use aggregate tables** for year-over-year comparisons to improve performance
8. **Implement slowly changing dimensions** (Type 2) for tracking historical changes in product attributes

## Additional Resources

- SQL Server indexing strategy: https://docs.microsoft.com/sql/relational-databases/indexes/
- Power BI DAX best practices: https://docs.microsoft.com/power-bi/guidance/dax-best-practices
- Star schema design patterns: Kimball Group's data warehouse toolkit
