---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for warehouse, fleet, and supply chain KPI harmonization
triggers:
  - "set up logifleet pulse supply chain analytics"
  - "deploy logistics data warehouse with power bi"
  - "configure multi-fact star schema for warehouse and fleet"
  - "implement supply chain analytics dashboard"
  - "create logistics intelligence reporting system"
  - "build warehouse and fleet KPI tracking"
  - "integrate logistics data modeling with sql server"
  - "set up real-time supply chain monitoring"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that combines:
- **MS SQL Server multi-fact star schema** for warehouse operations, fleet telemetry, and supplier data
- **Power BI dashboards** with cross-fact KPI harmonization
- **Real-time data ingestion** from WMS, telematics, and external APIs
- **Predictive analytics** for bottleneck detection and fleet maintenance
- **Role-based security** and automated alerting

The system unifies disparate logistics data streams into a single semantic layer, enabling queries that span warehouse operations, fleet performance, and supply chain metrics.

## Installation & Prerequisites

### Requirements
- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Network access to data sources (WMS, TMS, telematics APIs)

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Run the complete schema deployment script

USE master;
GO

-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTime DATETIME2 NOT NULL,
    [Year] INT NOT NULL,
    [Quarter] INT NOT NULL,
    [Month] INT NOT NULL,
    [Week] INT NOT NULL,
    [Day] INT NOT NULL,
    [Hour] INT NOT NULL,
    [Minute] INT NOT NULL,
    DayOfWeek NVARCHAR(20),
    IsWeekend BIT DEFAULT 0,
    FiscalPeriod NVARCHAR(10),
    CONSTRAINT UQ_DateTime UNIQUE (DateTime)
);
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID NVARCHAR(50) NOT NULL,
    LocationName NVARCHAR(255),
    LocationType NVARCHAR(50), -- 'Warehouse', 'RouteNode', 'CrossDock'
    Address NVARCHAR(500),
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6),
    CONSTRAINT UQ_LocationID UNIQUE (LocationID)
);
GO

CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) NOT NULL,
    ProductName NVARCHAR(255),
    Category NVARCHAR(100),
    Subcategory NVARCHAR(100),
    IsPerishable BIT DEFAULT 0,
    IsFragile BIT DEFAULT 0,
    Weight DECIMAL(10,2), -- kg
    Volume DECIMAL(10,3), -- cubic meters
    UnitValue DECIMAL(10,2), -- currency
    GravityScore DECIMAL(5,2), -- Warehouse Gravity Zones™
    CONSTRAINT UQ_SKU UNIQUE (SKU)
);
GO

CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID NVARCHAR(50) NOT NULL,
    SupplierName NVARCHAR(255),
    ReliabilityScore DECIMAL(5,2), -- 0-100
    AvgLeadTimeDays INT,
    DefectRatePercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2),
    CONSTRAINT UQ_SupplierID UNIQUE (SupplierID)
);
GO

