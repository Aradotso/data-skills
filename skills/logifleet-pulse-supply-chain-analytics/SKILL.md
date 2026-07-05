---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing template for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - set up logifleet pulse supply chain analytics
  - configure power bi logistics dashboard
  - implement warehouse fleet data model
  - deploy sql server supply chain schema
  - create logistics intelligence warehouse
  - build cross-modal supply chain analytics
  - integrate warehouse and fleet telemetry data
  - set up logicore analytics adaptive supply chain
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill provides expertise in deploying and using LogiFleet Pulse, a comprehensive MS SQL Server and Power BI data warehousing template for logistics, fleet management, and supply chain analytics. The project implements a multi-fact star schema that harmonizes warehouse operations, fleet telemetry, inventory management, and external data feeds into a unified semantic layer.

## What LogiFleet Pulse Does

LogiFleet Pulse is a **data modeling and visualization template** that:

- Provides a complete SQL Server star schema for logistics data warehousing
- Integrates warehouse operations (receiving, putaway, picking, packing, shipping)
- Tracks fleet telemetry (GPS, fuel consumption, vehicle diagnostics, route efficiency)
- Manages inventory with aging curves and gravity-based zone optimization
- Connects external data (weather, traffic, supplier performance)
- Delivers Power BI dashboards with real-time KPI monitoring
- Supports cross-fact KPI analysis (e.g., inventory turnover vs. fuel costs)
- Enables predictive bottleneck detection and fleet maintenance prioritization

Key architectural features:
- **Multi-fact star schema** with time-phased dimensions
- **Bridge tables** for many-to-many relationships (routes ↔ storage zones)
- **Composite shared dimensions** (time, geography, product hierarchy)
- **Role-based security** with row-level filtering
- **15-minute granularity** time dimension for real-time analysis

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telematics APIs, ERP systems
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance and create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the schema deployment script
-- The project includes complete DDL scripts for all tables

-- Example: Create DimTime dimension (15-minute granularity)
CREATE TABLE dbo.DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay TINYINT NOT NULL,
    MinuteSlot TINYINT NOT NULL, -- 0, 15, 30, 45
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    MonthNumber TINYINT NOT NULL,
    MonthName VARCHAR(10) NOT NULL,
    Quarter TINYINT NOT NULL,
    FiscalYear SMALLINT NOT NULL,
    FiscalPeriod SMALLINT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
);

-- Create clustered columnstore index for fact tables
CREATE TABLE dbo.FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) NOT NULL,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    EmployeeKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- 'RECEIVE', 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    Quantity INT NOT NULL,
    DurationMinutes DECIMAL(10,2) NOT NULL,
    ZoneGravityScore DECIMAL(5,2) NULL,
    DwellTimeHours DECIMAL(10,2) NULL,
    INDEX CCI_FactWarehouseOperations CLUSTERED COLUMNSTORE
);

-- Create fact table for fleet operations
CREATE TABLE dbo.FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) NOT NULL,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginLocationKey INT NOT NULL,
    DestinationLocationKey INT NOT NULL,
    DistanceKm DECIMAL(10,2) NOT NULL,
    FuelLiters DECIMAL(10,2) NOT NULL,
    IdleMinutes DECIMAL(10,2) NOT NULL,
    LoadingMinutes DECIMAL(10,2) NOT NULL,
    UnloadingMinutes DECIMAL(10,2) NOT NULL,
    LoadWeightKg DECIMAL(10,2) NOT NULL,
    AverageSpeedKmh DECIMAL(5,2) NOT NULL,
    MaxSpeedKmh DECIMAL(5,2) NOT NULL,
    DelayMinutes DECIMAL(10,2) NULL,
    DelayReasonKey INT NULL,
    INDEX CCI_FactFleetTrips CLUSTERED COLUMNSTORE
);
```

### Step 2: Create Dimension Tables

```sql
-- Product dimension with gravity scoring
CREATE TABLE dbo.DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    ProductSKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200) NOT NULL,
    Category VARCHAR(100) NOT NULL,
    Subcategory VARCHAR(100) NULL,
    UnitWeight DECIMAL(10,3) NOT NULL,
    UnitVolume DECIMAL(10,3) NOT NULL,
    IsFragile BIT NOT NULL DEFAULT 0,
    IsPerishable BIT NOT NULL DEFAULT 0,
    StandardCost DECIMAL(12,2) NOT NULL,
    GravityScore DECIMAL(5,2) NOT NULL, -- Calculated: velocity + value + fragility
    EffectiveDate DATE NOT NULL,
    ExpirationDate DATE NULL,
    IsCurrent BIT NOT NULL DEFAULT 1
);

