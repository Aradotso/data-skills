---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform with multi-fact star schema for warehouse, fleet, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse analytics platform"
  - "deploy supply chain data warehouse schema"
  - "configure Power BI logistics dashboard"
  - "create warehouse gravity zone analytics"
  - "implement fleet telemetry tracking"
  - "build cross-modal supply chain reports"
  - "setup logistics KPI harmonization"
  - "integrate warehouse and fleet data models"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform built on MS SQL Server and Power BI. It implements a multi-fact star schema data warehouse that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer for real-time supply chain analytics.

**Core capabilities:**
- Multi-fact data warehouse with time-phased dimensions
- Unified KPI layer linking warehouse and fleet metrics
- Warehouse Gravity Zones for spatial optimization
- Real-time fleet triage and predictive bottleneck detection
- Power BI dashboards with role-based security

## Project Structure

```
LogiCore-Analytics-Adaptive-Supply-Chain-Pulse/
├── SQL/
│   ├── schema_create.sql          # Core database schema
│   ├── dimensions.sql              # Dimension tables
│   ├── facts.sql                   # Fact tables
│   ├── views.sql                   # Analytical views
│   ├── stored_procedures.sql       # Data loading procedures
│   └── indexes.sql                 # Performance optimization
├── PowerBI/
│   ├── LogiFleet_Pulse_Master.pbit # Power BI template
│   └── semantic_layer.json         # Model metadata
├── config_sample.json              # Configuration template
└── docs/
    ├── data_dictionary.md
    └── deployment_guide.md
```

## Installation & Setup

### 1. Database Deployment

**Prerequisites:**
- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Minimum 10GB free disk space for initial deployment

**Deploy the schema:**

```sql
-- Connect to your SQL Server instance
-- Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Execute schema scripts in order
:r SQL\schema_create.sql
:r SQL\dimensions.sql
:r SQL\facts.sql
:r SQL\views.sql
:r SQL\stored_procedures.sql
:r SQL\indexes.sql
GO
```

**Alternative: Execute via command line:**

```bash
# Windows
sqlcmd -S localhost -d master -i SQL\schema_create.sql
sqlcmd -S localhost -d LogiFleetPulse -i SQL\dimensions.sql
sqlcmd -S localhost -d LogiFleetPulse -i SQL\facts.sql

# Linux/Mac with environment variables
sqlcmd -S $SQL_SERVER -U $SQL_USER -P $SQL_PASSWORD -d LogiFleetPulse -i SQL/schema_create.sql
```

### 2. Configuration

Create `config.json` from the template:

```json
{
  "database": {
    "server": "${SQL_SERVER}",
    "database": "LogiFleetPulse",
    "authentication": "integrated",
    "connectionTimeout": 30
  },
  "dataSources": {
    "wms": {
      "type": "api",
      "endpoint": "${WMS_API_ENDPOINT}",
      "apiKey": "${WMS_API_KEY}",
      "refreshInterval": 900
    },
    "telematics": {
      "type": "api",
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "apiKey": "${TELEMATICS_API_KEY}",
      "refreshInterval": 900
    },
    "weather": {
      "type": "api",
      "endpoint": "https://api.openweathermap.org/data/2.5",
      "apiKey": "${WEATHER_API_KEY}",
      "refreshInterval": 3600
    }
  },
  "alerts": {
    "emailServer": "${SMTP_SERVER}",
    "emailFrom": "${ALERT_EMAIL_FROM}",
    "teamsWebhook": "${TEAMS_WEBHOOK_URL}"
  }
}
```

### 3. Power BI Setup

```powershell
# Install Power BI Desktop (Windows)
winget install Microsoft.PowerBI

# Open the template
Start-Process "PowerBI\LogiFleet_Pulse_Master.pbit"
```

**Connect to data source:**
1. Open `LogiFleet_Pulse_Master.pbit`
2. When prompted, enter SQL Server connection details:
   - Server: `your-server-name`
   - Database: `LogiFleetPulse`
3. Choose authentication method (Windows/SQL Server)
4. Click "Load" to import the semantic model

## Core Database Schema

### Key Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE dbo.DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2(0) NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME(0) NOT NULL,
    HourOfDay TINYINT NOT NULL,
    MinuteBucket TINYINT NOT NULL, -- 0, 15, 30, 45
    DayOfWeek TINYINT NOT NULL,
    DayOfMonth TINYINT NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    MonthOfYear TINYINT NOT NULL,
    Quarter TINYINT NOT NULL,
    FiscalYear SMALLINT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
);

