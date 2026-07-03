---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform for warehouse operations, fleet management, and supply chain KPI analysis
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "create SQL star schema for warehouse operations"
  - "build fleet telemetry data model"
  - "implement cross-fact supply chain KPIs"
  - "deploy logistics intelligence warehouse"
  - "optimize warehouse gravity zones with SQL"
  - "integrate fleet and inventory analytics"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is a comprehensive logistics intelligence platform that unifies warehouse operations, fleet telemetry, and supply chain analytics through a multi-fact star schema in MS SQL Server with Power BI visualizations. It provides cross-modal KPI harmonization, predictive bottleneck detection, and real-time operational insights across the entire logistics chain.

## What It Does

- **Unified Data Model**: Multi-fact star schema linking warehouse operations, fleet trips, and cross-dock activities
- **Real-Time Dashboards**: Power BI dashboards with 15-minute refresh intervals
- **Predictive Analytics**: ML-powered bottleneck detection and fleet triage scoring
- **Warehouse Gravity Zones**: Spatial optimization based on pick frequency, value, and fragility
- **Cross-Fact KPIs**: Query relationships between inventory turnover and fleet utilization
- **Role-Based Security**: Row-level security for different organizational roles

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, telematics API, ERP system

### Step 1: Deploy SQL Schema

```sql
-- Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create time dimension (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTimeStamp DATETIME2 NOT NULL,
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayOfMonth TINYINT NOT NULL,
    MonthOfYear TINYINT NOT NULL,
    FiscalQuarter TINYINT NOT NULL,
    FiscalYear SMALLINT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
)
GO

-- Create columnstore index for time dimension
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_DimTime_CS 
ON DimTime (DateTimeStamp, HourOfDay, DayOfWeek)
GO

-- Create geography dimension with hierarchy
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationCode VARCHAR(20) NOT NULL UNIQUE,
    LocationName NVARCHAR(100) NOT NULL,
    LocationType VARCHAR(20) NOT NULL, -- Warehouse, Route Node, Distribution Center
    Address NVARCHAR(200),
    City NVARCHAR(50),
    StateProvince NVARCHAR(50),
    Country NVARCHAR(50),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone VARCHAR(50),
    IsActive BIT NOT NULL DEFAULT 1
)
GO

-- Create product gravity dimension
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(50),
    SubCategory NVARCHAR(50),
    UnitWeight DECIMAL(10,3),
    VelocityScore DECIMAL(5,2), -- 0-100 based on pick frequency
    ValueScore DECIMAL(5,2), -- 0-100 based on unit price
    FragilityScore DECIMAL(5,2), -- 0-100 based on damage rate
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    RecommendedZone VARCHAR(20), -- High-Gravity, Medium-Gravity, Low-Gravity
    IsActive BIT NOT NULL DEFAULT 1
)
GO

-- Create supplier reliability dimension
CREATE TABLE DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierCode VARCHAR(20) NOT NULL UNIQUE,
    SupplierName NVARCHAR(100) NOT NULL,
    LeadTimeDaysAvg DECIMAL(5,1),
    LeadTimeDaysStdDev DECIMAL(5,2),
    DefectPercentage DECIMAL(5,3),
    OnTimeDeliveryRate DECIMAL(5,2), -- 0-100
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityScore AS (OnTimeDeliveryRate * 0.6 + (100 - DefectPercentage * 10) * 0.4) PERSISTED,
    IsActive BIT NOT NULL DEFAULT 1
)
GO

-- Create warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT NULL,
    OperationType VARCHAR(20) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OperationStartTime DATETIME2 NOT NULL,
    OperationEndTime DATETIME2 NOT NULL,
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime) PERSISTED,
    QuantityProcessed INT NOT NULL,
    WorkerID VARCHAR(20),
    EquipmentID VARCHAR(20),
    StorageZone VARCHAR(20),
    DwellTimeHours DECIMAL(10,2) NULL,
    ErrorOccurred BIT NOT NULL DEFAULT 0,
    ErrorType VARCHAR(50) NULL,
    
    CONSTRAINT FK_WarehouseOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WarehouseOps_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WarehouseOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_WarehouseOps_Supplier FOREIGN KEY (SupplierKey) REFERENCES DimSupplierReliability(SupplierKey)
)
GO

-- Create columnstore index for warehouse operations
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouseOps_CS
ON FactWarehouseOperations (TimeKey, GeographyKey, ProductKey, OperationType, DurationMinutes)
GO

-- Create fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    VehicleID VARCHAR(20) NOT NULL,
    DriverID VARCHAR(20) NOT NULL,
    TripStartTime DATETIME2 NOT NULL,
    TripEndTime DATETIME2 NOT NULL,
    TripDurationMinutes AS DATEDIFF(MINUTE, TripStartTime, TripEndTime) PERSISTED,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,3),
    IdleTimeMinutes DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    AverageSpeedKPH DECIMAL(6,2),
    WeightKG DECIMAL(10,2),
    DelayedByWeather BIT DEFAULT 0,
    DelayedByTraffic BIT DEFAULT 0,
    MaintenanceAlertTriggered BIT DEFAULT 0,
    MaintenanceAlertType VARCHAR(50) NULL,
    
    CONSTRAINT FK_FleetTrips_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrips_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FleetTrips_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
)
GO

-- Create columnstore index for fleet trips
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleetTrips_CS
ON FactFleetTrips (TimeKey, OriginGeographyKey, DestinationGeographyKey, VehicleID, TripDurationMinutes)
GO

-- Create cross-dock operations fact table
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT NULL,
    OutboundTripKey BIGINT NULL,
    ReceiptTime DATETIME2 NOT NULL,
    ShipmentTime DATETIME2 NOT NULL,
    CrossDockDurationMinutes AS DATEDIFF(MINUTE, ReceiptTime, ShipmentTime) PERSISTED,
    QuantityTransferred INT NOT NULL,
    TemporaryStorageUsed BIT DEFAULT 0,
    
    CONSTRAINT FK_CrossDock_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CrossDock_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_CrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_CrossDock_InboundTrip FOREIGN KEY (InboundTripKey) REFERENCES FactFleetTrips(TripKey),
    CONSTRAINT FK_CrossDock_OutboundTrip FOREIGN KEY (OutboundTripKey) REFERENCES FactFleetTrips(TripKey)
)
GO
```

