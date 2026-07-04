---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing and fleet logistics analytics engine for supply chain intelligence
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy logistics data warehouse with Power BI"
  - "create multi-fact star schema for warehouse and fleet data"
  - "build supply chain KPI dashboard"
  - "configure LogiFleet analytics engine"
  - "implement warehouse gravity zones"
  - "set up fleet telemetry analytics"
  - "create cross-modal logistics intelligence"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a multi-modal logistics intelligence engine that combines:
- **MS SQL Server data warehouse** with multi-fact star schema
- **Power BI dashboards** for real-time logistics visualization
- **Warehouse operations analytics** (receiving, putaway, picking, packing, shipping)
- **Fleet telemetry integration** (GPS, fuel consumption, driver behavior)
- **Cross-fact KPI harmonization** linking inventory, routes, and performance metrics
- **Predictive bottleneck detection** and adaptive fleet triage

The platform unifies warehouse micro-operations, fleet telemetry, inventory aging, and external signals into a single semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- ODBC drivers for external data sources (if integrating WMS/TMS)
- Azure Synapse Analytics (optional, for big data enrichment)

### 1. Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    DateValue DATE NOT NULL,
    TimeValue TIME NOT NULL,
    HourOfDay INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayName NVARCHAR(20),
    WeekOfYear INT NOT NULL,
    MonthNumber INT NOT NULL,
    MonthName NVARCHAR(20),
    QuarterNumber INT NOT NULL,
    Year INT NOT NULL,
    FiscalPeriod NVARCHAR(10),
    IsWeekend BIT NOT NULL,
    IsHoliday BIT DEFAULT 0
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID NVARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50) NOT NULL, -- 'Warehouse', 'Distribution Center', 'Route Node'
    AddressLine1 NVARCHAR(200),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    PostalCode NVARCHAR(20),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone NVARCHAR(50),
    IsActive BIT DEFAULT 1
);

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(100) NOT NULL UNIQUE,
    ProductName NVARCHAR(300),
    Category NVARCHAR(100),
    Subcategory NVARCHAR(100),
    Weight DECIMAL(10,2), -- kg
    Volume DECIMAL(10,2), -- cubic meters
    Fragility NVARCHAR(20), -- 'Low', 'Medium', 'High'
    TemperatureControl NVARCHAR(50), -- 'Ambient', 'Refrigerated', 'Frozen'
    VelocityScore DECIMAL(5,2), -- Pick frequency score
    ValueScore DECIMAL(5,2), -- Revenue impact score
    GravityZone NVARCHAR(50), -- 'High-Gravity', 'Medium-Gravity', 'Low-Gravity'
    GravityScore AS (VelocityScore * 0.6 + ValueScore * 0.4) PERSISTED,
    IsActive BIT DEFAULT 1
);

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID NVARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(300),
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectPercentage DECIMAL(5,2),
    OnTimeDeliveryRate DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2),
    ReliabilityRating AS (
        (OnTimeDeliveryRate * 0.4) + 
        ((100 - DefectPercentage) * 0.3) + 
        (ComplianceScore * 0.3)
    ) PERSISTED,
    IsActive BIT DEFAULT 1
);

