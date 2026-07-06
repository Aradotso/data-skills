---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing template for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "implement multi-fact star schema for warehouse data"
  - "deploy SQL Server logistics data warehouse"
  - "create fleet management analytics with Power BI"
  - "build supply chain KPI dashboard"
  - "integrate warehouse and fleet telemetry data"
  - "design logistics data model with time dimensions"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What It Does

LogiFleet Pulse is a comprehensive data warehousing and analytics template for logistics operations. It provides:

- **Multi-fact star schema** for warehouse operations, fleet trips, and cross-dock activities
- **Time-phased dimensions** with 15-minute granularity for real-time tracking
- **Power BI dashboard templates** for unified logistics visualization
- **Cross-fact KPI harmonization** linking inventory, fleet, and supplier metrics
- **SQL Server stored procedures** for incremental data loading and alerting
- **Role-based security** with row-level access controls

The project bridges warehouse management systems (WMS), telematics feeds, supplier portals, and external APIs into a single semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Step 1: Deploy SQL Schema

```sql
-- Create the main database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    TimeStamp DATETIME2(0) NOT NULL,
    [Hour] TINYINT NOT NULL,
    [Minute] TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayOfMonth TINYINT NOT NULL,
    [Month] TINYINT NOT NULL,
    [Quarter] TINYINT NOT NULL,
    [Year] SMALLINT NOT NULL,
    FiscalPeriod VARCHAR(10),
    Is15MinBucket BIT DEFAULT 1
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    NodeCode VARCHAR(50) NOT NULL UNIQUE,
    NodeName NVARCHAR(200) NOT NULL,
    NodeType VARCHAR(20) CHECK (NodeType IN ('Warehouse', 'RoutePoint', 'CrossDock', 'Supplier')),
    [Address] NVARCHAR(500),
    City NVARCHAR(100),
    [State] NVARCHAR(100),
    Country NVARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50)
);

CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName NVARCHAR(300) NOT NULL,
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitWeight DECIMAL(10,4),
    UnitVolume DECIMAL(10,4),
    Fragility TINYINT CHECK (Fragility BETWEEN 1 AND 10),
    TemperatureZone VARCHAR(20),
    GravityScore DECIMAL(5,2) -- Calculated: velocity * value / fragility
);

CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode VARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200) NOT NULL,
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2)
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    WarehouseKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProduct(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplier(SupplierKey),
    OperationType VARCHAR(20) CHECK (OperationType IN ('Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping')),
    Quantity INT NOT NULL,
    DwellTimeMinutes INT,
    CycleTimeMinutes DECIMAL(10,2),
    ZoneCode VARCHAR(20),
    BatchNumber VARCHAR(100),
    QualityCheckPassed BIT
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKeyStart INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    TimeKeyEnd INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleID VARCHAR(50) NOT NULL,
    OriginKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    RouteID VARCHAR(100),
    DistanceKm DECIMAL(10,2),
    FuelLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightKg DECIMAL(10,2),
    TripStatus VARCHAR(20) CHECK (TripStatus IN ('Completed', 'InProgress', 'Delayed', 'Cancelled')),
    DelayReasonCode VARCHAR(50),
    DriverID VARCHAR(50)
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKeyIn INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    TimeKeyOut INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    CrossDockKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProduct(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    Quantity INT NOT NULL,
    ProcessingTimeMinutes INT
);

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey) INCLUDE (WarehouseKey, ProductKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey) INCLUDE (TimeKey, Quantity);
CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKeyStart) INCLUDE (VehicleID, RouteID);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID) INCLUDE (TimeKeyStart, TripStatus);
```

### Step 2: Populate Time Dimension

```sql
-- Generate 15-minute time buckets for 3 years
DECLARE @StartDate DATETIME2 = '2024-01-01 00:00:00';
DECLARE @EndDate DATETIME2 = '2027-12-31 23:45:00';
DECLARE @CurrentDate DATETIME2 = @StartDate;
DECLARE @TimeKey INT = 1;

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO DimTime (TimeKey, TimeStamp, [Hour], [Minute], DayOfWeek, DayOfMonth, [Month], [Quarter], [Year])
    VALUES (
        @TimeKey,
        @CurrentDate,
        DATEPART(HOUR, @CurrentDate),
        DATEPART(MINUTE, @CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        DATEPART(DAY, @CurrentDate),
        DATEPART(MONTH, @CurrentDate),
        DATEPART(QUARTER, @CurrentDate),
        DATEPART(YEAR, @CurrentDate)
    );
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate);
    SET @TimeKey = @TimeKey + 1;
END;
```

