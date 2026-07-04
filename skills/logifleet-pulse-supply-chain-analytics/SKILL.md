---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform for warehouse operations, fleet telemetry, and cross-modal supply chain analytics
triggers:
  - set up logifleet pulse logistics dashboard
  - configure supply chain analytics warehouse
  - implement multi-fact star schema for logistics
  - deploy power bi logistics intelligence platform
  - create warehouse gravity zone analytics
  - integrate fleet telemetry with warehouse data
  - build cross-modal supply chain kpi dashboard
  - troubleshoot logifleet pulse sql schema
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a multi-modal logistics intelligence engine built on MS SQL Server and Power BI. It unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a semantic data layer with a custom multi-fact star schema. The platform enables cross-fact KPI analysis, predictive bottleneck detection, and real-time operational dashboards for supply chain optimization.

**Core Capabilities:**
- Multi-fact star schema with time-phased dimensions
- Warehouse gravity zone optimization
- Fleet telemetry and route optimization
- Cross-dock operations tracking
- Real-time Power BI dashboards (15-minute refresh)
- Predictive analytics for bottlenecks and maintenance
- Role-based access control with row-level security

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Network access to WMS, TMS, and telemetry data sources

### Step 1: Deploy SQL Schema

```sql
-- 1. Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- 2. Create core dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    FullDateTime DATETIME2 NOT NULL,
    Hour TINYINT NOT NULL,
    Minute TINYINT NOT NULL,
    DayOfWeek VARCHAR(10) NOT NULL,
    DayOfMonth TINYINT NOT NULL,
    Month TINYINT NOT NULL,
    Quarter TINYINT NOT NULL,
    FiscalYear INT NOT NULL,
    IsWeekend BIT NOT NULL,
    TimeSlot VARCHAR(20) NOT NULL -- '00:00-00:15', '00:15-00:30', etc.
);

CREATE CLUSTERED INDEX IX_DimTime_DateTime ON DimTime(FullDateTime);

-- 3. Create geography dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL UNIQUE,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- 'Warehouse', 'Route Node', 'Distribution Center'
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(10,8),
    Longitude DECIMAL(11,8),
    TimezoneOffset INT, -- UTC offset in minutes
    IsActive BIT DEFAULT 1
);

CREATE INDEX IX_DimGeography_Type ON DimGeography(LocationType);

-- 4. Create product gravity dimension
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(200) NOT NULL,
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitValue DECIMAL(10,2),
    IsFragile BIT DEFAULT 0,
    RequiresRefrigeration BIT DEFAULT 0,
    VelocityScore DECIMAL(5,2), -- 0-100 based on pick frequency
    ValueScore DECIMAL(5,2), -- 0-100 based on unit value
    FragilityScore DECIMAL(5,2), -- 0-100 based on handling requirements
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    RecommendedZone VARCHAR(50),
    LastRecalculated DATETIME2
);

CREATE INDEX IX_DimProductGravity_Gravity ON DimProductGravity(GravityScore DESC);

-- 5. Create supplier reliability dimension
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200) NOT NULL,
    AvgLeadTimeDays DECIMAL(6,2),
    LeadTimeVariance DECIMAL(6,2),
    DefectRate DECIMAL(5,4), -- 0.0000 to 1.0000
    OnTimeDeliveryRate DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityScore AS (
        (1 - DefectRate) * 30 + 
        OnTimeDeliveryRate * 40 + 
        (ComplianceScore / 100) * 30
    ) PERSISTED,
    LastEvaluated DATETIME2
);

-- 6. Create warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT NULL FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType VARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationStartTime DATETIME2 NOT NULL,
    OperationEndTime DATETIME2 NOT NULL,
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime) PERSISTED,
    Quantity INT NOT NULL,
    StorageZone VARCHAR(50),
    PickPath VARCHAR(200),
    OperatorID VARCHAR(50),
    EquipmentID VARCHAR(50),
    DwellTimeHours DECIMAL(8,2), -- Time item sat before being moved
    IsDelayed BIT DEFAULT 0,
    DelayReasonCode VARCHAR(50),
    TransactionValue DECIMAL(12,2)
);

CREATE CLUSTERED COLUMNSTORE INDEX IX_FactWarehouse_CCI ON FactWarehouseOperations;
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);

-- 7. Create fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50) NOT NULL,
    TripStartTime DATETIME2 NOT NULL,
    TripEndTime DATETIME2 NOT NULL,
    TripDurationMinutes AS DATEDIFF(MINUTE, TripStartTime, TripEndTime) PERSISTED,
    DistanceMiles DECIMAL(8,2),
    FuelConsumedGallons DECIMAL(6,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    AvgSpeedMPH DECIMAL(5,2),
    MaxSpeedMPH DECIMAL(5,2),
    HarshBrakingEvents INT,
    RapidAccelerationEvents INT,
    LoadWeightLbs INT,
    WeatherCondition VARCHAR(50),
    TrafficDelayMinutes INT,
    IsOnTime BIT DEFAULT 1,
    FuelEfficiencyMPG AS (DistanceMiles / NULLIF(FuelConsumedGallons, 0)) PERSISTED,
    TripCost DECIMAL(10,2)
);

CREATE CLUSTERED COLUMNSTORE INDEX IX_FactFleet_CCI ON FactFleetTrips;
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID);
CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);

-- 8. Create cross-dock fact table (links warehouse and fleet)
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT NULL FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT NULL FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    ArrivalTime DATETIME2 NOT NULL,
    DepartureTime DATETIME2 NOT NULL,
    CrossDockDurationMinutes AS DATEDIFF(MINUTE, ArrivalTime, DepartureTime) PERSISTED,
    Quantity INT NOT NULL,
    BypassedStorage BIT DEFAULT 1,
    HandlingCost DECIMAL(8,2)
);

CREATE CLUSTERED COLUMNSTORE INDEX IX_FactCrossDock_CCI ON FactCrossDock;
```