### Step 2: Create Views for Cross-Fact Analysis

```sql
-- View: Integrated supply chain performance
CREATE VIEW vw_IntegratedSupplyChainPerformance AS
SELECT 
    t.DateTimeStamp,
    t.FiscalQuarter,
    t.FiscalYear,
    g.LocationName AS Warehouse,
    g.City,
    g.Country,
    p.SKU,
    p.ProductName,
    p.Category,
    p.GravityScore,
    p.RecommendedZone,
    
    -- Warehouse metrics
    AVG(wo.DurationMinutes) AS AvgOperationDurationMinutes,
    SUM(wo.QuantityProcessed) AS TotalQuantityProcessed,
    AVG(wo.DwellTimeHours) AS AvgDwellTimeHours,
    SUM(CASE WHEN wo.ErrorOccurred = 1 THEN 1 ELSE 0 END) AS ErrorCount,
    
    -- Fleet metrics (for shipments from this warehouse)
    AVG(ft.TripDurationMinutes) AS AvgTripDurationMinutes,
    AVG(ft.FuelConsumedLiters) AS AvgFuelConsumedLiters,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTimeMinutes,
    SUM(CASE WHEN ft.DelayedByWeather = 1 THEN 1 ELSE 0 END) AS WeatherDelayCount,
    SUM(CASE WHEN ft.MaintenanceAlertTriggered = 1 THEN 1 ELSE 0 END) AS MaintenanceAlertCount,
    
    -- Cross-dock metrics
    AVG(cd.CrossDockDurationMinutes) AS AvgCrossDockDurationMinutes,
    SUM(cd.QuantityTransferred) AS TotalCrossDockQuantity

FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips ft ON g.GeographyKey = ft.OriginGeographyKey 
    AND CAST(wo.OperationEndTime AS DATE) = CAST(ft.TripStartTime AS DATE)
LEFT JOIN FactCrossDock cd ON wo.GeographyKey = cd.GeographyKey 
    AND wo.ProductKey = cd.ProductKey
    AND CAST(wo.OperationEndTime AS DATE) = CAST(cd.ReceiptTime AS DATE)

GROUP BY 
    t.DateTimeStamp, t.FiscalQuarter, t.FiscalYear,
    g.LocationName, g.City, g.Country,
    p.SKU, p.ProductName, p.Category, p.GravityScore, p.RecommendedZone
GO

-- View: Fleet efficiency with warehouse correlation
CREATE VIEW vw_FleetEfficiencyWarehouseCorrelation AS
SELECT
    ft.VehicleID,
    ft.DriverID,
    t.DateTimeStamp,
    orig.LocationName AS OriginWarehouse,
    dest.LocationName AS DestinationWarehouse,
    
    ft.TripDurationMinutes,
    ft.DistanceKM,
    ft.FuelConsumedLiters,
    ft.IdleTimeMinutes,
    (ft.IdleTimeMinutes * 100.0 / NULLIF(ft.TripDurationMinutes, 0)) AS IdleTimePercentage,
    
    -- Fuel efficiency
    (ft.FuelConsumedLiters / NULLIF(ft.DistanceKM, 0)) AS FuelEfficiencyLitersPer100KM,
    
    -- Origin warehouse dwell time (for same day shipments)
    AVG(wo.DwellTimeHours) AS OriginWarehouseDwellTime,
    
    -- Weight and load factor
    ft.WeightKG,
    
    -- Delay indicators
    ft.DelayedByWeather,
    ft.DelayedByTraffic,
    ft.MaintenanceAlertTriggered

FROM FactFleetTrips ft
INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
INNER JOIN DimGeography orig ON ft.OriginGeographyKey = orig.GeographyKey
INNER JOIN DimGeography dest ON ft.DestinationGeographyKey = dest.GeographyKey
LEFT JOIN FactWarehouseOperations wo ON ft.OriginGeographyKey = wo.GeographyKey
    AND CAST(ft.TripStartTime AS DATE) = CAST(wo.OperationEndTime AS DATE)
    AND wo.OperationType = 'Shipping'

GROUP BY
    ft.VehicleID, ft.DriverID, t.DateTimeStamp,
    orig.LocationName, dest.LocationName,
    ft.TripDurationMinutes, ft.DistanceKM, ft.FuelConsumedLiters,
    ft.IdleTimeMinutes, ft.WeightKG,
    ft.DelayedByWeather, ft.DelayedByTraffic, ft.MaintenanceAlertTriggered
GO
```