### Step 3: Create Data Loading Stored Procedures

```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @SourceTable NVARCHAR(200),
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Incremental load from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, SupplierKey,
        OperationType, Quantity, DwellTimeMinutes, CycleTimeMinutes,
        ZoneCode, BatchNumber, QualityCheckPassed
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        src.OperationType,
        src.Quantity,
        src.DwellTimeMinutes,
        src.CycleTimeMinutes,
        src.ZoneCode,
        src.BatchNumber,
        src.QualityCheckPassed
    FROM @SourceTable src
    INNER JOIN DimTime t ON DATEADD(MINUTE, 
        (DATEDIFF(MINUTE, '2024-01-01', src.OperationTimestamp) / 15) * 15, 
        '2024-01-01') = t.TimeStamp
    INNER JOIN DimGeography g ON src.WarehouseCode = g.NodeCode
    INNER JOIN DimProduct p ON src.SKU = p.SKU
    LEFT JOIN DimSupplier s ON src.SupplierCode = s.SupplierCode
    WHERE src.OperationTimestamp > @LastLoadTime;
    
    RETURN @@ROWCOUNT;
END;
GO

CREATE PROCEDURE usp_CalculateProductGravity
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update gravity scores based on velocity, value, and fragility
    UPDATE p
    SET GravityScore = (
        -- Velocity factor (picks per day)
        (SELECT COUNT(*) / NULLIF(DATEDIFF(DAY, MIN(t.TimeStamp), MAX(t.TimeStamp)), 0)
         FROM FactWarehouseOperations w
         INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
         WHERE w.ProductKey = p.ProductKey 
           AND w.OperationType = 'Picking'
           AND t.TimeStamp >= DATEADD(DAY, -30, GETDATE())
        ) * 0.5
        +
        -- Fragility inverse factor
        (11 - p.Fragility) * 0.3
        +
        -- Random value placeholder (replace with actual value)
        5.0 * 0.2
    )
    FROM DimProduct p;
END;
GO
```

### Step 4: Configure Power BI Connection

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `${SQL_SERVER_HOST}`
4. Database: `LogiFleetPulse`
5. Data Connectivity mode: **Import** (for small datasets) or **DirectQuery** (for real-time)
6. Advanced options → SQL statement:

```sql
-- Use views for cleaner Power BI queries
SELECT 
    w.OperationKey,
    t.TimeStamp,
    t.[Year],
    t.[Quarter],
    t.[Month],
    g.NodeName AS Warehouse,
    g.City,
    g.Country,
    p.SKU,
    p.ProductName,
    p.Category,
    p.GravityScore,
    w.OperationType,
    w.Quantity,
    w.DwellTimeMinutes,
    w.CycleTimeMinutes
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
INNER JOIN DimProduct p ON w.ProductKey = p.ProductKey
WHERE t.TimeStamp >= DATEADD(MONTH, -6, GETDATE());
```

## Key DAX Measures for Power BI

### Warehouse Performance

```dax
// Total Dwell Time (Hours)
TotalDwellHours = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeMinutes] / 60
)

// Average Cycle Time by Operation
AvgCycleTime = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[CycleTimeMinutes]),
    ALLEXCEPT(FactWarehouseOperations, FactWarehouseOperations[OperationType])
)

// Inventory Turnover Rate
TurnoverRate = 
DIVIDE(
    CALCULATE(
        SUM(FactWarehouseOperations[Quantity]),
        FactWarehouseOperations[OperationType] = "Shipping"
    ),
    CALCULATE(
        AVERAGE(FactWarehouseOperations[Quantity]),
        FactWarehouseOperations[OperationType] = "Receiving"
    )
)

// High-Gravity Zone Utilization
HighGravityUtilization = 
CALCULATE(
    COUNT(FactWarehouseOperations[OperationKey]),
    DimProduct[GravityScore] > 7
) / COUNT(FactWarehouseOperations[OperationKey])
```

### Fleet Analytics

```dax
// Fleet Efficiency Score
FleetEfficiency = 
VAR TotalDistance = SUM(FactFleetTrips[DistanceKm])
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
VAR TotalTripTime = SUMX(
    FactFleetTrips,
    DATEDIFF(
        RELATED(DimTimeStart[TimeStamp]),
        RELATED(DimTimeEnd[TimeStamp]),
        MINUTE
    )
)
RETURN
    (TotalDistance / (TotalTripTime - TotalIdleTime)) * 100

// Fuel Consumption Rate (Liters per 100km)
FuelRate = 
DIVIDE(
    SUM(FactFleetTrips[FuelLiters]),
    SUM(FactFleetTrips[DistanceKm])
) * 100

// On-Time Delivery Percentage
OnTimeDelivery% = 
CALCULATE(
    COUNTROWS(FactFleetTrips),
    FactFleetTrips[TripStatus] = "Completed",
    FactFleetTrips[DelayReasonCode] = BLANK()
) / COUNTROWS(FactFleetTrips)
```

