---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing and logistics intelligence platform for supply chain analytics with multi-fact star schema
triggers:
  - set up LogiFleet Pulse supply chain analytics
  - deploy LogiCore Analytics warehouse schema
  - configure Power BI logistics dashboard
  - implement multi-fact star schema for fleet data
  - create supply chain intelligence dashboards
  - build warehouse gravity zone analytics
  - integrate fleet telemetry with warehouse operations
  - optimize logistics data modeling in SQL Server
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What It Does

LogiFleet Pulse (LogiCore Analytics) is a comprehensive data warehousing and analytics platform for supply chain and logistics operations. It combines:

- **Multi-fact star schema** data modeling in MS SQL Server
- **Power BI dashboards** for real-time logistics intelligence
- **Cross-modal analytics** linking warehouse operations, fleet telemetry, inventory, and external data
- **Predictive bottleneck detection** and fleet optimization
- **Warehouse Gravity Zones** for spatial optimization based on pick frequency and item characteristics

The platform ingests data from WMS, telematics, supplier portals, and external APIs to provide unified logistics visibility.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (supports PolyBase for external tables)
- Power BI Desktop (latest version)
- Access to source systems (WMS, TMS, telemetry feeds)

### Step 1: Deploy SQL Schema

```sql
-- Execute the schema deployment script
-- This creates the core data warehouse structure
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimensional tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    FullDateTime DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    Hour TINYINT NOT NULL,
    Minute TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayOfMonth TINYINT NOT NULL,
    Month TINYINT NOT NULL,
    Quarter TINYINT NOT NULL,
    Year SMALLINT NOT NULL,
    FiscalPeriod NVARCHAR(20),
    IsBusinessHour BIT NOT NULL,
    TimeSlot15Min NVARCHAR(10)
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID NVARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50), -- 'Warehouse', 'Distribution Center', 'Route Node'
    AddressLine1 NVARCHAR(200),
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    PostalCode NVARCHAR(20),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone NVARCHAR(50)
);

CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(100) NOT NULL UNIQUE,
    ProductName NVARCHAR(500),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,2),
    IsFragile BIT,
    IsPerishable BIT,
    ShelfLifeDays INT,
    UnitValue DECIMAL(10,2),
    GravityScore DECIMAL(5,2), -- Calculated: velocity × value × fragility factor
    VelocityClass NVARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValidFrom DATETIME NOT NULL DEFAULT GETDATE(),
    ValidTo DATETIME NULL
);

CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID NVARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200),
    Country NVARCHAR(100),
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    OnTimeDeliveryRate DECIMAL(5,4),
    ReliabilityScore DECIMAL(5,2), -- Calculated composite score
    ValidFrom DATETIME NOT NULL DEFAULT GETDATE(),
    ValidTo DATETIME NULL
);

CREATE TABLE DimVehicle (
    VehicleKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID NVARCHAR(50) NOT NULL UNIQUE,
    VehicleType NVARCHAR(50),
    Make NVARCHAR(50),
    Model NVARCHAR(50),
    Year SMALLINT,
    Capacity DECIMAL(10,2),
    FuelType NVARCHAR(30),
    Status NVARCHAR(30), -- 'Active', 'Maintenance', 'Retired'
    AssignedWarehouseKey INT
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType NVARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationID NVARCHAR(100),
    Quantity INT NOT NULL,
    CycleTimeMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    ZoneID NVARCHAR(50),
    GravityZone NVARCHAR(20), -- 'High', 'Medium', 'Low'
    PickPath NVARCHAR(500),
    EmployeeID NVARCHAR(50),
    EquipmentID NVARCHAR(50),
    CONSTRAINT FK_WarehouseOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WarehouseOps_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WarehouseOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripID NVARCHAR(100),
    DistanceKM DECIMAL(10,2),
    DurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    FuelConsumed DECIMAL(10,2),
    LoadWeight DECIMAL(10,2),
    LoadValue DECIMAL(12,2),
    DriverID NVARCHAR(50),
    SpeedAvg DECIMAL(5,2),
    SpeedMax DECIMAL(5,2),
    DelayMinutes DECIMAL(10,2),
    DelayReason NVARCHAR(200),
    WeatherCondition NVARCHAR(50),
    TrafficLevel NVARCHAR(20),
    CONSTRAINT FK_FleetTrips_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrips_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_FleetTrips_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FleetTrips_Dest FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE TABLE FactInventory (
    InventoryKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT,
    BatchNumber NVARCHAR(100),
    QuantityOnHand INT NOT NULL,
    QuantityReserved INT,
    QuantityAvailable INT,
    AgeInDays INT,
    ZoneLocation NVARCHAR(100),
    LastMovementDate DATETIME,
    ExpiryDate DATETIME,
    CONSTRAINT FK_Inventory_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Inventory_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_Inventory_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_Inventory_Supplier FOREIGN KEY (SupplierKey) REFERENCES DimSupplier(SupplierKey)
);

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_Geography ON FactWarehouseOperations(GeographyKey);

CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Vehicle ON FactFleetTrips(VehicleKey);
CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Origin ON FactFleetTrips(OriginGeographyKey);

CREATE NONCLUSTERED INDEX IX_FactInventory_Time ON FactInventory(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactInventory_Product ON FactInventory(ProductKey);
CREATE NONCLUSTERED INDEX IX_FactInventory_AgeInDays ON FactInventory(AgeInDays);

GO
```