-- Geography dimension (hierarchical)
CREATE TABLE dbo.DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationCode VARCHAR(50) NOT NULL UNIQUE,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(20) NOT NULL, -- 'WAREHOUSE', 'DEPOT', 'CUSTOMER', 'SUPPLIER'
    Latitude DECIMAL(9,6) NULL,
    Longitude DECIMAL(9,6) NULL,
    AddressLine1 VARCHAR(200) NULL,
    City VARCHAR(100) NOT NULL,
    StateProvince VARCHAR(100) NULL,
    PostalCode VARCHAR(20) NULL,
    Country VARCHAR(100) NOT NULL,
    Region VARCHAR(100) NOT NULL, -- 'NORTH_AMERICA', 'EUROPE', etc.
    Continent VARCHAR(50) NOT NULL
);

-- Supplier reliability dimension
CREATE TABLE dbo.DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200) NOT NULL,
    LeadTimeDaysAvg DECIMAL(5,1) NOT NULL,
    LeadTimeDaysStdDev DECIMAL(5,2) NOT NULL,
    DefectRatePercent DECIMAL(5,2) NOT NULL,
    OnTimeDeliveryPercent DECIMAL(5,2) NOT NULL,
    ComplianceScore DECIMAL(5,2) NOT NULL, -- 0-100
    ReliabilityTier VARCHAR(10) NOT NULL, -- 'PLATINUM', 'GOLD', 'SILVER', 'BRONZE'
    LastEvaluationDate DATE NOT NULL
);
```

### Step 3: Populate Time Dimension

```sql
-- Stored procedure to populate time dimension
CREATE PROCEDURE dbo.PopulateDimTime
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    DECLARE @TimeKey INT;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        -- Generate 15-minute slots for each day
        DECLARE @MinuteSlot INT = 0;
        WHILE @MinuteSlot < 1440 -- 24 hours * 60 minutes
        BEGIN
            SET @TimeKey = CAST(FORMAT(@CurrentDateTime, 'yyyyMMdd') AS INT) * 10000 + (@MinuteSlot / 15);
            
            INSERT INTO dbo.DimTime (
                TimeKey, FullDateTime, Date, TimeOfDay, HourOfDay, MinuteSlot,
                DayOfWeek, DayName, WeekOfYear, MonthNumber, MonthName,
                Quarter, FiscalYear, FiscalPeriod, IsWeekend, IsHoliday
            )
            VALUES (
                @TimeKey,
                DATEADD(MINUTE, @MinuteSlot, @CurrentDateTime),
                CAST(@CurrentDateTime AS DATE),
                CAST(DATEADD(MINUTE, @MinuteSlot, CAST(@CurrentDateTime AS DATETIME2)) AS TIME),
                @MinuteSlot / 60,
                (@MinuteSlot % 60) / 15 * 15,
                DATEPART(WEEKDAY, @CurrentDateTime),
                DATENAME(WEEKDAY, @CurrentDateTime),
                DATEPART(WEEK, @CurrentDateTime),
                MONTH(@CurrentDateTime),
                DATENAME(MONTH, @CurrentDateTime),
                DATEPART(QUARTER, @CurrentDateTime),
                YEAR(@CurrentDateTime),
                (MONTH(@CurrentDateTime) - 1) / 3 + 1,
                CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END,
                0 -- Set to 1 for actual holidays via separate update
            );
            
            SET @MinuteSlot = @MinuteSlot + 15;
        END;
        
        SET @CurrentDateTime = DATEADD(DAY, 1, @CurrentDateTime);
    END;
END;
GO

