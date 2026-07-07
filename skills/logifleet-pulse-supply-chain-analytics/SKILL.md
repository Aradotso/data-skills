---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI supply chain analytics platform with multi-fact star schema for warehouse and fleet logistics intelligence
triggers:
  - "set up LogiFleet Pulse analytics platform"
  - "implement supply chain data warehouse with Power BI"
  - "create multi-fact star schema for logistics"
  - "build warehouse and fleet intelligence dashboard"
  - "configure LogiCore Analytics supply chain system"
  - "integrate logistics data warehouse with SQL Server"
  - "deploy real-time fleet and warehouse analytics"
  - "implement cross-modal supply chain KPI tracking"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is an advanced data warehousing and business intelligence platform that unifies warehouse operations, fleet telemetry, and supply chain metrics into a single semantic layer. Built on MS SQL Server with Power BI visualization, it provides:

- **Multi-fact star schema** linking warehouse operations, fleet trips, and cross-dock activities
- **Time-phased dimensions** with 15-minute granularity for real-time analytics
- **Cross-fact KPI harmonization** to correlate inventory metrics with fleet performance
- **Warehouse Gravity Zones™** for spatial optimization based on pick frequency and item value
- **Predictive bottleneck detection** using temporal elasticity modeling
- **Role-based dashboards** with row-level security

The platform is designed for 3PL operators, retail chains, distributors, and any organization managing complex logistics operations.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telematics APIs
- SQL Server Management Studio (SSMS) recommended

### Step 1: Deploy SQL Schema

```sql
-- Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create dimension tables first
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Hour TINYINT,
    DayOfWeek TINYINT,
    DayOfMonth TINYINT,
    Month TINYINT,
    Quarter TINYINT,
    Year INT,
    FiscalPeriod INT,
    IsWeekend BIT,
    TimeSlot15Min VARCHAR(5) -- e.g., "14:45"
)

CREATE CLUSTERED INDEX IX_DimTime_DateTime ON DimTime(FullDateTime)

CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    NodeID VARCHAR(50) UNIQUE NOT NULL,
    NodeName NVARCHAR(200),
    NodeType VARCHAR(20), -- 'Warehouse', 'Route', 'Customer', 'Supplier'
    Continent NVARCHAR(50),
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    City NVARCHAR(100),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    IsActive BIT DEFAULT 1
)

CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    Subcategory NVARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT,
    IsPerishable BIT,
    ShelfLifeDays INT,
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value / fragility
    VelocityTier VARCHAR(10), -- 'Fast', 'Medium', 'Slow'
    ValueTier VARCHAR(10), -- 'High', 'Medium', 'Low'
    LastRecalculated DATETIME2
)

CREATE TABLE DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierID VARCHAR(50) UNIQUE NOT NULL,
    SupplierName NVARCHAR(200),
    Country NVARCHAR(100),
    LeadTimeDaysAvg DECIMAL(5,1),
    LeadTimeDaysStdDev DECIMAL(5,2),
    DefectRatePct DECIMAL(5,2),
    OnTimeDeliveryPct DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityGrade VARCHAR(2), -- A+, A, B, C, D
    LastEvaluated DATETIME2
)

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    WarehouseKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(20), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationID VARCHAR(50) UNIQUE,
    Quantity INT,
    DurationMinutes DECIMAL(8,2),
    DwellHours DECIMAL(10,2), -- Time in storage before next movement
    StorageZone VARCHAR(50),
    AssignedGravityZone VARCHAR(20),
    EmployeeID VARCHAR(50),
    BatchNumber VARCHAR(50),
    CostUSD DECIMAL(12,2)
)

CREATE NONCLUSTERED INDEX IX_FactWH_Time ON FactWarehouseOperations(TimeKey)
CREATE NONCLUSTERED INDEX IX_FactWH_Product ON FactWarehouseOperations(ProductKey)
CREATE NONCLUSTERED INDEX IX_FactWH_Type ON FactWarehouseOperations(OperationType)

CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    StartTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    EndTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    TripID VARCHAR(50) UNIQUE,
    DistanceKm DECIMAL(10,2),
    DurationMinutes DECIMAL(10,2),
    IdleMinutes DECIMAL(8,2),
    FuelLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(12,2),
    SpeedAvgKmh DECIMAL(6,2),
    SpeedMaxKmh DECIMAL(6,2),
    WeatherCondition VARCHAR(50),
    TrafficDelay BIT,
    MaintenanceAlert BIT,
    CostUSD DECIMAL(12,2)
)

CREATE NONCLUSTERED INDEX IX_FactFleet_StartTime ON FactFleetTrips(StartTimeKey)
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID)
CREATE NONCLUSTERED INDEX IX_FactFleet_Route ON FactFleetTrips(OriginKey, DestinationKey)

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    ArrivalTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    DepartureTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    CrossDockID VARCHAR(50) UNIQUE,
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    Quantity INT,
    DwellMinutes DECIMAL(8,2), -- Time between inbound and outbound
    TemperatureCompliance BIT,
    QualityCheckPassed BIT
)
```

