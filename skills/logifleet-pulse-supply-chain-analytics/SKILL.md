---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing framework for logistics, fleet, and supply chain analytics with multi-fact star schema
triggers:
  - "set up logifleet pulse supply chain analytics"
  - "deploy logifleet warehouse and fleet data model"
  - "configure power bi logistics dashboard"
  - "implement multi-fact star schema for supply chain"
  - "create warehouse gravity zones analytics"
  - "build fleet optimization data warehouse"
  - "set up cross-modal logistics intelligence"
  - "integrate sql server logistics data pipeline"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and analytics framework for logistics and supply chain operations. It provides:

- **Multi-fact star schema** for warehouse operations, fleet trips, and cross-dock activities
- **Time-phased dimensions** (15-minute granularity) for operational visibility
- **Power BI dashboards** with real-time refresh capabilities
- **Predictive analytics** for bottleneck detection and fleet optimization
- **Warehouse Gravity Zones™** - spatial optimization based on pick frequency, fragility, and lead time
- **Cross-fact KPI harmonization** linking inventory, fleet, and operations metrics

The system integrates data from WMS, telematics, GPS feeds, supplier portals, and external APIs to provide unified logistics intelligence.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (recommended) or Azure SQL Database
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telematics feeds

### Step 1: Deploy SQL Schema