### Step 2: Create Aggregation Views

```sql
-- Create unified KPI view for cross-fact analysis
CREATE VIEW vw_UnifiedLogisticsKPIs AS
SELECT 
    t.FullDateTime,
    t.DateKey,
    g.LocationName,
    g.Region,
    p.SKU,
    p.ProductName,
    p.Category,
    p.GravityScore,
    
    -- Warehouse metrics
    SUM(CASE WHEN wo.OperationType = 'Picking' THEN wo.Quantity ELSE 0 END) AS PickedQuantity,
    AVG(CASE WHEN wo.OperationType = 'Picking' THEN wo.CycleTimeMinutes ELSE NULL END) AS AvgPickTime,
    AVG(wo.DwellTimeHours) AS AvgDwellTime,
    
    -- Inventory metrics
    SUM(inv.QuantityOnHand) AS TotalOnHand,
    AVG(inv.AgeInDays) AS AvgInventoryAge,
    
    -- Fleet metrics (joined through geography)
    COUNT(DISTINCT ft.TripID) AS NumberOfTrips,
    SUM(ft.DistanceKM) AS TotalDistanceKM,
    SUM(ft.IdleTimeMinutes) AS TotalIdleTime,
    SUM(ft.FuelConsumed) AS TotalFuelConsumed

FROM DimTime t
LEFT JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
LEFT JOIN FactInventory inv ON t.TimeKey = inv.TimeKey
LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.TimeKey
LEFT JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
LEFT JOIN DimProduct p ON wo.ProductKey = p.ProductKey

GROUP BY 
    t.FullDateTime, t.DateKey, g.LocationName, g.Region,
    p.SKU, p.ProductName, p.Category, p.GravityScore;
GO
```

### Step 3: Create Stored Procedures for Data Loading

```sql
-- Incremental load procedure for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last 24 hours if no parameter provided
    IF @LastLoadDateTime IS NULL
        SET @LastLoadDateTime = DATEADD(HOUR, -24, GETDATE());
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OperationID, Quantity, CycleTimeMinutes, DwellTimeHours,
        ZoneID, GravityZone, PickPath, EmployeeID, EquipmentID
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        src.OperationType,
        src.OperationID,
        src.Quantity,
        src.CycleTimeMinutes,
        src.DwellTimeHours,
        src.ZoneID,
        src.GravityZone,
        src.PickPath,
        src.EmployeeID,
        src.EquipmentID
    FROM ExternalDataSource.WarehouseOperations src
    INNER JOIN DimTime t ON CAST(src.OperationDateTime AS DATETIME) = t.FullDateTime
    INNER JOIN DimGeography g ON src.LocationID = g.LocationID
    INNER JOIN DimProduct p ON src.SKU = p.SKU
    WHERE src.OperationDateTime > @LastLoadDateTime
    AND NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations 
        WHERE OperationID = src.OperationID
    );
    
    RETURN @@ROWCOUNT;
END;
GO

-- Calculate gravity scores for products
CREATE PROCEDURE usp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    UPDATE p
    SET 
        GravityScore = (
            (velocity_rank * 0.4) + 
            (value_rank * 0.3) + 
            (fragility_score * 0.3)
        ) * 100,
        VelocityClass = CASE 
            WHEN velocity_rank >= 0.7 THEN 'Fast'
            WHEN velocity_rank >= 0.3 THEN 'Medium'
            ELSE 'Slow'
        END
    FROM DimProduct p
    INNER JOIN (
        SELECT 
            ProductKey,
            PERCENT_RANK() OVER (ORDER BY TotalPicks DESC) AS velocity_rank,
            PERCENT_RANK() OVER (ORDER BY UnitValue DESC) AS value_rank,
            (CAST(IsFragile AS DECIMAL) * 0.5 + CAST(IsPerishable AS DECIMAL) * 0.5) AS fragility_score
        FROM DimProduct p2
        LEFT JOIN (
            SELECT 
                ProductKey,
                COUNT(*) AS TotalPicks
            FROM FactWarehouseOperations
            WHERE OperationType = 'Picking'
            AND TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime) -- Last 7 days (15-min intervals)
            GROUP BY ProductKey
        ) picks ON p2.ProductKey = picks.ProductKey
    ) calc ON p.ProductKey = calc.ProductKey;
END;
GO
```