### Step 2: Populate Time Dimension

```sql
-- Generate time dimension for 2 years (15-minute intervals)
DECLARE @StartDate DATETIME2 = '2025-01-01 00:00:00'
DECLARE @EndDate DATETIME2 = '2027-01-01 00:00:00'
DECLARE @CurrentDate DATETIME2 = @StartDate

WHILE @CurrentDate < @EndDate
BEGIN
    INSERT INTO DimTime (
        TimeKey, 
        FullDateTime, 
        Hour, 
        DayOfWeek, 
        DayOfMonth, 
        Month, 
        Quarter, 
        Year,
        FiscalPeriod,
        IsWeekend,
        TimeSlot15Min
    )
    VALUES (
        CONVERT(INT, FORMAT(@CurrentDate, 'yyyyMMddHHmm')),
        @CurrentDate,
        DATEPART(HOUR, @CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        DATEPART(DAY, @CurrentDate),
        DATEPART(MONTH, @CurrentDate),
        DATEPART(QUARTER, @CurrentDate),
        DATEPART(YEAR, @CurrentDate),
        -- Fiscal period logic (example: month + 3)
        CASE WHEN DATEPART(MONTH, @CurrentDate) + 3 > 12 
             THEN DATEPART(MONTH, @CurrentDate) - 9 
             ELSE DATEPART(MONTH, @CurrentDate) + 3 
        END,
        CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1, 7) THEN 1 ELSE 0 END,
        FORMAT(@CurrentDate, 'HH:mm')
    )
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate)
END
```

### Step 3: Create Data Loading Stored Procedures

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example integration with external WMS (adjust to your source)
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType, 
        OperationID, Quantity, DurationMinutes, DwellHours,
        StorageZone, AssignedGravityZone, EmployeeID, BatchNumber, CostUSD
    )
    SELECT 
        CONVERT(INT, FORMAT(wms.OperationTimestamp, 'yyyyMMddHHmm')),
        geo.GeographyKey,
        prod.ProductKey,
        wms.OpType,
        wms.OpID,
        wms.Qty,
        wms.Duration,
        wms.Dwell,
        wms.Zone,
        prod.VelocityTier, -- Map to gravity zone
        wms.EmpID,
        wms.Batch,
        wms.Cost
    FROM ExternalWMS.dbo.Operations wms
    INNER JOIN DimGeography geo ON wms.WarehouseCode = geo.NodeID
    INNER JOIN DimProductGravity prod ON wms.SKU = prod.SKU
    WHERE wms.OperationTimestamp > @LastLoadTime
END
GO

-- Calculate gravity scores (run periodically)
CREATE PROCEDURE usp_RecalculateProductGravity
AS
BEGIN
    UPDATE DimProductGravity
    SET GravityScore = (
        -- Example formula: (pick frequency * value) / (fragility factor * volume)
        SELECT 
            (COUNT(*) * AVG(CostUSD)) / 
            (CASE WHEN IsFragile = 1 THEN 2.0 ELSE 1.0 END * UnitVolume)
        FROM FactWarehouseOperations
        WHERE ProductKey = DimProductGravity.ProductKey
          AND OperationType = 'Picking'
          AND TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMddHHmm'))
    ),
    VelocityTier = CASE 
        WHEN GravityScore >= 80 THEN 'Fast'
        WHEN GravityScore >= 40 THEN 'Medium'
        ELSE 'Slow'
    END,
    LastRecalculated = GETDATE()
