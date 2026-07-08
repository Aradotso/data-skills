---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing template for logistics, fleet, and warehouse analytics with multi-fact star schema
triggers:
  - set up logistics analytics dashboard
  - deploy supply chain data warehouse
  - configure fleet and warehouse analytics
  - implement LogiFleet Pulse power bi template
  - create multi-fact star schema for logistics
  - build warehouse gravity zone analytics
  - set up cross-modal supply chain reporting
  - deploy logicore analytics sql schema
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and analytics template for logistics operations. It provides:

- **Multi-fact star schema** for warehouse operations, fleet trips, and cross-dock activities
- **Power BI dashboard templates** for real-time logistics visualization
- **Dimensional modeling** with time-phased dimensions (15-minute granularity)
- **Cross-fact KPI harmonization** linking inventory, fleet, and operational metrics
- **Predictive analytics** for bottleneck detection and fleet maintenance prioritization
- **Warehouse Gravity Zones** for spatial optimization based on SKU velocity

The platform integrates data from WMS systems, telematics/GPS, supplier portals, weather APIs, and customer orders into a unified semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Database access with CREATE TABLE, VIEW, PROCEDURE permissions
- Optional: Azure Synapse Analytics for big data enrichment

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Create a new database for LogiFleet Pulse
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Deploy dimension tables first
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    [Year] INT NOT NULL,
    [Quarter] INT NOT NULL,
    [Month] INT NOT NULL,
    [Day] INT NOT NULL,
    [Hour] INT NOT NULL,
    [Minute] INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayOfWeekName NVARCHAR(20),
    IsWeekend BIT NOT NULL,
    FiscalYear INT NOT NULL,
    FiscalQuarter INT NOT NULL,
    TimeSlot15Min INT NOT NULL -- 0-95 (96 slots per day)
);

-- Create clustered columnstore index for fact tables
CREATE CLUSTERED COLUMNSTORE INDEX CCI_DimTime ON DimTime;

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    Continent NVARCHAR(50),
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    WarehouseCode NVARCHAR(20),
    WarehouseName NVARCHAR(200),
    RouteNode NVARCHAR(50),
    Latitude DECIMAL(10,8),
    Longitude DECIMAL(11,8),
    TimeZone NVARCHAR(50),
    IsActive BIT DEFAULT 1
);

-- DimProductGravity: Products with velocity-based gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity + value + fragility
    VelocityClass NVARCHAR(20), -- Fast/Medium/Slow
    ValueTier NVARCHAR(20), -- High/Medium/Low
    FragilityIndex INT, -- 1-10
    OptimalZoneType NVARCHAR(50), -- High-gravity, Mid-gravity, Low-gravity
    UpdatedDate DATETIME DEFAULT GETDATE()
);

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode NVARCHAR(50) UNIQUE NOT NULL,
    SupplierName NVARCHAR(200),
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectPercentage DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityRating NVARCHAR(20), -- Excellent/Good/Fair/Poor
    LastAssessmentDate DATETIME
);
```

### Step 2: Create Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity metrics
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT NULL,
    OperationType NVARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    QuantityHandled INT,
    DwellTimeMinutes INT,
    CycleTimeSeconds INT,
    ZoneCode NVARCHAR(20),
    GravityZoneType NVARCHAR(50),
    OperatorID NVARCHAR(50),
    BatchID NVARCHAR(50),
    OrderID NVARCHAR(50),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    FOREIGN KEY (SupplierKey) REFERENCES DimSupplierReliability(SupplierKey)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOps ON FactWarehouseOperations;

-- FactFleetTrips: Vehicle and route telemetry
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKeyStart INT NOT NULL,
    TimeKeyEnd INT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    DistanceKM DECIMAL(10,2),
    FuelLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    LoadWeightKG DECIMAL(10,2),
    RouteSegmentID NVARCHAR(50),
    DelayMinutes INT,
    DelayReason NVARCHAR(100), -- Weather, Traffic, Mechanical, Loading
    TirePressurePSI DECIMAL(5,2),
    EngineHealthScore DECIMAL(5,2), -- 0-100
    FOREIGN KEY (TimeKeyStart) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips ON FactFleetTrips;

-- FactCrossDock: Direct transfer operations
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKeyInbound INT NOT NULL,
    TimeKeyOutbound INT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT NULL,
    OutboundTripKey BIGINT NULL,
    QuantityTransferred INT,
    DwellTimeMinutes INT, -- Time between inbound and outbound
    TransferZone NVARCHAR(50),
    FOREIGN KEY (TimeKeyInbound) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactCrossDock ON FactCrossDock;
```

