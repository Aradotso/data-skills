---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehousing platform with multi-fact star schema for warehouse and fleet analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "deploy multi-fact star schema for warehouse operations"
  - "implement fleet telemetry data model"
  - "create supply chain KPI dashboards"
  - "integrate warehouse and fleet analytics"
  - "build logistics data warehouse with SQL Server"
  - "optimize cross-dock operations analytics"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced logistics intelligence platform that integrates warehouse operations, fleet telemetry, and supply chain data into a unified semantic layer. Built on MS SQL Server and Power BI, it uses a multi-fact star schema with time-phased dimensions to enable cross-functional KPI analysis and predictive bottleneck detection.

**Core capabilities:**
- Multi-fact data warehouse (warehouse operations, fleet trips, cross-dock transfers)
- Unified semantic layer linking inventory, fleet, and external data
- Real-time Power BI dashboards with 15-minute refresh cycles
- Predictive analytics for bottleneck detection and fleet maintenance
- Role-based access control with row-level security

## Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, TMS, telematics APIs
- Power BI Pro license (for sharing/collaboration)

## Installation

### 1. Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    TimeOfDay TIME NOT NULL,
    Hour TINYINT NOT NULL,
    MinuteBucket TINYINT NOT NULL, -- 0, 15, 30, 45
    DayOfWeek TINYINT NOT NULL,
    DayOfMonth TINYINT NOT NULL,
    Month TINYINT NOT NULL,
    Quarter TINYINT NOT NULL,
    Year SMALLINT NOT NULL,
    FiscalPeriod VARCHAR(20),
    IsBusinessDay BIT NOT NULL
)
GO

CREATE CLUSTERED COLUMNSTORE INDEX CCI_DimTime ON DimTime
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL UNIQUE,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- Warehouse, Route Node, Cross-Dock
    Address VARCHAR(500),
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone VARCHAR(50),
    IsActive BIT NOT NULL DEFAULT 1
)
GO

CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(300) NOT NULL,
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    Brand VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT NOT NULL DEFAULT 0,
    RequiresRefrigeration BIT NOT NULL DEFAULT 0,
    GravityScore DECIMAL(5,2), -- Calculated: velocity + value + fragility
    ValuePerUnit DECIMAL(10,2),
    IsActive BIT NOT NULL DEFAULT 1
)
GO

CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(300) NOT NULL,
    Country VARCHAR(100),
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2), -- Standard deviation
    DefectRate DECIMAL(5,4), -- Percentage as decimal
    ComplianceScore DECIMAL(3,2), -- 0.00 to 1.00
    IsActive BIT NOT NULL DEFAULT 1
)
GO