-- DimProductGravity: Product classification with gravity scoring
CREATE TABLE dbo.DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY,
    SKU NVARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100) NOT NULL,
    SubCategory NVARCHAR(100),
    GravityScore DECIMAL(5,2) NOT NULL, -- Calculated score
    VelocityClass NVARCHAR(20) NOT NULL, -- Fast/Medium/Slow
    ValueClass NVARCHAR(20) NOT NULL, -- High/Medium/Low
    FragilityScore TINYINT NOT NULL, -- 1-10
    OptimalZone NVARCHAR(50), -- Recommended storage zone
    EffectiveDate DATE NOT NULL,
    ExpirationDate DATE NOT NULL,
    IsCurrent BIT NOT NULL DEFAULT 1
);

-- DimGeography: Hierarchical location dimension
CREATE TABLE dbo.DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY,
    LocationCode NVARCHAR(50) NOT NULL UNIQUE,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50) NOT NULL, -- Warehouse/DC/Store/Route
    Address NVARCHAR(500),
    City NVARCHAR(100),
    State NVARCHAR(100),
    Country NVARCHAR(100),
    PostalCode NVARCHAR(20),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    ParentLocationKey INT NULL,
    CONSTRAINT FK_Geography_Parent FOREIGN KEY (ParentLocationKey)
        REFERENCES dbo.DimGeography(GeographyKey)
);
```

### Key Fact Tables

```sql
-- FactWarehouseOperations: All warehouse activities
CREATE TABLE dbo.FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    LocationKey INT NOT NULL,
    OperationType NVARCHAR(50) NOT NULL, -- Receiving/Putaway/Picking/Packing/Shipping
    OrderID NVARCHAR(50),
    BatchID NVARCHAR(50),
    QuantityHandled INT NOT NULL,
    DurationMinutes DECIMAL(10,2) NOT NULL,
    DwellTimeMinutes DECIMAL(10,2), -- Time in location before next move
    ZoneCode NVARCHAR(50),
    EmployeeID NVARCHAR(50),
    EquipmentID NVARCHAR(50),
    CompletionStatus NVARCHAR(20) NOT NULL, -- Completed/Delayed/Cancelled
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES dbo.DimTime(TimeKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES dbo.DimProductGravity(ProductKey),
    CONSTRAINT FK_WH_Location FOREIGN KEY (LocationKey) REFERENCES dbo.DimGeography(GeographyKey)
);

-- FactFleetTrips: Fleet and route telemetry
CREATE TABLE dbo.FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY,
    TripID NVARCHAR(50) NOT NULL,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT NOT NULL,
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50) NOT NULL,
    OriginLocationKey INT NOT NULL,
    DestinationLocationKey INT NOT NULL,
    DistanceKM DECIMAL(10,2) NOT NULL,
    DurationMinutes DECIMAL(10,2) NOT NULL,
    IdleTimeMinutes DECIMAL(10,2) NOT NULL,
    FuelConsumedLiters DECIMAL(10,2) NOT NULL,
    LoadWeightKG DECIMAL(10,2),
    StopCount TINYINT NOT NULL,
    DelayMinutes DECIMAL(10,2),
    DelayReason NVARCHAR(100),
    WeatherCondition NVARCHAR(50),
    TrafficSeverity TINYINT, -- 1-5 scale
    CONSTRAINT FK_Fleet_StartTime FOREIGN KEY (StartTimeKey) REFERENCES dbo.DimTime(TimeKey),
    CONSTRAINT FK_Fleet_EndTime FOREIGN KEY (EndTimeKey) REFERENCES dbo.DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Origin FOREIGN KEY (OriginLocationKey) REFERENCES dbo.DimGeography(GeographyKey),
    CONSTRAINT FK_Fleet_Destination FOREIGN KEY (DestinationLocationKey) REFERENCES dbo.DimGeography(GeographyKey)
);