### Step 4: Configure Data Sources

Create a configuration file `config.json` (not checked into version control):

```json
{
  "connections": {
    "sql_server": {
      "server": "${SQL_SERVER_HOST}",
      "database": "LogiFleetPulse",
      "username": "${SQL_SERVER_USER}",
      "password": "${SQL_SERVER_PASSWORD}",
      "driver": "ODBC Driver 17 for SQL Server"
    },
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}"
    },
    "telematics_api": {
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "api_key": "${TELEMATICS_API_KEY}"
    }
  },
  "refresh_intervals": {
    "warehouse_operations": 15,
    "fleet_trips": 15,
    "inventory": 60
  }
}
```

## Power BI Dashboard Configuration

### Connect to Data Source

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Connect to your LogiFleetPulse database
4. Import the following tables:
   - `vw_UnifiedLogisticsKPIs` (for cross-fact analysis)
   - All dimension tables (DimTime, DimGeography, DimProduct, etc.)
   - Fact tables as needed

### Key DAX Measures

```dax
// Total Dwell Time by Gravity Zone
TotalDwellTime = 
CALCULATE(
    SUM(FactWarehouseOperations[DwellTimeHours]),
    ALLEXCEPT(FactWarehouseOperations, FactWarehouseOperations[GravityZone])
)

// Fleet Utilization Rate
FleetUtilizationRate = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
)

// Inventory Turnover Ratio
InventoryTurnover = 
DIVIDE(
    CALCULATE(SUM(FactWarehouseOperations[Quantity]), 
              FactWarehouseOperations[OperationType] = "Shipping"),
    AVERAGE(FactInventory[QuantityOnHand]),
    0
)

// Predictive Bottleneck Index (composite score)
BottleneckIndex = 
VAR DwellScore = DIVIDE([TotalDwellTime], 48, 0) * 30 // 48hr threshold
VAR IdleScore = DIVIDE(SUM(FactFleetTrips[IdleTimeMinutes]), 
                       SUM(FactFleetTrips[DurationMinutes]), 0) * 40
VAR InventoryAgeScore = DIVIDE(AVERAGE(FactInventory[AgeInDays]), 90, 0) * 30 // 90 day threshold
RETURN DwellScore + IdleScore + InventoryAgeScore

// Cross-Fact KPI: Dwell Time vs Fleet Idle Cost
DwellImpactOnFleet = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR AvgIdleTime = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR IdleCostPerMinute = 0.5 // Configurable parameter
RETURN AvgDwell * AvgIdleTime * IdleCostPerMinute

// Time Intelligence: Week-over-Week Change
WoW_PickRate = 
VAR CurrentWeek = 
    CALCULATE(
        DIVIDE(SUM(FactWarehouseOperations[Quantity]), 
               COUNT(FactWarehouseOperations[OperationKey])),
        FactWarehouseOperations[OperationType] = "Picking"
    )
VAR PreviousWeek = 
    CALCULATE(
        DIVIDE(SUM(FactWarehouseOperations[Quantity]), 
               COUNT(FactWarehouseOperations[OperationKey])),
        FactWarehouseOperations[OperationType] = "Picking",
        DATEADD(DimTime[FullDateTime], -7, DAY)
    )
RETURN DIVIDE(CurrentWeek - PreviousWeek, PreviousWeek, 0)
```

### Row-Level Security Setup

```dax
// RLS role: Regional Manager (can only see their region)
[Region] = USERNAME()

// RLS role: Warehouse Supervisor (can only see their warehouse)
[LocationName] IN 
    LOOKUPVALUE(
        UserWarehouseAccess[WarehouseName],
        UserWarehouseAccess[UserEmail],
        USERNAME()
    )
```

## Common Analytical Queries

### Find Delayed Shipments with High Dwell Time