```sql
-- Execute the core schema deployment script
-- This creates all fact tables, dimensions, views, and stored procedures

USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables first
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTimeValue DATETIME NOT NULL,
    TimeSlot15Min INT NOT NULL,
    HourOfDay INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayOfMonth INT NOT NULL,
    WeekOfYear INT NOT NULL,
    MonthNumber INT NOT NULL,
    QuarterNumber INT NOT NULL,
    FiscalYear INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT DEFAULT 0,
    CONSTRAINT UQ_DimTime_DateTime UNIQUE(DateTimeValue)
);
GO

CREATE NONCLUSTERED INDEX IX_DimTime_Date ON DimTime(DateTimeValue);
CREATE NONCLUSTERED INDEX IX_DimTime_FiscalYear_Quarter ON DimTime(FiscalYear, QuarterNumber);
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationCode NVARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50) NOT NULL, -- Warehouse, Distribution Center, Route Node
    AddressLine1 NVARCHAR(200),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    PostalCode NVARCHAR(20),
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone NVARCHAR(50),
    IsActive BIT DEFAULT 1,
    CONSTRAINT UQ_DimGeography_Code UNIQUE(LocationCode)
);
GO

CREATE NONCLUSTERED INDEX IX_DimGeography_Type ON DimGeography(LocationType);
CREATE NONCLUSTERED INDEX IX_DimGeography_Country_Region ON DimGeography(Country, Region);
GO

CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(100) NOT NULL,
    ProductName NVARCHAR(300) NOT NULL,
    ProductCategory NVARCHAR(100),
    ProductSubcategory NVARCHAR(100),
    UnitOfMeasure NVARCHAR(50),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsPerishable BIT DEFAULT 0,
    ShelfLifeDays INT,
    FragilityScore DECIMAL(5,2), -- 0-10 scale
    UnitValue DECIMAL(12,2),
    IsActive BIT DEFAULT 1,
    CONSTRAINT UQ_DimProduct_SKU UNIQUE(SKU)
);
GO

CREATE NONCLUSTERED INDEX IX_DimProduct_Category ON DimProduct(ProductCategory, ProductSubcategory);
GO

CREATE TABLE DimProductGravity (
    ProductGravityKey INT PRIMARY KEY IDENTITY(1,1),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProduct(ProductKey),
    EffectiveDate DATE NOT NULL,
    GravityScore DECIMAL(5,2) NOT NULL, -- Composite score: velocity + value + fragility
    VelocityRank INT, -- Based on pick frequency
    ValueRank INT, -- Based on unit value
    RecommendedZoneType NVARCHAR(50), -- Hot, Warm, Cold
    OptimalDistanceFromDock DECIMAL(6,2), -- Meters
    CalculationTimestamp DATETIME DEFAULT GETDATE(),
    CONSTRAINT UQ_DimProductGravity_Product_Date UNIQUE(ProductKey, EffectiveDate)
);
GO

CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode NVARCHAR(50) NOT NULL,
    SupplierName NVARCHAR(200) NOT NULL,
    SupplierType NVARCHAR(50),
    Country NVARCHAR(100),
    LeadTimeDaysAvg DECIMAL(6,2),
    LeadTimeDaysStdDev DECIMAL(6,2),
    DefectRatePercent DECIMAL(5,2),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100 scale
    IsActive BIT DEFAULT 1,
    CONSTRAINT UQ_DimSupplier_Code UNIQUE(SupplierCode)
);
GO

CREATE TABLE DimFleet (
    FleetKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID NVARCHAR(50) NOT NULL,
    VehicleType NVARCHAR(50) NOT NULL, -- Van, Truck, Semi, Refrigerated
    Make NVARCHAR(100),
    Model NVARCHAR(100),
    ModelYear INT,
    FuelType NVARCHAR(50),
    MaxLoadCapacityKg DECIMAL(10,2),
    MaxLoadCapacityM3 DECIMAL(10,2),
    IsRefrigerated BIT DEFAULT 0,
    AcquisitionDate DATE,
    LastMaintenanceDate DATE,
    MaintenanceIntervalKm INT,
    CurrentOdometerKm INT,
    IsActive BIT DEFAULT 1,
    CONSTRAINT UQ_DimFleet_VehicleID UNIQUE(VehicleID)
);
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProduct(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplier(SupplierKey),
    OperationType NVARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OrderID NVARCHAR(100),
    QuantityUnits INT NOT NULL,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes DECIMAL(10,2),
    ZoneCode NVARCHAR(50),
    DistanceWalkedMeters DECIMAL(10,2),
    OperatorID NVARCHAR(50),
    BatchID NVARCHAR(100),
    IsAnomalous BIT DEFAULT 0,
    AnomalyNotes NVARCHAR(500),
    CreatedTimestamp DATETIME DEFAULT GETDATE()
);
GO

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Geography ON FactWarehouseOperations(GeographyKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Type_Time ON FactWarehouseOperations(OperationType, TimeKey);
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    FleetKey INT NOT NULL FOREIGN KEY REFERENCES DimFleet(FleetKey),
    StartTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    EndTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TripID NVARCHAR(100) NOT NULL,
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    LoadVolumeM3 DECIMAL(10,2),
    DriverID NVARCHAR(50),
    RouteCode NVARCHAR(50),
    WeatherCondition NVARCHAR(50),
    TrafficDelay INT DEFAULT 0, -- Minutes
    MaintenanceIssue BIT DEFAULT 0,
    MaintenanceNotes NVARCHAR(500),
    IsCompleted BIT DEFAULT 0,
    CreatedTimestamp DATETIME DEFAULT GETDATE()
);
GO

CREATE NONCLUSTERED INDEX IX_FactTrips_Fleet ON FactFleetTrips(FleetKey);
CREATE NONCLUSTERED INDEX IX_FactTrips_StartTime ON FactFleetTrips(StartTimeKey);
CREATE NONCLUSTERED INDEX IX_FactTrips_Route ON FactFleetTrips(OriginGeographyKey, DestinationGeographyKey);
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProduct(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    TransferID NVARCHAR(100),
    QuantityUnits INT NOT NULL,
    DockTimeMinutes INT,
    BypassStorage BIT DEFAULT 1,
    TemperatureCompliance BIT DEFAULT 1,
    QualityCheckPassed BIT DEFAULT 1,
    CreatedTimestamp DATETIME DEFAULT GETDATE()
);
GO

CREATE NONCLUSTERED INDEX IX_FactCrossDock_Time ON FactCrossDock(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactCrossDock_Geography ON FactCrossDock(GeographyKey);
GO
```