### Step 2: Configure Data Sources

```sql
-- Create stored procedure for incremental data loading
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTime DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last 24 hours if not specified
    IF @LastLoadTime IS NULL
        SET @LastLoadTime = DATEADD(HOUR, -24, GETDATE());
    
    -- Insert from external data source (configure connection separately)
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, OperationStartTime, OperationEndTime,
        Quantity, StorageZone, OperatorID, EquipmentID,
        DwellTimeHours, IsDelayed, DelayReasonCode, TransactionValue
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        wms.OperationType,
        wms.StartTime,
        wms.EndTime,
        wms.Quantity,
        wms.Zone,
        wms.OperatorID,
        wms.EquipmentID,
        wms.DwellHours,
        CASE WHEN wms.DwellHours > 72 THEN 1 ELSE 0 END,
        wms.DelayReason,
        wms.TransValue
    FROM ExternalWMS.dbo.Operations wms
    INNER JOIN DimTime t ON DATEADD(MINUTE, (t.Hour * 60 + t.Minute), CAST(CAST(wms.StartTime AS DATE) AS DATETIME2)) = wms.StartTime
    INNER JOIN DimGeography g ON g.LocationID = wms.WarehouseID
    INNER JOIN DimProductGravity p ON p.SKU = wms.SKU
    LEFT JOIN DimSupplierReliability s ON s.SupplierID = wms.SupplierID
    WHERE wms.StartTime >= @LastLoadTime;
    
    -- Update gravity scores based on new velocity patterns
    EXEC usp_RecalculateGravityScores;
END;
GO

-- Create procedure to recalculate gravity scores
CREATE PROCEDURE usp_RecalculateGravityScores
AS
BEGIN
    -- Update velocity scores based on last 30 days of picking frequency
    WITH VelocityCalc AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickCount,
            PERCENT_RANK() OVER (ORDER BY COUNT(*)) * 100 AS VelocityPercentile
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
          AND OperationStartTime >= DATEADD(DAY, -30, GETDATE())
        GROUP BY ProductKey
    )
    UPDATE p
    SET 
        p.VelocityScore = COALESCE(v.VelocityPercentile, 0),
        p.LastRecalculated = GETDATE()
    FROM DimProductGravity p
    LEFT JOIN VelocityCalc v ON p.ProductKey = v.ProductKey;
    
    -- Update recommended zones based on gravity score
    UPDATE DimProductGravity
    SET RecommendedZone = CASE
        WHEN GravityScore >= 75 THEN 'Zone-A-High-Gravity'
        WHEN GravityScore >= 50 THEN 'Zone-B-Medium-Gravity'
        WHEN GravityScore >= 25 THEN 'Zone-C-Low-Gravity'
        ELSE 'Zone-D-Reserve'
    END;
END;
GO
```

