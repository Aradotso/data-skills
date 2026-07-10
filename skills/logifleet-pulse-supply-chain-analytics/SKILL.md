---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server data warehouse and Power BI dashboards for real-time logistics, fleet, and warehouse intelligence
triggers:
  - "set up logifleet pulse supply chain analytics"
  - "deploy logistics data warehouse with power bi"
  - "configure warehouse and fleet tracking dashboards"
  - "implement multi-fact star schema for logistics"
  - "create supply chain kpi reports"
  - "integrate warehouse and fleet telemetry data"
  - "build logistics intelligence platform"
  - "set up cross-modal supply chain analytics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing solution combining MS SQL Server and Power BI for real-time logistics intelligence. It implements a multi-fact star schema that unifies:

- **Warehouse operations** (receiving, putaway, picking, packing, shipping, dwell time)
- **Fleet telemetry** (GPS, fuel consumption, idle time, maintenance events)
- **Inventory management** (aging curves, gravity zones, turnover rates)
- **Supplier reliability** (lead times, defect rates, compliance)
- **External signals** (weather, traffic, demand patterns)

The platform provides predictive bottleneck detection, adaptive fleet triage, and cross-fact KPI harmonization through a unified semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- Access to data sources (WMS, TMS, ERP, telemetry APIs)

### Step 1: Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the schema creation script
-- Execute the provided SQL scripts in this order:
-- 1. 01_create_dimensions.sql
-- 2. 02_create_facts.sql
-- 3. 03_create_relationships.sql
-- 4. 04_create_views.sql
-- 5. 05_create_stored_procedures.sql
```

### Step 3: Configure Data Sources

Create a configuration file for connection strings:

```json
{
  "connections": {
    "sql_server": "Server=${SQL_SERVER};Database=LogiFleetPulse;User Id=${SQL_USER};Password=${SQL_PASSWORD};",
    "wms_api": "${WMS_API_ENDPOINT}",
    "fleet_telemetry": "${FLEET_API_ENDPOINT}",
    "weather_api": "${WEATHER_API_KEY}"
  },
  "refresh_intervals": {
    "warehouse_ops": 15,
    "fleet_trips": 15,
    "inventory": 60
  }
}
```

### Step 4: Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI Template
3. Select `LogiFleet_Pulse_Master.pbit`
4. Enter SQL Server connection details when prompted
5. Configure row-level security roles

## Core Data Model Structure

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2,
    DateKey INT,
    TimeSlot TIME,
    HourOfDay INT,
    DayOfWeek NVARCHAR(10),
    WeekOfYear INT,
    MonthName NVARCHAR(10),
    Quarter INT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT
);

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationCode NVARCHAR(50),
    LocationName NVARCHAR(200),
    LocationType NVARCHAR(50), -- Warehouse, Route Node, Depot
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    RegionKey INT,
    RegionName NVARCHAR(100),
    CountryCode NVARCHAR(3),
    CountryName NVARCHAR(100),
    Continent NVARCHAR(50)
);

-- DimProductGravity: Product classification with gravity score
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU NVARCHAR(50),
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * fragility
    VelocityClass NVARCHAR(20), -- Fast, Medium, Slow
    ValueTier NVARCHAR(20), -- High, Medium, Low
    IsFragile BIT,
    RequiresColdStorage BIT,
    OptimalZoneID NVARCHAR(50)
);

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY,
    SupplierCode NVARCHAR(50),
    SupplierName NVARCHAR(200),
    LeadTimeDaysAvg DECIMAL(5,1),
    LeadTimeVariance DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,2),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2),
    ReliabilityTier NVARCHAR(20) -- Gold, Silver, Bronze, Watch
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Detailed warehouse activity
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType NVARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    QuantityHandled INT,
    CycleTimeMinutes DECIMAL(8,2),
    DwellTimeHours DECIMAL(8,2),
    StorageZoneID NVARCHAR(50),
    OperatorID NVARCHAR(50),
    EquipmentID NVARCHAR(50),
    BatchNumber NVARCHAR(100),
    OrderID NVARCHAR(100),
    IsAnomalous BIT,
    AnomalyReason NVARCHAR(500)
);

-- FactFleetTrips: Vehicle journey and telemetry
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY,
    TimeKeyStart INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    TimeKeyEnd INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID NVARCHAR(50),
    DriverID NVARCHAR(50),
    RouteID NVARCHAR(50),
    DistanceKM DECIMAL(8,2),
    DurationMinutes DECIMAL(8,2),
    IdleTimeMinutes DECIMAL(8,2),
    FuelConsumedLiters DECIMAL(8,2),
    AverageSpeedKMH DECIMAL(5,2),
    LoadWeightKG DECIMAL(10,2),
    DeliveryCount INT,
    OnTimeDeliveries INT,
    WeatherCondition NVARCHAR(50),
    TrafficDelayMinutes DECIMAL(6,2),
    MaintenanceAlerts INT
);

-- FactCrossDock: Direct transfer operations
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY,
    TimeKeyInbound INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    TimeKeyOutbound INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundVehicleID NVARCHAR(50),
    OutboundVehicleID NVARCHAR(50),
    QuantityTransferred INT,
    TransferTimeMinutes DECIMAL(8,2),
    DockBayID NVARCHAR(20)
);
```