### Step 2: Create Stored Procedures for Data Loading

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- If no timestamp provided, load all data from today
    IF @LastLoadTimestamp IS NULL
        SET @LastLoadTimestamp = CAST(GETDATE() AS DATE);
    
    -- Insert new operations from staging table (assumes external data source)
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, OrderID, QuantityUnits, DwellTimeMinutes,
        ProcessingTimeMinutes, ZoneCode, DistanceWalkedMeters, OperatorID
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        stg.OperationType,
        stg.OrderID,
        stg.QuantityUnits,
        DATEDIFF(MINUTE, stg.OperationStartTime, stg.OperationEndTime) AS DwellTimeMinutes,
        stg.ProcessingTimeMinutes,
        stg.ZoneCode,
        stg.DistanceWalkedMeters,
        stg.OperatorID
    FROM StagingWarehouseOperations stg
    INNER JOIN DimTime t ON CAST(stg.OperationStartTime AS DATETIME) = t.DateTimeValue
    INNER JOIN DimGeography g ON stg.LocationCode = g.LocationCode
    INNER JOIN DimProduct p ON stg.SKU = p.SKU
    LEFT JOIN DimSupplier s ON stg.SupplierCode = s.SupplierCode
    WHERE stg.LoadTimestamp > @LastLoadTimestamp;
    
    RETURN @@ROWCOUNT;
END;
GO

-- Stored procedure to calculate product gravity scores
CREATE PROCEDURE sp_CalculateProductGravity
    @CalculationDate DATE = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    IF @CalculationDate IS NULL
        SET @CalculationDate = CAST(GETDATE() AS DATE);
    
    -- Calculate velocity rank (last 90 days)
    WITH PickFrequency AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickCount,
            RANK() OVER (ORDER BY COUNT(*) DESC) AS VelocityRank
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
            AND TimeKey IN (SELECT TimeKey FROM DimTime WHERE DateTimeValue >= DATEADD(DAY, -90, @CalculationDate))
        GROUP BY ProductKey
    ),
    ValueRanking AS (
        SELECT 
            ProductKey,
            UnitValue,
            RANK() OVER (ORDER BY UnitValue DESC) AS ValueRank
        FROM DimProduct
        WHERE IsActive = 1
    )
    INSERT INTO DimProductGravity (
        ProductKey, EffectiveDate, GravityScore, VelocityRank, ValueRank, RecommendedZoneType
    )
    SELECT 
        p.ProductKey,
        @CalculationDate,
        -- Composite score: 50% velocity, 30% value, 20% fragility
        (COALESCE(pf.VelocityRank, 999) * 0.5 + COALESCE(vr.ValueRank, 999) * 0.3 + p.FragilityScore * 0.2) AS GravityScore,
        pf.VelocityRank,
        vr.ValueRank,
        CASE 
            WHEN COALESCE(pf.VelocityRank, 999) <= 100 THEN 'Hot'
            WHEN COALESCE(pf.VelocityRank, 999) <= 500 THEN 'Warm'
            ELSE 'Cold'
        END AS RecommendedZoneType
    FROM DimProduct p
    LEFT JOIN PickFrequency pf ON p.ProductKey = pf.ProductKey
    LEFT JOIN ValueRanking vr ON p.ProductKey = vr.ProductKey
    WHERE p.IsActive = 1;
    
    RETURN @@ROWCOUNT;
END;
GO