END
GO
```

### Step 4: Configure Alert System

```sql
CREATE TABLE AlertThresholds (
    AlertID INT IDENTITY(1,1) PRIMARY KEY,
    AlertName NVARCHAR(100),
    MetricType VARCHAR(50), -- 'FleetIdlePct', 'DwellTime', 'Temperature', etc.
    ThresholdValue DECIMAL(10,2),
    ComparisonOperator VARCHAR(2), -- '>', '<', '='
    AlertSeverity VARCHAR(10), -- 'Critical', 'Warning', 'Info'
    NotificationEmail NVARCHAR(255),
    IsActive BIT DEFAULT 1
)

CREATE PROCEDURE usp_CheckAlerts
AS
BEGIN
    -- Example: Fleet idle time alert
    INSERT INTO AlertLog (AlertID, TriggeredTime, MetricValue, AffectedEntity)
    SELECT 
        a.AlertID,
        GETDATE(),
        (SUM(f.IdleMinutes) * 100.0 / SUM(f.DurationMinutes)) AS IdlePct,
        f.VehicleID
    FROM FactFleetTrips f
    CROSS JOIN AlertThresholds a
    WHERE a.MetricType = 'FleetIdlePct'
      AND a.IsActive = 1
      AND f.StartTimeKey >= CONVERT(INT, FORMAT(DATEADD(HOUR, -24, GETDATE()), 'yyyyMMddHHmm'))
    GROUP BY f.VehicleID, a.AlertID, a.ThresholdValue, a.ComparisonOperator
    HAVING (SUM(f.IdleMinutes) * 100.0 / SUM(f.DurationMinutes)) > a.ThresholdValue
       AND a.ComparisonOperator = '>'
END
GO

-- Schedule this with SQL Server Agent (every 15 minutes)
```

## Power BI Configuration

### Connecting to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `your-server-name`
4. Database: `LogiFleetPulse`
5. Data Connectivity mode: **DirectQuery** (for real-time) or **Import** (for performance)

### Recommended Measures (DAX)

```dax
// Total Warehouse Operations
TotalOperations = COUNTROWS(FactWarehouseOperations)

// Average Dwell Time (hours)
AvgDwellTime = 
AVERAGE(FactWarehouseOperations[DwellHours])

// Fleet Idle Percentage
FleetIdlePct = 
DIVIDE(
    SUM(FactFleetTrips[IdleMinutes]),
    SUM(FactFleetTrips[DurationMinutes])
) * 100

// Cross-Dock Efficiency (target: < 2 hours)
CrossDockEfficiency = 
CALCULATE(
    COUNTROWS(FactCrossDock),
    FactCrossDock[DwellMinutes] < 120
) / COUNTROWS(FactCrossDock)

// Warehouse Gravity Zone Compliance
GravityZoneCompliance = 
VAR ExpectedZone = RELATED(DimProductGravity[VelocityTier])
VAR ActualZone = FactWarehouseOperations[AssignedGravityZone]
RETURN
CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    ExpectedZone = ActualZone
) / COUNTROWS(FactWarehouseOperations)

// Predictive Bottleneck Index (simplified)
BottleneckIndex = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellHours])
VAR StdDevDwell = STDEV.P(FactWarehouseOperations[DwellHours])
VAR CurrentDwell = MAX(FactWarehouseOperations[DwellHours])
RETURN
IF(
    CurrentDwell > AvgDwell + (2 * StdDevDwell),
    "High Risk",
    IF(
        CurrentDwell > AvgDwell + StdDevDwell,
        "Medium Risk",
        "Normal"
    )
)

// Fleet Fuel Efficiency (km per liter)
FuelEfficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKm]),
    SUM(FactFleetTrips[FuelLiters])
)