CREATE TABLE DimFleet (
    FleetKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID VARCHAR(50) NOT NULL UNIQUE,
    VehicleType VARCHAR(50) NOT NULL, -- Truck, Van, Trailer
    Make VARCHAR(50),
    Model VARCHAR(50),
    Year SMALLINT,
    Capacity DECIMAL(10,2),
    FuelType VARCHAR(30),
    IsActive BIT NOT NULL DEFAULT 1
)
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT NULL,
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    Quantity INT NOT NULL,
    DwellTimeMinutes INT NULL,
    ProcessingTimeMinutes INT NOT NULL,
    StorageZone VARCHAR(50),
    OperatorID VARCHAR(50),
    OrderID VARCHAR(100),
    BatchNumber VARCHAR(100),
    CONSTRAINT FK_FWO_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FWO_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FWO_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_FWO_Supplier FOREIGN KEY (SupplierKey) REFERENCES DimSupplier(SupplierKey)
)
GO

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOperations ON FactWarehouseOperations
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    StartTimeKey INT NOT NULL,
    EndTimeKey INT NOT NULL,
    FleetKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DistanceKM DECIMAL(10,2) NOT NULL,
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DriverID VARCHAR(50),
    OrderCount INT,
    TotalWeight DECIMAL(10,2),
    RouteType VARCHAR(50), -- Direct, Multi-Stop, Return
    DelayMinutes INT DEFAULT 0,
    DelayReason VARCHAR(200),
    CONSTRAINT FK_FFT_StartTime FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FFT_EndTime FOREIGN KEY (EndTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FFT_Fleet FOREIGN KEY (FleetKey) REFERENCES DimFleet(FleetKey),
    CONSTRAINT FK_FFT_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FFT_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
)
GO

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips ON FactFleetTrips
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    ArrivalTimeKey INT NOT NULL,
    DepartureTimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT NULL,
    OutboundTripKey BIGINT NULL,
    Quantity INT NOT NULL,
    DwellTimeMinutes INT NOT NULL,
    CONSTRAINT FK_FCD_ArrivalTime FOREIGN KEY (ArrivalTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FCD_DepartureTime FOREIGN KEY (DepartureTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FCD_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FCD_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
)
GO

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactCrossDock ON FactCrossDock
GO
```

### 2. Create Supporting Objects

```sql
-- View for cross-fact KPI analysis
CREATE VIEW vw_WarehouseFleetIntegration AS
SELECT 
    dt.Date,
    dt.Hour,
    dg.LocationName AS Warehouse,
    dp.Category AS ProductCategory,
    SUM(fwo.Quantity) AS WarehouseVolume,
    AVG(fwo.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(DISTINCT fft.TripKey) AS FleetTrips,
    AVG(fft.IdleTimeMinutes) AS AvgIdleTime,
    SUM(fft.FuelConsumedLiters) AS TotalFuel
FROM FactWarehouseOperations fwo
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
INNER JOIN DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
INNER JOIN DimProduct dp ON fwo.ProductKey = dp.ProductKey
LEFT JOIN FactFleetTrips fft ON fft.OriginGeographyKey = fwo.GeographyKey 
    AND fft.StartTimeKey = fwo.TimeKey
WHERE fwo.OperationType = 'Shipping'
GROUP BY dt.Date, dt.Hour, dg.LocationName, dp.Category
GO

-- Stored procedure for incremental data loading
CREATE PROCEDURE usp_LoadWarehouseOperations
    @StartDate DATETIME2,
    @EndDate DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from staging table or external source
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, Quantity, DwellTimeMinutes, 
        ProcessingTimeMinutes, StorageZone, OperatorID, OrderID, BatchNumber
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        ds.SupplierKey,
        s.OperationType,
        s.Quantity,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
        s.ProcessingTimeMinutes,
        s.StorageZone,
        s.OperatorID,
        s.OrderID,
        s.BatchNumber
    FROM Staging_WarehouseOps s
    INNER JOIN DimTime dt ON dt.DateTime = DATEADD(MINUTE, 
        (DATEPART(MINUTE, s.StartTime) / 15) * 15, 
        DATEADD(HOUR, DATEDIFF(HOUR, 0, s.StartTime), 0))
    INNER JOIN DimGeography dg ON dg.LocationID = s.LocationID
    INNER JOIN DimProduct dp ON dp.SKU = s.SKU
    LEFT JOIN DimSupplier ds ON ds.SupplierID = s.SupplierID
    WHERE s.StartTime >= @StartDate AND s.StartTime < @EndDate
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f
            WHERE f.OrderID = s.OrderID AND f.OperationType = s.OperationType
        )
END
GO

-- Alert stored procedure for threshold monitoring
CREATE PROCEDURE usp_CheckLogisticsAlerts AS
BEGIN
    -- High dwell time alert
    SELECT 
        'High Dwell Time' AS AlertType,
        dg.LocationName,
        dp.SKU,
        AVG(fwo.DwellTimeMinutes) AS AvgDwellTime
    FROM FactWarehouseOperations fwo
    INNER JOIN DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
    INNER JOIN DimProduct dp ON fwo.ProductKey = dp.ProductKey
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    WHERE dt.Date >= CAST(GETDATE() AS DATE)
        AND fwo.OperationType = 'Picking'
    GROUP BY dg.LocationName, dp.SKU
    HAVING AVG(fwo.DwellTimeMinutes) > 72 * 60 -- 72 hours
    
    UNION ALL
    
    -- Fleet idle time alert
    SELECT 
        'Excessive Idle Time' AS AlertType,
        df.VehicleID,
        NULL AS SKU,
        AVG(fft.IdleTimeMinutes) AS AvgIdleTime
    FROM FactFleetTrips fft
    INNER JOIN DimFleet df ON fft.FleetKey = df.FleetKey
    INNER JOIN DimTime dt ON fft.StartTimeKey = dt.TimeKey
    WHERE dt.Date >= CAST(GETDATE() AS DATE)
    GROUP BY df.VehicleID
    HAVING AVG(fft.IdleTimeMinutes) > (AVG(fft.EndTimeKey - fft.StartTimeKey) * 15 * 0.15) -- >15% of trip
END
GO
```

### 3. Configure Power BI Template

1. Download the `.pbit` template file from the repository
2. Open in Power BI Desktop
3. When prompted, enter your SQL Server connection string:
   - Server: `your-server-name.database.windows.net` or `localhost`
   - Database: `LogiFleetPulse`
4. Choose authentication method (Windows or SQL Server)
5. Load the data model

## Configuration

### Connection Configuration

Create a `config.json` file for external data source connections:

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}",
    "driver": "ODBC Driver 17 for SQL Server"
  },
  "telemetry_api": {
    "endpoint": "${TELEMETRY_API_URL}",
    "api_key": "${TELEMETRY_API_KEY}",
    "refresh_interval_minutes": 15
  },
  "weather_api": {
    "endpoint": "${WEATHER_API_URL}",
    "api_key": "${WEATHER_API_KEY}"
  },
  "powerbi": {
    "workspace_id": "${POWERBI_WORKSPACE_ID}",
    "dataset_refresh_schedule": "*/15 * * * *"
  }
}
```

### Role-Based Security

```sql
-- Create security table
CREATE TABLE SecurityRoles (
    RoleID INT PRIMARY KEY IDENTITY(1,1),
    RoleName VARCHAR(50) NOT NULL,
    UserEmail VARCHAR(200) NOT NULL,
    AllowedLocations VARCHAR(MAX), -- JSON array of LocationIDs
    AllowedCategories VARCHAR(MAX) -- JSON array of categories
)
GO

-- Example role assignments
INSERT INTO SecurityRoles (RoleName, UserEmail, AllowedLocations, AllowedCategories)
VALUES 
    ('Executive', 'ceo@company.com', NULL, NULL), -- Full access
    ('RegionalManager', 'manager@company.com', '["WH-001", "WH-002"]', NULL),
    ('CategoryManager', 'catmgr@company.com', NULL, '["Electronics", "Appliances"]')
GO

-- Create row-level security function
CREATE FUNCTION fn_SecurityFilter(@UserEmail VARCHAR(200))
RETURNS TABLE
AS
RETURN
    SELECT 
        GeographyKey,
        ProductKey
    FROM SecurityRoles sr
    CROSS APPLY (
        SELECT GeographyKey FROM DimGeography
        WHERE sr.AllowedLocations IS NULL 
            OR LocationID IN (SELECT value FROM OPENJSON(sr.AllowedLocations))
    ) g
    CROSS APPLY (
        SELECT ProductKey FROM DimProduct
        WHERE sr.AllowedCategories IS NULL
            OR Category IN (SELECT value FROM OPENJSON(sr.AllowedCategories))
    ) p
    WHERE sr.UserEmail = @UserEmail
GO

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityFilter(CURRENT_USER) ON FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityFilter(CURRENT_USER) ON FactCrossDock
WITH (STATE = ON)
GO
```

## Key DAX Measures for Power BI

```dax
// Warehouse Dwell Time (weighted by quantity)
Avg Dwell Time = 
DIVIDE(
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[DwellTimeMinutes] * FactWarehouseOperations[Quantity]
    ),
    SUM(FactWarehouseOperations[Quantity])
)

// Fleet Utilization Rate
Fleet Utilization % = 
VAR TotalTime = 
    SUMX(
        FactFleetTrips,
        (FactFleetTrips[EndTimeKey] - FactFleetTrips[StartTimeKey]) * 15
    )
VAR IdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN
    DIVIDE(TotalTime - IdleTime, TotalTime, 0)

// Cross-Fact KPI: Cost per Delivered Unit
Cost Per Unit = 
VAR FuelCost = SUMX(FactFleetTrips, FactFleetTrips[FuelConsumedLiters]) * 1.5 // $1.50/liter
VAR WarehouseCost = SUMX(FactWarehouseOperations, FactWarehouseOperations[ProcessingTimeMinutes]) * 0.5 // $0.50/min
VAR TotalUnits = CALCULATE(SUM(FactWarehouseOperations[Quantity]), FactWarehouseOperations[OperationType] = "Shipping")
RETURN
    DIVIDE(FuelCost + WarehouseCost, TotalUnits, 0)

// Predictive Bottleneck Index
Bottleneck Risk Score = 
VAR DwellScore = 
    IF([Avg Dwell Time] > 4320, 100, [Avg Dwell Time] / 4320 * 100) // 4320 min = 72 hours
VAR IdleScore = 
    IF([Fleet Utilization %] < 0.70, (1 - [Fleet Utilization %]) * 100, 0)
VAR DelayScore = 
    DIVIDE(SUM(FactFleetTrips[DelayMinutes]), COUNTROWS(FactFleetTrips), 0) * 2
RETURN
    (DwellScore * 0.4) + (IdleScore * 0.3) + (DelayScore * 0.3)

// Warehouse Gravity Score (velocity + value + fragility)
Product Gravity = 
VAR Velocity = 
    DIVIDE(
        CALCULATE(SUM(FactWarehouseOperations[Quantity]), FactWarehouseOperations[OperationType] = "Picking"),
        DISTINCTCOUNT(DimTime[Date]),
        0
    )
VAR Value = AVERAGE(DimProduct[ValuePerUnit])
VAR Fragility = IF(SELECTEDVALUE(DimProduct[IsFragile]) = TRUE, 1.5, 1)
RETURN
    (Velocity * 0.5 + Value * 0.3) * Fragility

// Time-Phased Comparison (YoY, QoQ)
YoY Dwell Change % = 
VAR CurrentDwell = [Avg Dwell Time]
VAR PriorYearDwell = 
    CALCULATE(
        [Avg Dwell Time],
        DATEADD(DimTime[Date], -1, YEAR)
    )
RETURN
    DIVIDE(CurrentDwell - PriorYearDwell, PriorYearDwell, 0)
```

## Common Usage Patterns

### Pattern 1: Daily Operational Dashboard

```sql
-- Query for real-time dashboard (scheduled every 15 minutes)
SELECT 
    dt.Date,
    dt.Hour,
    dg.LocationName,
    COUNT(DISTINCT fwo.OperationKey) AS TotalOperations,
    SUM(CASE WHEN fwo.OperationType = 'Receiving' THEN fwo.Quantity ELSE 0 END) AS ReceivedQty,
    SUM(CASE WHEN fwo.OperationType = 'Shipping' THEN fwo.Quantity ELSE 0 END) AS ShippedQty,
    AVG(CASE WHEN fwo.OperationType = 'Picking' THEN fwo.ProcessingTimeMinutes END) AS AvgPickTime,
    COUNT(DISTINCT fft.TripKey) AS ActiveTrips,
    SUM(fft.DelayMinutes) AS TotalDelayMinutes
FROM DimTime dt
LEFT JOIN FactWarehouseOperations fwo ON dt.TimeKey = fwo.TimeKey
LEFT JOIN DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
LEFT JOIN FactFleetTrips fft ON dt.TimeKey = fft.StartTimeKey
WHERE dt.Date = CAST(GETDATE() AS DATE)
    AND dt.Hour >= DATEPART(HOUR, GETDATE()) - 2
GROUP BY dt.Date, dt.Hour, dg.LocationName
ORDER BY dt.Hour DESC, dg.LocationName
```

### Pattern 2: Predictive Maintenance Queue

```sql
-- Identify vehicles requiring maintenance based on telemetry patterns
WITH VehicleMetrics AS (
    SELECT 
        df.VehicleID,
        df.VehicleType,
        COUNT(fft.TripKey) AS TripCount,
        SUM(fft.DistanceKM) AS TotalDistance,
        AVG(fft.FuelConsumedLiters / NULLIF(fft.DistanceKM, 0)) AS AvgFuelEfficiency,
        SUM(fft.IdleTimeMinutes) AS TotalIdleTime,
        SUM(fft.DelayMinutes) AS TotalDelays
    FROM DimFleet df
    INNER JOIN FactFleetTrips fft ON df.FleetKey = fft.FleetKey
    INNER JOIN DimTime dt ON fft.StartTimeKey = dt.TimeKey
    WHERE dt.Date >= DATEADD(DAY, -30, GETDATE())
    GROUP BY df.VehicleID, df.VehicleType
),
MaintenancePriority AS (
    SELECT 
        VehicleID,
        VehicleType,
        TotalDistance,
        AvgFuelEfficiency,
        (TotalIdleTime * 0.3 + TotalDelays * 0.5 + 
         CASE WHEN AvgFuelEfficiency > 0.12 THEN 100 ELSE 0 END) AS PriorityScore
    FROM VehicleMetrics
)
SELECT TOP 10
    VehicleID,
    VehicleType,
    TotalDistance,
    ROUND(AvgFuelEfficiency, 4) AS FuelEfficiency,
    PriorityScore
FROM MaintenancePriority
WHERE PriorityScore > 50
ORDER BY PriorityScore DESC
```

### Pattern 3: Cross-Dock Optimization

```sql
-- Identify products with excessive cross-dock dwell time
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.Category,
    dg.LocationName AS CrossDockFacility,
    AVG(fcd.DwellTimeMinutes) AS AvgDwellMinutes,
    COUNT(fcd.CrossDockKey) AS TransferCount,
    SUM(fcd.Quantity) AS TotalQuantity
FROM FactCrossDock fcd
INNER JOIN DimProduct dp ON fcd.ProductKey = dp.ProductKey
INNER JOIN DimGeography dg ON fcd.GeographyKey = dg.GeographyKey
INNER JOIN DimTime dt ON fcd.ArrivalTimeKey = dt.TimeKey
WHERE dt.Date >= DATEADD(DAY, -7, GETDATE())
GROUP BY dp.SKU, dp.ProductName, dp.Category, dg.LocationName
HAVING AVG(fcd.DwellTimeMinutes) > 120 -- More than 2 hours
ORDER BY AVG(fcd.DwellTimeMinutes) DESC
```

### Pattern 4: Supplier Performance Scorecard

```sql
-- Calculate supplier reliability metrics
WITH SupplierMetrics AS (
    SELECT 
        ds.SupplierName,
        ds.Country,
        COUNT(DISTINCT fwo.OrderID) AS OrderCount,
        AVG(fwo.ProcessingTimeMinutes) AS AvgProcessingTime,
        AVG(fwo.DwellTimeMinutes) AS AvgDwellTime,
        ds.LeadTimeDays,
        ds.LeadTimeVariance,
        ds.DefectRate,
        ds.ComplianceScore
    FROM DimSupplier ds
    INNER JOIN FactWarehouseOperations fwo ON ds.SupplierKey = fwo.SupplierKey
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    WHERE dt.Date >= DATEADD(MONTH, -3, GETDATE())
        AND fwo.OperationType = 'Receiving'
    GROUP BY ds.SupplierName, ds.Country, ds.LeadTimeDays, 
             ds.LeadTimeVariance, ds.DefectRate, ds.ComplianceScore
)
SELECT 
    SupplierName,
    Country,
    OrderCount,
    ROUND(AvgProcessingTime, 2) AS AvgProcessingTime,
    ROUND(AvgDwellTime, 2) AS AvgDwellTime,
    LeadTimeDays,
    ROUND(LeadTimeVariance, 2) AS LeadTimeVariance,
    ROUND(DefectRate * 100, 2) AS DefectRatePercent,
    ROUND(ComplianceScore * 100, 2) AS CompliancePercent,
    -- Composite reliability score
    ROUND(
        (1 - DefectRate) * 0.4 + 
        ComplianceScore * 0.3 + 
        (1 - LEAST(LeadTimeVariance / 10, 1)) * 0.3,
        3
    ) AS ReliabilityScore
FROM SupplierMetrics
ORDER BY ReliabilityScore DESC
```

## Integration Examples

### Python Integration for Data Ingestion

```python
import pyodbc
import os
import requests
from datetime import datetime, timedelta

# Database connection
conn_str = (
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.getenv('SQL_SERVER_HOST')};"
    f"DATABASE=LogiFleetPulse;"
    f"UID={os.getenv('SQL_USER')};"
    f"PWD={os.getenv('SQL_PASSWORD')}"
)

def ingest_telemetry_data():
    """Fetch fleet telemetry data and insert into staging table"""
    conn = pyodbc.connect(conn_str)
    cursor = conn.cursor()
    
    # Fetch from telemetry API
    api_url = os.getenv('TELEMETRY_API_URL')
    api_key = os.getenv('TELEMETRY_API_KEY')
    
    response = requests.get(
        f"{api_url}/trips/recent",
        headers={"Authorization": f"Bearer {api_key}"},
        params={"since": (datetime.now() - timedelta(minutes=15)).isoformat()}
    )
    
    trips = response.json()
    
    # Insert into staging table
    for trip in trips:
        cursor.execute("""
            INSERT INTO Staging_FleetTrips (
                VehicleID, StartTime, EndTime, OriginLocationID, 
                DestinationLocationID, DistanceKM, FuelConsumedLiters, 
                IdleTimeMinutes, DriverID
            )
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            trip['vehicle_id'],
            trip['start_time'],
            trip['end_time'],
            trip['origin_location'],
            trip['destination_location'],
            trip['distance_km'],
            trip['fuel_consumed'],
            trip['idle_time_minutes'],
            trip['driver_id']
        ))
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Ingested {len(trips)} trips")

def calculate_gravity_scores():
    """Recalculate product gravity scores based on recent activity"""
    conn = pyodbc.connect(conn_str)
    cursor = conn.cursor()
    
    cursor.execute("""
        UPDATE dp
        SET dp.GravityScore = (
            -- Velocity component (picks per day)
            (SELECT AVG(daily_qty) * 0.5
             FROM (
                 SELECT SUM(fwo.Quantity) / NULLIF(COUNT(DISTINCT dt.Date), 0) AS daily_qty
                 FROM FactWarehouseOperations fwo
                 INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
                 WHERE fwo.ProductKey = dp.ProductKey
                     AND fwo.OperationType = 'Picking'
                     AND dt.Date >= DATEADD(DAY, -30, GETDATE())
             ) v) +
            -- Value component
            (dp.ValuePerUnit * 0.3) +
            -- Fragility multiplier
            (CASE WHEN dp.IsFragile = 1 THEN 1.5 ELSE 1 END)
        )
        FROM DimProduct dp
        WHERE dp.IsActive = 1
    """)
    
    conn.commit()
    print(f"Updated {cursor.rowcount} product gravity scores")
    
    cursor.close()
    conn.close()

if __name__ == "__main__":
    ingest_telemetry_data()
    calculate_gravity_scores()
```

### Power Automate Flow for Alerts

```json
{
  "definition": {
    "triggers": {
      "Recurrence": {
        "type": "Recurrence",
        "recurrence": {
          "frequency": "Hour",
          "interval": 1
        }
      }
    },
    "actions": {
      "Execute_Alert_Procedure": {
        "type": "SqlServer",
        "inputs": {
          "server": "${SQL_SERVER_HOST}",
          "database": "LogiFleetPulse",
          "authentication": "SQL",
          "username": "${SQL_USER}",
          "password": "${SQL_PASSWORD}",
          "query": "EXEC usp_