-- Stored procedure for automated alerting
CREATE PROCEDURE sp_GenerateOperationalAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertLog TABLE (
        AlertType NVARCHAR(100),
        AlertMessage NVARCHAR(500),
        Severity NVARCHAR(20),
        EntityID NVARCHAR(100)
    );
    
    -- Alert 1: Fleet idling time > 15% of trip duration
    INSERT INTO @AlertLog
    SELECT 
        'Fleet Excessive Idle' AS AlertType,
        'Vehicle ' + df.VehicleID + ' has idle time ' + CAST(IdleTimeMinutes AS NVARCHAR) + ' minutes (' + 
        CAST(CAST(IdleTimeMinutes * 100.0 / NULLIF(DurationMinutes, 0) AS INT) AS NVARCHAR) + '%)' AS AlertMessage,
        'High' AS Severity,
        TripID AS EntityID
    FROM FactFleetTrips fft
    INNER JOIN DimFleet df ON fft.FleetKey = df.FleetKey
    WHERE IsCompleted = 1
        AND IdleTimeMinutes * 100.0 / NULLIF(DurationMinutes, 0) > 15
        AND StartTimeKey IN (SELECT TimeKey FROM DimTime WHERE DateTimeValue >= DATEADD(DAY, -1, GETDATE()));
    
    -- Alert 2: Warehouse dwell time > 72 hours
    INSERT INTO @AlertLog
    SELECT 
        'Excessive Dwell Time' AS AlertType,
        'SKU ' + dp.SKU + ' at ' + dg.LocationName + ' has dwell time ' + CAST(DwellTimeMinutes / 60 AS NVARCHAR) + ' hours' AS AlertMessage,
        'Medium' AS Severity,
        OrderID AS EntityID
    FROM FactWarehouseOperations fwo
    INNER JOIN DimProduct dp ON fwo.ProductKey = dp.ProductKey
    INNER JOIN DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
    WHERE DwellTimeMinutes > 4320 -- 72 hours
        AND TimeKey IN (SELECT TimeKey FROM DimTime WHERE DateTimeValue >= DATEADD(DAY, -1, GETDATE()));
    
    -- Alert 3: Cross-dock temperature compliance failure
    INSERT INTO @AlertLog
    SELECT 
        'Temperature Compliance Failure' AS AlertType,
        'Product ' + dp.SKU + ' at ' + dg.LocationName + ' failed temperature check' AS AlertMessage,
        'Critical' AS Severity,
        TransferID AS EntityID
    FROM FactCrossDock fcd
    INNER JOIN DimProduct dp ON fcd.ProductKey = dp.ProductKey
    INNER JOIN DimGeography dg ON fcd.GeographyKey = dg.GeographyKey
    WHERE TemperatureCompliance = 0
        AND dp.IsPerishable = 1
        AND TimeKey IN (SELECT TimeKey FROM DimTime WHERE DateTimeValue >= DATEADD(DAY, -1, GETDATE()));
    
    -- Output all alerts
    SELECT * FROM @AlertLog
    ORDER BY 
        CASE Severity 
            WHEN 'Critical' THEN 1
            WHEN 'High' THEN 2
            WHEN 'Medium' THEN 3
            ELSE 4
        END;
END;
GO
```

### Step 3: Configure Data Sources

Create a configuration file (not included in repo):

```json
{
  "connections": {
    "sqlServer": {
      "server": "${SQL_SERVER_HOST}",
      "database": "LogiFleetPulse",
      "authentication": "ActiveDirectoryIntegrated",
      "encrypt": true
    },
    "wmsApi": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "apiKey": "${WMS_API_KEY}",
      "refreshIntervalMinutes": 15
    },
    "telematicsApi": {
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "apiKey": "${TELEMATICS_API_KEY}",
      "refreshIntervalMinutes": 5
    },
    "weatherApi": {
      "endpoint": "${WEATHER_API_ENDPOINT}",
      "apiKey": "${WEATHER_API_KEY}",
      "refreshIntervalMinutes": 60
    }
  },
  "alerting": {
    "emailRecipients": ["${ALERT_EMAIL_RECIPIENTS}"],
    "smsRecipients": ["${ALERT_SMS_RECIPIENTS}"],
    "teamsWebhook": "${TEAMS_WEBHOOK_URL}"
  }
}
```

### Step 4: Load Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. When prompted, enter your SQL Server connection details
3. The template includes pre-configured:
   - Data model with relationships
   - DAX measures for KPIs
   - Dashboard pages with visualizations
   - Row-level security roles

## Key DAX Measures

The Power BI template includes these critical measures:

```dax
-- Total Warehouse Operations
Total Operations = COUNTROWS(FactWarehouseOperations)

-- Average Dwell Time (hours)
Avg Dwell Time Hours = 
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) / 60

-- Fleet Utilization %
Fleet Utilization % = 
    DIVIDE(
        CALCULATE(
            COUNTROWS(FactFleetTrips),
            FactFleetTrips[IsCompleted] = TRUE
        ),
        COUNTROWS(DimFleet) * 24, -- Assume 24 possible hours per vehicle
        0
    ) * 100

-- Fleet Idle Time %
Fleet Idle % = 
    DIVIDE(
        SUM(FactFleetTrips[IdleTimeMinutes]),
        SUM(FactFleetTrips[DurationMinutes]),
        0
    ) * 100

-- Cross-Dock Efficiency
CrossDock Bypass Rate = 
    DIVIDE(
        CALCULATE(
            COUNTROWS(FactCrossDock),
            FactCrossDock[BypassStorage] = TRUE
        ),
        COUNTROWS(FactCrossDock),
        0
    ) * 100