-- Execute to populate 2 years of time data
EXEC dbo.PopulateDimTime '2025-01-01', '2027-12-31';
```

### Step 4: Configure Data Source Connections

Create a configuration file for external data sources:

```json
{
  "dataSources": {
    "wms": {
      "type": "SQL_SERVER",
      "server": "${WMS_SQL_SERVER}",
      "database": "${WMS_DATABASE}",
      "authentication": "SQL_AUTH",
      "username": "${WMS_SQL_USER}",
      "password": "${WMS_SQL_PASSWORD}"
    },
    "telemetry": {
      "type": "REST_API",
      "baseUrl": "${FLEET_TELEMETRY_API_URL}",
      "authType": "BEARER_TOKEN",
      "token": "${FLEET_API_TOKEN}",
      "refreshInterval": 900
    },
    "weather": {
      "type": "REST_API",
      "baseUrl": "https://api.openweathermap.org/data/2.5",
      "apiKey": "${WEATHER_API_KEY}"
    }
  },
  "incremental": {
    "warehouseOps": {
      "watermarkColumn": "LastModifiedDate",
      "frequency": "15min"
    },
    "fleetTrips": {
      "watermarkColumn": "TripEndDateTime",
      "frequency": "15min"
    }
  }
}
```

### Step 5: Create ETL Stored Procedures

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE dbo.LoadFactWarehouseOperations
    @LastWatermark DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO dbo.FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, EmployeeKey,
        OperationType, Quantity, DurationMinutes, ZoneGravityScore, DwellTimeHours
    )
    SELECT 
        dt.TimeKey,
        dw.WarehouseKey,
        dp.ProductKey,
        de.EmployeeKey,
        src.OperationType,
        src.Quantity,
        src.DurationMinutes,
        dp.GravityScore,
        DATEDIFF(HOUR, src.OperationDateTime, src.CompletionDateTime)
    FROM ExternalSource.WarehouseOperations src
    INNER JOIN dbo.DimTime dt 
        ON CAST(src.OperationDateTime AS DATE) = dt.Date
        AND DATEPART(HOUR, src.OperationDateTime) = dt.HourOfDay
        AND (DATEPART(MINUTE, src.OperationDateTime) / 15) * 15 = dt.MinuteSlot
    INNER JOIN dbo.DimWarehouse dw ON src.WarehouseCode = dw.WarehouseCode
    INNER JOIN dbo.DimProduct dp ON src.SKU = dp.ProductSKU AND dp.IsCurrent = 1
    INNER JOIN dbo.DimEmployee de ON src.EmployeeID = de.EmployeeID
    WHERE src.LastModifiedDate > @LastWatermark;
    
    -- Update watermark
    UPDATE dbo.ETLWatermarks
    SET WatermarkValue = GETUTCDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
GO

-- Calculate gravity score for products
CREATE PROCEDURE dbo.UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            COUNT(*) AS PickFrequency,
            AVG(wo.DurationMinutes) AS AvgPickTime,
            AVG(wo.DwellTimeHours) AS AvgDwellTime
        FROM dbo.DimProduct p
        LEFT JOIN dbo.FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
        WHERE wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMdd') AS INT) * 10000
        GROUP BY p.ProductKey
    )
    UPDATE p
    SET GravityScore = 
        (COALESCE(pm.PickFrequency, 0) * 0.4) + -- Velocity weight
        (p.StandardCost / NULLIF((SELECT MAX(StandardCost) FROM dbo.DimProduct), 0) * 100 * 0.3) + -- Value weight
        (CASE WHEN p.IsFragile = 1 THEN 20 ELSE 0 END) + -- Fragility weight
        (CASE WHEN p.IsPerishable = 1 THEN 15 ELSE 0 END) - -- Perishability weight
        (COALESCE(pm.AvgDwellTime, 0) * 0.1) -- Penalize slow movers
    FROM dbo.DimProduct p
    LEFT JOIN ProductMetrics pm ON p.ProductKey = pm.ProductKey
    WHERE p.IsCurrent = 1;
END;
GO
```

## Power BI Configuration

### Step 6: Connect Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter SQL Server connection parameters:
   - Server: `${SQL_SERVER_NAME}`
   - Database: `LogiFleetPulse`
   - Authentication: Windows or SQL Server

### Power BI DAX Measures