CREATE TABLE DimVehicle (
    VehicleKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID NVARCHAR(50) NOT NULL,
    VehicleType NVARCHAR(50), -- 'Truck', 'Van', 'Refrigerated'
    Capacity DECIMAL(10,2), -- kg
    FuelType NVARCHAR(50),
    YearMade INT,
    LastMaintenanceDate DATE,
    CONSTRAINT UQ_VehicleID UNIQUE (VehicleID)
);
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT,
    OperationType NVARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity INT,
    DwellTimeMinutes INT,
    CycleTimeMinutes INT,
    StorageZone NVARCHAR(50),
    OperatorID NVARCHAR(50),
    BatchNumber NVARCHAR(100),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    FOREIGN KEY (SupplierKey) REFERENCES DimSupplier(SupplierKey)
);
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    LoadWeightKG DECIMAL(10,2),
    DriverID NVARCHAR(50),
    DelayMinutes INT,
    DelayReason NVARCHAR(255),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    TransferTimeMinutes INT,
    Quantity INT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    FOREIGN KEY (InboundTripKey) REFERENCES FactFleetTrips(TripKey),
    FOREIGN KEY (OutboundTripKey) REFERENCES FactFleetTrips(TripKey)
);
GO

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey) INCLUDE (Quantity, DwellTimeMinutes);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey) INCLUDE (OperationType, Quantity);
CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey) INCLUDE (FuelConsumedLiters, IdleTimeMinutes);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleKey) INCLUDE (DistanceKM, DelayMinutes);
GO
```

### Step 2: Create Time Dimension Population Procedure

```sql
-- Procedure to populate DimTime with 15-minute granularity
CREATE PROCEDURE PopulateDimTime
    @StartDate DATETIME2,
    @EndDate DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        INSERT INTO DimTime (DateTime, [Year], [Quarter], [Month], [Week], [Day], [Hour], [Minute], DayOfWeek, IsWeekend, FiscalPeriod)
        VALUES (
            @CurrentDateTime,
            YEAR(@CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            MONTH(@CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            DAY(@CurrentDateTime),
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(MINUTE, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END,
            'FY' + CAST(YEAR(DATEADD(MONTH, 6, @CurrentDateTime)) AS NVARCHAR) + 'Q' + CAST(DATEPART(QUARTER, DATEADD(MONTH, 6, @CurrentDateTime)) AS NVARCHAR)
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

-- Populate time dimension for 2 years
EXEC PopulateDimTime '2025-01-01', '2027-12-31';
GO
```

### Step 3: Create Data Loading Stored Procedures

```sql
-- Example: Load warehouse operations from staging table
CREATE PROCEDURE LoadWarehouseOperations
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, Quantity, DwellTimeMinutes, CycleTimeMinutes,
        StorageZone, OperatorID, BatchNumber
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        stg.OperationType,
        stg.Quantity,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DwellTimeMinutes,
        stg.CycleTimeMinutes,
        stg.StorageZone,
        stg.OperatorID,
        stg.BatchNumber
    FROM StagingWarehouseOps stg
    INNER JOIN DimTime t ON CAST(stg.StartTime AS DATETIME2) = t.DateTime
    INNER JOIN DimGeography g ON stg.LocationID = g.LocationID
    INNER JOIN DimProduct p ON stg.SKU = p.SKU
    LEFT JOIN DimSupplier s ON stg.SupplierID = s.SupplierID
    WHERE NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations f
        WHERE f.TimeKey = t.TimeKey
          AND f.ProductKey = p.ProductKey
          AND f.BatchNumber = stg.BatchNumber
    );
END;
GO

-- Example: Load fleet trip data
CREATE PROCEDURE LoadFleetTrips
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        TimeKey, VehicleKey, OriginGeographyKey, DestinationGeographyKey,
        TripStartTime, TripEndTime, DistanceKM, FuelConsumedLiters,
        IdleTimeMinutes, LoadingTimeMinutes, UnloadingTimeMinutes,
        LoadWeightKG, DriverID, DelayMinutes, DelayReason
    )
    SELECT 
        t.TimeKey,
        v.VehicleKey,
        go.GeographyKey AS OriginKey,
        gd.GeographyKey AS DestinationKey,
        stg.TripStartTime,
        stg.TripEndTime,
        stg.DistanceKM,
        stg.FuelConsumedLiters,
        stg.IdleTimeMinutes,
        stg.LoadingTimeMinutes,
        stg.UnloadingTimeMinutes,
        stg.LoadWeightKG,
        stg.DriverID,
        stg.DelayMinutes,
        stg.DelayReason
    FROM StagingFleetTrips stg
    INNER JOIN DimTime t ON CAST(stg.TripStartTime AS DATETIME2) = t.DateTime
    INNER JOIN DimVehicle v ON stg.VehicleID = v.VehicleID
    INNER JOIN DimGeography go ON stg.OriginLocationID = go.LocationID
    INNER JOIN DimGeography gd ON stg.DestinationLocationID = gd.LocationID
    WHERE NOT EXISTS (
        SELECT 1 FROM FactFleetTrips f
        WHERE f.VehicleKey = v.VehicleKey
          AND f.TripStartTime = stg.TripStartTime
    );
END;
GO
```

## Key Analytical Views

### Cross-Fact KPI View

```sql
-- Unified view combining warehouse and fleet metrics
CREATE VIEW vw_UnifiedLogisticsKPI AS
SELECT 
    t.[Year],
    t.[Month],
    g.Region,
    g.LocationName,
    -- Warehouse metrics
    SUM(CASE WHEN w.OperationType = 'Shipping' THEN w.Quantity ELSE 0 END) AS TotalShipped,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    -- Fleet metrics
    COUNT(DISTINCT f.TripKey) AS TotalTrips,
    SUM(f.FuelConsumedLiters) AS TotalFuelConsumed,
    AVG(f.IdleTimeMinutes) AS AvgIdleTime,
    SUM(f.DelayMinutes) AS TotalDelayMinutes,
    -- Cross-fact KPIs
    CASE 
        WHEN COUNT(DISTINCT f.TripKey) > 0 
        THEN SUM(CASE WHEN w.OperationType = 'Shipping' THEN w.Quantity ELSE 0 END) * 1.0 / COUNT(DISTINCT f.TripKey)
        ELSE 0 
    END AS ShipmentsPerTrip,
    CASE 
        WHEN SUM(f.DistanceKM) > 0 
        THEN SUM(f.FuelConsumedLiters) / SUM(f.DistanceKM)
        ELSE 0 
    END AS FuelEfficiencyLitersPerKM
FROM DimTime t
LEFT JOIN FactWarehouseOperations w ON t.TimeKey = w.TimeKey
LEFT JOIN FactFleetTrips f ON t.TimeKey = f.TimeKey
LEFT JOIN DimGeography g ON COALESCE(w.GeographyKey, f.OriginGeographyKey) = g.GeographyKey
GROUP BY t.[Year], t.[Month], g.Region, g.LocationName;
GO
```

### Warehouse Gravity Zone Analysis

```sql
-- Identify products that should be reassigned to different gravity zones
CREATE VIEW vw_GravityZoneOptimization AS
WITH ProductVelocity AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        p.GravityScore AS CurrentGravityScore,
        COUNT(*) AS PickFrequency,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        SUM(w.Quantity) AS TotalQuantityMoved
    FROM FactWarehouseOperations w
    INNER JOIN DimProduct p ON w.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE w.OperationType = 'Picking'
      AND t.DateTime >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY p.ProductKey, p.SKU, p.ProductName, p.GravityScore
)
SELECT 
    SKU,
    ProductName,
    CurrentGravityScore,
    PickFrequency,
    AvgDwellTime,
    TotalQuantityMoved,
    -- Calculate recommended gravity score
    CASE 
        WHEN PickFrequency > 100 AND AvgDwellTime < 60 THEN 90.0
        WHEN PickFrequency > 50 AND AvgDwellTime < 120 THEN 70.0
        WHEN PickFrequency > 20 THEN 50.0
        ELSE 30.0
    END AS RecommendedGravityScore,
    CASE 
        WHEN ABS(CurrentGravityScore - 
            CASE 
                WHEN PickFrequency > 100 AND AvgDwellTime < 60 THEN 90.0
                WHEN PickFrequency > 50 AND AvgDwellTime < 120 THEN 70.0
                WHEN PickFrequency > 20 THEN 50.0
                ELSE 30.0
            END) > 20 THEN 'Reassignment Needed'
        ELSE 'Optimal'
    END AS ZoneStatus
FROM ProductVelocity;
GO
```

### Fleet Maintenance Priority Queue

```sql
-- Proactive fleet maintenance based on telemetry and revenue impact
CREATE VIEW vw_FleetMaintenancePriority AS
WITH VehiclePerformance AS (
    SELECT 
        v.VehicleKey,
        v.VehicleID,
        v.VehicleType,
        v.LastMaintenanceDate,
        COUNT(*) AS TripCount,
        SUM(f.DistanceKM) AS TotalDistance,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        SUM(f.DelayMinutes) AS TotalDelayMinutes,
        AVG(f.FuelConsumedLiters / NULLIF(f.DistanceKM, 0)) AS FuelEfficiency
    FROM FactFleetTrips f
    INNER JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY v.VehicleKey, v.VehicleID, v.VehicleType, v.LastMaintenanceDate
)
SELECT 
    VehicleID,
    VehicleType,
    LastMaintenanceDate,
    DATEDIFF(DAY, LastMaintenanceDate, GETDATE()) AS DaysSinceLastMaintenance,
    TripCount,
    TotalDistance,
    AvgIdleTime,
    TotalDelayMinutes,
    FuelEfficiency,
    -- Priority score (higher = more urgent)
    (DATEDIFF(DAY, LastMaintenanceDate, GETDATE()) * 0.3 +
     TotalDelayMinutes * 0.5 +
     AvgIdleTime * 0.2) AS MaintenancePriorityScore
FROM VehiclePerformance
ORDER BY MaintenancePriorityScore DESC;
GO
```

## Power BI Configuration

### Step 1: Connect to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `YOUR_SQL_SERVER_INSTANCE`
4. Database: `LogiFleetPulse`
5. Data Connectivity mode: **DirectQuery** (for real-time) or **Import** (for performance)

### Step 2: Import Pre-Built Measures

```dax
-- Total Shipments (Measure)
Total Shipments = 
CALCULATE(
    SUM(FactWarehouseOperations[Quantity]),
    FactWarehouseOperations[OperationType] = "Shipping"
)

-- Average Dwell Time (Measure)
Avg Dwell Time (Hours) = 
DIVIDE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    60,
    0
)