```sql
SELECT 
    ft.TripID,
    ft.DelayMinutes,
    ft.DelayReason,
    wo.DwellTimeHours,
    p.SKU,
    p.ProductName,
    g_origin.LocationName AS OriginWarehouse,
    g_dest.LocationName AS Destination,
    t.FullDateTime AS TripStartTime
FROM FactFleetTrips ft
INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
INNER JOIN DimGeography g_origin ON ft.OriginGeographyKey = g_origin.GeographyKey
INNER JOIN DimGeography g_dest ON ft.DestinationGeographyKey = g_dest.GeographyKey
INNER JOIN FactWarehouseOperations wo ON g_origin.GeographyKey = wo.GeographyKey
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
WHERE ft.DelayMinutes > 30
  AND wo.DwellTimeHours > 72
  AND ft.WeatherCondition IS NOT NULL
  AND wo.OperationType = 'Shipping'
  AND t.FullDateTime >= DATEADD(DAY, -90, GETDATE())
ORDER BY ft.DelayMinutes DESC, wo.DwellTimeHours DESC;
```

### Warehouse Gravity Zone Optimization Analysis

```sql
SELECT 
    wo.GravityZone,
    p.VelocityClass,
    COUNT(DISTINCT wo.OperationID) AS TotalOperations,
    AVG(wo.CycleTimeMinutes) AS AvgCycleTime,
    AVG(wo.DwellTimeHours) AS AvgDwellTime,
    SUM(wo.Quantity) AS TotalQuantity,
    CASE 
        WHEN wo.GravityZone = 'High' AND p.VelocityClass = 'Fast' THEN 'Optimal'
        WHEN wo.GravityZone = 'Low' AND p.VelocityClass = 'Slow' THEN 'Optimal'
        ELSE 'Misaligned'
    END AS ZoneAlignment
FROM FactWarehouseOperations wo
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationType = 'Picking'
  AND wo.TimeKey >= (SELECT MAX(TimeKey) - 2016 FROM DimTime) -- Last 3 weeks
GROUP BY wo.GravityZone, p.VelocityClass
ORDER BY wo.GravityZone, p.VelocityClass;
```

### Fleet Maintenance Priority Queue

```sql
SELECT 
    v.VehicleID,
    v.VehicleType,
    v.Make,
    v.Model,
    COUNT(ft.TripKey) AS TripCount,
    SUM(ft.DistanceKM) AS TotalDistance,
    SUM(ft.IdleTimeMinutes) AS TotalIdleTime,
    AVG(ft.LoadValue) AS AvgLoadValue,
    MAX(ft.SpeedMax) AS MaxSpeed,
    -- Priority score: higher for high-value loads with high usage
    (SUM(ft.DistanceKM) * 0.3 + 
     SUM(ft.IdleTimeMinutes) * 0.2 + 
     AVG(ft.LoadValue) * 0.5) AS MaintenancePriority
FROM DimVehicle v
INNER JOIN FactFleetTrips ft ON v.VehicleKey = ft.VehicleKey
WHERE v.Status = 'Active'
  AND ft.TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime) -- Last week
GROUP BY v.VehicleID, v.VehicleType, v.Make, v.Model
ORDER BY MaintenancePriority DESC;
```

### Supplier Reliability Impact on Dwell Time

```sql
SELECT 
    s.SupplierName,
    s.ReliabilityScore,
    s.LeadTimeDays,
    s.LeadTimeVariance,
    AVG(inv.AgeInDays) AS AvgInventoryAge,
    AVG(wo.DwellTimeHours) AS AvgDwellTime,
    COUNT(DISTINCT inv.BatchNumber) AS BatchCount
FROM DimSupplier s
INNER JOIN FactInventory inv ON s.SupplierKey = inv.SupplierKey
INNER JOIN DimProduct p ON inv.ProductKey = p.ProductKey
INNER JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey 
    AND inv.GeographyKey = wo.GeographyKey
WHERE inv.TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime) -- Last 30 days
GROUP BY s.SupplierName, s.ReliabilityScore, s.LeadTimeDays, s.LeadTimeVariance
HAVING AVG(wo.DwellTimeHours) > 48
ORDER BY s.ReliabilityScore ASC, AVG(wo.DwellTimeHours) DESC;
```

## Automated Alerting