-- FactCrossDock: Direct transfer operations
CREATE TABLE dbo.FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY,
    InboundTripKey BIGINT NOT NULL,
    OutboundTripKey BIGINT NOT NULL,
    ProductKey INT NOT NULL,
    TransferTimeKey INT NOT NULL,
    LocationKey INT NOT NULL,
    Quantity INT NOT NULL,
    TransferDurationMinutes DECIMAL(10,2) NOT NULL,
    CONSTRAINT FK_CD_InboundTrip FOREIGN KEY (InboundTripKey) REFERENCES dbo.FactFleetTrips(TripKey),
    CONSTRAINT FK_CD_OutboundTrip FOREIGN KEY (OutboundTripKey) REFERENCES dbo.FactFleetTrips(TripKey),
    CONSTRAINT FK_CD_Product FOREIGN KEY (ProductKey) REFERENCES dbo.DimProductGravity(ProductKey),
    CONSTRAINT FK_CD_Time FOREIGN KEY (TransferTimeKey) REFERENCES dbo.DimTime(TimeKey),
    CONSTRAINT FK_CD_Location FOREIGN KEY (LocationKey) REFERENCES dbo.DimGeography(GeographyKey)
);
```

## Data Loading Patterns

### Incremental Load Stored Procedure

```sql
CREATE PROCEDURE dbo.usp_LoadWarehouseOperations
    @LastLoadTime DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last successful load if not specified
    IF @LastLoadTime IS NULL
        SELECT @LastLoadTime = MAX(LoadDateTime) 
        FROM dbo.LoadControl 
        WHERE TableName = 'FactWarehouseOperations' 
        AND Status = 'Success';
    
    -- Insert new operations since last load
    INSERT INTO dbo.FactWarehouseOperations (
        TimeKey, ProductKey, LocationKey, OperationType,
        OrderID, BatchID, QuantityHandled, DurationMinutes,
        DwellTimeMinutes, ZoneCode, EmployeeID, EquipmentID,
        CompletionStatus
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        s.OperationType,
        s.OrderID,
        s.BatchID,
        s.Quantity,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime),
        s.DwellTimeMinutes,
        s.ZoneCode,
        s.EmployeeID,
        s.EquipmentID,
        s.Status
    FROM staging.WarehouseOperations s
    INNER JOIN dbo.DimTime t ON s.StartTime = t.FullDateTime
    INNER JOIN dbo.DimProductGravity p ON s.SKU = p.SKU AND p.IsCurrent = 1
    INNER JOIN dbo.DimGeography g ON s.LocationCode = g.LocationCode
    WHERE s.LastModified > @LastLoadTime
    AND s.IsProcessed = 0;
    
    -- Mark staging records as processed
    UPDATE staging.WarehouseOperations
    SET IsProcessed = 1, ProcessedDate = GETDATE()
    WHERE LastModified > @LastLoadTime
    AND IsProcessed = 0;
    
    -- Log completion
    INSERT INTO dbo.LoadControl (TableName, LoadDateTime, RowsProcessed, Status)
    VALUES ('FactWarehouseOperations', GETDATE(), @@ROWCOUNT, 'Success');
END;
GO
```

### Execute incremental load:

```sql
-- Manual execution
EXEC dbo.usp_LoadWarehouseOperations;

-- With specific last load time
EXEC dbo.usp_LoadWarehouseOperations @LastLoadTime = '2026-07-07 06:00:00';
```

## Key Analytical Views

### Cross-Fact KPI View

```sql
CREATE VIEW dbo.vw_WarehouseFleetKPI
AS
SELECT 
    t.DateKey,
    t.MonthOfYear,
    t.Quarter,
    t.FiscalYear,
    g.Country,
    g.State,
    g.LocationName,
    p.Category,
    p.VelocityClass,
    
    -- Warehouse metrics
    SUM(wo.QuantityHandled) AS TotalQuantityHandled,
    AVG(wo.DurationMinutes) AS AvgHandlingTimeMinutes,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
    COUNT(DISTINCT wo.OrderID) AS OrderCount,
    
    -- Fleet metrics
    SUM(ft.DistanceKM) AS TotalDistanceKM,
    SUM(ft.FuelConsumedLiters) AS TotalFuelConsumed,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTimeMinutes,
    AVG(ft.DelayMinutes) AS AvgDelayMinutes,
    
    -- Cross-fact KPIs
    CASE 
        WHEN SUM(ft.DistanceKM) > 0 
        THEN SUM(ft.FuelConsumedLiters) / SUM(ft.DistanceKM) 
        ELSE 0 
    END AS FuelEfficiencyLiterPerKM,
    
    CASE 
        WHEN SUM(wo.QuantityHandled) > 0 
        THEN SUM(ft.DistanceKM) / SUM(wo.QuantityHandled) 
        ELSE 0 
    END AS TransportDistancePerUnit
    
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN dbo.DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN dbo.DimGeography g ON wo.LocationKey = g.GeographyKey
LEFT JOIN dbo.FactFleetTrips ft ON wo.OrderID = ft.TripID 
    AND ft.OriginLocationKey = g.GeographyKey