### Step 3: Set Up Power BI Connection

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Enter server name and database: `LogiFleetPulse`
4. Advanced options → SQL statement (optional for custom views):

```sql
-- Custom view for cross-fact KPI analysis
CREATE VIEW vw_CrossFactKPIs AS
SELECT 
    t.FiscalYear,
    t.Quarter,
    t.Month,
    g.Region,
    g.Country,
    -- Warehouse metrics
    COUNT(DISTINCT CASE WHEN w.OperationType = 'Picking' THEN w.OperationKey END) AS TotalPicks,
    AVG(CASE WHEN w.OperationType = 'Picking' THEN w.DurationMinutes END) AS AvgPickTimeMin,
    AVG(w.DwellTimeHours) AS AvgDwellHours,
    -- Fleet metrics
    COUNT(DISTINCT f.TripKey) AS TotalTrips,
    AVG(f.FuelEfficiencyMPG) AS AvgFuelEfficiency,
    AVG(f.IdleTimeMinutes) AS AvgIdleTimeMin,
    SUM(f.TripCost) AS TotalFleetCost,
    -- Cross-dock efficiency
    COUNT(DISTINCT cd.CrossDockKey) AS TotalCrossDocks,
    AVG(cd.CrossDockDurationMinutes) AS AvgCrossDockMin,
    -- Combined efficiency score
    (
        (1 - AVG(f.IdleTimeMinutes) / NULLIF(AVG(f.TripDurationMinutes), 0)) * 0.4 +
        (1 - AVG(w.DwellTimeHours) / 72.0) * 0.3 +
        (AVG(f.FuelEfficiencyMPG) / 10.0) * 0.3
    ) * 100 AS CompositeEfficiencyScore
FROM DimTime t
LEFT JOIN FactWarehouseOperations w ON t.TimeKey = w.TimeKey
LEFT JOIN FactFleetTrips f ON t.TimeKey = f.TimeKey
LEFT JOIN FactCrossDock cd ON t.TimeKey = cd.TimeKey
LEFT JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
GROUP BY t.FiscalYear, t.Quarter, t.Month, g.Region, g.Country;
GO
```

### Step 4: Configure Row-Level Security (RLS)

```sql
-- Create security table
CREATE TABLE SecurityUserRoles (
    UserEmail VARCHAR(200) PRIMARY KEY,
    Role VARCHAR(50) NOT NULL, -- 'Executive', 'Supervisor', 'Operator'
    AllowedRegions VARCHAR(MAX), -- Comma-separated list or 'ALL'
    AllowedWarehouses VARCHAR(MAX), -- Comma-separated list or 'ALL'
    CreatedDate DATETIME2 DEFAULT GETDATE()
);

-- Insert sample roles
INSERT INTO SecurityUserRoles (UserEmail, Role, AllowedRegions, AllowedWarehouses)
VALUES 
    ('executive@company.com', 'Executive', 'ALL', 'ALL'),
    ('supervisor.west@company.com', 'Supervisor', 'West', 'ALL'),
    ('operator.wh01@company.com', 'Operator', 'West', 'WH-01');

-- Create RLS predicate function
CREATE FUNCTION fn_SecurityPredicate(@Region VARCHAR(100))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS AccessAllowed
    WHERE 
        EXISTS (
            SELECT 1 
            FROM dbo.SecurityUserRoles 
            WHERE UserEmail = USER_NAME()
              AND (AllowedRegions = 'ALL' OR @Region IN (SELECT value FROM STRING_SPLIT(AllowedRegions, ',')))
        );
GO

-- Apply security policy to geography dimension
CREATE SECURITY POLICY GeographySecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(Region)
ON dbo.DimGeography
WITH (STATE = ON);
GO
```

