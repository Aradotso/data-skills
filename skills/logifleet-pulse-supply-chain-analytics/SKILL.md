---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI data warehouse for logistics, fleet management, and supply chain intelligence with multi-fact star schema
triggers:
  - set up logistics analytics dashboard
  - configure supply chain data warehouse
  - implement fleet management analytics
  - create warehouse operations reporting
  - build logistics intelligence platform
  - design multi-fact star schema for supply chain
  - integrate power bi with logistics data
  - optimize warehouse and fleet KPIs
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is an advanced data warehousing and analytics platform for logistics operations. It combines MS SQL Server for data storage and Power BI for visualization, implementing a multi-fact star schema that harmonizes warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer for real-time supply chain intelligence.

## What This Project Does

- **Unified Data Model**: Multi-fact star schema linking warehouse, fleet, and supply chain data
- **Real-Time Dashboarding**: Power BI dashboards with 15-minute refresh cycles
- **Cross-Fact KPI Analysis**: Correlate metrics across warehousing, fleet, and inventory domains
- **Predictive Analytics**: Bottleneck detection and maintenance prioritization
- **Spatial Optimization**: Warehouse "gravity zones" based on pick frequency and item value
- **Role-Based Access**: Row-level security for multi-team deployment

## Installation and Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Database Deployment

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the schema**:
```sql
-- Connect to your SQL Server instance and create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the schema creation scripts (example structure)
-- Execute in order: dimensions, facts, views, stored procedures
```

3. **Configure data sources**:
```json
{
  "connections": {
    "wms_database": "Server=${WMS_SQL_SERVER};Database=${WMS_DATABASE};User Id=${SQL_USER};Password=${SQL_PASSWORD};",
    "fleet_api": "https://api.fleetmanagement.example.com",
    "api_key": "${FLEET_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

### Power BI Setup

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter your SQL Server connection details when prompted
3. Configure refresh schedules in Power BI Service (if publishing)

## Core Data Model

### Dimension Tables

```sql
-- Time dimension with 15-minute granularity
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    [Date] DATE NOT NULL,
    [Hour] TINYINT NOT NULL,
    [Minute] TINYINT NOT NULL,
    QuarterHourBucket TINYINT NOT NULL,
    DayOfWeek NVARCHAR(10),
    IsWeekend BIT,
    FiscalPeriod NVARCHAR(20)
);

-- Product with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU NVARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity × value × fragility
    PickFrequencyRank INT,
    OptimalZoneAssignment NVARCHAR(20)
);

-- Geographic hierarchy
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationCode NVARCHAR(20) UNIQUE,
    LocationName NVARCHAR(100),
    LocationType NVARCHAR(20), -- Warehouse, RouteNode, Distribution Center
    Region NVARCHAR(50),
    Country NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);

-- Supplier reliability metrics
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY,
    SupplierCode NVARCHAR(20) UNIQUE,
    SupplierName NVARCHAR(200),
    LeadTimeVarianceDays DECIMAL(5,2),
    DefectPercentage DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2),
    LastEvaluationDate DATE
);
```

### Fact Tables

```sql
-- Warehouse operations fact
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    LocationKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OperationType NVARCHAR(20), -- Receiving, Putaway, Picking, Packing, Shipping
    DwellTimeMinutes INT,
    PickingCycleSeconds INT,
    PackingCycleSeconds INT,
    QuantityHandled INT,
    ErrorCount INT
);

-- Fleet trips fact
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY,
    VehicleKey INT FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    DriverKey INT FOREIGN KEY REFERENCES DimDriver(DriverKey),
    StartTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    EndTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginLocationKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationLocationKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    DistanceKilometers DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    MaintenanceAlertFlag BIT
);