```dax
-- Cross-fact KPI: Dwell time per SKU vs Fleet Idle Cost
DwellTime_vs_FleetIdleCost = 
VAR DwellHours = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeHours]),
        ALLEXCEPT(DimProduct, DimProduct[ProductSKU])
    )
VAR IdleCostPerHour = 25 -- USD per hour vehicle idle
VAR FleetIdleHours = 
    CALCULATE(
        SUM(FactFleetTrips[IdleMinutes]) / 60,
        FILTER(
            FactFleetTrips,
            RELATED(DimProduct[ProductSKU]) = SELECTEDVALUE(DimProduct[ProductSKU])
        )
    )
RETURN
    DwellHours * FleetIdleHours * IdleCostPerHour

-- Predictive Bottleneck Index
BottleneckIndex = 
VAR CurrentUtilization = 
    DIVIDE(
        CALCULATE(COUNT(FactWarehouseOperations[OperationKey])),
        [MaxCapacity]
    )
VAR TrendFactor = 
    CALCULATE(
        DIVIDE(
            COUNT(FactWarehouseOperations[OperationKey]),
            CALCULATE(
                COUNT(FactWarehouseOperations[OperationKey]),
                DATEADD(DimTime[Date], -7, DAY)
            )
        ) - 1,
        REMOVEFILTERS(DimTime)
    )
VAR SeasonalityFactor = 
    CALCULATE(
        AVERAGEX(
            FILTER(
                ALLSELECTED(DimTime),
                DimTime[MonthNumber] = SELECTEDVALUE(DimTime[MonthNumber])
            ),
            [DailyOperations]
        ) / [AnnualAvgDailyOperations]
    )
RETURN
    (CurrentUtilization * 0.5) + (TrendFactor * 0.3) + (SeasonalityFactor * 0.2)

-- Fleet Maintenance Priority Score
MaintenancePriorityScore = 
VAR VehicleAge = DATEDIFF(DimVehicle[PurchaseDate], TODAY(), YEAR)
VAR MileageFactor = DimVehicle[CurrentMileage] / DimVehicle[MaxRecommendedMileage]
VAR RevenueAtRisk = 
    CALCULATE(
        SUM(FactFleetTrips[LoadWeightKg]) * [AvgRevenuePerKg],
        FILTER(
            FactFleetTrips,
            FactFleetTrips[VehicleKey] = SELECTEDVALUE(DimVehicle[VehicleKey]) &&
            FactFleetTrips[StartTimeKey] >= [TodayKey]
        )
    )
VAR DiagnosticScore = 
    (DimVehicle[TirePressureWarning] * 0.3) +
    (DimVehicle[EngineWarning] * 0.5) +
    (DimVehicle[BrakeWarning] * 0.2)
RETURN
    (VehicleAge * 10) + (MileageFactor * 30) + (RevenueAtRisk / 1000) + (DiagnosticScore * 40)

-- Warehouse Gravity Zone Efficiency
ZoneEfficiency = 
DIVIDE(
    CALCULATE(
        SUM(FactWarehouseOperations[Quantity]),
        DimProduct[GravityScore] > 70
    ),
    CALCULATE(
        SUM(FactWarehouseOperations[DurationMinutes]),
        DimProduct[GravityScore] > 70
    )
) * 100
```

### Row-Level Security (RLS)

```dax
-- In Power BI, create roles with DAX filters

-- Role: Regional Manager (North America only)
[Region Manager - North America] =
    DimGeography[Region] = "NORTH_AMERICA"

-- Role: Warehouse Supervisor (specific warehouse only)
[Warehouse Supervisor] =
    DimWarehouse[WarehouseCode] = USERPRINCIPALNAME()

-- Role: Fleet Manager (specific vehicle group)
[Fleet Manager] =
    DimVehicle[FleetGroup] IN 
        LOOKUPVALUE(
            UserFleetAccess[FleetGroup],
            UserFleetAccess[Email],
            USERPRINCIPALNAME()
        )
```

## Common Patterns & Usage

### Pattern 1: Incremental ETL Execution

```sql
-- Schedule this via SQL Server Agent every 15 minutes
DECLARE @LastRun DATETIME2;

SELECT @LastRun = WatermarkValue
FROM dbo.ETLWatermarks
WHERE TableName = 'FactWarehouseOperations';

EXEC dbo.LoadFactWarehouseOperations @LastWatermark = @LastRun;
EXEC dbo.LoadFactFleetTrips @LastWatermark = @LastRun;
EXEC dbo.UpdateProductGravityScores;
```

### Pattern 2: Cross-Fact Analysis Query

```sql
-- Find products with high warehouse dwell time AND high fleet idle cost
SELECT 
    p.ProductSKU,
    p.ProductName,
    AVG(wo.DwellTimeHours) AS AvgDwellHours,
    SUM(ft.IdleMinutes) AS TotalFleetIdleMinutes,
    (AVG(wo.DwellTimeHours) * SUM(ft.IdleMinutes) / 60 * 25) AS EstimatedIdleCost
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.DimProduct p ON wo.ProductKey = p.ProductKey
INNER JOIN dbo.FactFleetTrips ft ON wo.ProductKey = ft.ProductKey
WHERE wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd') AS INT) * 10000
GROUP BY p.ProductSKU, p.ProductName
HAVING AVG(wo.DwellTimeHours) > 48
ORDER BY EstimatedIdleCost DESC;
```