## Key SQL Queries & Patterns

### Cross-Fact KPI: Dwell Time vs Fleet Idle Cost

```sql
-- Calculate correlation between warehouse dwell time and fleet idling
WITH WarehouseDwell AS (
    SELECT 
        dt.DateKey,
        dg.RegionName,
        AVG(fwo.DwellTimeHours) AS AvgDwellHours,
        SUM(fwo.QuantityHandled) AS TotalUnits
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
    WHERE fwo.OperationType IN ('Putaway', 'Picking')
    GROUP BY dt.DateKey, dg.RegionName
),
FleetIdle AS (
    SELECT 
        dt.DateKey,
        dg.RegionName,
        AVG(fft.IdleTimeMinutes) AS AvgIdleMinutes,
        SUM(fft.IdleTimeMinutes * 0.5) AS EstimatedIdleCostUSD -- $0.50/minute
    FROM FactFleetTrips fft
    INNER JOIN DimTime dt ON fft.TimeKeyStart = dt.TimeKey
    INNER JOIN DimGeography dg ON fft.OriginGeographyKey = dg.GeographyKey
    GROUP BY dt.DateKey, dg.RegionName
)
SELECT 
    wd.DateKey,
    wd.RegionName,
    wd.AvgDwellHours,
    fi.AvgIdleMinutes,
    fi.EstimatedIdleCostUSD,
    -- Correlation flag: high dwell + high idle = potential bottleneck
    CASE 
        WHEN wd.AvgDwellHours > 48 AND fi.AvgIdleMinutes > 45 THEN 'Critical'
        WHEN wd.AvgDwellHours > 24 AND fi.AvgIdleMinutes > 30 THEN 'Warning'
        ELSE 'Normal'
    END AS BottleneckStatus
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.DateKey = fi.DateKey AND wd.RegionName = fi.RegionName
ORDER BY fi.EstimatedIdleCostUSD DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products in suboptimal storage zones
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    dp.OptimalZoneID AS RecommendedZone,
    fwo.StorageZoneID AS CurrentZone,
    COUNT(*) AS PickOperations,
    AVG(fwo.CycleTimeMinutes) AS AvgPickTime,
    -- Calculate potential time savings
    CASE 
        WHEN dp.OptimalZoneID != fwo.StorageZoneID 
        THEN AVG(fwo.CycleTimeMinutes) * 0.25 -- 25% improvement estimate
        ELSE 0 
    END AS EstimatedTimeSavingsMinutes
FROM FactWarehouseOperations fwo
INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
WHERE fwo.OperationType = 'Picking'
    AND fwo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateKey = CONVERT(INT, CONVERT(VARCHAR(8), DATEADD(DAY, -30, GETDATE()), 112)))
GROUP BY dp.SKU, dp.ProductName, dp.GravityScore, dp.OptimalZoneID, fwo.StorageZoneID
HAVING dp.OptimalZoneID != fwo.StorageZoneID
    AND COUNT(*) > 50 -- Focus on high-volume items
ORDER BY EstimatedTimeSavingsMinutes DESC;
```