### Step 3: Create ETL Stored Procedures

```sql
-- Incremental load procedure for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new records from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, QuantityHandled, DwellTimeMinutes,
        CycleTimeSeconds, ZoneCode, GravityZoneType,
        OperatorID, BatchID, OrderID
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        stg.OperationType,
        stg.QuantityHandled,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DwellTimeMinutes,
        DATEDIFF(SECOND, stg.StartTime, stg.EndTime) AS CycleTimeSeconds,
        stg.ZoneCode,
        p.OptimalZoneType,
        stg.OperatorID,
        stg.BatchID,
        stg.OrderID
    FROM StagingWarehouseOps stg
    INNER JOIN DimTime t ON CAST(stg.OperationDateTime AS DATE) = CAST(t.FullDateTime AS DATE)
        AND DATEPART(HOUR, stg.OperationDateTime) = t.[Hour]
        AND (DATEPART(MINUTE, stg.OperationDateTime) / 15) * 15 = t.[Minute]
    INNER JOIN DimGeography g ON stg.WarehouseCode = g.WarehouseCode
    INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
    LEFT JOIN DimSupplierReliability s ON stg.SupplierCode = s.SupplierCode
    WHERE stg.LoadedDateTime > @LastLoadDateTime;
    
    -- Update gravity scores based on recent velocity
    UPDATE DimProductGravity
    SET GravityScore = (
        SELECT 
            (COUNT(*) / 30.0) * 10 + -- Velocity component
            (AVG(CASE WHEN wo.OperationType = 'Picking' THEN 1 ELSE 0 END) * 50) + -- Value proxy
            (p.FragilityIndex * 5) -- Fragility component
        FROM FactWarehouseOperations wo
        WHERE wo.ProductKey = DimProductGravity.ProductKey
            AND wo.TimeKey >= (SELECT MAX(TimeKey) - (30 * 96) FROM DimTime) -- Last 30 days
    ),
    UpdatedDate = GETDATE()
    WHERE ProductKey IN (
        SELECT DISTINCT ProductKey 
        FROM FactWarehouseOperations 
        WHERE TimeKey >= (SELECT MAX(TimeKey) - (30 * 96) FROM DimTime)
    );
END;
GO

-- Procedure for fleet maintenance prioritization
CREATE PROCEDURE usp_GenerateFleetMaintenanceQueue
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT TOP 100
        f.VehicleID,
        COUNT(DISTINCT f.TripKey) AS RecentTrips,
        AVG(f.EngineHealthScore) AS AvgEngineHealth,
        AVG(f.TirePressurePSI) AS AvgTirePressure,
        SUM(f.IdleTimeMinutes) AS TotalIdleTime,
        SUM(f.DelayMinutes) AS TotalDelayMinutes,
        -- Calculate revenue impact proxy
        SUM(wo.QuantityHandled * p.GravityScore) AS RevenueImpactScore,
        -- Prioritization score (lower = higher priority)
        (100 - AVG(f.EngineHealthScore)) * 2 + 
        (CASE WHEN AVG(f.TirePressurePSI) < 30 THEN 50 ELSE 0 END) +
        (SUM(wo.QuantityHandled * p.GravityScore) / 1000.0) AS PriorityScore
    FROM FactFleetTrips f
    LEFT JOIN FactWarehouseOperations wo ON f.TripKey = wo.OperationKey -- Linking via batch/order
    LEFT JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE f.TimeKeyStart >= (SELECT MAX(TimeKey) - (7 * 96) FROM DimTime) -- Last 7 days
    GROUP BY f.VehicleID
    HAVING AVG(f.EngineHealthScore) < 80 OR AVG(f.TirePressurePSI) < 32
    ORDER BY PriorityScore DESC;
END;
GO
```