### Pattern 3: Temporal Elasticity Simulation

```sql
-- Simulate impact of increasing warehouse capacity from 80% to 95%
WITH CurrentCapacity AS (
    SELECT 
        w.WarehouseCode,
        COUNT(*) AS CurrentOps,
        w.MaxCapacity,
        CAST(COUNT(*) AS FLOAT) / w.MaxCapacity AS CurrentUtilization
    FROM dbo.FactWarehouseOperations wo
    INNER JOIN dbo.DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
    WHERE wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMdd') AS INT) * 10000
    GROUP BY w.WarehouseCode, w.MaxCapacity
),
SimulatedCapacity AS (
    SELECT 
        WarehouseCode,
        CurrentOps,
        MaxCapacity,
        CurrentUtilization,
        0.95 AS TargetUtilization,
        (MaxCapacity * 0.95 - CurrentOps) AS AdditionalCapacity,
        (MaxCapacity * 0.95 - CurrentOps) * 15 AS EstimatedAdditionalMinutes
    FROM CurrentCapacity
)
SELECT 
    sc.*,
    AVG(ft.IdleMinutes) AS AvgFleetIdlePerTrip,
    (sc.EstimatedAdditionalMinutes / NULLIF(AVG(ft.IdleMinutes), 0)) AS PotentialAdditionalTrips
FROM SimulatedCapacity sc
CROSS JOIN (
    SELECT AVG(IdleMinutes) AS IdleMinutes
    FROM dbo.FactFleetTrips
) ft
ORDER BY sc.AdditionalCapacity DESC;
```

### Pattern 4: Anomaly Detection with Tagging

```sql
-- Create anomaly tracking table
CREATE TABLE dbo.AnomalyLog (
    AnomalyID BIGINT IDENTITY(1,1) PRIMARY KEY,
    DetectedDateTime DATETIME2 DEFAULT GETUTCDATE(),
    FactTable VARCHAR(100) NOT NULL,
    FactKey BIGINT NOT NULL,
    AnomalyType VARCHAR(50) NOT NULL,
    Severity VARCHAR(20) NOT NULL, -- 'LOW', 'MEDIUM', 'HIGH', 'CRITICAL'
    Description VARCHAR(500),
    DetectedBy VARCHAR(100), -- System or username
    VerifiedBy VARCHAR(100) NULL,
    VerifiedDateTime DATETIME2 NULL,
    RootCause VARCHAR(500) NULL,
    IsResolved BIT DEFAULT 0
);

-- Detect dwell time anomalies (3 standard deviations)
INSERT INTO dbo.AnomalyLog (FactTable, FactKey, AnomalyType, Severity, Description, DetectedBy)
SELECT 
    'FactWarehouseOperations',
    wo.OperationKey,
    'DWELL_TIME_SPIKE',
    CASE 
        WHEN wo.DwellTimeHours > (avg_dwell.AvgDwell + 3 * std_dwell.StdDev) THEN 'CRITICAL'
        WHEN wo.DwellTimeHours > (avg_dwell.AvgDwell + 2 * std_dwell.StdDev) THEN 'HIGH'
        ELSE 'MEDIUM'
    END,
    'Dwell time of ' + CAST(wo.DwellTimeHours AS VARCHAR(10)) + ' hours exceeds normal range',
    'SYSTEM'
FROM dbo.FactWarehouseOperations wo
CROSS APPLY (
    SELECT AVG(DwellTimeHours) AS AvgDwell
    FROM dbo.FactWarehouseOperations
    WHERE ProductKey = wo.ProductKey
) avg_dwell
CROSS APPLY (
    SELECT STDEV(DwellTimeHours) AS StdDev
    FROM dbo.FactWarehouseOperations
    WHERE ProductKey = wo.ProductKey
) std_dwell
WHERE wo.DwellTimeHours > (avg_dwell.AvgDwell + 2 * std_dwell.StdDev)
    AND wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -1, GETDATE()), 'yyyyMMdd') AS INT) * 10000;
```

## Alerting & Notifications

### Pattern 5: Automated Alert System