### Predictive Fleet Maintenance Triage

```sql
-- Score vehicles by maintenance priority based on telemetry and load value
WITH VehicleMetrics AS (
    SELECT 
        VehicleID,
        COUNT(*) AS TripCount,
        SUM(DistanceKM) AS TotalDistanceKM,
        AVG(FuelConsumedLiters / NULLIF(DistanceKM, 0)) AS AvgFuelEfficiency,
        SUM(MaintenanceAlerts) AS TotalAlerts,
        AVG(LoadWeightKG) AS AvgLoadWeight
    FROM FactFleetTrips
    WHERE TimeKeyStart >= (SELECT TimeKey FROM DimTime WHERE DateKey = CONVERT(INT, CONVERT(VARCHAR(8), DATEADD(DAY, -90, GETDATE()), 112)))
    GROUP BY VehicleID
),
HighValueLoads AS (
    SELECT 
        fft.VehicleID,
        COUNT(DISTINCT fft.TripKey) AS HighValueTripCount
    FROM FactFleetTrips fft
    INNER JOIN FactWarehouseOperations fwo ON fft.RouteID = fwo.OrderID
    INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    WHERE dp.ValueTier = 'High'
    GROUP BY fft.VehicleID
)
SELECT 
    vm.VehicleID,
    vm.TotalDistanceKM,
    vm.TotalAlerts,
    vm.AvgFuelEfficiency,
    ISNULL(hvl.HighValueTripCount, 0) AS HighValueTrips,
    -- Priority score (higher = more urgent)
    (vm.TotalAlerts * 10) + 
    (CASE WHEN vm.AvgFuelEfficiency > 12 THEN 20 ELSE 0 END) + -- Poor fuel efficiency
    (ISNULL(hvl.HighValueTripCount, 0) * 5) AS MaintenancePriorityScore
FROM VehicleMetrics vm
LEFT JOIN HighValueLoads hvl ON vm.VehicleID = hvl.VehicleID
WHERE vm.TotalAlerts > 0 OR vm.AvgFuelEfficiency > 12
ORDER BY MaintenancePriorityScore DESC;
```

### Time-Phased Demand Elasticity

```sql
-- Analyze demand patterns across 15-minute time slots
SELECT 
    dt.HourOfDay,
    dt.DayOfWeek,
    dp.Category,
    COUNT(*) AS OrderCount,
    SUM(fwo.QuantityHandled) AS TotalUnits,
    AVG(fwo.CycleTimeMinutes) AS AvgProcessingTime,
    -- Elasticity indicator: variance from daily average
    (SUM(fwo.QuantityHandled) - AVG(SUM(fwo.QuantityHandled)) OVER (PARTITION BY dp.Category)) / 
        NULLIF(STDEV(SUM(fwo.QuantityHandled)) OVER (PARTITION BY dp.Category), 0) AS ElasticityZ
FROM FactWarehouseOperations fwo
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
WHERE fwo.OperationType IN ('Picking', 'Packing')
    AND dt.DateKey >= CONVERT(INT, CONVERT(VARCHAR(8), DATEADD(DAY, -60, GETDATE()), 112))
GROUP BY dt.HourOfDay, dt.DayOfWeek, dp.Category
ORDER BY dp.Category, dt.DayOfWeek, dt.HourOfDay;
```

## Stored Procedures for ETL

### Incremental Load for Warehouse Operations