GROUP BY 
    t.DateKey, t.MonthOfYear, t.Quarter, t.FiscalYear,
    g.Country, g.State, g.LocationName,
    p.Category, p.VelocityClass;
GO
```

### Warehouse Gravity Zone Analysis

```sql
CREATE VIEW dbo.vw_GravityZoneOptimization
AS
SELECT 
    p.ProductKey,
    p.SKU,
    p.ProductName,
    p.Category,
    p.GravityScore,
    p.OptimalZone AS RecommendedZone,
    wo.ZoneCode AS CurrentZone,
    COUNT(DISTINCT wo.OperationKey) AS PickCount,
    AVG(wo.DurationMinutes) AS AvgPickTimeMinutes,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
    SUM(wo.QuantityHandled) AS TotalQuantity,
    
    -- Velocity calculation (picks per day)
    COUNT(DISTINCT wo.OperationKey) / 
        NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0) 
        AS VelocityPicksPerDay,
    
    -- Zone mismatch flag
    CASE 
        WHEN p.OptimalZone <> wo.ZoneCode THEN 'Misaligned'
        ELSE 'Aligned'
    END AS ZoneAlignment
    
FROM dbo.DimProductGravity p
INNER JOIN dbo.FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
WHERE wo.OperationType = 'Picking'
AND p.IsCurrent = 1
AND t.FullDateTime >= DATEADD(MONTH, -3, GETDATE()) -- Last 3 months
GROUP BY 
    p.ProductKey, p.SKU, p.ProductName, p.Category,
    p.GravityScore, p.OptimalZone, wo.ZoneCode;
GO
```

## Alerting System

### Create Alert Threshold Table

```sql
CREATE TABLE dbo.AlertThresholds (
    AlertID INT PRIMARY KEY IDENTITY,
    AlertName NVARCHAR(100) NOT NULL,
    MetricName NVARCHAR(100) NOT NULL,
    ThresholdValue DECIMAL(18,4) NOT NULL,
    ComparisonOperator NVARCHAR(10) NOT NULL, -- >, <, =, >=, <=
    SeverityLevel NVARCHAR(20) NOT NULL, -- Critical/Warning/Info
    NotificationChannels NVARCHAR(200) NOT NULL, -- Email/Teams/SMS
    IsActive BIT NOT NULL DEFAULT 1
);

-- Example thresholds
INSERT INTO dbo.AlertThresholds (AlertName, MetricName, ThresholdValue, ComparisonOperator, SeverityLevel, NotificationChannels)
VALUES 
    ('High Fleet Idle Time', 'IdleTimePercentage', 15.0, '>', 'Warning', 'Email,Teams'),
    ('Excessive Dwell Time', 'DwellTimeHours', 72.0, '>', 'Critical', 'Email,Teams,SMS'),
    ('Low Fuel Efficiency', 'FuelEfficiencyLiterPerKM', 0.35, '>', 'Warning', 'Email'),
    ('Delivery Delays', 'DelayMinutes', 30.0, '>', 'Critical', 'Email,Teams');