### Step 4: Configure Power BI Connection

```sql
-- Create a service account view for Power BI (row-level security)
CREATE VIEW vw_PowerBI_WarehouseOperations
AS
SELECT 
    wo.OperationKey,
    t.FullDateTime AS OperationDateTime,
    t.[Year], t.[Quarter], t.[Month], t.[Day],
    t.DayOfWeekName,
    g.Continent, g.Country, g.Region, g.WarehouseName,
    p.SKU, p.ProductName, p.Category, p.GravityScore,
    wo.OperationType,
    wo.QuantityHandled,
    wo.DwellTimeMinutes,
    wo.CycleTimeSeconds,
    wo.GravityZoneType
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE g.IsActive = 1;
GO

CREATE VIEW vw_PowerBI_FleetTrips
AS
SELECT 
    f.TripKey,
    ts.FullDateTime AS TripStartDateTime,
    te.FullDateTime AS TripEndDateTime,
    og.WarehouseName AS OriginWarehouse,
    dg.WarehouseName AS DestinationWarehouse,
    f.VehicleID,
    f.DistanceKM,
    f.FuelLiters,
    f.IdleTimeMinutes,
    f.LoadWeightKG,
    f.DelayMinutes,
    f.DelayReason,
    f.EngineHealthScore,
    (f.FuelLiters / NULLIF(f.DistanceKM, 0)) AS FuelEfficiency
FROM FactFleetTrips f
INNER JOIN DimTime ts ON f.TimeKeyStart = ts.TimeKey
LEFT JOIN DimTime te ON f.TimeKeyEnd = te.TimeKey
INNER JOIN DimGeography og ON f.OriginGeographyKey = og.GeographyKey
INNER JOIN DimGeography dg ON f.DestinationGeographyKey = dg.GeographyKey;
GO
```

## Power BI Template Configuration

### Step 1: Open Template

1. Download `LogiFleet_Pulse_Master.pbit` from the repository
2. Open in Power BI Desktop
3. When prompted, enter SQL Server connection details:
   - Server: `${SQL_SERVER_HOST}`
   - Database: `LogiFleetPulse`
   - Authentication: Use SQL Server or Windows Authentication

### Step 2: Configure Data Refresh

```dax
// Create a parameter for incremental refresh
let
    Source = Sql.Database("${SQL_SERVER_HOST}", "LogiFleetPulse"),
    LastRefreshTime = DateTime.LocalNow(),
    IncrementalFilter = Table.SelectRows(Source, each [LoadedDateTime] > LastRefreshTime)
in
    IncrementalFilter
```

### Step 3: Key DAX Measures

```dax
// Total Dwell Time (warehouse operations)
Total_DwellTime_Hours = 
CALCULATE(
    SUM(FactWarehouseOperations[DwellTimeMinutes]) / 60,
    FactWarehouseOperations[OperationType] IN {"Receiving", "Putaway"}
)

// Average Dwell Time by Gravity Zone
Avg_DwellTime_ByGravityZone = 
AVERAGEX(
    VALUES(FactWarehouseOperations[GravityZoneType]),
    CALCULATE(AVERAGE(FactWarehouseOperations[DwellTimeMinutes]))
)

// Fleet Idle Percentage
Fleet_IdlePercentage = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[IdleTimeMinutes]) + 
    SUMX(FactFleetTrips, DATEDIFF(FactFleetTrips[TripStartDateTime], FactFleetTrips[TripEndDateTime], MINUTE))
) * 100

// Cross-Fact KPI: Dwell Cost per SKU vs Fleet Idle Cost
Combined_LogisticsCost = 
VAR DwellCost = [Total_DwellTime_Hours] * 25 // $25/hour warehouse cost
VAR FleetIdleCost = SUM(FactFleetTrips[IdleTimeMinutes]) / 60 * 45 // $45/hour fleet cost
RETURN DwellCost + FleetIdleCost

// Predictive Bottleneck Index (simplified)
Bottleneck_Risk_Score = 
VAR CurrentDwellAvg = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR HistoricalAvg = CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATEADD(DimTime[FullDateTime], -30, DAY)
)
VAR DwellIncrease = DIVIDE(CurrentDwellAvg - HistoricalAvg, HistoricalAvg, 0)
VAR FleetDelayAvg = AVERAGE(FactFleetTrips[DelayMinutes])
RETURN (DwellIncrease * 50) + (FleetDelayAvg * 2) // Weighted composite score
```