```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Merge from staging table (populated by external ETL process)
    MERGE FactWarehouseOperations AS target
    USING (
        SELECT 
            dt.TimeKey,
            dg.GeographyKey,
            dp.ProductKey,
            ds.SupplierKey,
            stg.OperationType,
            stg.QuantityHandled,
            stg.CycleTimeMinutes,
            stg.DwellTimeHours,
            stg.StorageZoneID,
            stg.OperatorID,
            stg.EquipmentID,
            stg.BatchNumber,
            stg.OrderID,
            -- Anomaly detection: cycle time > 3 std devs from mean
            CASE 
                WHEN stg.CycleTimeMinutes > (
                    SELECT AVG(CycleTimeMinutes) + 3*STDEV(CycleTimeMinutes) 
                    FROM StagingWarehouseOps 
                    WHERE OperationType = stg.OperationType
                ) THEN 1 
                ELSE 0 
            END AS IsAnomalous,
            CASE 
                WHEN stg.CycleTimeMinutes > (SELECT AVG(CycleTimeMinutes) + 3*STDEV(CycleTimeMinutes) FROM StagingWarehouseOps WHERE OperationType = stg.OperationType) 
                THEN 'Excessive cycle time detected'
                ELSE NULL 
            END AS AnomalyReason
        FROM StagingWarehouseOps stg
        INNER JOIN DimTime dt ON CAST(stg.OperationDateTime AS DATETIME2) BETWEEN dt.FullDateTime AND DATEADD(MINUTE, 15, dt.FullDateTime)
        INNER JOIN DimGeography dg ON stg.LocationCode = dg.LocationCode
        INNER JOIN DimProductGravity dp ON stg.SKU = dp.SKU
        LEFT JOIN DimSupplierReliability ds ON stg.SupplierCode = ds.SupplierCode
        WHERE stg.OperationDateTime > @LastLoadTime
    ) AS source
    ON 1=0 -- Always insert (append-only fact table)
    WHEN NOT MATCHED THEN
        INSERT (TimeKey, GeographyKey, ProductKey, SupplierKey, OperationType, 
                QuantityHandled, CycleTimeMinutes, DwellTimeHours, StorageZoneID, 
                OperatorID, EquipmentID, BatchNumber, OrderID, IsAnomalous, AnomalyReason)
        VALUES (source.TimeKey, source.GeographyKey, source.ProductKey, source.SupplierKey, 
                source.OperationType, source.QuantityHandled, source.CycleTimeMinutes, 
                source.DwellTimeHours, source.StorageZoneID, source.OperatorID, 
                source.EquipmentID, source.BatchNumber, source.OrderID, 
                source.IsAnomalous, source.AnomalyReason);
    
    -- Update last load timestamp
    UPDATE ETLControl 
    SET LastLoadTime = GETDATE() 
    WHERE TableName = 'FactWarehouseOperations';
END;
```

### Automated Alerting Procedure

```sql
CREATE PROCEDURE usp_CheckKPIThresholds
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Check for fleet idling threshold breach
    INSERT INTO AlertLog (AlertType, Severity, Message, CreatedAt)
    SELECT 
        'Fleet Idling',
        'Warning',
        'Vehicle ' + VehicleID + ' exceeded 15% idle time on route ' + RouteID,
        GETDATE()
    FROM FactFleetTrips
    WHERE TimeKeyStart >= (SELECT MAX(TimeKey) - 96 FROM DimTime) -- Last 24 hours
        AND (IdleTimeMinutes / NULLIF(DurationMinutes, 0)) > 0.15
        AND VehicleID NOT IN (SELECT VehicleID FROM AlertLog WHERE AlertType = 'Fleet Idling' AND CreatedAt > DATEADD(HOUR, -4, GETDATE()));
    
    -- Check for warehouse dwell time anomalies
    INSERT INTO AlertLog (AlertType, Severity, Message, CreatedAt)
    SELECT 
        'Warehouse Dwell',
        'Critical',
        'SKU ' + dp.SKU + ' in zone ' + fwo.StorageZoneID + ' has dwell time of ' + CAST(fwo.DwellTimeHours AS VARCHAR) + ' hours',
        GETDATE()
    FROM FactWarehouseOperations fwo
    INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    WHERE fwo.DwellTimeHours > 72
        AND fwo.OperationType = 'Putaway'
        AND fwo.TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime)
        AND NOT EXISTS (
            SELECT 1 FROM AlertLog 
            WHERE AlertType = 'Warehouse Dwell' 
                AND Message LIKE '%' + dp.SKU + '%' 
                AND CreatedAt > DATEADD(HOUR, -8, GETDATE())
        );
END;
```

## Power BI Configuration

### Connecting to SQL Server

In Power BI Desktop:

1. Home → Get Data → SQL Server
2. Server: `${SQL_SERVER}`
3. Database: `LogiFleetPulse`
4. Data Connectivity mode: DirectQuery (for real-time) or Import (for performance)
5. Advanced options:
   - Command timeout: 30 minutes
   - SQL statement: Leave blank (import full schema)

### Key DAX Measures

```dax
// Fleet Utilization Percentage
Fleet Utilization % = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

// Warehouse Throughput (units per hour)
Warehouse Throughput = 
DIVIDE(
    SUM(FactWarehouseOperations[QuantityHandled]),
    SUM(FactWarehouseOperations[CycleTimeMinutes]) / 60,
    0
)

// Predictive Bottleneck Index (0-100 scale)
Bottleneck Index = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR AvgIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR DwellZ = DIVIDE(AvgDwell - 24, 12, 0) // Z-score normalized to 24h mean, 12h std
VAR IdleZ = DIVIDE(AvgIdle - 30, 15, 0)
RETURN 
    MIN(100, MAX(0, (DwellZ + IdleZ) * 25 + 50))

// On-Time Delivery Rate
OTD Rate % = 
DIVIDE(
    SUM(FactFleetTrips[OnTimeDeliveries]),
    SUM(FactFleetTrips[DeliveryCount]),
    0
) * 100

// Cost Per KM (including idle cost)
Cost Per KM = 
VAR FuelCost = SUM(FactFleetTrips[FuelConsumedLiters]) * 1.5 // $1.50/liter
VAR IdleCost = SUM(FactFleetTrips[IdleTimeMinutes]) * 0.5 // $0.50/minute
VAR TotalDistance = SUM(FactFleetTrips[DistanceKM])
RETURN DIVIDE(FuelCost + IdleCost, TotalDistance, 0)
```

### Row-Level Security (RLS)

```dax
// Create role: Regional Managers
[RegionName] = USERNAME()

// Create role: Warehouse Supervisors
[LocationCode] IN (
    SELECTCOLUMNS(
        FILTER(UserPermissions, UserPermissions[UserEmail] = USERNAME()),
        "LocationCode", UserPermissions[LocationCode]
    )
)

// Create role: Executive (no filter - see all)
1 = 1
```

## Common Troubleshooting

### Performance Issues

**Problem**: Queries taking >10 seconds in Power BI

**Solution**:
```sql
-- Add columnstore indexes to fact tables
CREATE COLUMNSTORE INDEX IX_FactWarehouseOps_CS 
ON FactWarehouseOperations (TimeKey, GeographyKey, ProductKey);

CREATE COLUMNSTORE INDEX IX_FactFleetTrips_CS 
ON FactFleetTrips (TimeKeyStart, OriginGeographyKey, DestinationGeographyKey);

-- Partition large fact tables by date
ALTER TABLE FactWarehouseOperations
ADD CONSTRAINT PK_FactWarehouseOps 
PRIMARY KEY CLUSTERED (OperationKey, TimeKey);

-- Create partition function and scheme
CREATE PARTITION FUNCTION PF_TimeKey (INT)
AS RANGE RIGHT FOR VALUES 
(20260101, 20260201, 20260301, 20260401, 20260501, 20260601, 
 20260701, 20260801, 20260901, 20261001, 20261101, 20261201);
```

### Data Refresh Failures

**Problem**: Power BI Gateway timeout

**Solution**:
- Switch to DirectQuery mode for large tables
- Create aggregated summary views:

```sql
CREATE VIEW vw_DailyFleetSummary AS
SELECT 
    dt.DateKey,
    fft.VehicleID,
    dg.RegionName,
    SUM(fft.DistanceKM) AS TotalDistanceKM,
    AVG(fft.IdleTimeMinutes) AS AvgIdleMinutes,
    SUM(fft.FuelConsumedLiters) AS TotalFuelLiters,
    COUNT(*) AS TripCount
FROM FactFleetTrips fft
INNER JOIN DimTime dt ON fft.TimeKeyStart = dt.TimeKey
INNER JOIN DimGeography dg ON fft.OriginGeographyKey = dg.GeographyKey
GROUP BY dt.DateKey, fft.VehicleID, dg.RegionName;
```