```sql
-- Create alert configuration table
CREATE TABLE AlertConfiguration (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName NVARCHAR(200),
    KPIName NVARCHAR(100),
    ThresholdOperator NVARCHAR(10), -- '>', '<', '>=', '<=', '='
    ThresholdValue DECIMAL(18,2),
    AlertRecipients NVARCHAR(MAX), -- Comma-separated emails
    IsActive BIT DEFAULT 1
);

-- Alert execution procedure
CREATE PROCEDURE usp_ExecuteAlerts
AS
BEGIN
    DECLARE @AlertName NVARCHAR(200);
    DECLARE @KPIName NVARCHAR(100);
    DECLARE @CurrentValue DECIMAL(18,2);
    DECLARE @ThresholdValue DECIMAL(18,2);
    DECLARE @Recipients NVARCHAR(MAX);
    
    -- Fleet idle time alert
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips ft
        INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -1, GETDATE())
        HAVING AVG(CAST(ft.IdleTimeMinutes AS DECIMAL) / ft.DurationMinutes) > 0.15
    )
    BEGIN
        -- Send alert (integrate with email service or Teams webhook)
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = '${ALERT_RECIPIENTS}',
            @subject = 'LogiFleet Alert: Fleet Idle Time Exceeded',
            @body = 'Fleet idle time has exceeded 15% in the last hour.',
            @profile_name = 'LogiFleetAlerts';
    END;
    
    -- Inventory age alert
    IF EXISTS (
        SELECT 1 
        FROM FactInventory inv
        INNER JOIN DimTime t ON inv.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -1, GETDATE())
        AND inv.AgeInDays > 90
        GROUP BY inv.ProductKey
        HAVING SUM(inv.QuantityOnHand) > 0
    )
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = '${ALERT_RECIPIENTS}',
            @subject = 'LogiFleet Alert: Aged Inventory Detected',
            @body = 'Products with inventory age > 90 days detected.',
            @profile_name = 'LogiFleetAlerts';
    END;
END;
GO

-- Schedule alert execution (SQL Server Agent Job)
-- Run every 15 minutes
```

## Integration Patterns

### WMS Data Ingestion (Python Example)

```python
import pyodbc
import requests
import os
from datetime import datetime, timedelta

# Connection configuration
conn_str = (
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.getenv('SQL_SERVER_HOST')};"
    f"DATABASE=LogiFleetPulse;"
    f"UID={os.getenv('SQL_SERVER_USER')};"
    f"PWD={os.getenv('SQL_SERVER_PASSWORD')}"
)

def ingest_warehouse_operations():
    """Fetch warehouse operations from WMS API and load into SQL Server"""
    
    # Get last load timestamp
    conn = pyodbc.connect(conn_str)
    cursor = conn.cursor()
    
    cursor.execute("""
        SELECT ISNULL(MAX(FullDateTime), DATEADD(DAY, -1, GETDATE()))
        FROM DimTime t
        INNER JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
    """)
    last_load = cursor.fetchone()[0]
    
    # Fetch new data from WMS API
    wms_endpoint = os.getenv('WMS_API_ENDPOINT')
    wms_api_key = os.getenv('WMS_API_KEY')
    
    response = requests.get(
        f"{wms_endpoint}/operations",
        headers={"Authorization": f"Bearer {wms_api_key}"},
        params={"since": last_load.isoformat()}
    )
    
    operations = response.json()
    
    # Bulk insert
    for op in operations:
        cursor.execute("""
            EXEC usp_LoadWarehouseOperations @LastLoadDateTime = ?
        """, last_load)
    
    conn.commit()
    cursor.close()
    conn.close()

def calculate_gravity_scores():
    """Recalculate product gravity scores"""
    conn = pyodbc.connect(conn_str)
    cursor = conn.cursor()
    cursor.execute("EXEC usp_UpdateProductGravityScores")
    conn.commit()
    cursor.close()
    conn.close()

if __name__ == "__main__":
    ingest_warehouse_operations()
    calculate_gravity_scores()
```

### Telematics Feed Processing

```python
import pyodbc
import os
import json

def process_telematics_stream(telemetry_data):
    """Process real-time telematics data and insert into FactFleetTrips"""
    
    conn = pyodbc.connect(conn_str)
    cursor = conn.cursor()
    
    for event in telemetry_data:
        # Map to dimension keys
        cursor.execute("""
            DECLARE @TimeKey INT, @VehicleKey INT, @OriginKey INT, @DestKey INT;
            
            SELECT @TimeKey = TimeKey FROM DimTime 
            WHERE FullDateTime = ?;
            
            SELECT @VehicleKey = VehicleKey FROM DimVehicle 
            WHERE VehicleID = ?;
            
            SELECT @OriginKey = GeographyKey FROM DimGeography 
            WHERE LocationID = ?;
            
            SELECT @DestKey = GeographyKey FROM DimGeography 
            WHERE LocationID = ?;
            
            INSERT INTO FactFleetTrips (
                TimeKey, VehicleKey, OriginGeographyKey, DestinationGeographyKey,
                Trip
