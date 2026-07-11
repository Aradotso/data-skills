---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for logistics intelligence with multi-fact star schema, fleet optimization, and warehouse operations analytics
triggers:
  - set up logistics analytics dashboard
  - create supply chain data warehouse
  - build power bi logistics visualization
  - implement fleet tracking analytics
  - design warehouse operations reporting
  - configure multi-fact star schema for logistics
  - analyze fleet and warehouse KPIs
  - deploy logistics intelligence platform
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform built on MS SQL Server and Power BI. It provides:

- **Multi-fact star schema** for warehouse operations, fleet trips, and cross-dock activities
- **Unified semantic layer** linking warehouse, fleet, and external data sources
- **Real-time dashboards** with 15-minute refresh intervals
- **Predictive analytics** for bottleneck detection and fleet maintenance
- **Warehouse Gravity Zones** for spatial optimization based on pick frequency and item value
- **Cross-fact KPI harmonization** (e.g., inventory turnover vs. fuel burn rates)

The platform integrates data from WMS systems, telematics/GPS feeds, supplier portals, weather APIs, and customer order history.

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SSMS (SQL Server Management Studio) or Azure Data Studio
- Access to warehouse/fleet data sources (WMS, TMS, telematics)

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance via SSMS
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Deploy dimension tables first
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    WeekOfYear INT NOT NULL,
    DayOfMonth INT NOT NULL,
    DayOfWeek INT NOT NULL,
    HourOfDay INT NOT NULL,
    MinuteBucket INT NOT NULL, -- 0, 15, 30, 45
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT,
    IsHoliday BIT
);
CREATE CLUSTERED INDEX IX_DimTime_DateTime ON DimTime(FullDateTime);

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeoKey INT PRIMARY KEY IDENTITY(1,1),
    NodeID VARCHAR(50) UNIQUE NOT NULL,
    NodeName VARCHAR(200) NOT NULL,
    NodeType VARCHAR(50), -- 'Warehouse', 'Route Node', 'Distribution Center'
    Continent VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    City VARCHAR(100),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    ParentNodeID VARCHAR(50)
);
CREATE NONCLUSTERED INDEX IX_DimGeo_NodeID ON DimGeography(NodeID);

-- DimProductGravity: Products with velocity-based gravity scores
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) UNIQUE NOT NULL,
    ProductName VARCHAR(500),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitValue DECIMAL(18,2),
    Fragility VARCHAR(20), -- 'Low', 'Medium', 'High'
    GravityScore DECIMAL(5,2), -- Calculated: velocity + value + fragility
    PickFrequencyRank INT,
    ReplenishmentLeadTime INT -- Days
);
CREATE NONCLUSTERED INDEX IX_Product_SKU ON DimProductGravity(SKU);

-- DimSupplierReliability
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) UNIQUE NOT NULL,
    SupplierName VARCHAR(300),
    LeadTimeMean INT, -- Days
    LeadTimeVariance DECIMAL(8,2),
    DefectPercentage DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    LastAuditDate DATE
);

-- FactWarehouseOperations: Core warehouse activity metrics
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeoKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeoKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled INT,
    DwellTimeMinutes INT, -- Time from receipt to shipment
    PickCycleSeconds INT,
    PackCycleSeconds INT,
    StorageZone VARCHAR(50),
    OperatorID VARCHAR(50),
    ErrorFlag BIT DEFAULT 0
);
CREATE NONCLUSTERED INDEX IX_FactWH_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWH_Product ON FactWarehouseOperations(ProductKey);

-- FactFleetTrips: Vehicle and route performance
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeoKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeoKey),
    DestGeoKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeoKey),
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    TripDurationMinutes INT,
    LoadWeightKG DECIMAL(10,2),
    WeatherDelayFlag BIT DEFAULT 0,
    TrafficDelayFlag BIT DEFAULT 0
);
CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID);