## Common Integration Patterns

### Pattern 1: Staging Table Ingestion from WMS

```sql
-- Create staging table for external WMS data
CREATE TABLE StagingWarehouseOps (
    StagingID BIGINT IDENTITY(1,1) PRIMARY KEY,
    OperationDateTime DATETIME NOT NULL,
    WarehouseCode NVARCHAR(20),
    SKU NVARCHAR(50),
    SupplierCode NVARCHAR(50),
    OperationType NVARCHAR(50),
    QuantityHandled INT,
    StartTime DATETIME,
    EndTime DATETIME,
    ZoneCode NVARCHAR(20),
    OperatorID NVARCHAR(50),
    BatchID NVARCHAR(50),
    OrderID NVARCHAR(50),
    LoadedDateTime DATETIME DEFAULT GETDATE()
);

-- Bulk insert from CSV export
BULK INSERT StagingWarehouseOps
FROM '\\fileserver\exports\wms_operations_daily.csv'
WITH (
    FIRSTROW = 2,
    FIELDTERMINATOR = ',',
    ROWTERMINATOR = '\n',
    TABLOCK
);

-- Run ETL procedure
EXEC usp_LoadWarehouseOperations @LastLoadDateTime = '2026-07-08 00:00:00';
```

### Pattern 2: Real-Time Fleet Telemetry via External Table

```sql
-- Configure PolyBase for streaming data (SQL Server 2019+)
CREATE EXTERNAL DATA SOURCE FleetTelemetryAPI
WITH (
    TYPE = EXTERNAL_GENERICS_TYPE,
    LOCATION = 'https://api.fleet-telematics.com/v2/trips',
    CREDENTIAL = FleetAPI_Credential
);

-- Create external table for real-time ingestion
CREATE EXTERNAL TABLE ExtFleetTelemetry (
    VehicleID NVARCHAR(50),
    TripStartTime DATETIME,
    Latitude DECIMAL(10,8),
    Longitude DECIMAL(11,8),
    FuelLevel DECIMAL(5,2),
    EngineHealth DECIMAL(5,2)
)
WITH (
    DATA_SOURCE = FleetTelemetryAPI,
    FILE_FORMAT = JSON_FORMAT
);

-- Merge into fact table every 15 minutes (via SQL Agent job)
MERGE INTO FactFleetTrips AS target
USING (
    SELECT 
        VehicleID,
        MIN(TripStartTime) AS StartTime,
        MAX(TripStartTime) AS EndTime,
        AVG(EngineHealth) AS AvgEngineHealth
    FROM ExtFleetTelemetry
    WHERE TripStartTime >= DATEADD(MINUTE, -15, GETDATE())
    GROUP BY VehicleID
) AS source
ON target.VehicleID = source.VehicleID 
    AND target.TimeKeyStart = (SELECT TimeKey FROM DimTime WHERE FullDateTime = source.StartTime)
WHEN MATCHED THEN
    UPDATE SET 
        TimeKeyEnd = (SELECT TimeKey FROM DimTime WHERE FullDateTime = source.EndTime),
        EngineHealthScore = source.AvgEngineHealth
WHEN NOT MATCHED THEN
    INSERT (TimeKeyStart, VehicleID, EngineHealthScore, OriginGeographyKey, DestinationGeographyKey)
    VALUES (
        (SELECT TimeKey FROM DimTime WHERE FullDateTime = source.StartTime),
        source.VehicleID,
        source.AvgEngineHealth,
        1, -- Default origin, update based on GPS
        1  -- Default destination
    );
```

### Pattern 3: Gravity Zone Optimization Query