-- Cross-dock operations
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundLocationKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OutboundLocationKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TransferTimeMinutes INT,
    QuantityTransferred INT
);
```

## Key SQL Queries and Patterns

### Cross-Fact KPI Analysis

```sql
-- Correlate dwell time with fleet idling for specific products
WITH WarehouseDwell AS (
    SELECT 
        wo.ProductKey,
        AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
        p.SKU,
        p.GravityScore
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.[Date] >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY wo.ProductKey, p.SKU, p.GravityScore
),
FleetIdle AS (
    SELECT 
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        g.Region
    FROM FactFleetTrips ft
    INNER JOIN DimGeography g ON ft.DestinationLocationKey = g.GeographyKey
    INNER JOIN DimTime t ON ft.EndTimeKey = t.TimeKey
    WHERE t.[Date] >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY g.Region
)
SELECT 
    wd.SKU,
    wd.AvgDwellTime,
    wd.GravityScore,
    fi.AvgIdleTime,
    fi.Region
FROM WarehouseDwell wd
CROSS JOIN FleetIdle fi
WHERE wd.GravityScore < 50 -- Low gravity = slow movers
ORDER BY wd.AvgDwellTime DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products that should be reassigned to different zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZoneAssignment AS CurrentZone,
    CASE 
        WHEN p.GravityScore >= 80 THEN 'HighGravity_DockProximity'
        WHEN p.GravityScore >= 50 THEN 'MediumGravity_MidWarehouse'
        ELSE 'LowGravity_BackStorage'
    END AS RecommendedZone,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(*) AS PickCount
FROM DimProductGravity p
INNER JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.[Date] >= DATEADD(MONTH, -3, GETDATE())
    AND wo.OperationType = 'Picking'
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.OptimalZoneAssignment
HAVING AVG(wo.DwellTimeMinutes) > 120 -- Items sitting too long
ORDER BY p.GravityScore DESC;
```

### Predictive Bottleneck Detection

```sql
-- Stored procedure for bottleneck scoring
CREATE PROCEDURE sp_CalculateBottleneckIndex
AS
BEGIN
    WITH TimeSeriesData AS (
        SELECT 
            t.[Date],
            t.[Hour],
            g.LocationName,
            COUNT(*) AS OperationCount,
            AVG(wo.DwellTimeMinutes) AS AvgDwell,
            STDEV(wo.DwellTimeMinutes) AS StdDevDwell
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        INNER JOIN DimGeography g ON wo.LocationKey = g.GeographyKey
        WHERE t.[Date] >= DATEADD(DAY, -30, GETDATE())
        GROUP BY t.[Date], t.[Hour], g.LocationName
    ),
    AnomalyScores AS (
        SELECT 
            *,
            CASE 
                WHEN AvgDwell > (SELECT AVG(AvgDwell) + 2 * STDEV(AvgDwell) FROM TimeSeriesData) 
                THEN 1 
                ELSE 0 
            END AS IsAnomaly
        FROM TimeSeriesData
    )
    SELECT 
        LocationName,
        [Hour],
        SUM(IsAnomaly) AS AnomalyCount,
        AVG(AvgDwell) AS AvgDwellTime,
        RANK() OVER (ORDER BY SUM(IsAnomaly) DESC) AS BottleneckRank
    FROM AnomalyScores
    GROUP BY LocationName, [Hour]
    ORDER BY BottleneckRank;
END;
GO
```

### Incremental Data Loading

```sql
-- Stored procedure for incremental ETL
CREATE PROCEDURE sp_LoadWarehouseOperationsIncremental
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new records from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, LocationKey, OperationType, 
        DwellTimeMinutes, PickingCycleSeconds, PackingCycleSeconds, 
        QuantityHandled, ErrorCount
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        stg.OperationType,
        stg.DwellTimeMinutes,
        stg.PickingCycleSeconds,
        stg.PackingCycleSeconds,
        stg.QuantityHandled,
        stg.ErrorCount
    FROM StagingWarehouseOps stg
    INNER JOIN DimTime t ON stg.OperationDateTime = t.FullDateTime
    INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
    INNER JOIN DimGeography g ON stg.LocationCode = g.LocationCode
    WHERE stg.OperationDateTime > @LastLoadDateTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f
            WHERE f.TimeKey = t.TimeKey 
                AND f.ProductKey = p.ProductKey
                AND f.LocationKey = g.GeographyKey
        );
    
    -- Update LastLoadDateTime tracking
    UPDATE ETLControl
    SET LastLoadDateTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