## Key Power BI DAX Measures

```dax
// Warehouse dwell time with alert threshold
Avg Dwell Time (Hours) = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeHours]),
    FactWarehouseOperations[OperationType] <> "Shipping"
)

Dwell Alert = 
IF(
    [Avg Dwell Time (Hours)] > 72,
    "⚠️ Exceeds 72hr threshold",
    "✓ Within tolerance"
)

// Fleet idle percentage
Fleet Idle % = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes]),
    0
) * 100

// Cross-fact: Dwell time impact on fleet cost
Dwell-to-Fleet Cost Correlation = 
VAR HighDwellProducts = 
    CALCULATETABLE(
        VALUES(DimProductGravity[ProductKey]),
        FactWarehouseOperations[DwellTimeHours] > 72
    )
RETURN
    CALCULATE(
        SUM(FactFleetTrips[TripCost]),
        TREATAS(HighDwellProducts, DimProductGravity[ProductKey])
    )

// Gravity zone performance
Gravity Zone Efficiency = 
VAR ActualZone = SELECTEDVALUE(FactWarehouseOperations[StorageZone])
VAR RecommendedZone = SELECTEDVALUE(DimProductGravity[RecommendedZone])
RETURN
    IF(
        ActualZone = RecommendedZone,
        100,
        MAX(0, 100 - (COUNTROWS(FactWarehouseOperations) * 5))
    )

// Predictive bottleneck index
Bottleneck Risk Score = 
VAR DwellScore = MIN([Avg Dwell Time (Hours)] / 72 * 40, 40)
VAR IdleScore = MIN([Fleet Idle %] * 0.3, 30)
VAR DelayRate = DIVIDE(
    CALCULATE(COUNTROWS(FactWarehouseOperations), FactWarehouseOperations[IsDelayed] = TRUE()),
    COUNTROWS(FactWarehouseOperations),
    0
) * 30
RETURN
    DwellScore + IdleScore + DelayRate

// Temporal elasticity simulation
Warehouse Utilization Impact = 
VAR CurrentCapacity = 0.80
VAR SimulatedCapacity = 0.95
VAR CapacityRatio = SimulatedCapacity / CurrentCapacity
RETURN
    [Fleet Idle %] * (1 + (CapacityRatio - 1) * 0.6) // Assuming 60% correlation

// Supplier reliability weighted average
Weighted Supplier Score = 
SUMX(
    VALUES(DimSupplierReliability[SupplierKey]),
    DimSupplierReliability[ReliabilityScore] * 
    CALCULATE(COUNTROWS(FactWarehouseOperations))
) / COUNTROWS(FactWarehouseOperations)
```

## Automated Alerting System