-- Warehouse Gravity Score Average
Avg Gravity Score = 
    CALCULATE(
        AVERAGE(DimProductGravity[GravityScore]),
        FILTER(
            DimProductGravity,
            DimProductGravity[EffectiveDate] = MAX(DimProductGravity[EffectiveDate])
        )
    )

-- Supplier Reliability Index
Supplier Reliability = 
    AVERAGE(DimSupplier[OnTimeDeliveryPercent]) * 0.5 + 
    (100 - AVERAGE(DimSupplier[DefectRatePercent])) * 0.3 +
    AVERAGE(DimSupplier[ComplianceScore]) * 0.2

-- Predictive Bottleneck Index (simplified version)
Bottleneck Index = 
    VAR DwellScore = [Avg Dwell Time Hours] / 72 * 100
    VAR IdleScore = [Fleet Idle %]
    VAR TempFailScore = 
        DIVIDE(
            CALCULATE(
                COUNTROWS(FactCrossDock),
                FactCrossDock[TemperatureCompliance] = FALSE
            ),
            COUNTROWS(FactCrossDock),
            0
        ) * 100
    RETURN
        (DwellScore * 0.4 + IdleScore * 0.4 + TempFailScore * 0.2)

-- On-Time Delivery Rate
On-Time Delivery % = 
    VAR PlannedMinutes = SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[TrafficDelay])
    VAR ActualMinutes = SUM(FactFleetTrips[DurationMinutes])
    RETURN
        DIVIDE(
            CALCULATE(
                COUNTROWS(FactFleetTrips),
                ActualMinutes <= PlannedMinutes * 1.1 -- 10% tolerance
            ),
            COUNTROWS(FactFleetTrips),
            0
        ) * 100

-- Cost per Trip
Cost per Trip = 
    VAR FuelCostPerLiter = 1.5 -- Configurable parameter
    VAR DriverHourlyRate = 25 -- Configurable parameter
    RETURN
        SUM(FactFleetTrips[FuelConsumedLiters]) * FuelCostPerLiter +
        SUM(FactFleetTrips[DurationMinutes]) / 60 * DriverHourlyRate

-- Time Intelligence: YTD Operations
YTD Operations = 
    CALCULATE(
        [Total Operations],
        DATESYTD(DimTime[DateTimeValue])
    )

-- Time Intelligence: Prior Year Same Period
PYSP Operations = 
    CALCULATE(
        [Total Operations],
        SAMEPERIODLASTYEAR(DimTime[DateTimeValue])
    )

-- Growth Rate
Operations Growth % = 
    DIVIDE(
        [YTD Operations] - [PYSP Operations],
        [PYSP Operations],
        0
    ) * 100
```

## Common Query Patterns

### Query 1: Top 10 Products by Pick Frequency

```sql
SELECT TOP 10
    p.SKU,
    p.ProductName,
    p.ProductCategory,
    COUNT(*) AS PickCount,
    AVG(fwo.ProcessingTimeMinutes) AS AvgProcessingTime,
    pg.GravityScore,
    pg.RecommendedZoneType
FROM FactWarehouseOperations fwo
INNER JOIN DimProduct p ON fwo.ProductKey = p.ProductKey
INNER JOIN DimProductGravity pg ON p.ProductKey = pg.ProductKey
    AND pg.EffectiveDate = (SELECT MAX(EffectiveDate) FROM DimProductGravity)
WHERE fwo.OperationType = 'Picking'
    AND fwo.TimeKey IN (
        SELECT TimeKey FROM DimTime 
        WHERE DateTimeValue >= DATEADD(DAY, -30, GETDATE())
    )
