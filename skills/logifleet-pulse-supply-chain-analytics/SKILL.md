---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI logistics intelligence platform with multi-fact star schema for warehouse, fleet, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure logistics data warehouse with multi-fact schema"
  - "create Power BI dashboard for fleet and warehouse operations"
  - "implement warehouse gravity zones optimization"
  - "build cross-modal supply chain KPI dashboard"
  - "deploy SQL Server logistics intelligence platform"
  - "integrate warehouse and fleet telemetry data"
  - "set up predictive bottleneck detection for logistics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is an advanced logistics intelligence platform that combines MS SQL Server data warehousing with Power BI visualization to create a unified view of supply chain operations. It uses a multi-fact star schema to integrate:

- Warehouse operations (receiving, putaway, picking, packing, shipping)
- Fleet telemetry (GPS, fuel consumption, vehicle diagnostics)
- Inventory management (aging curves, turnover rates)
- Supplier reliability metrics
- External data feeds (weather, traffic)

The platform provides cross-fact KPI harmonization, predictive bottleneck detection, and spatial optimization through "Warehouse Gravity Zones."

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS)
- Appropriate database permissions (CREATE TABLE, CREATE VIEW, CREATE PROCEDURE)

### Step 1: Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance via SSMS
-- Create a new database for the logistics platform
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the schema deployment script (typically in /sql/schema/)
-- This creates all dimension and fact tables
```

### Step 3: Configure Data Sources

Create a configuration file based on the provided template:

```json
{
  "sql_server": {
    "server": "YOUR_SERVER_NAME",
    "database": "LogiFleetPulse",
    "authentication": "sql",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_api": "${TELEMATICS_API_ENDPOINT}",
    "weather_api_key": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

### Step 4: Load Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter your SQL Server connection details when prompted
3. The model will automatically detect fact tables and build relationships

## Core SQL Schema Structure

### Dimension Tables

```sql
-- DimTime: Time-phased dimension with 15-minute granularity
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME NOT NULL,
    Hour TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    Month TINYINT NOT NULL,
    Quarter TINYINT NOT NULL,
    FiscalYear INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
);

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL,
    LocationName VARCHAR(100) NOT NULL,
    LocationType VARCHAR(20) NOT NULL, -- Warehouse, Route Node, Customer Site
    StreetAddress VARCHAR(200),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50)
);

-- DimProductGravity: Product classification with velocity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200) NOT NULL,
    Category VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated from velocity, value, fragility
    VelocityClass VARCHAR(20), -- Fast, Medium, Slow
    ValueClass VARCHAR(20), -- High, Medium, Low
    FragilityScore DECIMAL(3,2),
    OptimalStorageZone VARCHAR(50),
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,4)
);

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200) NOT NULL,
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectPercentage DECIMAL(5,2),
    OnTimeDeliveryRate DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2),
    ReliabilityRating VARCHAR(20) -- A, B, C, D
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Warehouse activity metrics
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OrderID VARCHAR(50),
    QuantityHandled INT NOT NULL,
    DwellTimeMinutes INT, -- Time spent in warehouse before next operation
    CycleTimeSeconds INT, -- Time to complete operation
    ErrorFlag BIT DEFAULT 0,
    OperatorID VARCHAR(50),
    StorageZone VARCHAR(50),
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

-- FactFleetTrips: Fleet and route telemetry
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    TripDistanceKm DECIMAL(10,2),
    TripDurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(8,2),
    LoadWeightKg DECIMAL(10,2),
    OnTimeFlag BIT,
    DelayReasonCode VARCHAR(50),
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_Fleet_Dest FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

-- FactCrossDock: Cross-docking operations
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    QuantityTransferred INT NOT NULL,
    DwellTimeMinutes INT, -- Time between inbound and outbound
    CONSTRAINT FK_CD_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CD_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_CD_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);
```

## Key SQL Queries and Patterns

### Cross-Fact KPI: Warehouse Dwell Time vs Fleet Idle Time

```sql
-- Correlate warehouse dwell time with subsequent fleet idle time
-- to identify bottlenecks in the handoff process
WITH WarehouseDwell AS (
    SELECT 
        w.ProductKey,
        w.GeographyKey,
        DATEPART(HOUR, dt.DateTime) AS HourOfDay,
        AVG(w.DwellTimeMinutes) AS AvgDwellMinutes,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations w
    JOIN DimTime dt ON w.TimeKey = dt.TimeKey
    WHERE w.OperationType = 'Packing'
        AND dt.DateTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY w.ProductKey, w.GeographyKey, DATEPART(HOUR, dt.DateTime)
),
FleetIdle AS (
    SELECT 
        f.OriginGeographyKey AS GeographyKey,
        DATEPART(HOUR, dt.DateTime) AS HourOfDay,
        AVG(f.IdleTimeMinutes) AS AvgIdleMinutes,
        COUNT(*) AS TripCount
    FROM FactFleetTrips f
    JOIN DimTime dt ON f.TimeKey = dt.TimeKey
    WHERE dt.DateTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY f.OriginGeographyKey, DATEPART(HOUR, dt.DateTime)
)
SELECT 
    g.LocationName,
    wd.HourOfDay,
    wd.AvgDwellMinutes,
    fi.AvgIdleMinutes,
    (wd.AvgDwellMinutes + fi.AvgIdleMinutes) AS TotalWaitTime,
    wd.OperationCount,
    fi.TripCount
FROM WarehouseDwell wd
JOIN FleetIdle fi ON wd.GeographyKey = fi.GeographyKey 
    AND wd.HourOfDay = fi.HourOfDay
JOIN DimGeography g ON wd.GeographyKey = g.GeographyKey
ORDER BY TotalWaitTime DESC;
```

### Warehouse Gravity Zone Analysis

```sql
-- Calculate optimal storage zones based on product gravity scores
-- and current pick frequency
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalStorageZone,
    w.StorageZone AS CurrentStorageZone,
    COUNT(*) AS PickFrequency,
    AVG(w.CycleTimeSeconds) AS AvgPickTimeSeconds,
    CASE 
        WHEN p.OptimalStorageZone != w.StorageZone THEN 'Needs Relocation'
        ELSE 'Optimally Placed'
    END AS RecommendationStatus
FROM FactWarehouseOperations w
JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
JOIN DimTime dt ON w.TimeKey = dt.TimeKey
WHERE w.OperationType = 'Picking'
    AND dt.DateTime >= DATEADD(DAY, -90, GETDATE())
GROUP BY 
    p.SKU, 
    p.ProductName, 
    p.GravityScore, 
    p.OptimalStorageZone, 
    w.StorageZone
HAVING COUNT(*) > 10 -- Only consider products with meaningful pick frequency
ORDER BY PickFrequency DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify potential bottlenecks by analyzing variance in cycle times
-- and flagging operations that exceed 2 standard deviations
WITH CycleTimeStats AS (
    SELECT 
        GeographyKey,
        OperationType,
        AVG(CAST(CycleTimeSeconds AS FLOAT)) AS AvgCycleTime,
        STDEV(CAST(CycleTimeSeconds AS FLOAT)) AS StdDevCycleTime
    FROM FactWarehouseOperations
    WHERE TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE DateTime >= DATEADD(DAY, -30, GETDATE()))
    GROUP BY GeographyKey, OperationType
)
SELECT 
    g.LocationName,
    w.OperationType,
    dt.DateTime,
    w.CycleTimeSeconds,
    cts.AvgCycleTime,
    cts.StdDevCycleTime,
    (w.CycleTimeSeconds - cts.AvgCycleTime) / NULLIF(cts.StdDevCycleTime, 0) AS ZScore,
    CASE 
        WHEN (w.CycleTimeSeconds - cts.AvgCycleTime) / NULLIF(cts.StdDevCycleTime, 0) > 2 
        THEN 'High Risk'
        WHEN (w.CycleTimeSeconds - cts.AvgCycleTime) / NULLIF(cts.StdDevCycleTime, 0) > 1 
        THEN 'Medium Risk'
        ELSE 'Normal'
    END AS BottleneckRisk
FROM FactWarehouseOperations w
JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
JOIN DimTime dt ON w.TimeKey = dt.TimeKey
JOIN CycleTimeStats cts ON w.GeographyKey = cts.GeographyKey 
    AND w.OperationType = cts.OperationType
WHERE dt.DateTime >= DATEADD(DAY, -7, GETDATE())
    AND (w.CycleTimeSeconds - cts.AvgCycleTime) / NULLIF(cts.StdDevCycleTime, 0) > 1
ORDER BY ZScore DESC;
```

### Fleet Fuel Efficiency Analysis

```sql
-- Calculate fuel efficiency per route and identify optimization opportunities
SELECT 
    og.LocationName AS Origin,
    dg.LocationName AS Destination,
    COUNT(*) AS TripCount,
    AVG(f.TripDistanceKm) AS AvgDistanceKm,
    AVG(f.FuelConsumedLiters) AS AvgFuelLiters,
    AVG(f.FuelConsumedLiters / NULLIF(f.TripDistanceKm, 0)) AS AvgLitersPer100Km,
    AVG(f.IdleTimeMinutes) AS AvgIdleMinutes,
    SUM(CASE WHEN f.OnTimeFlag = 1 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS OnTimePercentage
FROM FactFleetTrips f
JOIN DimGeography og ON f.OriginGeographyKey = og.GeographyKey
JOIN DimGeography dg ON f.DestinationGeographyKey = dg.GeographyKey
JOIN DimTime dt ON f.TimeKey = dt.TimeKey
WHERE dt.DateTime >= DATEADD(DAY, -60, GETDATE())
GROUP BY og.LocationName, dg.LocationName
HAVING COUNT(*) >= 5 -- Only routes with meaningful sample size
ORDER BY AvgLitersPer100Km DESC;
```

## Stored Procedures for Automation

### Incremental Data Load Procedure

```sql
CREATE PROCEDURE usp_IncrementalLoadWarehouseOps
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new warehouse operations since last load
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OrderID, QuantityHandled, DwellTimeMinutes, 
        CycleTimeSeconds, ErrorFlag, OperatorID, StorageZone
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        src.OperationType,
        src.OrderID,
        src.QuantityHandled,
        src.DwellTimeMinutes,
        src.CycleTimeSeconds,
        src.ErrorFlag,
        src.OperatorID,
        src.StorageZone
    FROM ExternalWarehouseData src -- This would be your staging/external table
    JOIN DimTime dt ON CAST(src.OperationDateTime AS DATETIME) = dt.DateTime
    JOIN DimGeography dg ON src.LocationID = dg.LocationID
    JOIN DimProductGravity dp ON src.SKU = dp.SKU
    WHERE src.OperationDateTime > @LastLoadDateTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f
            WHERE f.OrderID = src.OrderID 
                AND f.TimeKey = dt.TimeKey
                AND f.ProductKey = dp.ProductKey
        );
    
    SELECT @@ROWCOUNT AS RowsInserted;
END;
GO
```

### Automated Alert Procedure

```sql
CREATE PROCEDURE usp_GenerateBottleneckAlerts
    @ThresholdMultiplier DECIMAL(3,1) = 2.0
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Identify operations exceeding cycle time thresholds
    WITH CycleTimeStats AS (
        SELECT 
            GeographyKey,
            OperationType,
            AVG(CAST(CycleTimeSeconds AS FLOAT)) AS AvgCycleTime,
            STDEV(CAST(CycleTimeSeconds AS FLOAT)) AS StdDevCycleTime
        FROM FactWarehouseOperations
        WHERE TimeKey >= (SELECT MIN(TimeKey) FROM DimTime 
                          WHERE DateTime >= DATEADD(DAY, -30, GETDATE()))
        GROUP BY GeographyKey, OperationType
    )
    INSERT INTO AlertQueue (AlertType, Severity, LocationName, Message, CreatedDateTime)
    SELECT 
        'Bottleneck' AS AlertType,
        CASE 
            WHEN (w.CycleTimeSeconds - cts.AvgCycleTime) / NULLIF(cts.StdDevCycleTime, 0) > 3 
            THEN 'Critical'
            WHEN (w.CycleTimeSeconds - cts.AvgCycleTime) / NULLIF(cts.StdDevCycleTime, 0) > 2 
            THEN 'High'
            ELSE 'Medium'
        END AS Severity,
        g.LocationName,
        CONCAT(
            w.OperationType, 
            ' cycle time (', w.CycleTimeSeconds, 's) exceeds threshold by ',
            CAST((w.CycleTimeSeconds - cts.AvgCycleTime) / NULLIF(cts.StdDevCycleTime, 0) AS DECIMAL(5,2)),
            ' standard deviations'
        ) AS Message,
        GETDATE() AS CreatedDateTime
    FROM FactWarehouseOperations w
    JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    JOIN DimTime dt ON w.TimeKey = dt.TimeKey
    JOIN CycleTimeStats cts ON w.GeographyKey = cts.GeographyKey 
        AND w.OperationType = cts.OperationType
    WHERE dt.DateTime >= DATEADD(HOUR, -2, GETDATE())
        AND (w.CycleTimeSeconds - cts.AvgCycleTime) / NULLIF(cts.StdDevCycleTime, 0) > @ThresholdMultiplier;
    
    SELECT @@ROWCOUNT AS AlertsGenerated;
END;
GO
```

## Power BI DAX Measures

### Cross-Fact KPI: Total Logistics Cost Per Order

```dax
// Combines warehouse labor cost and fleet fuel cost
Total Logistics Cost = 
VAR WarehouseCost = 
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[CycleTimeSeconds] / 3600 * 25 // $25/hour labor rate
    )
VAR FleetCost = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[FuelConsumedLiters] * 1.50 // $1.50/liter fuel cost
    )
RETURN WarehouseCost + FleetCost
```

### Fleet Utilization Rate

```dax
Fleet Utilization Rate = 
DIVIDE(
    SUMX(
        FactFleetTrips,
        FactFleetTrips[TripDurationMinutes] - FactFleetTrips[IdleTimeMinutes]
    ),
    SUMX(
        FactFleetTrips,
        FactFleetTrips[TripDurationMinutes]
    ),
    0
)
```

### Warehouse Gravity Score Compliance

```dax
// Percentage of picks occurring in optimal gravity zones
Gravity Compliance % = 
VAR OptimalPicks = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[StorageZone] = RELATED(DimProductGravity[OptimalStorageZone]),
        FactWarehouseOperations[OperationType] = "Picking"
    )
VAR TotalPicks = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[OperationType] = "Picking"
    )
RETURN DIVIDE(OptimalPicks, TotalPicks, 0)
```

### Predictive Bottleneck Index

```dax
// Weighted score based on cycle time variance and error rate
Bottleneck Index = 
VAR CycleTimeVar = 
    VAR AvgCycle = AVERAGE(FactWarehouseOperations[CycleTimeSeconds])
    VAR StdDev = STDEV.P(FactWarehouseOperations[CycleTimeSeconds])
    RETURN DIVIDE(StdDev, AvgCycle, 0)
VAR ErrorRate = 
    DIVIDE(
        CALCULATE(COUNTROWS(FactWarehouseOperations), FactWarehouseOperations[ErrorFlag] = TRUE()),
        COUNTROWS(FactWarehouseOperations),
        0
    )
RETURN (CycleTimeVar * 0.6) + (ErrorRate * 0.4) // Weighted composite
```

## Configuration Best Practices

### SQL Server Optimization

```sql
-- Create indexes for common query patterns
CREATE NONCLUSTERED INDEX IX_WH_Time_Geography_Product 
ON FactWarehouseOperations (TimeKey, GeographyKey, ProductKey)
INCLUDE (OperationType, DwellTimeMinutes, CycleTimeSeconds);

CREATE NONCLUSTERED INDEX IX_Fleet_Time_Origin_Dest
ON FactFleetTrips (TimeKey, OriginGeographyKey, DestinationGeographyKey)
INCLUDE (IdleTimeMinutes, FuelConsumedLiters, OnTimeFlag);

-- Partition fact tables by time for improved query performance
CREATE PARTITION FUNCTION PF_LogiFleet_Monthly (INT)
AS RANGE RIGHT FOR VALUES (
    -- Create monthly partitions based on TimeKey ranges
    202401, 202402, 202403, 202404, 202405, 202406
);

CREATE PARTITION SCHEME PS_LogiFleet_Monthly
AS PARTITION PF_LogiFleet_Monthly
ALL TO ([PRIMARY]);
```

### Row-Level Security Setup

```sql
-- Create security predicate for role-based access
CREATE FUNCTION dbo.fn_SecurityPredicate(@LocationID VARCHAR(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN 
    SELECT 1 AS AccessGranted
    WHERE 
        @LocationID IN (
            SELECT LocationID 
            FROM dbo.UserLocationAccess 
            WHERE UserName = USER_NAME()
        )
        OR IS_MEMBER('db_owner') = 1;
GO

-- Apply security policy to geography dimension
CREATE SECURITY POLICY LocationAccessPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(LocationID)
ON DimGeography
WITH (STATE = ON);
GO
```

### Power BI Refresh Schedule

Configure automated refresh in Power BI Service:

1. Navigate to dataset settings in Power BI Service
2. Set scheduled refresh to align with SQL data load schedule (e.g., every 15 minutes during business hours)
3. Configure gateway connection for on-premises SQL Server
4. Enable incremental refresh for large fact tables:

```dax
// Power BI incremental refresh parameter
RangeStart = DateTime(2024, 1, 1, 0, 0, 0)
RangeEnd = DateTime(2025, 12, 31, 23, 59, 59)

// Filter in Power Query
= Table.SelectRows(
    FactWarehouseOperations,
    each [OperationDateTime] >= RangeStart and [OperationDateTime] < RangeEnd
)
```

## Common Troubleshooting

### Issue: Cross-Fact Queries Running Slow

**Solution**: Verify composite indexes exist and consider materialized views:

```sql
-- Create materialized view for frequently joined facts
CREATE VIEW vw_WarehouseFleetCrossReference
WITH SCHEMABINDING
AS
SELECT 
    w.GeographyKey,
    w.TimeKey,
    w.ProductKey,
    COUNT_BIG(*) AS WarehouseOpCount,
    AVG(w.DwellTimeMinutes) AS AvgDwellMinutes,
    SUM(w.QuantityHandled) AS TotalQuantity
FROM dbo.FactWarehouseOperations w
GROUP BY w.GeographyKey, w.TimeKey, w.ProductKey;

CREATE UNIQUE CLUSTERED INDEX IX_Materialized
ON vw_WarehouseFleetCrossReference (GeographyKey, TimeKey, ProductKey);
```

### Issue: Power BI Dashboard Not Refreshing

**Solution**: Check gateway connection and credentials:

```powershell
# Test SQL connection from gateway machine
Test-NetConnection -ComputerName YOUR_SQL_SERVER -Port 1433

# Verify service account has read permissions
sqlcmd -S YOUR_SQL_SERVER -d LogiFleetPulse -Q "SELECT SYSTEM_USER"
```

### Issue: Gravity Score Calculation Inconsistencies

**Solution**: Recalculate gravity scores using stored procedure:

```sql
CREATE PROCEDURE usp_RecalculateGravityScores
AS
BEGIN
    UPDATE DimProductGravity
    SET GravityScore = (
        (VelocityWeight * 0.5) + 
        (ValueWeight * 0.3) + 
        ((1 - FragilityScore) * 0.2) -- Inverse fragility
    )
    FROM (
        SELECT 
            ProductKey,
            CASE VelocityClass
                WHEN 'Fast' THEN 1.0
                WHEN 'Medium' THEN 0.6
                WHEN 'Slow' THEN 0.3
            END AS VelocityWeight,
            CASE ValueClass
                WHEN 'High' THEN 1.0
                WHEN 'Medium' THEN 0.6
                WHEN 'Low' THEN 0.3
            END AS ValueWeight
        FROM DimProductGravity
    ) AS Weights
    WHERE DimProductGravity.ProductKey = Weights.ProductKey;
END;
GO
```

### Issue: Alert Queue Overflowing

**Solution**: Archive old alerts and adjust thresholds:

```sql
-- Archive alerts older than 90 days
INSERT INTO AlertQueueArchive
SELECT * FROM AlertQueue 
WHERE CreatedDateTime < DATEADD(DAY, -90, GETDATE());

DELETE FROM AlertQueue 
WHERE CreatedDateTime < DATEADD(DAY, -90, GETDATE());

-- Adjust alert thresholds to reduce noise
EXEC usp_GenerateBottleneckAlerts @ThresholdMultiplier = 2.5; -- Increase from 2.0
```

## Integration Patterns

### WMS API Integration (Python Example)

```python
import requests
import pyodbc
import os
from datetime import datetime

# Connection to SQL Server
conn_str = (
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.getenv('SQL_SERVER')};"
    f"DATABASE=LogiFleetPulse;"
    f"UID={os.getenv('SQL_USERNAME')};"
    f"PWD={os.getenv('SQL_PASSWORD')}"
)

conn = pyodbc.connect(conn_str)
cursor = conn.cursor()

# Fetch data from WMS API
wms_response = requests.get(
    os.getenv('WMS_API_ENDPOINT'),
    headers={'Authorization': f"Bearer {os.getenv('WMS_API_TOKEN')}"}
)

operations = wms_response.json()['operations']

# Insert into staging table
for op in operations:
    cursor.execute("""
        INSERT INTO ExternalWarehouseData (
            OperationDateTime, LocationID, SKU, OperationType,
            OrderID, QuantityHandled, DwellTimeMinutes, 
            CycleTimeSeconds, OperatorID, StorageZone
        ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    """, (
        datetime.fromisoformat(op['timestamp']),
        op['location_id'],
        op['sku'],
        op['operation_type'],
        op['order_id'],
        op['quantity'],
        op.get('dwell_time_minutes', 0),
        op['cycle_time_seconds'],
        op['operator_id'],
        op['storage_zone']
    ))

conn.commit()

# Trigger incremental load procedure
cursor.execute("EXEC usp_IncrementalLoadWarehouseOps @LastLoadDateTime = ?", 
               (datetime.now(),))
conn.commit()

cursor.close()
conn.close()
```

### Telematics API Integration (Node.js Example)

```javascript
const sql = require('mssql');
const axios = require('axios');

const config = {
    server: process.env.SQL_SERVER,
    database: 'LogiFleetPulse',
    user: process.env.SQL_USERNAME,
    password: process.env.SQL_PASSWORD,
    options: {
        encrypt: true,
        trustServerCertificate: false
    }
};

async function loadTelematicsData() {
    try {
        const pool = await sql.connect(config);
        
        // Fetch from telematics API
        const response = await axios.get(process.env.TELEMATICS_API_ENDPOINT, {
            headers: { 'Authorization': `Bearer ${process.env.TELEMATICS_API_TOKEN}` }
        });
        
        