```sql
-- Create alert configuration table
CREATE TABLE dbo.AlertConfig (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName VARCHAR(100) NOT NULL,
    AlertType VARCHAR(50) NOT NULL,
    ThresholdValue DECIMAL(18,2),
    NotificationEmail VARCHAR(500), -- Comma-separated
    IsActive BIT DEFAULT 1
);

-- Stored procedure for alert execution
CREATE PROCEDURE dbo.ExecuteAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Fleet idle time alert
    DECLARE @FleetIdleThreshold DECIMAL(5,2);
    SELECT @FleetIdleThreshold = ThresholdValue
    FROM dbo.AlertConfig
    WHERE AlertName = 'FLEET_IDLE_EXCESSIVE' AND IsActive = 1;
    
    IF EXISTS (
        SELECT 1
        FROM dbo.FactFleetTrips ft
        INNER JOIN dbo.DimVehicle v ON ft.VehicleKey = v.VehicleKey
        WHERE ft.StartTimeKey >= CAST(FORMAT(GETDATE(), 'yyyyMMdd') AS INT) * 10000
            AND (ft.IdleMinutes / NULLIF((ft.EndTimeKey - ft.StartTimeKey) / 100, 0)) > @FleetIdleThreshold
    )
    BEGIN
        DECLARE @EmailBody NVARCHAR(MAX) = 
            'ALERT: Fleet vehicles exceeding ' + CAST(@FleetIdleThreshold AS VARCHAR(10)) + 
            '% idle time threshold.' + CHAR(13) + CHAR(13) +
            'Affected vehicles:' + CHAR(13);
        
        SELECT @EmailBody = @EmailBody + 
            v.VehicleCode + ': ' + CAST((ft.IdleMinutes / NULLIF((ft.EndTimeKey - ft.StartTimeKey) / 100, 0) * 100) AS VARCHAR(10)) + 
            '% idle' + CHAR(13)
        FROM dbo.FactFleetTrips ft
        INNER JOIN dbo.DimVehicle v ON ft.VehicleKey = v.VehicleKey
        WHERE ft.StartTimeKey >= CAST(FORMAT(GETDATE(), 'yyyyMMdd') AS INT) * 10000
            AND (ft.IdleMinutes / NULLIF((ft.EndTimeKey - ft.StartTimeKey) / 100, 0)) > @FleetIdleThreshold;
        
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = '${LOGISTICS_ALERT_EMAIL}',
            @subject = 'LogiFleet Alert: Fleet Idle Time Excessive',
            @body = @EmailBody;
    END;
END;
GO
```

## Configuration Reference

### Environment Variables

Set these environment variables for external integrations:

```bash
# SQL Server connection
export SQL_SERVER_NAME="your-server.database.windows.net"
export SQL_DATABASE="LogiFleetPulse"
export SQL_USER="logifleet_user"
export SQL_PASSWORD="your-secure-password"

# WMS integration
export WMS_SQL_SERVER="wms-server.company.com"
export WMS_DATABASE="WarehouseDB"
export WMS_SQL_USER="readonly_user"
export WMS_SQL_PASSWORD="wms-password"

# Fleet telemetry API
export FLEET_TELEMETRY_API_URL="https://api.fleetprovider.com/v2"
export FLEET_API_TOKEN="your-api-token"

# Weather API
export WEATHER_API_KEY="your-openweathermap-key"

# Email notifications
export LOGISTICS_ALERT_EMAIL="logistics-alerts@company.com"
export SMTP_SERVER="smtp.company.com"
export SMTP_PORT="587"
```

### Key SQL Server Settings

```sql
-- Enable polybase for external data sources
EXEC sp_configure 'polybase enabled', 1;
RECONFIGURE;

-- Configure memory allocation for columnstore indexes
EXEC sp_configure 'max server memory (MB)', 32768;
RECONFIGURE;

-- Enable query store for performance monitoring
ALTER DATABASE LogiFleetPulse SET QUERY_STORE = ON;
```

## Troubleshooting

### Issue: Power BI refresh fails with timeout

**Solution**: Implement incremental refresh in Power BI:

1. Add RangeStart and RangeEnd parameters in Power Query:
```m
// In Power Query
let
    Source = Sql.Database(
        SQL_SERVER_NAME, 
        "LogiFleetPulse",
        [Query = "SELECT * FROM FactWarehouseOperations WHERE 
                  FullDateTime >= '" & DateTime.ToText(RangeStart, "yyyy-MM-dd HH:mm:ss") & "' AND 
                  FullDateTime < '" & DateTime.ToText(RangeEnd, "yyyy-MM-dd HH:mm:ss") & "'"]
    )
in
    Source
```

2. Configure incremental refresh policy:
   - Store last 2 years
   - Refresh last 7 days
   - Detect data changes

### Issue: Slow cross-fact queries

**Solution**: Create indexed views for common joins:

```sql
CREATE VIEW dbo.vw_WarehouseFleetC