-- FactCrossDock: Cross-dock transfers
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    QuantityTransferred INT,
    DockDwellMinutes INT,
    DirectTransferFlag BIT DEFAULT 0
);
```

### Step 2: Populate Dimension Tables

```sql
-- Populate DimTime with 15-minute granularity for 3 years
DECLARE @StartDate DATETIME = '2024-01-01 00:00:00';
DECLARE @EndDate DATETIME = '2027-12-31 23:45:00';
DECLARE @CurrentDate DATETIME = @StartDate;

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO DimTime (
        TimeKey, FullDateTime, Year, Quarter, Month, WeekOfYear,
        DayOfMonth, DayOfWeek, HourOfDay, MinuteBucket, IsWeekend
    )
    VALUES (
        CAST(FORMAT(@CurrentDate, 'yyyyMMddHHmm') AS INT),
        @CurrentDate,
        YEAR(@CurrentDate),
        DATEPART(QUARTER, @CurrentDate),
        MONTH(@CurrentDate),
        DATEPART(WEEK, @CurrentDate),
        DAY(@CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        DATEPART(HOUR, @CurrentDate),
        (DATEPART(MINUTE, @CurrentDate) / 15) * 15,
        CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1,7) THEN 1 ELSE 0 END
    );
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate);
END;
```

### Step 3: Create ETL Stored Procedures

```sql
-- Incremental load procedure for warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    IF @LastLoadTime IS NULL
        SET @LastLoadTime = DATEADD(MINUTE, -30, GETDATE());
    
    -- Insert from staging/source table
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeoKey, ProductKey, OperationType,
        QuantityHandled, DwellTimeMinutes, PickCycleSeconds,
        PackCycleSeconds, StorageZone, OperatorID
    )
    SELECT 
        t.TimeKey,
        g.GeoKey,
        p.ProductKey,
        src.OperationType,
        src.Quantity,
        DATEDIFF(MINUTE, src.ReceiveTime, src.ShipTime),
        src.PickSeconds,
        src.PackSeconds,
        src.Zone,
        src.Operator
    FROM StagingWarehouseOps src
    INNER JOIN DimTime t ON FORMAT(src.OperationTime, 'yyyyMMddHHmm') = t.TimeKey
    INNER JOIN DimGeography g ON src.WarehouseID = g.NodeID
    INNER JOIN DimProductGravity p ON src.SKU = p.SKU
    WHERE src.OperationTime > @LastLoadTime;
    
    -- Update gravity scores based on new pick frequency
    UPDATE p
    SET 
        PickFrequencyRank = ranked.NewRank,
        GravityScore = (ranked.NewRank * 0.4) + (p.UnitValue * 0.3) + 
                       (CASE p.Fragility WHEN 'High' THEN 30 WHEN 'Medium' THEN 20 ELSE 10 END)
    FROM DimProductGravity p
    INNER JOIN (
        SELECT 
            ProductKey,
            ROW_NUMBER() OVER (ORDER BY COUNT(*) DESC) AS NewRank
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
            AND TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
        GROUP BY ProductKey
    ) ranked ON p.ProductKey = ranked.ProductKey;
END;
GO