GROUP BY p.SKU, p.ProductName, p.ProductCategory, pg.GravityScore, pg.RecommendedZoneType
ORDER BY PickCount DESC;
```

### Query 2: Fleet Performance with Idle Time Analysis

```sql
SELECT 
    df.VehicleID,
    df.VehicleType,
    COUNT(DISTINCT fft.TripID) AS TotalTrips,
    SUM(fft.DistanceKm) AS TotalDistanceKm,
    SUM(fft.DurationMinutes) / 60.0 AS TotalHours,
    SUM(fft.IdleTimeMinutes) / 60.0 AS IdleHours,
    SUM(fft.IdleTimeMinutes) * 100.0 / NULLIF(SUM(fft.DurationMinutes), 0) AS IdleTimePercent,
    SUM(fft.FuelConsumedLiters) AS TotalFuelLiters,
    SUM(fft.FuelConsumedLiters) / NULLIF(SUM(fft.DistanceKm), 0) * 100 AS FuelEfficiencyL100km,
    AVG(fft.LoadWeightKg) AS AvgLoadKg,
    SUM(CASE WHEN fft.MaintenanceIssue = 1 THEN 1 ELSE 0 END) AS MaintenanceIncidents
FROM DimFleet df
LEFT JOIN FactFleetTrips fft ON df.FleetKey = fft.FleetKey
WHERE fft.IsCompleted = 1
    AND fft.StartTimeKey IN (
        SELECT TimeKey FROM DimTime 
        WHERE DateTimeValue >= DATEADD(DAY, -30, GETDATE())
    )
GROUP BY df.VehicleID, df.VehicleType
HAVING COUNT(DISTINCT fft.TripID) > 0
ORDER BY IdleTimePercent DESC;
```

### Query 3: Cross-Fact Analysis - Dwell Time Impact on Fleet Performance

```sql
-- Products with high dwell time and their subsequent delivery performance
WITH HighDwellProducts AS (
    SELECT 
        fwo.ProductKey,
        p.SKU,
        AVG(fwo.DwellTimeMinutes) AS AvgDwellMinutes
    FROM FactWarehouseOperations fwo
    INNER JOIN DimProduct p ON fwo.ProductKey = p.ProductKey
    WHERE fwo.TimeKey IN (
        SELECT TimeKey FROM DimTime 
        WHERE DateTimeValue >= DATEADD(DAY, -30, GETDATE())
    )
    GROUP BY fwo.ProductKey, p.SKU
    HAVING AVG(fwo.DwellTimeMinutes) > 1440 -- More than 24 hours
)
SELECT 
    hdp.SKU,
    hdp.AvgDwellMinutes / 60.0 AS AvgDwellHours,
    COUNT(DISTINCT fft.TripID) AS RelatedTrips,
    AVG(fft.IdleTimeMinutes) AS AvgTripIdleMinutes,
    AVG(fft.TrafficDelay) AS AvgTrafficDelayMinutes,
    AVG(fft.DurationMinutes - fft.TrafficDelay) AS AvgActualTripMinutes,
    SUM(CASE WHEN fcd.TemperatureCompliance = 0 THEN 1 ELSE 0 END) AS TempFailures
FROM HighDwellProducts hdp
INNER JOIN FactCrossDock fcd ON hdp.ProductKey = fcd.ProductKey
INNER JOIN FactFleetTrips fft ON fcd.OutboundTripKey = fft.TripKey
WHERE fft.IsCompleted = 1
GROUP BY hdp.SKU, hdp.AvgDwellMinutes
ORDER BY hdp.AvgDwellMinutes DESC;
```

### Query 4: Warehouse Gravity Zone Optimization Recommendations

```sql
-- Identify products in wrong zones based on gravity score
SELECT 
    p.SKU,
    p.ProductName,
    fwo.ZoneCode AS CurrentZone,
    pg.RecommendedZoneType,
    pg.GravityScore,
    COUNT(*) AS PickCountLast30Days,
    AVG(fwo.DistanceWalkedMeters) AS AvgDistanceWalked,
    -- Calculate potential savings if moved to recommended zone
    AVG(fwo.DistanceWalkedMeters) - pg.OptimalDistanceFromDock AS PotentialSavingsMeters
FROM FactWarehouseOperations fwo
INNER JOIN DimProduct p ON fwo.ProductKey = p.ProductKey
INNER JOIN DimProductGravity pg ON p.ProductKey = pg.ProductKey
    AND pg.EffectiveDate = (SELECT MAX(EffectiveDate) FROM DimProductGravity)
WHERE fwo.OperationType = 'Picking'
    AND fwo.TimeKey IN (
        SELECT TimeKey FROM DimTime