### Cross-Fact KPIs

```dax
// Dwell Time Impact on Delivery Delay
DwellDelayCorrelation = 
VAR HighDwellProducts = 
    CALCULATETABLE(
        VALUES(DimProduct[ProductKey]),
        FactWarehouseOperations[DwellTimeMinutes] > 72 * 60
    )
RETURN
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FactFleetTrips[TripStatus] = "Delayed",
        TREATAS(HighDwellProducts, DimProduct[ProductKey])
    )

// Cost Per Unit Shipped
CostPerUnit = 
VAR TotalFuelCost = SUM(FactFleetTrips[FuelLiters]) * 1.5  -- Assume $1.5/liter
VAR TotalUnitsShipped = 
    CALCULATE(
        SUM(FactWarehouseOperations[Quantity]),
        FactWarehouseOperations[OperationType] = "Shipping"
    )
RETURN
    DIVIDE(TotalFuelCost, TotalUnitsShipped, 0)
```

## Configuration

### Connection Settings

Store database credentials securely:

```json
{
  "data_sources": {
    "sql_server": {
      "host": "${SQL_SERVER_HOST}",
      "database": "LogiFleetPulse",
      "authentication": "sql_login",
      "username": "${SQL_USER}",
      "password": "${SQL_PASSWORD}"
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
  "refresh_schedule": {
    "warehouse_operations": "*/15 * * * *",
    "fleet_trips": "*/15 * * * *",
    "dimensions": "0 2 * * *"
  }
}
```

### Row-Level Security

```sql
-- Create role-based security
CREATE TABLE SecurityRoles (
    RoleID INT PRIMARY KEY IDENTITY(1,1),
    RoleName VARCHAR(50) NOT NULL,
    Username VARCHAR(100) NOT NULL,
    AllowedWarehouses VARCHAR(MAX), -- JSON array: ["WH001", "WH002"]
    AllowedRegions VARCHAR(MAX)
);

-- Create RLS function
CREATE FUNCTION dbo.fn_SecurityPredicate(@WarehouseKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS fn_SecurityPredicateResult
    WHERE 
        @WarehouseKey IN (
            SELECT g.GeographyKey
            FROM dbo.DimGeography g
            INNER JOIN dbo.SecurityRoles s ON g.NodeCode IN (
                SELECT value FROM OPENJSON(s.AllowedWarehouses)
            )
            WHERE s.Username = USER_NAME()
        )
        OR USER_NAME() = 'AdminUser';

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(WarehouseKey)
ON dbo.FactWarehouseOperations
WITH (STATE = ON);
```

## Common Patterns

### Pattern 1: Incremental ETL from External API

```python
import pyodbc
import requests
from datetime import datetime, timedelta
import os

# Connect to SQL Server
conn = pyodbc.connect(
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.getenv('SQL_SERVER_HOST')};"
    f"DATABASE=LogiFleetPulse;"
    f"UID={os.getenv('SQL_USER')};"
    f"PWD={os.getenv('SQL_PASSWORD')}"
)
cursor = conn.cursor()

# Get last load timestamp
cursor.execute("SELECT MAX(TimeStamp) FROM DimTime t INNER JOIN FactWarehouseOperations w ON t.TimeKey = w.TimeKey")
last_load = cursor.fetchone()[0] or datetime.now() - timedelta(days=30)

# Fetch new data from WMS API
response = requests.get(
    f"{os.getenv('WMS_API_ENDPOINT')}/operations",
    headers={"Authorization": f"Bearer {os.getenv('WMS_API_KEY')}"},
    params={"since": last_load.isoformat()}
)

operations = response.json()

# Batch insert
for op in operations:
    cursor.execute("""
        INSERT INTO FactWarehouseOperations 
        (TimeKey, WarehouseKey, ProductKey, OperationType, Quantity, DwellTimeMinutes, CycleTimeMinutes, ZoneCode)
        SELECT 
            t.TimeKey,
            g.GeographyKey,
            p.ProductKey,
            ?,
            ?,
            ?,
            ?,
            ?
        FROM DimTime t
        CROSS JOIN DimGeography g
        CROSS JOIN DimProduct p
        WHERE t.TimeStamp = ? AND g.NodeCode = ? AND p.SKU = ?
    """, (
        op['operation_type'],
        op['quantity'],
        op['dwell_minutes'],
        op['cycle_minutes'],
        op['zone_code'],
        op['timestamp'],
        op['warehouse_code'],
        op['sku']
    ))

conn.commit()
cursor.close()
conn.close()
```