CREATE TABLE DimVehicle (
    VehicleKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID NVARCHAR(50) NOT NULL UNIQUE,
    VehicleType NVARCHAR(50), -- 'Van', 'Truck', 'Semi', 'Refrigerated'
    Capacity DECIMAL(10,2), -- kg
    FuelType NVARCHAR(50),
    YearManufactured INT,
    CurrentMileage INT,
    MaintenanceStatus NVARCHAR(50),
    IsActive BIT DEFAULT 1
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT NULL FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType NVARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OrderID NVARCHAR(100),
    Quantity INT,
    DwellTimeMinutes INT, -- Time in current zone
    CycleTimeMinutes INT, -- Time to complete operation
    StorageZone NVARCHAR(100),
    OperatorID NVARCHAR(50),
    ErrorOccurred BIT DEFAULT 0,
    ErrorType NVARCHAR(100),
    ProcessingCost DECIMAL(10,2)
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT NOT NULL FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TripID NVARCHAR(100) NOT NULL,
    DriverID NVARCHAR(50),
    DistanceKM DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKG DECIMAL(10,2),
    OnTimeDelivery BIT,
    DelayMinutes INT,
    DelayReason NVARCHAR(200),
    WeatherCondition NVARCHAR(100),
    TrafficLevel NVARCHAR(50),
    TripCost DECIMAL(10,2)
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT NULL FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT NULL FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    TransferID NVARCHAR(100),
    Quantity INT,
    TransferTimeMinutes INT,
    TemperatureCompliant BIT,
    DamageReported BIT
);

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Geography ON FactWarehouseOperations(GeographyKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Origin ON FactFleetTrips(OriginGeographyKey);
```

### 2. Populate Time Dimension

```sql
-- Stored procedure to populate time dimension with 15-minute granularity
CREATE PROCEDURE PopulateTimeDimension
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    DECLARE @TimeKey INT;
    
    WHILE @CurrentDateTime < @EndDate
    BEGIN
        SET @TimeKey = CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT);
        
        INSERT INTO DimTime (
            TimeKey, FullDateTime, DateValue, TimeValue,
            HourOfDay, DayOfWeek, DayName, WeekOfYear,
            MonthNumber, MonthName, QuarterNumber, Year,
            FiscalPeriod, IsWeekend
        )
        VALUES (
            @TimeKey,
            @CurrentDateTime,
            CAST(@CurrentDateTime AS DATE),
            CAST(@CurrentDateTime AS TIME),
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            DATEPART(MONTH, @CurrentDateTime),
            DATENAME(MONTH, @CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            DATEPART(YEAR, @CurrentDateTime),
            'FY' + CAST(YEAR(@CurrentDateTime) AS NVARCHAR(4)) + 'Q' + CAST(DATEPART(QUARTER, @CurrentDateTime) AS NVARCHAR(1)),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1,7) THEN 1 ELSE 0 END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

-- Execute to populate 2 years of time data
EXEC PopulateTimeDimension '2025-01-01', '2027-01-01';
```

### 3. Configure Data Source Connections

Create a configuration file for external integrations:

```sql
-- Create external data source configuration table
CREATE TABLE DataSourceConfig (
    ConfigKey NVARCHAR(100) PRIMARY KEY,
    ConfigValue NVARCHAR(MAX),
    Description NVARCHAR(500),
    LastUpdated DATETIME2 DEFAULT GETDATE()
);

-- Insert connection configurations (use environment variables in production)
INSERT INTO DataSourceConfig (ConfigKey, ConfigValue, Description)
VALUES 
    ('WMS_API_ENDPOINT', '${WMS_API_URL}', 'Warehouse Management System API endpoint'),
    ('TELEMETRY_API_ENDPOINT', '${TELEMETRY_API_URL}', 'Fleet telemetry API endpoint'),
    ('WEATHER_API_ENDPOINT', '${WEATHER_API_URL}', 'Weather data API endpoint'),
    ('REFRESH_INTERVAL_MINUTES', '15', 'Data refresh interval in minutes');
```

### 4. Import Power BI Template

```bash
# Download the repository
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse

# Open the Power BI template file
# LogiFleet_Pulse_Master.pbit
```

In Power BI Desktop:
1. Open `LogiFleet_Pulse_Master.pbit`
2. Enter SQL Server connection details when prompted
3. Configure row-level security roles
4. Publish to Power BI Service (optional)

## Key SQL Patterns

### Cross-Fact KPI Query: Dwell Time vs Fleet Idle Cost

```sql
-- Analyze relationship between warehouse dwell time and fleet idle costs
WITH WarehouseDwell AS (
    SELECT 
        t.DateValue,
        p.GravityZone,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations w
    JOIN DimTime t ON w.TimeKey = t.TimeKey
    JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE t.DateValue >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY t.DateValue, p.GravityZone
),
FleetIdle AS (
    SELECT 
        t.DateValue,
        AVG(f.IdleTimeMinutes * 1.0 / NULLIF(f.DurationMinutes, 0)) AS AvgIdlePercent,
        SUM(f.IdleTimeMinutes * 0.5) AS TotalIdleCost -- $0.50 per minute assumption
    FROM FactFleetTrips f
    JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.DateValue >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY t.DateValue
)
SELECT 
    w.DateValue,
    w.GravityZone,
    w.AvgDwellTime,
    w.OperationCount,
    f.AvgIdlePercent * 100 AS IdlePercentage,
    f.TotalIdleCost,
    -- Correlation indicator
    CASE 
        WHEN w.AvgDwellTime > 240 AND f.AvgIdlePercent > 0.15 
        THEN 'High Risk - Both Elevated'
        WHEN w.AvgDwellTime > 240 OR f.AvgIdlePercent > 0.15 
        THEN 'Medium Risk - One Elevated'
        ELSE 'Normal Operation'
    END AS RiskLevel
FROM WarehouseDwell w
LEFT JOIN FleetIdle f ON w.DateValue = f.DateValue
ORDER BY w.DateValue DESC, w.GravityZone;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products that should be moved to different gravity zones
WITH ProductPerformance AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        p.GravityZone AS CurrentZone,
        COUNT(*) AS PickCount,
        AVG(w.CycleTimeMinutes) AS AvgCycleTime,
        SUM(w.Quantity * p.ValueScore) AS TotalRevenueImpact
    FROM FactWarehouseOperations w
    JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE w.OperationType = 'Picking'
        AND w.TimeKey >= CAST(FORMAT(DATEADD(MONTH, -1, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY p.ProductKey, p.SKU, p.ProductName, p.GravityZone
)
SELECT 
    SKU,
    ProductName,
    CurrentZone,
    PickCount,
    AvgCycleTime,
    TotalRevenueImpact,
    CASE 
        WHEN PickCount > 100 AND TotalRevenueImpact > 10000 AND CurrentZone != 'High-Gravity'
        THEN 'Move to High-Gravity Zone'
        WHEN PickCount < 20 AND CurrentZone = 'High-Gravity'
        THEN 'Move to Low-Gravity Zone'
        WHEN PickCount BETWEEN 20 AND 100 AND CurrentZone IN ('High-Gravity', 'Low-Gravity')
        THEN 'Move to Medium-Gravity Zone'
        ELSE 'Keep in Current Zone'
    END AS Recommendation,
    CASE 
        WHEN PickCount > 100 AND TotalRevenueImpact > 10000 AND CurrentZone != 'High-Gravity'
        THEN AvgCycleTime * 0.3 -- Estimated 30% time savings
        ELSE 0
    END AS EstimatedTimeSavingsMinutes
FROM ProductPerformance
WHERE AvgCycleTime > 0
ORDER BY TotalRevenueImpact DESC;
```

### Predictive Bottleneck Detection

```sql
-- Detect potential bottlenecks using recent trends
CREATE PROCEDURE DetectBottlenecks
AS
BEGIN
    -- Warehouse bottlenecks
    SELECT 
        'Warehouse' AS Area,
        g.LocationName,
        w.StorageZone,
        COUNT(*) AS OperationCount,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        STDEV(w.DwellTimeMinutes) AS DwellTimeStdDev,
        SUM(CASE WHEN w.ErrorOccurred = 1 THEN 1 ELSE 0 END) AS ErrorCount,
        CASE 
            WHEN AVG(w.DwellTimeMinutes) > 180 AND COUNT(*) > 50
            THEN 'Critical - High Volume + High Dwell'
            WHEN STDEV(w.DwellTimeMinutes) > 60
            THEN 'Warning - High Variability'
            ELSE 'Normal'
        END AS BottleneckStatus
    FROM FactWarehouseOperations w
    JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateValue >= DATEADD(DAY, -7, GETDATE())
    GROUP BY g.LocationName, w.StorageZone
    HAVING AVG(w.DwellTimeMinutes) > 120 OR STDEV(w.DwellTimeMinutes) > 40
    
    UNION ALL
    
    -- Fleet bottlenecks
    SELECT 
        'Fleet' AS Area,
        g.LocationName,
        NULL AS StorageZone,
        COUNT(*) AS TripCount,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        STDEV(f.IdleTimeMinutes) AS IdleTimeStdDev,
        SUM(CASE WHEN f.OnTimeDelivery = 0 THEN 1 ELSE 0 END) AS DelayCount,
        CASE 
            WHEN AVG(f.IdleTimeMinutes * 1.0 / NULLIF(f.DurationMinutes, 0)) > 0.20
            THEN 'Critical - High Idle Time'
            WHEN SUM(CASE WHEN f.OnTimeDelivery = 0 THEN 1 ELSE 0 END) * 1.0 / COUNT(*) > 0.15
            THEN 'Warning - High Delay Rate'
            ELSE 'Normal'
        END AS BottleneckStatus
    FROM FactFleetTrips f
    JOIN DimGeography g ON f.DestinationGeographyKey = g.GeographyKey
    JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.DateValue >= DATEADD(DAY, -7, GETDATE())
    GROUP BY g.LocationName
    HAVING AVG(f.IdleTimeMinutes) > 30 OR SUM(CASE WHEN f.OnTimeDelivery = 0 THEN 1 ELSE 0 END) > 5
    
    ORDER BY BottleneckStatus DESC, AvgDwellTime DESC;
END;
GO

-- Execute bottleneck detection
EXEC DetectBottlenecks;
```

### Adaptive Fleet Triage Engine

```sql
-- Prioritize vehicle maintenance based on revenue impact
WITH VehicleRisk AS (
    SELECT 
        v.VehicleKey,
        v.VehicleID,
        v.VehicleType,
        v.CurrentMileage,
        v.MaintenanceStatus,
        COUNT(DISTINCT f.TripKey) AS RecentTrips,
        AVG(f.LoadWeightKG) AS AvgLoadWeight,
        SUM(f.TripCost) AS TotalRevenue,
        AVG(f.FuelConsumedLiters * 1.0 / NULLIF(f.DistanceKM, 0)) AS AvgFuelEfficiency,
        SUM(CASE WHEN f.OnTimeDelivery = 0 THEN 1 ELSE 0 END) AS DelayCount
    FROM DimVehicle v
    LEFT JOIN FactFleetTrips f ON v.VehicleKey = f.VehicleKey
    JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.DateValue >= DATEADD(DAY, -30, GETDATE())
        AND v.IsActive = 1
    GROUP BY v.VehicleKey, v.VehicleID, v.VehicleType, v.CurrentMileage, v.MaintenanceStatus
)
SELECT 
    VehicleID,
    VehicleType,
    CurrentMileage,
    MaintenanceStatus,
    RecentTrips,
    AvgLoadWeight,
    TotalRevenue,
    AvgFuelEfficiency,
    DelayCount,
    -- Triage score: higher = more urgent
    (
        (CASE WHEN CurrentMileage > 150000 THEN 30 ELSE 0 END) +
        (CASE WHEN MaintenanceStatus = 'Overdue' THEN 40 ELSE 0 END) +
        (CASE WHEN AvgFuelEfficiency > 0.12 THEN 20 ELSE 0 END) + -- Poor fuel efficiency
        (DelayCount * 5) +
        (TotalRevenue / 1000) -- Revenue impact multiplier
    ) AS TriageScore,
    CASE 
        WHEN MaintenanceStatus = 'Overdue' AND TotalRevenue > 50000
        THEN 'Critical - High Revenue Vehicle Overdue'
        WHEN MaintenanceStatus = 'Overdue'
        THEN 'High - Maintenance Overdue'
        WHEN AvgFuelEfficiency > 0.12 AND RecentTrips > 10
        THEN 'Medium - Poor Fuel Efficiency'
        ELSE 'Low - Monitor'
    END AS MaintenancePriority
FROM VehicleRisk
WHERE MaintenanceStatus IN ('Overdue', 'Due Soon') OR AvgFuelEfficiency > 0.10
ORDER BY TriageScore DESC;
```

## Automated Alerting

```sql
-- Create alert configuration table
CREATE TABLE AlertConfig (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName NVARCHAR(200) NOT NULL,
    AlertType NVARCHAR(50) NOT NULL, -- 'Warehouse', 'Fleet', 'Cross-Fact'
    Threshold DECIMAL(10,2),
    RecipientEmails NVARCHAR(MAX), -- Comma-separated
    IsActive BIT DEFAULT 1
);

-- Insert sample alert rules
INSERT INTO AlertConfig (AlertName, AlertType, Threshold, RecipientEmails)
VALUES 
    ('High Dwell Time Alert', 'Warehouse', 180, '${WAREHOUSE_MANAGER_EMAIL}'),
    ('Fleet Idle Time Alert', 'Fleet', 15, '${FLEET_MANAGER_EMAIL}'),
    ('Cross-Dock Delay Alert', 'Cross-Fact', 30, '${OPERATIONS_MANAGER_EMAIL}');

-- Stored procedure to check and send alerts
CREATE PROCEDURE CheckAndSendAlerts
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check warehouse dwell time alerts
    IF EXISTS (
        SELECT 1 
        FROM FactWarehouseOperations w
        JOIN DimTime t ON w.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -1, GETDATE())
            AND w.DwellTimeMinutes > (SELECT Threshold FROM AlertConfig WHERE AlertName = 'High Dwell Time Alert')
    )
    BEGIN
        SET @AlertMessage = 'Alert: High dwell time detected in warehouse operations. Immediate review required.';
        -- In production, integrate with email service or Azure Logic Apps
        PRINT @AlertMessage;
    END
    
    -- Check fleet idle time alerts
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips f
        JOIN DimTime t ON f.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -1, GETDATE())
            AND (f.IdleTimeMinutes * 100.0 / NULLIF(f.DurationMinutes, 0)) > (SELECT Threshold FROM AlertConfig WHERE AlertName = 'Fleet Idle Time Alert')
    )
    BEGIN
        SET @AlertMessage = 'Alert: High fleet idle time detected. Route optimization recommended.';
        PRINT @AlertMessage;
    END