GO
```

## Power BI DAX Measures

### Cross-Fact Composite KPIs

```dax
// Fleet Fuel Efficiency per Warehouse Operation
FuelPerOperation = 
DIVIDE(
    SUM(FactFleetTrips[FuelConsumedLiters]),
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationKey]),
        USERELATIONSHIP(DimTime[TimeKey], FactWarehouseOperations[TimeKey])
    ),
    0
)

// Weighted Gravity Score by Region
WeightedGravityScore = 
SUMX(
    SUMMARIZE(
        FactWarehouseOperations,
        DimProductGravity[GravityScore],
        DimGeography[Region],
        "PickCount", COUNT(FactWarehouseOperations[OperationKey])
    ),
    [GravityScore] * [PickCount]
) / SUM(FactWarehouseOperations[OperationKey])

// Bottleneck Probability Index
BottleneckProbability = 
VAR CurrentDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR HistoricalAvg = CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATESINPERIOD(DimTime[Date], LASTDATE(DimTime[Date]), -90, DAY)
)
VAR HistoricalStdDev = CALCULATE(
    STDEV.P(FactWarehouseOperations[DwellTimeMinutes]),
    DATESINPERIOD(DimTime[Date], LASTDATE(DimTime[Date]), -90, DAY)
)
RETURN
IF(
    CurrentDwell > HistoricalAvg + (2 * HistoricalStdDev),
    1,
    0
)
```

## Configuration and Security

### Row-Level Security (RLS)

```dax
// Power BI RLS for Regional Access
[Region] = USERNAME()

// Or for more complex scenarios:
VAR UserEmail = USERPRINCIPALNAME()
VAR UserRegion = LOOKUPVALUE(
    DimUserRegionMapping[Region],
    DimUserRegionMapping[Email],
    UserEmail
)
RETURN
    DimGeography[Region] = UserRegion || UserEmail = "admin@company.com"
```

### Automated Alerting

```sql
-- Create alert trigger
CREATE PROCEDURE sp_CheckKPIThresholds
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(500);
    
    -- Check fleet idling threshold
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips ft
        INNER JOIN DimTime t ON ft.EndTimeKey = t.TimeKey
        WHERE t.[Date] = CAST(GETDATE() AS DATE)
        GROUP BY ft.VehicleKey
        HAVING AVG(CAST(ft.IdleTimeMinutes AS FLOAT) / NULLIF(DATEDIFF(MINUTE, StartTimeKey, EndTimeKey), 0)) > 0.15
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idling time exceeded 15% threshold';
        -- Send via SQL Server Database Mail or log to alert table
        INSERT INTO AlertLog (AlertDateTime, AlertType, Message)
        VALUES (GETDATE(), 'FleetIdling', @AlertMessage);
    END
    
    -- Check warehouse dwell time anomalies
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.[Date] = CAST(GETDATE() AS DATE)
        GROUP BY wo.LocationKey
        HAVING AVG(wo.DwellTimeMinutes) > (
            SELECT AVG(DwellTimeMinutes) + 2 * STDEV(DwellTimeMinutes)
            FROM FactWarehouseOperations
        )
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Warehouse dwell time anomaly detected';
        INSERT INTO AlertLog (AlertDateTime, AlertType, Message)
        VALUES (GETDATE(), 'DwellAnomaly', @AlertMessage);
    END
END;
GO