-- Fleet maintenance priority calculation
CREATE PROCEDURE sp_CalculateFleetMaintenancePriority
AS
BEGIN
    -- Example: Create a maintenance queue based on telemetry + revenue impact
    SELECT 
        f.VehicleID,
        COUNT(DISTINCT f.TripKey) AS TripCount,
        SUM(f.IdleTimeMinutes) AS TotalIdleMinutes,
        AVG(f.FuelConsumedLiters / NULLIF(f.DistanceKM, 0)) AS AvgFuelEfficiency,
        SUM(wo.QuantityHandled * p.UnitValue) AS TotalCargoValue,
        -- Priority score: higher idle time + high cargo value = higher priority
        (SUM(f.IdleTimeMinutes) * 0.5) + (SUM(wo.QuantityHandled * p.UnitValue) * 0.0001) AS PriorityScore
    FROM FactFleetTrips f
    LEFT JOIN FactWarehouseOperations wo ON f.TimeKey = wo.TimeKey
    LEFT JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE f.TimeKey >= CAST(FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY f.VehicleID
    ORDER BY PriorityScore DESC;
END;
GO
```

### Step 4: Configure Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Update connection parameters:
   - Server: Your SQL Server instance
   - Database: `LogiFleetPulse`
3. Enter credentials (Windows Auth or SQL Auth)
4. The template auto-detects fact/dimension tables and builds relationships

### Step 5: Set Up Data Source Connections

Create a configuration file for external data sources:

```json
{
  "wms_connection": {
    "type": "sql",
    "server": "${WMS_SERVER}",
    "database": "${WMS_DATABASE}",
    "auth": "integrated"
  },
  "telematics_api": {
    "endpoint": "${TELEMATICS_API_URL}",
    "auth_token": "${TELEMATICS_API_KEY}",
    "refresh_interval_minutes": 15
  },
  "weather_api": {
    "endpoint": "https://api.weatherprovider.com/v1/forecast",
    "api_key": "${WEATHER_API_KEY}"
  },
  "alert_config": {
    "smtp_server": "${SMTP_SERVER}",
    "smtp_port": 587,
    "from_email": "${ALERT_FROM_EMAIL}",
    "recipients": ["logistics@company.com"]
  }
}
```

## Key SQL Queries and DAX Measures

### Cross-Fact KPI: Dwell Time vs Fleet Idle Time

```sql
-- Identify SKUs with high warehouse dwell time AND associated fleet idle time
SELECT 
    p.SKU,
    p.ProductName,
    AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwellMinutes,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    COUNT(DISTINCT wo.OperationKey) AS WarehouseOps,
    COUNT(DISTINCT ft.TripKey) AS FleetTrips
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN FactFleetTrips ft ON wo.TimeKey = ft.TimeKey
WHERE wo.OperationType = 'Shipping'
    AND wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
GROUP BY p.SKU, p.ProductName
HAVING AVG(wo.DwellTimeMinutes) > 120 -- More than 2 hours dwell
    AND AVG(ft.IdleTimeMinutes) > 45 -- More than 45 min fleet idle
ORDER BY AvgWarehouseDwellMinutes DESC;
```

### Warehouse Gravity Zone Recommendations

```sql
-- Recommend zone reassignments based on gravity scores
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    wo.StorageZone AS CurrentZone,
    CASE 
        WHEN p.GravityScore >= 80 THEN 'Zone-A (High Velocity)'
        WHEN p.GravityScore >= 50 THEN 'Zone-B (Medium Velocity)'
        ELSE 'Zone-C (Low Velocity)'
    END AS RecommendedZone,
    COUNT(*) AS PickCount,
    AVG(wo.PickCycleSeconds) AS AvgPickSeconds
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationType = 'Picking'
    AND wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
GROUP BY p.SKU, p.ProductName, p.GravityScore, wo.StorageZone
HAVING wo.StorageZone <> CASE 
    WHEN p.GravityScore >= 80 THEN 'Zone-A (High Velocity)'
    WHEN p.GravityScore >= 50 THEN 'Zone-B (Medium Velocity)'
    ELSE 'Zone-C (Low Velocity)'
END
ORDER BY p.GravityScore DESC;
```

### Power BI DAX Measures

```dax
// Total Warehouse Dwell Time (hours)
TotalDwellTimeHours = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeMinutes] / 60
)

// Fleet Fuel Efficiency (km per liter)
FleetFuelEfficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKM]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)

// Cross-Fact: Dwell Cost Per SKU
DwellCostPerSKU = 
VAR DwellHours = SUMX(FactWarehouseOperations, [DwellTimeMinutes] / 60)
VAR HourlyCost = 5 // $ per hour of storage
RETURN DwellHours * HourlyCost

// Predictive Bottleneck Index (simplified)
BottleneckIndex = 
VAR HighDwellCount = CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FactWarehouseOperations[DwellTimeMinutes] > 180
)
VAR HighIdleCount = CALCULATE(
    COUNTROWS(FactFleetTrips),
    FactFleetTrips[IdleTimeMinutes] > 60
)
RETURN (HighDwellCount * 0.6) + (HighIdleCount * 0.4)

// Time Intelligence: Dwell Time Same Period Last Year
DwellTimeSPLY = 
CALCULATE(
    [TotalDwellTimeHours],
    DATEADD(DimTime[FullDateTime], -1, YEAR)
)
```

## Configuration Patterns

### Row-Level Security Setup

```sql
-- Create security table
CREATE TABLE SecurityRoleMapping (
    RoleID INT PRIMARY KEY IDENTITY(1,1),
    RoleName VARCHAR(100),
    UserEmail VARCHAR(200),
    AllowedRegions VARCHAR(MAX), -- Comma-separated list
    AllowedCategories VARCHAR(MAX)
);

-- Example role assignment
INSERT INTO SecurityRoleMapping (RoleName, UserEmail, AllowedRegions, AllowedCategories)
VALUES 
    ('Regional Manager - EMEA', 'manager.emea@company.com', 'Europe', NULL),
    ('Category Lead - Perishables', 'lead.perishables@company.com', NULL, 'Food,Dairy');
```

Power BI RLS DAX:

```dax
// Create role: Regional Access
[Region] IN { SUBSTITUTE(USERPRINCIPALNAME(), "@company.com", "") }