```

### Alert Evaluation Procedure

```sql
CREATE PROCEDURE dbo.usp_EvaluateAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Fleet idle time alerts
    INSERT INTO dbo.AlertLog (AlertID, TriggeredDateTime, EntityID, MetricValue, Details)
    SELECT 
        a.AlertID,
        GETDATE(),
        ft.TripID,
        (ft.IdleTimeMinutes / NULLIF(ft.DurationMinutes, 0)) * 100,
        CONCAT('Trip ', ft.TripID, ' had ', 
               CAST(ft.IdleTimeMinutes AS NVARCHAR), ' minutes idle time (',
               CAST((ft.IdleTimeMinutes / NULLIF(ft.DurationMinutes, 0)) * 100 AS DECIMAL(5,2)), '%)')
    FROM dbo.FactFleetTrips ft
    INNER JOIN dbo.AlertThresholds a ON a.MetricName = 'IdleTimePercentage'
    WHERE (ft.IdleTimeMinutes / NULLIF(ft.DurationMinutes, 0)) * 100 > a.ThresholdValue
    AND ft.EndTimeKey >= (SELECT MAX(TimeKey) FROM dbo.DimTime WHERE FullDateTime >= DATEADD(HOUR, -1, GETDATE()))
    AND a.IsActive = 1;
    
    -- Warehouse dwell time alerts
    INSERT INTO dbo.AlertLog (AlertID, TriggeredDateTime, EntityID, MetricValue, Details)
    SELECT 
        a.AlertID,
        GETDATE(),
        p.SKU,
        wo.DwellTimeMinutes / 60.0, -- Convert to hours
        CONCAT('SKU ', p.SKU, ' in zone ', wo.ZoneCode, ' has dwell time of ',
               CAST(wo.DwellTimeMinutes / 60.0 AS DECIMAL(8,2)), ' hours')
    FROM dbo.FactWarehouseOperations wo
    INNER JOIN dbo.DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN dbo.AlertThresholds a ON a.MetricName = 'DwellTimeHours'
    WHERE (wo.DwellTimeMinutes / 60.0) > a.ThresholdValue
    AND wo.TimeKey >= (SELECT MAX(TimeKey) FROM dbo.DimTime WHERE FullDateTime >= DATEADD(HOUR, -1, GETDATE()))
    AND a.IsActive = 1
    AND p.IsCurrent = 1;
    
    -- Send notifications for new alerts
    EXEC dbo.usp_SendAlertNotifications;
END;
GO
```

## Power BI Integration

### DAX Measures for Cross-Fact Analysis

```dax
// Total Warehouse Operations
TotalOperations = COUNTROWS(FactWarehouseOperations)

// Average Handling Efficiency
AvgHandlingEfficiency = 
    DIVIDE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        SUM(FactWarehouseOperations[DurationMinutes]),
        0
    )

// Fleet Utilization Rate
FleetUtilizationRate = 
    VAR TotalTime = SUM(FactFleetTrips[DurationMinutes])
    VAR ActiveTime = TotalTime - SUM(FactFleetTrips[IdleTimeMinutes])
    RETURN DIVIDE(ActiveTime, TotalTime, 0)

// Cross-fact: Transport Cost Per Unit
TransportCostPerUnit = 
    VAR TotalDistance = SUM(FactFleetTrips[DistanceKM])
    VAR TotalUnits = SUM(FactWarehouseOperations[QuantityHandled])
    VAR CostPerKM = 0.75  // Configure as parameter
    RETURN DIVIDE(TotalDistance * CostPerKM, TotalUnits, 0)

// Gravity Zone Alignment Score
GravityAlignmentScore = 
    VAR TotalProducts = COUNTROWS(DimProductGravity)
    VAR AlignedProducts = 
        CALCULATE(
            COUNTROWS(DimProductGravity),
            DimProductGravity[OptimalZone] = FactWarehouseOperations[ZoneCode]
        )
    RETURN DIVIDE(AlignedProducts, TotalProducts, 0)

// Predictive Bottleneck Index (simplified)
BottleneckIndex = 
    VAR HighDwell = 
        CALCULATE(
            COUNTROWS(FactWarehouseOperations),
            FactWarehouseOperations[DwellTimeMinutes] > 4320  // 72 hours
        )
    VAR HighIdle = 
        CALCULATE(
            COUNTROWS(FactFleetTrips),
            FactFleetTrips[IdleTimeMinutes] / FactFleetTrips[DurationMinutes] > 0.15
        )
    RETURN (HighDwell + HighIdle) / (COUNTROWS(FactWarehouseOperations) + COUNTROWS(FactFleetTrips))
```

### Row-Level Security (RLS)

```dax
// In Power BI Desktop: Modeling > Manage Roles

// Role: Regional Manager
[DimGeography[Region]] = USERPRINCIPALNAME()

// Role: Warehouse Supervisor (specific location)
[DimGeography[LocationCode]] IN 
    LOOKUPVALUE(
        'UserLocations'[LocationCode],
        'UserLocations'[UserEmail], USERPRINCIPALNAME()
    )