### Missing Dimension Relationships

**Problem**: Products showing "Blank" in reports

**Solution**:
```sql
-- Identify orphaned fact records
SELECT DISTINCT fwo.ProductKey
FROM FactWarehouseOperations fwo
LEFT JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
WHERE dp.ProductKey IS NULL;

-- Create default dimension record for missing keys
INSERT INTO DimProductGravity (ProductKey, SKU, ProductName, GravityScore, VelocityClass, ValueTier)
VALUES (-1, 'UNKNOWN', 'Unknown Product', 0, 'Unclassified', 'Unknown', 0, 0, NULL);
```

## Advanced Patterns

### Slowly Changing Dimension (Type 2) for Suppliers

```sql
-- Capture supplier reliability changes over time
CREATE TABLE DimSupplierReliability_SCD (
    SupplierSCDKey INT PRIMARY KEY IDENTITY,
    SupplierKey INT,
    SupplierCode NVARCHAR(50),
    SupplierName NVARCHAR(200),
    LeadTimeDaysAvg DECIMAL(5,1),
    DefectRatePercent DECIMAL(5,2),
    ReliabilityTier NVARCHAR(20),
    ValidFrom DATETIME2,
    ValidTo DATETIME2,
    IsCurrent BIT
);

-- Update procedure for SCD Type 2
CREATE PROCEDURE usp_UpdateSupplierSCD
    @SupplierKey INT,
    @NewLeadTime DECIMAL(5,1),
    @NewDefectRate DECIMAL(5,2)
AS
BEGIN
    -- Expire current record
    UPDATE DimSupplierReliability_SCD
    SET ValidTo = GETDATE(), IsCurrent = 0
    WHERE SupplierKey = @SupplierKey AND IsCurrent = 1;
    
    -- Insert new record
    INSERT INTO DimSupplierReliability_SCD 
    (SupplierKey, SupplierCode, SupplierName, LeadTimeDaysAvg, DefectRatePercent, 
     ReliabilityTier, ValidFrom, ValidTo, IsCurrent)
    SELECT 
        SupplierKey, SupplierCode, SupplierName, @NewLeadTime, @NewDefectRate,
        CASE 
            WHEN @NewDefectRate < 1 AND @NewLeadTime < 7 THEN 'Gold'
            WHEN @NewDefectRate < 3 AND @NewLeadTime < 14 THEN 'Silver'
            WHEN @NewDefectRate < 5 AND @NewLeadTime < 21 THEN 'Bronze'
            ELSE 'Watch'
        END,
        GETDATE(), '9999-12-31', 1
    FROM DimSupplierReliability WHERE SupplierKey = @SupplierKey;
END;
```

### External API Integration (Python)

```python
# Load weather data into staging table
import pyodbc
import requests
import os
from datetime import datetime, timedelta

# Configuration
SQL_SERVER = os.environ['SQL_SERVER']
SQL_USER = os.environ['SQL_USER']
SQL_PASSWORD = os.environ['SQL_PASSWORD']
WEATHER_API_KEY = os.environ['WEATHER_API_KEY']

# Connect to SQL Server
conn = pyodbc.connect(
    f'DRIVER={{ODBC Driver 17 for SQL Server}};'
    f'SERVER={SQL_SERVER};DATABASE=LogiFleetPulse;'
    f'UID={SQL_USER};PWD={SQL_PASSWORD}'
)
cursor = conn.cursor()

# Fetch active route nodes
cursor.execute("SELECT LocationCode, Latitude, Longitude FROM DimGeography WHERE LocationType = 'Route Node'")
locations = cursor.fetchall()

# Get weather for each location (last 24 hours)
for loc in locations:
    response = requests.get(
        f'https://api.weatherapi.com/v1/history.json',
        params={
            'key': WEATHER_API_KEY,
            'q': f'{loc.Latitude},{loc.Longitude}',
            'dt': (datetime.now() - timedelta(days=1)).strftime('%Y-%m-%d')
        }
    )
    
    if response.status_code == 200:
        weather_data = response.json()
        for hour in weather_data['forecast']['