// Supplier Reliability Score
SupplierReliability = 
AVERAGE(DimSupplierReliability[ComplianceScore])
```

### Relationships in Power BI Model

```
FactWarehouseOperations[TimeKey] → DimTime[TimeKey]
FactWarehouseOperations[WarehouseKey] → DimGeography[GeographyKey]
FactWarehouseOperations[ProductKey] → DimProductGravity[ProductKey]

FactFleetTrips[StartTimeKey] → DimTime[TimeKey]
FactFleetTrips[OriginKey] → DimGeography[GeographyKey]
FactFleetTrips[DestinationKey] → DimGeography[GeographyKey] (inactive, use USERELATIONSHIP)

FactCrossDock[ProductKey] → DimProductGravity[ProductKey]
FactCrossDock[SupplierKey] → DimSupplierReliability[SupplierKey]
FactCrossDock[InboundTripKey] → FactFleetTrips[TripKey]
```

### Row-Level Security (RLS)

```dax
// Create role: "Warehouse Manager"
[DimGeography[NodeType]] = "Warehouse" 
&& [DimGeography[Region]] = USERPRINCIPALNAME()

// Create role: "Fleet Supervisor"
[FactFleetTrips[VehicleID]] IN 
    (SELECT VehicleID FROM AssignedVehicles WHERE Manager = USERPRINCIPALNAME())
```

## Key Usage Patterns

### Pattern 1: Cross-Fact Analysis

```sql
-- Find products with high warehouse dwell AND high fleet idle cost
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    AVG(w.DwellHours) AS AvgWarehouseDwell,
    AVG(f.IdleMinutes) AS AvgFleetIdle,
    SUM(w.CostUSD + f.CostUSD) AS TotalCost
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
INNER JOIN FactFleetTrips f ON w.TimeKey = f.StartTimeKey
WHERE w.TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm'))
GROUP BY p.SKU, p.ProductName, p.GravityScore
HAVING AVG(w.DwellHours) > 48 
   AND AVG(f.IdleMinutes) > 30
ORDER BY TotalCost DESC
```

### Pattern 2: Temporal Elasticity Query

```sql
-- Simulate warehouse capacity change impact
WITH CapacityScenarios AS (
    SELECT 
        'Current_80pct' AS Scenario,
        COUNT(*) AS Operations,
        AVG(DwellHours) AS AvgDwell,
        AVG(DurationMinutes) AS AvgDuration
    FROM FactWarehouseOperations
    WHERE TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMddHHmm'))
    
    UNION ALL
    
    -- Simulate 95% capacity (dwell increases by 25%)
    SELECT 
        'Projected_95pct',
        COUNT(*),
        AVG(DwellHours) * 1.25,
        AVG(DurationMinutes) * 1.15
    FROM FactWarehouseOperations
    WHERE TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMddHHmm'))
)
SELECT * FROM CapacityScenarios
```

### Pattern 3: Gravity Zone Optimization

```sql
-- Recommend zone reassignments
SELECT 
    p.SKU,
    p.ProductName,
    p.VelocityTier AS CurrentZone,
    CASE 
        WHEN PickFreq > 50 THEN 'Fast'
        WHEN PickFreq > 20 THEN 'Medium'
        ELSE 'Slow'
    END AS RecommendedZone,
    PickFreq,
    AvgDistanceToShipping
FROM (
    SELECT 
        ProductKey,
        COUNT(*) AS PickFreq,
        AVG(CAST(SUBSTRING(StorageZone, 2, 2) AS INT)) AS AvgDistanceToShipping
    FROM FactWarehouseOperations
    WHERE OperationType = 'Picking'
      AND TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm'))
    GROUP BY ProductKey
) AS metrics
INNER JOIN DimProductGravity p ON metrics.ProductKey = p.ProductKey
WHERE p.VelocityTier != CASE 
    WHEN PickFreq > 50 THEN 'Fast'
    WHEN PickFreq > 20 THEN 'Medium'
    ELSE 'Slow'