-- Fleet Utilization Rate (Measure)
Fleet Utilization % = 
VAR TotalTripTime = SUM(FactFleetTrips[TripEndTime]) - SUM(FactFleetTrips[TripStartTime])
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN
DIVIDE(
    TotalTripTime - TotalIdleTime,
    TotalTripTime,
    0
) * 100

-- Cross-Fact: Fuel Cost per Shipment (Measure)
Fuel Cost per Shipment = 
VAR FuelCost = SUM(FactFleetTrips[FuelConsumedLiters]) * 1.50 -- Assuming $1.50/liter
VAR TotalShipments = CALCULATE(
    SUM(FactWarehouseOperations[Quantity]),
    FactWarehouseOperations[OperationType] = "Shipping"
)
RETURN
DIVIDE(FuelCost, TotalShipments, 0)

-- Predictive Bottleneck Index (Measure)
Bottleneck Index = 
VAR HighDwellCount = CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FactWarehouseOperations[DwellTimeMinutes] > 240
)
VAR HighDelayCount = CALCULATE(
    COUNTROWS(FactFleetTrips),
    FactFleetTrips[DelayMinutes] > 30
)
RETURN
(HighDwellCount + HighDelayCount) / COUNTROWS(FactWarehouseOperations) * 100
```

### Step 3: Create Role-Based Security

```dax
-- Create role "Warehouse Manager" in Power BI
-- Row-Level Security filter on DimGeography:
[LocationType] = "Warehouse" && [Region] = USERNAME()