### Pattern 2: Real-Time Fleet Monitoring

```sql
-- Create view for live fleet dashboard
CREATE VIEW vw_LiveFleetStatus AS
SELECT 
    f.VehicleID,
    f.RouteID,
    o.NodeName AS CurrentLocation,
    d.NodeName AS Destination,
    t.TimeStamp AS LastUpdate,
    f.LoadWeightKg,
    f.TripStatus,
    CASE 
        WHEN f.IdleTimeMinutes > 30 THEN 'Alert'
        WHEN f.IdleTimeMinutes > 15 THEN 'Warning'
        ELSE 'Normal'
    END AS IdleStatus,
    DATEDIFF(MINUTE, t.TimeStamp, GETDATE()) AS MinutesSinceUpdate
FROM FactFleetTrips f
INNER JOIN DimTime t ON f.TimeKeyStart = t.TimeKey
INNER JOIN DimGeography o ON f.OriginKey = o.GeographyKey
INNER JOIN DimGeography d ON f.DestinationKey = d.GeographyKey
WHERE f.TripStatus = 'InProgress'
  AND t.TimeStamp >= DATEADD(HOUR, -2, GETDATE());
```

### Pattern 3: Predictive Bottleneck Detection

```sql
-- Identify zones with increasing dwell time trend
WITH DwellTrend AS (
    SELECT 
        w.ZoneCode,
        p.Category,
        t.[Year],
        t.[Month],
        AVG(w.DwellTimeMinutes) AS AvgDwell,
        LAG(AVG(w.DwellTimeMinutes)) OVER (
            PARTITION BY w.ZoneCode, p.Category 
            ORDER BY t.[Year], t.[Month]
        ) AS PrevMonthDwell
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    INNER JOIN DimProduct p ON w.ProductKey = p.ProductKey
    WHERE t.TimeStamp >= DATEADD(MONTH, -6, GETDATE())
    GROUP BY w.ZoneCode, p.Category, t.[Year], t.[Month]
)
SELECT 
    ZoneCode,
    Category,
    AvgDwell,
    PrevMonthDwell,
    ((AvgDwell - PrevMonthDwell) / NULLIF(PrevMonthDwell, 0)) * 100 AS GrowthPercent
FROM DwellTrend
WHERE AvgDwell > PrevMonthDwell * 1.2  -- 20% increase
ORDER BY GrowthPercent DESC;
```

## Troubleshooting

### Issue: Power BI Refresh Timeout

**Cause**: Large fact tables exceeding 30-minute refresh window

**Solution**: Implement incremental refresh in Power BI

```m
// Power Query M code for incremental refresh
let
    Source = Sql.Database("${SQL_SERVER_HOST}", "LogiFleetPulse"),
    FilteredRows = Table.SelectRows(Source, each [TimeStamp] >= RangeStart and [TimeStamp] < RangeEnd)
in
    FilteredRows
```

Then configure incremental refresh policy:
- Archive data: 2 years
- Refresh data: Last 7 days
- Detect data changes: Yes (based on TimeStamp column)

### Issue: Slow Cross-Fact Queries

**Cause**: Missing indexes on foreign key columns

**Solution**: Add covering indexes

```sql
-- Analyze query execution plan
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- Add indexes for common join patterns
CREATE NONCLUSTERED INDEX IX_Fact_Composite 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (Quantity, DwellTimeMinutes);

CREATE NONCLUSTERED INDEX IX_Fleet_Composite
ON FactFleetTrips(TimeKeyStart, VehicleID)
INCLUDE (DistanceKm, FuelLiters, TripStatus);

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

### Issue: Time Dimension Gaps

**Cause**: Data arriving outside 15-minute bucket boundaries

**Solution**: Use FLOOR rounding in ETL

```sql
-- Correct approach for time key lookup
DECLARE @EventTime DATETIME2 = '2025-03-15 14:37:22';