```sql
-- Identify products in suboptimal gravity zones
WITH ProductVelocity AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        p.OptimalZoneType AS RecommendedZone,
        wo.GravityZoneType AS CurrentZone,
        COUNT(*) AS PicksLast30Days,
        AVG(wo.CycleTimeSeconds) AS AvgCycleTime
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE wo.OperationType = 'Picking'
        AND t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY 
        p.ProductKey, p.SKU, p.ProductName, 
        p.OptimalZoneType, wo.GravityZoneType
)
SELECT 
    SKU,
    ProductName,
    CurrentZone,
    RecommendedZone,
    PicksLast30Days,
    AvgCycleTime,
    -- Calculate potential time savings
    (AvgCycleTime - 45) * PicksLast30Days AS PotentialSecondsSaved
FROM ProductVelocity
WHERE CurrentZone <> RecommendedZone
    AND PicksLast30Days > 10 -- Focus on high-impact SKUs
ORDER BY PotentialSecondsSaved DESC;
```

## Configuration Reference

### Environment Variables

```bash
# SQL Server connection
export SQL_SERVER_HOST="your-server.database.windows.net"
export SQL_SERVER_DATABASE="LogiFleetPulse"
export SQL_SERVER_USER="analytics_service"
export SQL_SERVER_PASSWORD="${SQL_PASSWORD}" # Use secret manager

# External API credentials
export FLEET_TELEMETRY_API_KEY="${FLEET_API_KEY}"
export WEATHER_API_KEY="${WEATHER_API_KEY}"

# Power BI Service
export POWERBI_WORKSPACE_ID="your-workspace-guid"
export POWERBI_DATASET_ID="your-dataset-guid"
```

### Row-Level Security Setup

```sql
-- Create security roles for Power BI
CREATE ROLE WarehouseManager;
CREATE ROLE FleetSupervisor;
CREATE ROLE Executive;

-- Grant permissions
GRANT SELECT ON vw_PowerBI_WarehouseOperations TO WarehouseManager;
GRANT SELECT ON vw_PowerBI_FleetTrips TO FleetSupervisor;
GRANT SELECT ON vw_PowerBI_WarehouseOperations TO Executive;
GRANT SELECT ON vw_PowerBI_FleetTrips TO Executive;

-- Implement dynamic security filter (in Power BI)
-- Create a DimUser table and link via email/username
CREATE TABLE DimUser (
    UserKey INT PRIMARY KEY IDENTITY(1,1),
    UserEmail NVARCHAR(200) UNIQUE,
    UserRole NVARCHAR(50),
    AllowedWarehouses NVARCHAR(MAX) -- JSON array: ["WH001", "WH002"]
);

-- In Power BI, create a DAX security filter:
-- [UserEmail] = USERPRINCIPALNAME()
-- AND DimGeography[WarehouseCode] IN JSONSELECTARRAY([AllowedWarehouses])
```

## Troubleshooting

### Issue: Slow query performance on fact tables

**Solution**: Ensure columnstore indexes are present and statistics are updated:

```sql
-- Rebuild columnstore indexes
ALTER INDEX CCI_FactWarehouseOps ON FactWarehouseOperations REBUILD;
ALTER INDEX CCI_FactFleetTrips ON FactFleetTrips REBUILD;

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;

-- Add filtered nonclustered indexes for common queries
CREATE NONCLUSTERED INDEX IX_WarehouseOps_TimeProduct 
ON FactWarehouseOperations (TimeKey, ProductKey) 
INCLUDE (DwellTimeMinutes, QuantityHandled);
```

### Issue: Power BI dashboard not refreshing automatically

**Solution**: Configure scheduled refresh in Power BI Service:

1. Publish the report to Power BI Service
2. Go to Dataset Settings → Scheduled Refresh
3. Set refresh frequency (max: 8 times/day on Pro, 48 times/day on Premium)
4. For real-time: Use DirectQuery mode instead of Import

```dax
// Switch to DirectQuery in Power Query
let
    Source = Sql.Database("${SQL_SERVER_HOST}", "LogiFleetPulse", [Query="SELECT * FROM vw_PowerBI_WarehouseOperations"]),
    DirectQueryMode = Table.Buffer(Source, [EnableDirectQueryMode=true])
in
    DirectQueryMode
```