```sql
-- Create alert configuration table
CREATE TABLE AlertConfiguration (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName VARCHAR(100) NOT NULL,
    MetricType VARCHAR(50) NOT NULL, -- 'DwellTime', 'FleetIdle', 'FuelEfficiency', etc.
    ThresholdValue DECIMAL(10,2) NOT NULL,
    ThresholdOperator VARCHAR(10) NOT NULL, -- '>', '<', '>=', '<=', '='
    NotificationEmails VARCHAR(MAX), -- Comma-separated
    IsActive BIT DEFAULT 1,
    LastTriggered DATETIME2
);

-- Insert sample alerts
INSERT INTO AlertConfiguration (AlertName, MetricType, ThresholdValue, ThresholdOperator, NotificationEmails)
VALUES 
    ('High Dwell Time Alert', 'DwellTime', 72, '>', 'operations@company.com,supervisor@company.com'),
    ('Low Fuel Efficiency Alert', 'FuelEfficiency', 6.0, '<', 'fleet@company.com'),
    ('Excessive Fleet Idle Alert', 'FleetIdle', 15, '>', 'fleet@company.com,supervisor@company.com');

-- Create stored procedure to check alerts
CREATE PROCEDURE usp_CheckAlerts
AS
BEGIN
    DECLARE @AlertResults TABLE (
        AlertName VARCHAR(100),
        CurrentValue DECIMAL(10,2),
        ThresholdValue DECIMAL(10,2),
        Recipients VARCHAR(MAX)
    );
    
    -- Check dwell time alerts
    INSERT INTO @AlertResults
    SELECT 
        ac.AlertName,
        AVG(w.DwellTimeHours),
        ac.ThresholdValue,
        ac.NotificationEmails
    FROM AlertConfiguration ac
    CROSS APPLY (
        SELECT AVG(DwellTimeHours) AS AvgDwell
        FROM FactWarehouseOperations
        WHERE OperationStartTime >= DATEADD(HOUR, -4, GETDATE())
    ) w
    WHERE ac.MetricType = 'DwellTime'
      AND ac.IsActive = 1
      AND (
          (ac.ThresholdOperator = '>' AND w.AvgDwell > ac.ThresholdValue) OR
          (ac.ThresholdOperator = '<' AND w.AvgDwell < ac.ThresholdValue)
      );
    
    -- Check fleet idle alerts
    INSERT INTO @AlertResults
    SELECT 
        ac.AlertName,
        (SUM(f.IdleTimeMinutes) * 100.0 / NULLIF(SUM(f.TripDurationMinutes), 0)),
        ac.ThresholdValue,
        ac.NotificationEmails
    FROM AlertConfiguration ac
    CROSS APPLY (
        SELECT IdleTimeMinutes, TripDurationMinutes
        FROM FactFleetTrips
        WHERE TripStartTime >= DATEADD(HOUR, -4, GETDATE())
    ) f
    WHERE ac.MetricType = 'FleetIdle'
      AND ac.IsActive = 1
    GROUP BY ac.AlertName, ac.ThresholdValue, ac.NotificationEmails, ac.ThresholdOperator
    HAVING (
        (ac.ThresholdOperator = '>' AND (SUM(f.IdleTimeMinutes) * 100.0 / NULLIF(SUM(f.TripDurationMinutes), 0)) > ac.ThresholdValue)
    );
    
    -- Send alerts (integrate with email service or message queue)
    SELECT * FROM @AlertResults;
END;
GO

-- Schedule this procedure via SQL Server Agent every 15 minutes
```

## Common Integration Patterns

### Pattern 1: Streaming Telemetry Ingestion

```sql
-- Create external data source for fleet telemetry API
CREATE EXTERNAL DATA SOURCE FleetTelemetryAPI
WITH (
    TYPE = GENERIC,
    LOCATION = 'https://api.fleetprovider.com',
    CREDENTIAL = FleetAPICredential -- Create this separately with API key
);

-- Create staging table for real-time data
CREATE TABLE StagingFleetTelemetry (
    VehicleID VARCHAR(50),
    Timestamp DATETIME2,
    Latitude DECIMAL(10,8),
    Longitude DECIMAL(11,8),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2),
    EngineTemp DECIMAL(5,2),
    TirePressureFL DECIMAL(4,1),
    TirePressureFR DECIMAL(4,1),
    TirePressureRL DECIMAL(4,1),
    TirePressureRR DECIMAL(4,1)
);

-- Procedure to process telemetry and generate maintenance queue
CREATE PROCEDURE usp_ProcessTelemetryAndTriageMaintenance
AS
BEGIN
    -- Calculate maintenance priority score
    WITH MaintenancePriority AS (
        SELECT 
            VehicleID,
            MAX(Timestamp) AS LastReported,
            AVG(EngineTemp) AS AvgEngineTemp,
            MIN(LEAST(TirePressureFL, TirePressureFR, TirePressureRL, TirePressureRR)) AS MinTirePressure,
            -- Score: higher = more urgent
            (CASE WHEN AVG(EngineTemp) > 220 THEN 40 ELSE 0 END) +
            (CASE WHEN MIN(LEAST(TirePressureFL, TirePressureFR, TirePressureRL, TirePressureRR)) < 30 THEN 30 ELSE 0 END) AS MechanicalScore
        FROM StagingFleetTelemetry
        WHERE Timestamp >= DATEADD(HOUR, -1, GETDATE())
        GROUP BY VehicleID
    ),
    RevenueImpact AS (
        SELECT 
            f.VehicleID,
            SUM(w.TransactionValue) AS CargoValue
        FROM FactFleetTrips f
        INNER JOIN FactCrossDock cd ON f.TripKey = cd.OutboundTripKey
        INNER JOIN FactWarehouseOperations w ON cd.ProductKey = w.ProductKey
        WHERE f.TripEndTime IS NULL -- In-progress trips
        GROUP BY f.VehicleID
    )
    SELECT 
        mp.VehicleID,
        mp.LastReported,
        mp.AvgEngineTemp,
        mp.MinTirePressure,
        mp.MechanicalScore,
        COALESCE(ri.CargoValue, 0) AS CurrentCargoValue,
        mp.MechanicalScore + (COALESCE(ri.CargoValue, 0) / 10000) AS TotalPriorityScore
    INTO #MaintenanceQueue
    FROM MaintenancePriority mp
    LEFT JOIN RevenueImpact ri ON mp.VehicleID = ri.VehicleID
    WHERE mp.MechanicalScore > 0
    ORDER BY TotalPriorityScore DESC;
    
    -- Output top priority vehicles
    SELECT * FROM #MaintenanceQueue;
END;
GO
```