END
```

### Pattern 4: External API Integration (Weather Impact)

```sql
-- Assuming weather data is loaded via SSIS or Python into a staging table
CREATE TABLE StagingWeather (
    Timestamp DATETIME2,
    Location VARCHAR(100),
    Condition VARCHAR(50),
    Temperature DECIMAL(5,2),
    Precipitation DECIMAL(5,2)
)

-- Correlate weather with delivery delays
SELECT 
    w.Condition,
    AVG(f.DurationMinutes - f.DistanceKm / 60.0) AS AvgDelayMinutes,
    COUNT(*) AS AffectedTrips
FROM FactFleetTrips f
INNER JOIN DimGeography dest ON f.DestinationKey = dest.GeographyKey
INNER JOIN StagingWeather w ON 
    CAST(f.EndTimeKey AS VARCHAR(12)) LIKE CONCAT(FORMAT(w.Timestamp, 'yyyyMMddHH'), '%')
    AND dest.City = w.Location
WHERE f.TrafficDelay = 1
GROUP BY w.Condition
ORDER BY AvgDelayMinutes DESC
```

## Data Ingestion Workflows

### Using SSIS for WMS Integration

```xml
<!-- SSIS Package Example: Load Warehouse Operations -->
<Package>
  <Executables>
    <Execute SQL Task Name="Get Last Load Time">
      SELECT MAX(OperationTimestamp) FROM LoadLog WHERE Source = 'WMS'
    </Execute>
    <Data Flow Task Name="Extract WMS Data">
      <Source Type="OLEDB">
        <ConnectionString>$(WMS_CONNECTION_STRING)</ConnectionString>
        <Query>
          SELECT * FROM Operations 
          WHERE Timestamp > ?
        </Query>
      </Source>
      <Transformation Type="Lookup">
        <LookupTable>DimGeography</LookupTable>
        <MatchColumn>WarehouseCode = NodeID</MatchColumn>
      </Transformation>
      <Destination Type="OLEDB">
        <Table>FactWarehouseOperations</Table>
      </Destination>
    </Data Flow Task>
  </Executables>
</Package>
```

### Python Script for Telematics API

```python
import pyodbc
import requests
import os
from datetime import datetime, timedelta

# Configuration
SQL_SERVER = os.getenv('SQL_SERVER')
SQL_DATABASE = 'LogiFleetPulse'
SQL_USER = os.getenv('SQL_USER')
SQL_PASSWORD = os.getenv('SQL_PASSWORD')
TELEMATICS_API_KEY = os.getenv('TELEMATICS_API_KEY')

# Connect to SQL Server
conn = pyodbc.connect(
    f'DRIVER={{ODBC Driver 17 for SQL Server}};'
    f'SERVER={SQL_SERVER};DATABASE={SQL_DATABASE};'
    f'UID={SQL_USER};PWD={SQL_PASSWORD}'
)
cursor = conn.cursor()

# Get last load timestamp
cursor.execute("SELECT MAX(EndTimeKey) FROM FactFleetTrips")
last_load = cursor.fetchone()[0]
last_load_dt = datetime.strptime(str(last_load), '%Y%m%d%H%M')

# Fetch telematics data
response = requests.get(
    'https://api.telematicsprovider.com/trips',
    headers={'Authorization': f'Bearer {TELEMATICS_API_KEY}'},
    params={
        'start_time': last_load_dt.isoformat(),
        'end_time': datetime.now().isoformat()
    }
)

trips = response.json()