-- Create role "Fleet Manager"
-- Row-Level Security filter on DimGeography:
[LocationType] = "RouteNode" || [LocationType] = "CrossDock"

-- Create role "Executive"
-- No filters (full access)
```

## Automated Alerting

### SQL Agent Job for Threshold Monitoring

```sql
-- Create alert table
CREATE TABLE AlertLog (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertTime DATETIME2 DEFAULT GETDATE(),
    AlertType NVARCHAR(100),
    AlertMessage NVARCHAR(MAX),
    Severity NVARCHAR(20), -- 'Critical', 'Warning', 'Info'
    IsResolved BIT DEFAULT 0
);
GO

-- Alert procedure for high idle time
CREATE PROCEDURE CheckFleetIdleTimeAlert
AS
BEGIN
    INSERT INTO AlertLog (AlertType, AlertMessage, Severity)
    SELECT 
        'Fleet High Idle Time',
        'Vehicle ' + v.VehicleID + ' exceeded 15% idle time on trip ' + CAST(f.TripKey AS NVARCHAR),
        'Warning'
    FROM FactFleetTrips f
    INNER JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(HOUR, -1, GETDATE())
      AND (f.IdleTimeMinutes * 1.0 / NULLIF(DATEDIFF(MINUTE, f.TripStartTime, f.TripEndTime), 0)) > 0.15
      AND NOT EXISTS (
          SELECT 1 FROM AlertLog a 
          WHERE a.AlertMessage LIKE '%' + CAST(f.TripKey AS NVARCHAR) + '%' 
            AND a.IsResolved = 0
      );
END;
GO

-- Alert procedure for warehouse dwell time
CREATE PROCEDURE CheckWarehouseDwellAlert
AS
BEGIN
    INSERT INTO AlertLog (AlertType, AlertMessage, Severity)
    SELECT 
        'Warehouse High Dwell Time',
        'SKU ' + p.SKU + ' at ' + g.LocationName + ' has dwell time of ' + CAST(w.DwellTimeMinutes AS NVARCHAR) + ' minutes',
        CASE 
            WHEN w.DwellTimeMinutes > 480 THEN 'Critical'
            WHEN w.DwellTimeMinutes > 240 THEN 'Warning'
            ELSE 'Info'
        END
    FROM FactWarehouseOperations w
    INNER JOIN DimProduct p ON w.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(HOUR, -1, GETDATE())
      AND w.DwellTimeMinutes > 240
      AND NOT EXISTS (
          SELECT 1 FROM AlertLog a 
          WHERE a.AlertMessage LIKE '%' + p.SKU + '%' 
            AND a.IsResolved = 0
      );
END;
GO