// Reference security table (if imported)
VAR UserRegions = 
    LOOKUPVALUE(
        SecurityRoleMapping[AllowedRegions],
        SecurityRoleMapping[UserEmail],
        USERPRINCIPALNAME()
    )
RETURN 
    CONTAINSSTRING(UserRegions, DimGeography[Region])
```

### Scheduled Alerts with SQL Agent

```sql
-- Create alert stored procedure
CREATE PROCEDURE sp_AlertHighDwellTime
AS
BEGIN
    DECLARE @AlertBody NVARCHAR(MAX);
    
    SELECT @AlertBody = STRING_AGG(
        CONCAT(
            'SKU: ', p.SKU, 
            ' | Product: ', p.ProductName,
            ' | Avg Dwell: ', CAST(AVG(wo.DwellTimeMinutes) AS VARCHAR), ' min',
            ' | Zone: ', wo.StorageZone
        ), 
        CHAR(13) + CHAR(10)
    )
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.TimeKey >= CAST(FORMAT(DATEADD(HOUR, -4, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY p.SKU, p.ProductName, wo.StorageZone
    HAVING AVG(wo.DwellTimeMinutes) > 240; -- Alert if > 4 hours
    
    IF @AlertBody IS NOT NULL
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'logistics@company.com',
            @subject = 'ALERT: High Warehouse Dwell Time Detected',
            @body = @AlertBody;
    END;
END;
GO

-- Schedule via SQL Server Agent (run every 4 hours)
```

## Common Integration Patterns

### Ingest Telematics Data via REST API

```sql
-- Create external table for JSON API data (SQL Server 2016+)
-- Requires PolyBase or OPENROWSET with Azure SQL

-- Alternative: Use SSIS package or Python script
-- Python example for periodic ingestion:
```

Python ETL script (`ingest_telematics.py`):

```python
import requests
import pyodbc
import os
from datetime import datetime

# Configuration from environment
API_ENDPOINT = os.getenv('TELEMATICS_API_URL')
API_KEY = os.getenv('TELEMATICS_API_KEY')
SQL_CONN_STR = os.getenv('SQL_CONNECTION_STRING')

def fetch_telematics_data():
    headers = {'Authorization': f'Bearer {API_KEY}'}
    response = requests.get(f'{API_ENDPOINT}/trips', headers=headers)
    response.raise_for_status()
    return response.json()

def insert_fleet_trips(trips_data):
    conn = pyodbc.connect(SQL_CONN_STR)
    cursor = conn.cursor()
    
    for trip in trips_data:
        # Match time dimension
        trip_time = datetime.fromisoformat(trip['start_time'])
        time_key = int(trip_time.strftime('%Y%m%d%H%M'))
        
        # Lookup geography keys
        cursor.execute(
            "SELECT GeoKey FROM DimGeography WHERE NodeID = ?", 
            trip['origin_node_id']
        )
        origin_key = cursor.fetchone()[0]
        
        cursor.execute(
            "SELECT GeoKey FROM DimGeography WHERE NodeID = ?", 
            trip['destination_node_id']
        )
        dest_key = cursor.fetchone()[0]
        
        # Insert fact record
        cursor.execute("""
            INSERT INTO FactFleetTrips 
            (TimeKey, OriginGeoKey, DestGeoKey, VehicleID, DriverID, 
             DistanceKM, FuelConsumedLiters, IdleTimeMinutes, TripDurationMinutes, LoadWeightKG)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            time_key, origin_key, dest_key, trip['vehicle_id'], trip['driver_id'],
            trip['distance_km'], trip['fuel_liters'], trip['idle_minutes'],
            trip['duration_minutes'], trip['load_weight_kg']
        ))
    
    conn.commit()
    conn.close()

if __name__ == '__main__':
    data = fetch_telematics_data()
    insert_fleet_trips(data['trips'])
    print(f"Loaded {len(data['trips'])} fleet trips")
```

### Power BI Incremental Refresh Configuration

In Power BI Desktop:

1. Add parameters `RangeStart` and `RangeEnd` (DateTime)
2. Filter fact tables on `DimTime[FullDateTime]`:
   ```
   = Table.SelectRows(
       FactWarehouseOperations, 
       each [TimeKey] >= Number.From(Text.From(RangeStart, "yyyyMMddHHmm")) 
       and [TimeKey] < Number.From(Text.From(RangeEnd, "yyyyMMddHHmm"))
   )
   ```
3. Configure incremental refresh policy in Power BI Service:
   - Archive data: 3 years
   - Incremental refresh: Last 7 days
   - Refresh frequency: Every 15 minutes

## Troubleshooting

### Issue: Slow Dashboard Refresh

**Solution:**
- Verify columnstore indexes on large fact tables:
  ```sql
  CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWH_Analytics
  ON FactWarehouseOperations (TimeKey, ProductKey, QuantityHandled, DwellTimeMinutes);
  ```
- Enable query result caching in Power BI
- Partition fact tables by year/quarter:
  ```sql
  ALTER TABLE FactWarehouseOperations
  SWITCH PARTITION $PARTITION.TimePartitionFunc(202601) TO StagingPartition;
  ```

### Issue: Missing Foreign Key Relationships

**Solution:**
- Ensure dimension keys exist before inserting facts:
  ```sql
  -- Validate before inserting
  INSERT INTO FactWarehouseOperations (...)
  SELECT ...
  FROM StagingTable s
  WHERE EXISTS (SELECT 1 FROM DimTime WHERE TimeKey = s.TimeKey)
    AND EXISTS (SELECT 1 FROM DimGeography WHERE GeoKey = s.GeoKey)
    AND EXISTS (SELECT 1 FROM DimProductGravity WHERE ProductKey = s.ProductKey);
  ```

### Issue: Gravity Scores Not Updating

**Solution:**
- Schedule `sp_LoadWarehouseOperations` to run every 30 minutes via SQL Agent
- Verify pick frequency calculation:
  ```sql
  SELECT ProductKey, COUNT(*) AS PickCount
  FROM FactWarehouseOperations
  WHERE OperationType = 'Picking'
  GROUP BY ProductKey
  ORDER BY PickCount DESC;
  ```

### Issue: Power BI RLS Not Working

**Solution:**
- Test RLS in Power BI Desktop: Modeling tab → View as Roles
- Verify user emails match USERPRINCIPALNAME() format
- Check security table has correct mappings:
  ```sql
  SELECT * FROM SecurityRoleMapping WHERE UserEmail = 'youruser@company.com';
  ```

## Advanced Usage: Temporal Elasticity Simulation

```sql
-- Scenario: What if warehouse capacity increases from 80% to 95%?
WITH CurrentCapacity AS (
    SELECT 
        AVG(CAST(QuantityHandled AS FLOAT) / 
            (SELECT MAX(QuantityHandled) FROM FactWarehouseOperations)) AS UtilizationPct
    FROM FactWarehouseOperations
    WHERE TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
),
SimulatedLoad AS (
    SELECT 
        ft.VehicleID,
        ft.TripKey,
        ft.IdleTimeMinutes,
        -- Simulate 15% increase in load due to higher warehouse throughput
        ft.LoadWeightKG * 1.15 AS SimulatedLoadKG,
        -- Estimate fuel increase (linear approximation)
        ft.FuelConsumedLiters * (1 + (0.15 * 0.3)) AS SimulatedFuelLiters
    FROM FactFleetTrips ft
    WHERE ft.TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
)
SELECT 
    AVG(IdleTimeMinutes) AS AvgIdleMinutes,
    AVG(SimulatedLoadKG) AS AvgLoadKG,
    AVG(SimulatedFuelLiters) AS AvgFuelLiters,
    SUM(SimulatedFuelLiters) - SUM(ft.FuelConsumedLiters) AS AdditionalFuelCost
FROM SimulatedLoad sl
INNER JOIN FactFleetTrips ft ON sl.TripKey = ft.TripKey;
```

## Best Practices

1. **Indexing Strategy:**
   - Clustered index on TimeKey for fact tables
   - Nonclustered indexes on frequently joined foreign keys
   - Columnstore indexes for analytical queries

2. **Data Quality:**
   - Implement CHECK constraints on fact tables (e.g., `QuantityHandled >= 0`)
   - Use triggers to audit critical dimension changes
   - Schedule data validation procedures daily

3. **Security:**
   - Never store API keys in SQL tables; use environment variables or Azure Key Vault
   - Implement RLS for all user-facing Power BI reports
   - Encrypt connection strings in configuration files

4. **Performance:**
   - Partition large fact tables by date range
   - Use stored procedures for complex ETL logic
   - Enable Power BI aggregation tables for frequently accessed metrics

5. **Documentation:**
   - Annotate all DAX measures with business logic comments
   - Maintain data dictionary mapping source systems to warehouse tables
   - Document security role assignments and refresh schedules