### Step 3: Create Stored Procedures for Data Loading

```sql
-- Stored procedure: Incremental warehouse operations load
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last 24 hours if no timestamp provided
    IF @LastLoadTimestamp IS NULL
        SET @LastLoadTimestamp = DATEADD(HOUR, -24, GETDATE());
    
    -- Load from staging table (assumes external data source already loaded here)
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, OperationStartTime, OperationEndTime,
        QuantityProcessed, WorkerID, EquipmentID, StorageZone,
        DwellTimeHours, ErrorOccurred, ErrorType
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        stg.OperationType,
        stg.OperationStartTime,
        stg.OperationEndTime,
        stg.QuantityProcessed,
        stg.WorkerID,
        stg.EquipmentID,
        stg.StorageZone,
        stg.DwellTimeHours,
        stg.ErrorOccurred,
        stg.ErrorType
    FROM StagingWarehouseOperations stg
    INNER JOIN DimTime t ON CAST(stg.OperationStartTime AS DATETIME2) >= t.DateTimeStamp
        AND CAST(stg.OperationStartTime AS DATETIME2) < DATEADD(MINUTE, 15, t.DateTimeStamp)
    INNER JOIN DimGeography g ON stg.LocationCode = g.LocationCode
    INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
    LEFT JOIN DimSupplierReliability s ON stg.SupplierCode = s.SupplierCode
    WHERE stg.OperationStartTime > @LastLoadTimestamp
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations wo
            WHERE wo.OperationStartTime = stg.OperationStartTime
                AND wo.GeographyKey = g.GeographyKey
                AND wo.ProductKey = p.ProductKey
        );
    
    PRINT 'Loaded ' + CAST(@@ROWCOUNT AS VARCHAR(10)) + ' warehouse operations records';
END
GO

-- Stored procedure: Update product gravity scores
CREATE PROCEDURE usp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate velocity score based on last 90 days pick frequency
    WITH PickFrequency AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickCount,
            NTILE(100) OVER (ORDER BY COUNT(*) DESC) AS VelocityPercentile
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
            AND OperationStartTime >= DATEADD(DAY, -90, GETDATE())
        GROUP BY ProductKey
    )
    UPDATE p
    SET 
        VelocityScore = ISNULL(pf.VelocityPercentile, 0),
        RecommendedZone = CASE 
            WHEN p.GravityScore >= 70 THEN 'High-Gravity'
            WHEN p.GravityScore >= 40 THEN 'Medium-Gravity'
            ELSE 'Low-Gravity'
        END
    FROM DimProductGravity p
    LEFT JOIN PickFrequency pf ON p.ProductKey = pf.ProductKey;
    
    PRINT 'Updated gravity scores for ' + CAST(@@ROWCOUNT AS VARCHAR(10)) + ' products';
END
GO

-- Stored procedure: Fleet maintenance alert scoring
CREATE PROCEDURE usp_FleetMaintenanceTriage
    @OutputTopN INT = 20
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate priority score for each vehicle based on recent telemetry
    WITH FleetMetrics AS (
        SELECT 
            VehicleID,
            COUNT(*) AS TripCount,
            AVG(IdleTimeMinutes * 100.0 / NULLIF(TripDurationMinutes, 0)) AS AvgIdlePercentage,
            SUM(CASE WHEN MaintenanceAlertTriggered = 1 THEN 1 ELSE 0 END) AS MaintenanceAlertCount,
            AVG(FuelConsumedLiters / NULLIF(DistanceKM, 0)) AS FuelEfficiency
        FROM FactFleetTrips
        WHERE TripStartTime >= DATEADD(DAY, -30, GETDATE())
        GROUP BY VehicleID
    ),
    HighValueCargo AS (
        SELECT 
            ft.VehicleID,
            AVG(p.GravityScore) AS AvgCargoGravity
        FROM FactFleetTrips ft
        INNER JOIN FactWarehouseOperations wo ON ft.OriginGeographyKey = wo.GeographyKey
            AND CAST(ft.TripStartTime AS DATE) = CAST(wo.OperationEndTime AS DATE)
        INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
        WHERE ft.TripStartTime >= DATEADD(DAY, -30, GETDATE())
        GROUP BY ft.VehicleID
    )
    SELECT TOP (@OutputTopN)
        fm.VehicleID,
        fm.TripCount,
        fm.AvgIdlePercentage,
        fm.MaintenanceAlertCount,
        fm.FuelEfficiency,
        hvc.AvgCargoGravity,
        -- Priority score: more alerts + worse fuel efficiency + higher cargo value = higher priority
        (fm.MaintenanceAlertCount * 30.0 + 
         (100 - fm.FuelEfficiency * 20) * 0.3 + 
         ISNULL(hvc.AvgCargoGravity, 0) * 0.5) AS PriorityScore
    FROM FleetMetrics fm
    LEFT JOIN HighValueCargo hvc ON fm.VehicleID = hvc.VehicleID
    WHERE fm.MaintenanceAlertCount > 0
    ORDER BY PriorityScore DESC;
END
GO
```