-- Schedule these procedures in SQL Server Agent (or Azure Automation)
-- Run every 15 minutes to match data refresh rate
```

## Common Integration Patterns

### Pattern 1: Real-Time Telematics Ingestion

```sql
-- External table for streaming GPS data (requires PolyBase or OPENROWSET)
-- Example using staging table with bulk insert

CREATE TABLE StagingTelemetry (
    VehicleID NVARCHAR(50),
    Timestamp DATETIME2,
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2),
    EngineTemp DECIMAL(5,2)
);
GO

-- Procedure to process telemetry and update fleet trips
CREATE PROCEDURE ProcessTelemetryData
AS
BEGIN
    -- Update ongoing trips with latest telemetry
    UPDATE f
    SET f.FuelConsumedLiters = v.OriginalFuelLevel - stg.FuelLevel
    FROM FactFleetTrips f
    INNER JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
    INNER JOIN StagingTelemetry stg ON v.VehicleID = stg.VehicleID
    WHERE f.TripEndTime IS NULL
      AND stg.Timestamp >= f.TripStartTime;
      
    -- Clear processed records
    DELETE FROM StagingTelemetry WHERE Timestamp < DATEADD(HOUR, -1, GETDATE());
END;
GO
```

### Pattern 2: External Weather API Integration

```sql
-- Table for weather impact analysis
CREATE TABLE ExternalWeatherData (
    WeatherKey INT PRIMARY KEY IDENTITY(1,1),
    GeographyKey INT,
    Timestamp DATETIME2,
    Condition NVARCHAR(50), -- 'Clear', 'Rain', 'Snow', 'Storm'
    Temperature DECIMAL(5,2),
    Visibility DECIMAL(5,2),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey)
);
GO

-- View correlating weather with fleet delays
CREATE VIEW vw_WeatherImpactAnalysis AS
SELECT 
    g.Region,
    w.Condition,
    COUNT(f.TripKey) AS AffectedTrips,
    AVG(f.DelayMinutes) AS AvgDelayMinutes,
    SUM(f.FuelConsumedLiters) AS TotalFuelConsumed
FROM FactFleetTrips f
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON f.OriginGeographyKey = g.GeographyKey
INNER JOIN ExternalWeatherData w ON g.GeographyKey = w.GeographyKey
    AND w.Timestamp BETWEEN f.TripStartTime AND f.TripEndTime
WHERE f.DelayMinutes > 0
GROUP BY g.Region, w.Condition;
GO
```

### Pattern 3: Python Integration for Predictive Modeling

```python
# Example: Connect to SQL Server and run predictive maintenance model
import pyodbc
import pandas as pd
from sklearn.ensemble import RandomForestClassifier
import os

# Connection string using environment variables
conn_str = (
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.getenv('SQL_SERVER')};"
    f"DATABASE=LogiFleetPulse;"
    f"UID={os.getenv('SQL_USER')};"
    f"PWD={os.getenv('SQL_PASSWORD')}"
)

conn = pyodbc.connect(conn_str)

# Extract fleet performance data
query = """
SELECT 
    v.VehicleID,
    v.YearMade,
    DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) AS DaysSinceLastMaintenance,
    AVG(f.IdleTimeMinutes) AS AvgIdleTime,
    AVG(f.FuelConsumedLiters / NULLIF(f.DistanceKM, 0)) AS FuelEfficiency,
    COUNT(*) AS TripCount,
    CASE WHEN MAX(f.DelayMinutes) > 60 THEN 1 ELSE 0 END AS MaintenanceNeeded
FROM FactFleetTrips f
INNER JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
WHERE t.DateTime >= DATEADD(MONTH, -3, GETDATE())
GROUP BY v.VehicleID, v.YearMade, v.LastMaintenanceDate
"""

df = pd.read_sql(query, conn)

# Train simple predictive model
features = ['YearMade', 'DaysSinceLastMaintenance', 'AvgIdleTime', 'FuelEfficiency', 'TripCount']
X = df[features]
y = df['MaintenanceNeeded']

model = RandomForestClassifier(n_estimators=100, random_state=42)
model.fit(X, y)

# Predict maintenance needs for all vehicles
df['PredictedMaintenanceNeeded'] = model.predict(X)
df['MaintenanceProbability'] = model.predict_proba(X)[:, 1]

# Write predictions back to SQL
cursor = conn.cursor()
for _, row in df.iterrows():
    cursor.execute("""
        UPDATE DimVehicle 
        SET MaintenanceProbability = ?