### Issue: Gravity scores not updating correctly

**Solution**: Debug the gravity score calculation:

```sql
-- Check recent picks and gravity recalculation
SELECT 
    p.SKU,
    p.GravityScore,
    p.UpdatedDate,
    COUNT(wo.OperationKey) AS RecentPicks,
    AVG(wo.CycleTimeSeconds) AS AvgCycleTime
FROM DimProductGravity p
LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
    AND wo.OperationType = 'Picking'
    AND wo.TimeKey >= (SELECT MAX(TimeKey) - (30 * 96) FROM DimTime)
WHERE p.UpdatedDate < DATEADD(DAY, -7, GETDATE()) -- Not updated in 7 days
GROUP BY p.SKU, p.GravityScore, p.UpdatedDate
ORDER BY RecentPicks DESC;

-- Manually trigger gravity recalculation
EXEC usp_LoadWarehouseOperations @LastLoadDateTime = '1900-01-01';
```

### Issue: Missing data from external APIs

**Solution**: Implement retry logic in ETL procedures:

```sql
CREATE PROCEDURE usp_LoadFleetTelemetry_WithRetry
AS
BEGIN
    DECLARE @RetryCount INT = 0;
    DECLARE @MaxRetries INT = 3;
    DECLARE @ErrorMsg NVARCHAR(4000);
    
    WHILE @RetryCount < @MaxRetries
    BEGIN
        BEGIN TRY
            -- Attempt to load from external table
            INSERT INTO FactFleetTrips (TimeKeyStart, VehicleID, EngineHealthScore, OriginGeographyKey, DestinationGeographyKey)
            SELECT 
                (SELECT TimeKey FROM DimTime WHERE FullDateTime = TripStartTime),
                VehicleID,
                EngineHealth,
                1, 1
            FROM ExtFleetTelemetry
            WHERE TripStartTime >= DATEADD(MINUTE, -15, GETDATE());
            
            BREAK; -- Success, exit loop
        END TRY
        BEGIN CATCH
            SET @RetryCount = @RetryCount + 1;
            SET @ErrorMsg = ERROR_MESSAGE();
            
            IF @RetryCount >= @MaxRetries
            BEGIN
                -- Log to error table
                INSERT INTO ETL_ErrorLog (ProcedureName, ErrorMessage, ErrorTime)
                VALUES ('usp_LoadFleetTelemetry_WithRetry', @ErrorMsg, GETDATE());
                
                THROW; -- Re-raise error
            END
            ELSE
            BEGIN
                WAITFOR DELAY '00:00:05'; -- Wait 5 seconds before retry
            END
        END CATCH
    END
END;
```

## Advanced Patterns

### Predictive Bottleneck Detection with Machine Learning

```sql
-- Create a view for ML feature engineering
CREATE VIEW vw_ML_BottleneckFeatures
AS
SELECT 
    t.[Year], t.[Month], t.[Day], t.[Hour], t.TimeSlot15Min,
    g.WarehouseName,
    COUNT(DISTINCT wo.OperationKey) AS OperationsCount,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    SUM(wo.QuantityHandled) AS TotalQuantity,
    AVG(CAST(t.IsWeekend AS INT)) AS IsWeekend,
    COUNT(DISTINCT CASE WHEN wo.OperationType = 'Receiving' THEN wo.OperationKey END) AS ReceivingOps,
    COUNT(DISTINCT CASE WHEN wo.OperationType = 'Picking' THEN wo.OperationKey END) AS PickingOps,
    -- Lag features for time series prediction
    LAG(AVG(wo.DwellTimeMinutes), 1) OVER (PARTITION BY g.WarehouseName ORDER BY t.TimeKey) AS DwellTime_Lag1,
    LAG(AVG(wo.DwellTimeMinutes), 4) OVER (PARTITION BY g.WarehouseName ORDER BY t.TimeKey) AS DwellTime_Lag1Hr
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
GROUP BY t.[Year], t.[Month], t.[Day], t.[Hour], t.TimeSlot15Min, g.WarehouseName, t.TimeKey