SELECT TimeKey
FROM DimTime
WHERE TimeStamp = DATEADD(MINUTE, 
    (DATEDIFF(MINUTE, '2024-01-01 00:00:00', @EventTime) / 15) * 15,
    '2024-01-01 00:00:00'
);
```

### Issue: Row-Level Security Not Applied

**Cause**: Power BI using service principal with elevated permissions

**Solution**: Map Power BI roles to SQL users

```sql
-- In Power BI Desktop, create roles
-- Model → Manage Roles → Create "WarehouseUser"
-- DAX filter: [WarehouseCode] = USERNAME()

-- In SQL Server, create corresponding users
CREATE USER WarehouseUser_WH001 FOR LOGIN WarehouseUser_WH001;
EXEC sp_addrolemember 'db_datareader', 'WarehouseUser_WH001';

-- Apply RLS filter
CREATE FUNCTION dbo.fn_PowerBISecurityPredicate(@WarehouseKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS Result
    WHERE USER_NAME() LIKE 'WarehouseUser_%' 
      AND RIGHT(USER_NAME(), 5) IN (
          SELECT NodeCode 
          FROM dbo.DimGeography 
          WHERE GeographyKey = @WarehouseKey
      );
```

## Advanced Features

### Temporal Elasticity Simulation

```sql
-- What-if analysis: 95% warehouse capacity impact
CREATE PROCEDURE usp_SimulateCapacityImpact
    @TargetCapacityPercent DECIMAL(5,2)
AS
BEGIN
    -- Create simulation table
    SELECT 
        w.WarehouseKey,
        g.NodeName,
        COUNT(*) AS CurrentOperations,
        SUM(w.Quantity) AS CurrentVolume,
        AVG(w.DwellTimeMinutes) AS CurrentAvgDwell,
        -- Project new dwell time based on queuing theory
        AVG(w.DwellTimeMinutes) * (1 / (1 - @TargetCapacityPercent/100)) AS ProjectedAvgDwell,
        -- Estimate additional fleet trips needed
        CEILING(
            (SUM(w.Quantity) * (@TargetCapacityPercent / 100) - SUM(w.Quantity)) 
            / AVG(f.LoadWeightKg)
        ) AS AdditionalTripsNeeded
    FROM FactWarehouseOperations w
    INNER JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
    CROSS APPLY (
        SELECT AVG(LoadWeightKg) AS LoadWeightKg
        FROM FactFleetTrips
        WHERE OriginKey = w.WarehouseKey
    ) f
    GROUP BY w.WarehouseKey, g.NodeName;
END;
```

### Automated Alert System

```sql
-- Create alert configuration table
CREATE TABLE AlertRules (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName VARCHAR(100),
    MetricType VARCHAR(50),
    ThresholdValue DECIMAL(18,2),
    ComparisonOperator VARCHAR(10),
    NotificationEmail VARCHAR(200)
);

-- Stored procedure to check and send alerts
CREATE PROCEDURE usp_CheckAlerts
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check fleet idling threshold
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips
        WHERE IdleTimeMinutes > 
            (SELECT ThresholdValue FROM AlertRules WHERE MetricType = 'FleetIdleMinutes')
          AND TripStatus = 'InProgress'
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Multiple vehicles exceeding idle time threshold';
        -- Integrate with email service or Teams webhook
        -- EXEC msdb.dbo.sp_send_dbmail ...
    END;
    
    -- Check dwell time by zone
    IF EXISTS (
        SELECT ZoneCode
        FROM FactWarehouseOperations
        WHERE DwellTimeMinutes > 
            (SELECT ThresholdValue FROM AlertRules WHERE MetricType = 'DwellTimeMinutes')
        GROUP BY ZoneCode
        HAVING COUNT(*) > 5
    )
    BEGIN
        SET @AlertMessage = 'ALERT: High dwell time detected in multiple zones';
        -- Send notification
    END;
END;

-- Schedule via SQL Agent Job (runs every 15 minutes)
```

## Best Practices

1. **Partition Large Fact Tables** by month for faster queries:

```sql
ALTER TABLE FactWarehouseOperations
ADD CONSTRAINT PK_FactWarehouseOperations_Partitioned
PRIMARY KEY (OperationKey, TimeKey);

CREATE PARTITION SCHEME PS_Monthly
AS PARTITION PF_Monthly
TO ([FG_2024_01], [FG_2024_02], ...);
```

2. **Use Columnstore Indexes** for analytical queries:

```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_Fact_Columnstore
ON FactWarehouseOperations (TimeKey, WarehouseKey, ProductKey, Quantity, DwellTimeMinutes);
```

3. **Separate OLTP and OLAP** workloads using read-only replicas or snapshots

4. **Document DAX measures** with clear naming conventions and COMMENTS property

5. **Version control Power BI files** using PBIX → PBIT templates with parameterized connections