### Step 4: Configure Automated Alerting

```sql
-- Create alert configuration table
CREATE TABLE AlertConfiguration (
    AlertID INT IDENTITY(1,1) PRIMARY KEY,
    AlertName NVARCHAR(100) NOT NULL,
    AlertType VARCHAR(50) NOT NULL, -- Warehouse, Fleet, CrossDock
    MetricName VARCHAR(100) NOT NULL,
    ThresholdOperator VARCHAR(10) NOT NULL, -- >, <, >=, <=, =
    ThresholdValue DECIMAL(18,4) NOT NULL,
    EmailRecipients NVARCHAR(500),
    IsActive BIT NOT NULL DEFAULT 1
)
GO

-- Insert sample alerts
INSERT INTO AlertConfiguration (AlertName, AlertType, MetricName, ThresholdOperator, ThresholdValue, EmailRecipients, IsActive)
VALUES 
    ('High Dwell Time Alert', 'Warehouse', 'AvgDwellTimeHours', '>', 72, 'ops@example.com', 1),
    ('Fleet Idle Time Alert', 'Fleet', 'IdleTimePercentage', '>', 15, 'fleet@example.com', 1),
    ('Cross-Dock Bottleneck', 'CrossDock', 'AvgCrossDockDurationMinutes', '>', 120, 'logistics@example.com', 1);

-- Stored procedure: Check alerts and send notifications
CREATE PROCEDURE usp_CheckAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertID INT, @AlertName NVARCHAR(100), @AlertType VARCHAR(50);
    DECLARE @MetricName VARCHAR(100), @ThresholdOperator VARCHAR(10), @ThresholdValue DECIMAL(18,4);
    DECLARE @EmailRecipients NVARCHAR(500);
    DECLARE @CurrentValue DECIMAL(18,4);
    DECLARE @AlertTriggered BIT;
    
    DECLARE alert_cursor CURSOR FOR
    SELECT AlertID, AlertName, AlertType, MetricName, ThresholdOperator, ThresholdValue, EmailRecipients
    FROM AlertConfiguration
    WHERE IsActive = 1;
    
    OPEN alert_cursor;
    FETCH NEXT FROM alert_cursor INTO @AlertID, @AlertName, @AlertType, @MetricName, @ThresholdOperator, @ThresholdValue, @EmailRecipients;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @AlertTriggered = 0;
        
        -- Example: Check warehouse dwell time
        IF @AlertType = 'Warehouse' AND @MetricName = 'AvgDwellTimeHours'
        BEGIN
            SELECT @CurrentValue = AVG(DwellTimeHours)
            FROM FactWarehouseOperations
            WHERE OperationStartTime >= DATEADD(HOUR, -24, GETDATE());
            
            IF (@ThresholdOperator = '>' AND @CurrentValue > @ThresholdValue)
                SET @AlertTriggered = 1;
        END
        
        -- Example: Check fleet idle time
        IF @AlertType = 'Fleet' AND @MetricName = 'IdleTimePercentage'
        BEGIN
            SELECT @CurrentValue = AVG(IdleTimeMinutes * 100.0 / NULLIF(TripDurationMinutes, 0))
            FROM FactFleetTrips
            WHERE TripStartTime >= DATEADD(HOUR, -24, GETDATE());
            
            IF (@ThresholdOperator = '>' AND @CurrentValue > @ThresholdValue)
                SET @AlertTriggered = 1;
        END
        
        -- Send email if alert triggered (use Database Mail or external service)
        IF @AlertTriggered = 1
        BEGIN
            -- Log alert (replace with actual email sending logic)
            PRINT 'ALERT: ' + @AlertName + ' - Current Value: ' + CAST(@CurrentValue AS VARCHAR(20));
            -- EXEC msdb.dbo.sp_send_dbmail @recipients = @EmailRecipients, @subject = @AlertName, @body = ...
        END
        
        FETCH NEXT FROM alert_cursor INTO @AlertID, @AlertName, @AlertType, @MetricName, @ThresholdOperator, @ThresholdValue, @EmailRecipients;
    END
    
    CLOSE alert_cursor;
    DEALLOCATE alert_cursor;
END
GO
```