END;
GO

-- Schedule this procedure to run every 15 minutes via SQL Server Agent
```

## Power BI DAX Measures

Key DAX measures to create in Power BI:

```dax
// Total Operations
Total Operations = COUNTROWS(FactWarehouseOperations)

// Average Dwell Time
Avg Dwell Time = 
AVERAGE(FactWarehouseOperations[DwellTimeMinutes])

// Dwell Time by Gravity Zone
Dwell Time by Zone = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    ALLEXCEPT(DimProductGravity, DimProductGravity[GravityZone])
)

// Fleet Efficiency
Fleet Efficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKM]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)

// On-Time Delivery Rate
On-Time Delivery % = 
DIVIDE(
    COUNTROWS(FILTER(FactFleetTrips, FactFleetTrips[OnTimeDelivery] = TRUE)),
    COUNTROWS(FactFleetTrips),
    0
) * 100

// Cross-Fact KPI: Cost per Unit Shipped
Cost per Unit = 
DIVIDE(
    SUM(FactWarehouseOperations[ProcessingCost]) + SUM(FactFleetTrips[TripCost]),
    SUM(FactWarehouseOperations[Quantity]),
    0
)

// Bottleneck Risk Score
Bottleneck Risk = 
VAR DwellScore = IF(AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) > 180, 40, 0)
VAR IdleScore = IF(AVERAGE(FactFleetTrips[IdleTimeMinutes]) > 30, 30, 0)
VAR ErrorScore = COUNTROWS(FILTER(FactWarehouseOperations, FactWarehouseOperations[ErrorOccurred] = TRUE)) * 5
RETURN DwellScore + IdleScore + ErrorScore