-- Schedule via SQL Server Agent
-- Or use Power Automate with SQL connector
```

## Common Patterns

### Multi-Fact Time Intelligence

```dax
// Compare warehouse operations vs fleet trips over time
OperationsVsTrips = 
VAR WarehouseOps = COUNT(FactWarehouseOperations[OperationKey])
VAR FleetTrips = CALCULATE(
    COUNT(FactFleetTrips[TripKey]),
    USERELATIONSHIP(DimTime[TimeKey], FactFleetTrips[StartTimeKey])
)
RETURN
DIVIDE(WarehouseOps, FleetTrips, 0)
```

### Dynamic Zone Recommendations

```sql
-- View for real-time zone recommendations
CREATE VIEW vw_ZoneRecommendations AS
SELECT 
    p.SKU,
    p.GravityScore,
    p.OptimalZoneAssignment AS CurrentZone,
    CASE 
        WHEN recent.AvgPicksPerDay > 50 AND p.GravityScore < 70 
        THEN 'UpgradeToHighGravity'
        WHEN recent.AvgPicksPerDay < 5 AND p.GravityScore > 40 
        THEN 'DowngradeToLowGravity'
        ELSE 'NoChange'
    END AS Recommendation
FROM DimProductGravity p
LEFT JOIN (
    SELECT 
        ProductKey,
        COUNT(*) / 30.0 AS AvgPicksPerDay
    FROM FactWarehouseOperations
    WHERE TimeKey >= (SELECT MAX(TimeKey) - 43200 FROM DimTime) -- Last 30 days
    GROUP BY ProductKey
) recent ON p.ProductKey = recent.ProductKey;
```

## Troubleshooting

### Slow Dashboard Refresh

```sql
-- Add indexes for common query patterns
CREATE NONCLUSTERED INDEX IX_FactWarehouse_TimeProduct 
ON FactWarehouseOperations (TimeKey, ProductKey) 
INCLUDE (DwellTimeMinutes, QuantityHandled);

CREATE NONCLUSTERED INDEX IX_FactFleet_TimeVehicle
ON FactFleetTrips (StartTimeKey, VehicleKey)
INCLUDE (IdleTimeMinutes, FuelConsumedLiters);

-- Partition large fact tables by date
ALTER PARTITION SCHEME PS_ByMonth 
NEXT USED [PRIMARY];
```

### Incorrect Cross-Fact Relationships

```dax
// Force correct relationship in DAX
FleetMetricsForWarehouseDate = 
CALCULATE(
    [FleetMeasure],
    USERELATIONSHIP(DimTime[TimeKey], FactFleetTrips[StartTimeKey]),
    TREATAS(VALUES(FactWarehouseOperations[TimeKey]), DimTime[TimeKey])
)
```

### Missing Data in Dimensions

```sql
-- Validate dimension integrity
SELECT 
    'FactWarehouseOperations' AS FactTable,
    COUNT(*) AS OrphanedRecords
FROM FactWarehouseOperations f
LEFT JOIN DimProductGravity p ON f.ProductKey = p.ProductKey
WHERE p.ProductKey IS NULL

UNION ALL

SELECT 
    'FactFleetTrips',
    COUNT(*)
FROM FactFleetTrips f
LEFT JOIN DimGeography g ON f.OriginLocationKey = g.GeographyKey
WHERE g.GeographyKey IS NULL;
```

## Environment Variables Reference

Required environment variables for configuration:

- `${WMS_SQL_SERVER}` - Warehouse Management System SQL Server address
- `${WMS_DATABASE}` - WMS database name
- `${SQL_USER}` - SQL authentication username
- `${SQL_PASSWORD}` - SQL authentication password
- `${FLEET_API_KEY}` - Fleet telemetry API authentication key
- `${POWERBI_WORKSPACE_ID}` - Power BI workspace ID for publishing
- `${SMTP_SERVER}` - SMTP server for alerting (optional)
- `${ALERT_EMAIL_TO}` - Alert recipient email addresses (optional)

## Best Practices

1. **Use staging tables** for all external data ingestion before merging into fact tables
2. **Implement SCD Type 2** for dimensions that track historical changes (e.g., product categories)
3. **Partition fact tables** by date range for large datasets (100M+ rows)
4. **Cache Power BI aggregations** for frequently-used cross-fact queries
5. **Document data lineage** for all calculated gravity scores and composite metrics
6. **Test RLS policies** before deploying to production
7. **Schedule incremental refreshes** during low-usage hours