### Pattern 2: Warehouse Gravity Zone Optimization

```sql
-- Analyze current zone assignments vs recommendations
WITH ZoneMismatch AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        p.GravityScore,
        p.RecommendedZone,
        w.StorageZone AS CurrentZone,
        COUNT(*) AS PickCount,
        AVG(w.DurationMinutes) AS AvgPickTime,
        SUM(w.TransactionValue) AS TotalValue
    FROM DimProductGravity p
    INNER JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
    WHERE w.OperationType = 'Picking'
      AND w.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
      AND w.StorageZone <> p.RecommendedZone
    GROUP BY p.ProductKey, p.SKU, p.ProductName, p.GravityScore, p.RecommendedZone, w.StorageZone
)
SELECT 
    SKU,
    ProductName,
    GravityScore,
    CurrentZone,
    RecommendedZone,
    PickCount,
    AvgPickTime,
    TotalValue,
    -- Estimate time savings if moved to recommended zone
    PickCount * (AvgPickTime - (AvgPickTime * 0.7)) AS EstimatedTimeSavingsMin,
    -- Prioritize by value and frequency
    (PickCount * TotalValue) AS RelocationPriority
FROM ZoneMismatch
ORDER BY RelocationPriority DESC;
```

### Pattern 3: Cross-Fact Query for Root Cause Analysis

```sql
-- Find correlation between supplier delays and fleet costs
WITH SupplierDelays AS (
    SELECT 
        s.SupplierKey,
        s.SupplierName,
        COUNT(CASE WHEN w.IsDelayed = 1 THEN 1 END) AS DelayedOperations,
        COUNT(*) AS TotalOperations,
        CAST(COUNT(CASE WHEN w.IsDelayed = 1 THEN 1 END) AS FLOAT) / COUNT(*) AS DelayRate
    FROM DimSupplierReliability s
    INNER JOIN FactWarehouseOperations w ON s.SupplierKey = w.SupplierKey
    WHERE w.OperationStartTime >= DATEADD(DAY, -90, GETDATE())
    GROUP BY s.SupplierKey, s.SupplierName
    HAVING COUNT(CASE WHEN w.IsDelayed = 1 THEN 1 END) > 10
),
FleetImpact AS (
    SELECT 
        w.SupplierKey,
        SUM(f.TripCost) AS AssociatedFleetCost,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        COUNT(DISTINCT f.TripKey) AS AffectedTrips
    FROM FactWarehouseOperations w
    INNER JOIN FactCrossDock cd ON w.ProductKey = cd.ProductKey
    INNER JOIN FactFleetTrips f ON cd.OutboundTripKey = f.TripKey
    WHERE w.IsDelayed = 1
      AND w.OperationStartTime >= DATEADD(DAY, -90, GETDATE())
    GROUP BY w.SupplierKey
)
SELECT 
    sd.SupplierName,
    sd.DelayedOperations,
    sd.TotalOperations,
    CAST(sd.DelayRate * 100 AS DECIMAL(5,2)) AS