## Power BI Integration

### Connect Power BI to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `your-server-name.database.windows.net` or `localhost`
4. Database: `LogiFleetPulse`
5. Data Connectivity mode: **Import** (for scheduled refresh) or **DirectQuery** (for real-time)

### DAX Measures for Cross-Fact KPIs

```dax
// Measure: Warehouse Efficiency Index
Warehouse_Efficiency_Index = 
VAR AvgDuration = AVERAGE(FactWarehouseOperations[DurationMinutes])
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR ErrorRate = DIVIDE(
    COUNTROWS(FILTER(FactWarehouseOperations, FactWarehouseOperations[ErrorOccurred] = TRUE())),
    COUNTROWS(FactWarehouseOperations),
    0
)
RETURN
    (100 - (AvgDuration / 10)) * 0.4 + 
    (100 - (AvgDwell / 10)) * 0.4 + 
    (100 - ErrorRate * 100) * 0.2

// Measure: Fleet Cost per KM
Fleet_Cost_Per_KM = 
VAR TotalFuel = SUM(FactFleetTrips[FuelConsumedLiters])
VAR TotalDistance = SUM(FactFleetTrips[DistanceKM])
VAR FuelCostPerLiter = 1.50 -- Update with actual cost or parameter
RETURN
    DIVIDE(TotalFuel * FuelCostPerLiter, TotalDistance, 0)

// Measure: Cross-Dock Velocity
CrossDock_Velocity = 
VAR AvgCrossDockTime = AVERAGE(FactCrossDock[CrossDockDurationMinutes])
RETURN
    DIVIDE(60, AvgCrossDockTime, 0) -- Units per hour

// Measure: Integrated Supply Chain Score
Integrated_Supply_Chain_Score = 
VAR WarehouseScore = [Warehouse_Efficiency_Index]
VAR FleetScore = 100 - ([Fleet_Cost_Per_KM] * 10) -- Normalize to 0-100
VAR CrossDockScore = DIVIDE([CrossDock_Velocity] * 5, 1, 0) -- Normalize
RETURN
    (WarehouseScore * 0.4 + FleetScore * 0.35 + CrossDockScore * 0.25)

// Measure: Gravity Zone Compliance
Gravity_Zone_Compliance = 
VAR ActualZone = SELECTEDVALUE(FactWarehouseOperations[StorageZone])
VAR RecommendedZone = SELECTEDVALUE(DimProductGravity[RecommendedZone])
VAR MatchCount = COUNTROWS(
    FILTER(
        FactWarehouseOperations,
        FactWarehouseOperations[StorageZone] = RELATED(DimProductGravity[RecommendedZone])
    )
)
VAR TotalCount = COUNTROWS(FactWarehouseOperations)
RETURN
    DIVIDE(MatchCount, TotalCount, 0) * 100

// Measure: Predictive Bottleneck Index (simplified heuristic)
Predictive_Bottleneck_Index = 
VAR HighDwellCount = COUNTROWS(
    FILTER(FactWarehouseOperations, FactWarehouseOperations[DwellTimeHours] > 48)
)
VAR HighIdleCount = COUNTROWS(
    FILTER(FactFleetTrips, (FactFleetTrips[IdleTimeMinutes] / FactFleetTrips[TripDurationMinutes]) > 0.15)
)
VAR TotalOperations = COUNTROWS(FactWarehouseOperations) + COUNTROWS(FactFleetTrips)
RETURN
    DIVIDE((HighDwellCount + HighIdleCount), TotalOperations, 0) * 100
```

### Power BI Report Structure

Create these report pages:

1. **Executive Dashboard**: Integrated Supply Chain Score, key KPIs, trend charts
2. **Warehouse Operations**: Dwell time heatmap, gravity zone compliance, throughput by hour
3. **Fleet Performance**: Cost per KM, idle time analysis, maintenance priority list
4. **Cross-Dock Analysis**: Velocity trends, bottleneck identification
5. **Predictive Insights**: Bottleneck index forecasts, maintenance triage scores

## Common Patterns

### Pattern 1: Query Dwell Time by SKU and Route Delay Correlation

```sql
-- Find SKUs with high dwell time that also caused fleet delays
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    AVG(wo.DwellTimeHours) AS AvgDwellTimeHours,
    COUNT(DISTINCT ft.TripKey) AS AffectedTrips,
    AVG(ft.TripDurationMinutes) AS AvgTripDurationMinutes,
    SUM(CASE WHEN ft.DelayedBy