// Time Intelligence: Month-over-Month Growth
MoM Growth % = 
VAR CurrentMonth = SUM(FactWarehouseOperations[Quantity])
VAR PreviousMonth = 
    CALCULATE(
        SUM(FactWarehouseOperations[Quantity]),
        DATEADD(DimTime[DateValue], -1, MONTH)
    )
RETURN 
    DIVIDE(CurrentMonth - PreviousMonth, PreviousMonth, 0) * 100
```

## Configuration Best Practices

### Row-Level Security (RLS)

```dax
// Create role: Warehouse Manager
// Filter DimGeography table
[LocationType] = "Warehouse" && [Country] = USERPRINCIPALNAME()

// Create role: Fleet Manager  
// Filter DimVehicle table
[VehicleType] IN {"Van", "Truck", "Semi"}

// Create role: Executive
// No filters - full access
```

### Incremental Data Refresh

Configure in Power BI Service for large datasets:

```
RangeStart: 01/01/2024
RangeEnd: Current Date
Refresh rows where:
    DimTime[DateValue] >= RangeStart
    AND DimTime[DateValue] < RangeEnd
Archive rows older than: 2 years
```

### Performance Optimization

```sql
-- Create columnstore indexes for fact tables (SQL Server 2016+)
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouse_Columnstore
ON FactWarehouseOperations (
    TimeKey, GeographyKey, ProductKey, OperationType,
    Quantity, DwellTimeMinutes, CycleTimeMinutes
);

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleet_Columnstore
ON FactFleetTrips (
    TimeKey, VehicleKey, OriginGeographyKey, DestinationGeographyKey,
    DistanceKM, DurationMinutes, IdleTimeMinutes, FuelConsumedLiters
);

-- Partition fact tables by time for better performance
CREATE PARTITION FUNCTION PF_TimeKey (INT)
AS RANGE RIGHT FOR VALUES (
    202501010000, 202504010000, 202507010000, 202510010000,
    202601010000, 202604010000
);

CREATE PARTITION SCHEME PS_TimeKey
AS PARTITION PF_TimeKey
ALL TO ([PRIMARY]);

-- Apply partitioning (requires recreating table)
CREATE TABLE FactWarehouseOperations_Partitioned (
    -- ... same schema as FactWarehouseOperations
) ON PS_TimeKey(TimeKey);
```

## Common Troubleshooting

### Issue: Power BI Refresh Fails

```sql
-- Check for orphaned records
SELECT 'Orphaned Products' AS IssueType, COUNT(*) AS Count
FROM FactWarehouseOperations w
LEFT JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE p.ProductKey IS NULL

UNION ALL

SELECT 'Orphaned Geographies', COUNT(*)
FROM FactWarehouseOperations w
LEFT JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
WHERE g.GeographyKey IS NULL;

-- Fix orphaned records
DELETE FROM FactWarehouseOperations
WHERE ProductKey NOT IN (SELECT ProductKey FROM DimProductGravity);
```

### Issue: Slow Query Performance

```