# Insert into fact table
for trip in trips:
    cursor.execute("""
        INSERT INTO FactFleetTrips (
            StartTimeKey, EndTimeKey, OriginKey, DestinationKey,
            VehicleID, DriverID, TripID, DistanceKm, DurationMinutes,
            IdleMinutes, FuelLiters, LoadWeightKg, SpeedAvgKmh,
            SpeedMaxKmh, WeatherCondition, TrafficDelay, 
            MaintenanceAlert, CostUSD
        )
        SELECT 
            CONVERT(INT, FORMAT(CAST(? AS DATETIME2), 'yyyyMMddHHmm')),
            CONVERT(INT, FORMAT(CAST(? AS DATETIME2), 'yyyyMMddHHmm')),
            (SELECT GeographyKey FROM DimGeography WHERE NodeID = ?),
            (SELECT GeographyKey FROM DimGeography WHERE NodeID = ?),
            ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?
    """, 
        trip['start_time'], trip['end_time'],
        trip['origin_code'], trip['destination_code'],
        trip['vehicle_id'], trip['driver_id'], trip['trip_id'],
        trip['distance_km'], trip['duration_minutes'],
        trip['idle_minutes'], trip['fuel_liters'],
        trip['load_weight_kg'], trip['speed_avg_kmh'],
        trip['speed_max_kmh'], trip.get('weather_condition'),
        trip.get('traffic_delay', False),
        trip.get('maintenance_alert', False),
        trip['cost_usd']
    )

conn.commit()
cursor.close()
conn.close()
```

## Configuration Management

### Environment Variables

```bash
# SQL Server Connection
export SQL_SERVER="your-server.database.windows.net"
export SQL_USER="logifleet_admin"
export SQL_PASSWORD="your-secure-password"

# External APIs
export TELEMATICS_API_KEY="your-telematics-api-key"
export WEATHER_API_KEY="your-weather-api-key"
export WMS_CONNECTION_STRING="Server=wms-server;Database=WMS;..."

# Alert Configuration
export ALERT_SMTP_SERVER="smtp.yourcompany.com"
export ALERT_FROM_EMAIL="alerts@yourcompany.com"
export ALERT_TO_EMAIL="logistics-team@yourcompany.com"

# Power BI
export POWERBI_WORKSPACE_ID="your-workspace-guid"
export POWERBI_DATASET_ID="your-dataset-guid"
```

### config.json Example

```json
{
  "database": {
    "server": "${SQL_SERVER}",
    "database": "LogiFleetPulse",
    "authentication": "sql",
    "connection_timeout": 30,
    "partition_strategy": "monthly"
  },
  "data_sources": {
    "wms": {
      "type": "oledb",
      "connection_string": "${WMS_CONNECTION_STRING}",
      "refresh_interval_minutes": 15
    },
    "telematics": {
      "type": "rest_api",
      "base_url": "https://api.telematicsprovider.com",
      "auth_type": "bearer",
      "api_key": "${TELEMATICS_API_KEY}",
      "refresh_interval_minutes": 5
    },
    "weather": {
      "type": "rest_api",
      "base_url": "https://api.weather.com/v3",
      "api_key": "${WEATHER_API_KEY}",
      "refresh_interval_minutes": 60
    }
  },
  "gravity_zones": {
    "recalculation_schedule": "daily",
    "fast_threshold_score": 80,
    "medium_threshold_score": 40,
    "lookback_days": 90
  },
  "alerts": {
    "smtp_server": "${ALERT_SMTP_SERVER}",
    "from_email": "${ALERT_FROM_EMAIL}",
    "to_email": "${ALERT_TO_EMAIL}",
    "thresholds": {
      "fleet_idle_pct": 15,
      "warehouse_dwell_hours": 72,
      "crossdock_dwell_minutes": 120,
      "fuel_efficiency_min_kmpl": 3.5
    }
  },
  "powerbi": {
    "refresh_schedule": "0 */15 * * * *",
    "rls_enabled": true,
    "embedding_enabled": false
  }
}
```

## Troubleshooting

### Issue: Power BI DirectQuery Timeout

**Symptom**: Dashboards show timeout errors when filtering large date ranges

**Solution**: 
```sql
-- Create aggregated views for common queries
CREATE VIEW vw_DailyWarehouseMetrics AS
SELECT 
    CAST(t.FullDateTime AS DATE) AS Date,
    g.NodeName AS Warehouse,
    p.Category,
    w.OperationType,
    COUNT(*) AS OperationCount,
    AVG(w.DwellHours) AS AvgDwellHours,
    SUM(w.Quantity) AS TotalQuantity,
    SUM(w.CostUSD) AS TotalCost
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
GROUP BY CAST(t.FullDateTime AS DATE), g