// Role: Executive (no filter - sees all data)
1 = 1
```

## Common Usage Patterns

### Query 1: Top 10 Products by Handling Time

```sql
SELECT TOP 10
    p.SKU,
    p.ProductName,
    p.Category,
    COUNT(*) AS PickCount,
    AVG(wo.DurationMinutes) AS AvgPickTimeMinutes,
    p.GravityScore,
    p.OptimalZone
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
WHERE wo.OperationType = 'Picking'
AND t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
AND p.IsCurrent = 1
GROUP BY p.SKU, p.ProductName, p.Category, p.GravityScore, p.OptimalZone
ORDER BY AVG(wo.DurationMinutes) DESC;
```

### Query 2: Fleet Performance by Route

```sql
SELECT 
    orig.LocationName AS Origin,
    dest.LocationName AS Destination,
    COUNT(*) AS TripCount,
    AVG(ft.DistanceKM) AS AvgDistanceKM,
    AVG(ft.DurationMinutes) AS AvgDurationMinutes,
    AVG(ft.FuelConsumedLiters) AS AvgFuelLiters,
    AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
    AVG(ft.DelayMinutes) AS AvgDelayMinutes,
    SUM(CASE WHEN ft.DelayMinutes > 0 THEN 1 ELSE 0 END) AS DelayedTripCount
FROM dbo.FactFleetTrips ft
INNER JOIN dbo.DimGeography orig ON ft.OriginLocationKey = orig.GeographyKey
INNER JOIN dbo.DimGeography dest ON ft.DestinationLocationKey = dest.GeographyKey
INNER JOIN dbo.DimTime t ON ft.StartTimeKey = t.TimeKey
WHERE t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY orig.LocationName, dest.LocationName
HAVING COUNT(*) >= 5  -- Routes with at least 5 trips
ORDER BY AVgDelayMinutes DESC;
```

### Query 3: Identify Zone Misalignment Opportunities

```sql
SELECT 
    p.SKU,
    p.ProductName,
    p.OptimalZone AS RecommendedZone,
    wo.ZoneCode AS CurrentZone,
    COUNT(*) AS PickCount,
    AVG(wo.DurationMinutes) AS CurrentAvgPickTime,
    p.GravityScore,
    -- Calculate potential time savings
    AVG(wo.DurationMinutes) * 0.20 AS EstimatedTimeSavingsMinutes  -- Assume 20% improvement
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
WHERE wo.OperationType = 'Picking'
AND p.OptimalZone <> wo.ZoneCode  -- Misaligned
AND t.FullDateTime >= DATEADD(MONTH, -1, GETDATE())
AND p.IsCurrent = 1
GROUP BY p.SKU, p.ProductName, p.OptimalZone, wo.ZoneCode, p.GravityScore
HAVING COUNT(*) >= 10  -- At least 10 picks in current zone
ORDER BY COUNT(*) DESC, p.GravityScore DESC;
```

## Troubleshooting

### Issue: Slow Query Performance

**Diagnosis:**
```sql
-- Check missing indexes
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    s.avg_fragmentation_in_percent,
    s.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'SAMPLED') s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE s.avg_fragmentation_in_percent > 30
AND s.page_count > 1000
ORDER BY s.avg_fragmentation_in_percent DESC;
```

**Solution:**
```sql
-- Rebuild fragmented indexes
ALTER INDEX ALL ON dbo.FactWarehouseOperations REBUILD;
ALTER INDEX ALL ON dbo.FactFleetTrips REBUILD;

-- Update statistics
UPDATE STATISTICS dbo.FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS dbo.FactFleetTrips WITH FULLSCAN;
```

### Issue: Power BI Refresh Failures

**Check refresh errors:**
```sql
-- Query load control table
SELECT TOP 20 *
FROM dbo.LoadControl
WHERE Status <> 'Success'
ORDER BY LoadDateTime DESC;
```

**Common fixes:**
- Verify SQL Server connectivity and credentials
- Check if incremental refresh is properly configured
- Ensure time dimension table is populated for current date range
- Validate foreign key constraints aren't blocking inserts

### Issue: Dimension SCD (Slowly Changing Dimension) Not Updating

**Verify current records:**
```sql
SELECT ProductKey, SKU, EffectiveDate, ExpirationDate, IsCurrent
FROM dbo.DimProductGravity
WHERE SKU = 'YOUR-SKU-CODE'
ORDER BY EffectiveDate DESC;
```

**Manual SCD update:**
```sql
-- Close current record
UPDATE dbo.DimProductGravity
SET ExpirationDate = GETDATE(), IsCurrent = 0
WHERE SKU = 'YOUR-SKU-CODE' AND I
