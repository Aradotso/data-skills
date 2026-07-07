---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for fleet, warehouse, and cross-modal supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy the logistics data warehouse schema"
  - "configure Power BI for fleet and warehouse dashboards"
  - "query cross-fact logistics KPIs"
  - "implement warehouse gravity zones"
  - "build supply chain star schema"
  - "integrate fleet telemetry with warehouse data"
  - "create predictive bottleneck detection queries"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a **multi-modal logistics intelligence engine** that combines warehouse operations, fleet telemetry, inventory management, and external data sources into a unified MS SQL Server data warehouse. It features:

- **Multi-fact star schema** with time-phased dimensions
- **Cross-fact KPI harmonization** (warehouse metrics + fleet metrics + supply chain data)
- **Power BI dashboards** for real-time logistics visualization
- **Warehouse Gravity Zones™** — spatial optimization based on pick frequency and value
- **Predictive bottleneck detection** using temporal elasticity modeling
- **Role-based access control** with row-level security

The platform enables queries like: *"Show shipments delayed by weather from cold-storage zones with >72h dwell time"* by bridging multiple fact tables through shared dimensions.

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Enterprise or Standard edition)
- Power BI Desktop (latest version)
- ODBC/OLEDB drivers for external data sources (WMS, TMS, telemetry APIs)
- SQL Server Management Studio (SSMS) recommended

### Deploy the SQL Schema

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Execute the schema creation script:**

```sql
-- Run in SSMS connected to your target database
-- File: schema/01_create_database.sql

USE master;
GO

CREATE DATABASE LogiFleetPulse
ON PRIMARY 
(
    NAME = LogiFleetPulse_Data,
    FILENAME = 'C:\SQLData\LogiFleetPulse.mdf',
    SIZE = 10GB,
    FILEGROWTH = 1GB
)
LOG ON
(
    NAME = LogiFleetPulse_Log,
    FILENAME = 'C:\SQLData\LogiFleetPulse.ldf',
    SIZE = 2GB,
    FILEGROWTH = 512MB
);
GO

ALTER DATABASE LogiFleetPulse SET RECOVERY SIMPLE;
GO
```

3. **Create the dimension tables:**

```sql
-- File: schema/02_create_dimensions.sql

USE LogiFleetPulse;
GO

-- Time dimension with 15-minute granularity
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    [Year] INT NOT NULL,
    [Quarter] INT NOT NULL,
    [Month] INT NOT NULL,
    [Day] INT NOT NULL,
    [Hour] INT NOT NULL,
    [Minute] INT NOT NULL,
    DayOfWeek NVARCHAR(10) NOT NULL,
    IsWeekend BIT NOT NULL,
    FiscalYear INT NOT NULL,
    FiscalQuarter INT NOT NULL,
    TimeSlot AS (CONVERT(VARCHAR(5), [Hour]) + ':' + 
                 CASE WHEN [Minute] < 15 THEN '00'
                      WHEN [Minute] < 30 THEN '15'
                      WHEN [Minute] < 45 THEN '30'
                      ELSE '45' END)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_DimTime ON DimTime;
GO

-- Geography dimension (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID NVARCHAR(50) NOT NULL UNIQUE,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50) NOT NULL, -- 'Warehouse', 'RouteNode', 'CustomerSite'
    AddressLine1 NVARCHAR(200),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    PostalCode NVARCHAR(20),
    Country NVARCHAR(100) NOT NULL,
    Region NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone NVARCHAR(50),
    IsActive BIT DEFAULT 1
);

CREATE NONCLUSTERED INDEX IX_DimGeography_Type ON DimGeography(LocationType);
CREATE NONCLUSTERED INDEX IX_DimGeography_Country ON DimGeography(Country, Region);
GO

-- Product dimension with gravity scoring
CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU NVARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT DEFAULT 0,
    RequiresRefrigeration BIT DEFAULT 0,
    UnitValue DECIMAL(18,2),
    GravityScore AS (
        CASE 
            WHEN UnitValue > 100 AND IsFragile = 1 THEN 100
            WHEN UnitValue > 50 THEN 75
            WHEN IsFragile = 1 THEN 60
            ELSE 25
        END
    ) PERSISTED,
    IsActive BIT DEFAULT 1
);

CREATE NONCLUSTERED INDEX IX_DimProduct_Category ON DimProduct(Category, SubCategory);
CREATE NONCLUSTERED INDEX IX_DimProduct_Gravity ON DimProduct(GravityScore DESC);
GO

-- Supplier dimension with reliability metrics
CREATE TABLE DimSupplier (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierID NVARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200) NOT NULL,
    Country NVARCHAR(100),
    AvgLeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,2),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore AS (
        (100 - ISNULL(DefectRatePercent, 0)) * 0.4 +
        ISNULL(OnTimeDeliveryPercent, 0) * 0.6
    ) PERSISTED,
    IsActive BIT DEFAULT 1
);
GO
```

4. **Create the fact tables:**

```sql
-- File: schema/03_create_facts.sql

USE LogiFleetPulse;
GO

-- Warehouse operations fact
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType NVARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity INT NOT NULL,
    DurationMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    ZoneLocation NVARCHAR(50),
    EmployeeID NVARCHAR(50),
    OrderID NVARCHAR(100),
    BatchNumber NVARCHAR(100),
    CONSTRAINT FK_FWO_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FWO_Geography FOREIGN KEY (WarehouseKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FWO_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

CREATE NONCLUSTERED INDEX IX_FWO_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FWO_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FWO_OperationType ON FactWarehouseOperations(OperationType);
GO

-- Fleet trips fact
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    DeparturTimeKey INT NOT NULL,
    ArrivalTimeKey INT NOT NULL,
    OriginKey INT NOT NULL,
    DestinationKey INT NOT NULL,
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    DistanceKM DECIMAL(10,2),
    DurationMinutes DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    AvgSpeedKMH AS (CASE WHEN DurationMinutes > 0 THEN (DistanceKM / DurationMinutes) * 60 ELSE 0 END) PERSISTED,
    LoadWeightKG DECIMAL(10,2),
    OrdersDelivered INT,
    DelayMinutes DECIMAL(10,2),
    DelayReason NVARCHAR(200),
    CONSTRAINT FK_FFT_DepartTime FOREIGN KEY (DeparturTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FFT_ArrivalTime FOREIGN KEY (ArrivalTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FFT_Origin FOREIGN KEY (OriginKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FFT_Destination FOREIGN KEY (DestinationKey) REFERENCES DimGeography(GeographyKey)
);

CREATE NONCLUSTERED INDEX IX_FFT_DepartTime ON FactFleetTrips(DeparturTimeKey);
CREATE NONCLUSTERED INDEX IX_FFT_Vehicle ON FactFleetTrips(VehicleID);
CREATE NONCLUSTERED INDEX IX_FFT_Route ON FactFleetTrips(OriginKey, DestinationKey);
GO

-- Cross-dock operations fact
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    InboundTimeKey INT NOT NULL,
    OutboundTimeKey INT NOT NULL,
    CrossDockLocationKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundSupplierKey INT NOT NULL,
    OutboundCustomerKey INT NOT NULL,
    Quantity INT NOT NULL,
    TransferTimeMinutes DECIMAL(10,2),
    InboundOrderID NVARCHAR(100),
    OutboundOrderID NVARCHAR(100),
    CONSTRAINT FK_FCD_InboundTime FOREIGN KEY (InboundTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FCD_OutboundTime FOREIGN KEY (OutboundTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FCD_Location FOREIGN KEY (CrossDockLocationKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FCD_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_FCD_Supplier FOREIGN KEY (InboundSupplierKey) REFERENCES DimSupplier(SupplierKey)
);

CREATE NONCLUSTERED INDEX IX_FCD_InboundTime ON FactCrossDock(InboundTimeKey);
CREATE NONCLUSTERED INDEX IX_FCD_Product ON FactCrossDock(ProductKey);
GO
```

### Configure Data Sources

1. **Create external data source connection file:**

```json
// config/data_sources.json
{
  "warehouse_management_system": {
    "type": "SQL_SERVER",
    "server": "${WMS_DB_SERVER}",
    "database": "${WMS_DB_NAME}",
    "auth": "integrated",
    "refresh_interval_minutes": 15
  },
  "fleet_telemetry_api": {
    "type": "REST_API",
    "endpoint": "${FLEET_API_ENDPOINT}",
    "auth_header": "Authorization: Bearer ${FLEET_API_TOKEN}",
    "polling_interval_minutes": 5
  },
  "weather_service": {
    "type": "REST_API",
    "endpoint": "https://api.openweathermap.org/data/2.5/forecast",
    "api_key": "${WEATHER_API_KEY}",
    "polling_interval_minutes": 60
  }
}
```

2. **Create ETL stored procedure for incremental loading:**

```sql
-- File: etl/sp_load_warehouse_operations.sql

USE LogiFleetPulse;
GO

CREATE OR ALTER PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentLoad DATETIME2 = GETDATE();
    
    IF @LastLoadDateTime IS NULL
        SET @LastLoadDateTime = DATEADD(HOUR, -1, @CurrentLoad);
    
    -- Insert from WMS staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey,
        WarehouseKey,
        ProductKey,
        OperationType,
        Quantity,
        DurationMinutes,
        DwellTimeHours,
        ZoneLocation,
        EmployeeID,
        OrderID,
        BatchNumber
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        s.Quantity,
        s.DurationMinutes,
        s.DwellTimeHours,
        s.ZoneLocation,
        s.EmployeeID,
        s.OrderID,
        s.BatchNumber
    FROM WMS_Staging.dbo.Operations s
    INNER JOIN DimTime t ON CAST(s.OperationDateTime AS DATE) = CAST(t.FullDateTime AS DATE)
        AND DATEPART(HOUR, s.OperationDateTime) = t.[Hour]
        AND DATEPART(MINUTE, s.OperationDateTime) / 15 = t.[Minute] / 15
    INNER JOIN DimGeography g ON s.WarehouseID = g.LocationID
    INNER JOIN DimProduct p ON s.SKU = p.SKU
    WHERE s.OperationDateTime >= @LastLoadDateTime
        AND s.OperationDateTime < @CurrentLoad;
    
    -- Log the load
    INSERT INTO ETL_LoadLog (ProcedureName, LoadDateTime, RowsAffected)
    VALUES ('sp_LoadWarehouseOperations', @CurrentLoad, @@ROWCOUNT);
END;
GO
```

## Key Queries & Analytics Patterns

### Cross-Fact KPI: Fleet Fuel Efficiency vs. Warehouse Dwell Time

```sql
-- Query: Correlate high dwell time SKUs with fleet fuel consumption on deliveries

WITH HighDwellProducts AS (
    SELECT 
        p.SKU,
        p.ProductName,
        AVG(fwo.DwellTimeHours) AS AvgDwellHours,
        SUM(fwo.Quantity) AS TotalUnits
    FROM FactWarehouseOperations fwo
    INNER JOIN DimProduct p ON fwo.ProductKey = p.ProductKey
    WHERE fwo.OperationType = 'Shipping'
        AND fwo.TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime) -- Last 30 days (15min slots)
    GROUP BY p.SKU, p.ProductName
    HAVING AVG(fwo.DwellTimeHours) > 48
),
FleetEfficiency AS (
    SELECT 
        fft.VehicleID,
        AVG(fft.FuelConsumedLiters / NULLIF(fft.DistanceKM, 0)) AS AvgFuelPerKM,
        AVG(fft.IdleTimeMinutes) AS AvgIdleMinutes,
        COUNT(*) AS TotalTrips
    FROM FactFleetTrips fft
    WHERE fft.DeparturTimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime)
    GROUP BY fft.VehicleID
)
SELECT 
    hdp.SKU,
    hdp.ProductName,
    hdp.AvgDwellHours,
    fe.VehicleID,
    fe.AvgFuelPerKM,
    fe.AvgIdleMinutes,
    hdp.AvgDwellHours * fe.AvgIdleMinutes AS WasteIndex -- Higher = more operational friction
FROM HighDwellProducts hdp
CROSS JOIN FleetEfficiency fe
WHERE fe.AvgFuelPerKM > (SELECT AVG(AvgFuelPerKM) FROM FleetEfficiency)
ORDER BY WasteIndex DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Query: Recommend SKU repositioning based on pick frequency and gravity score

WITH PickFrequency AS (
    SELECT 
        fwo.ProductKey,
        COUNT(*) AS PickCount,
        AVG(fwo.DurationMinutes) AS AvgPickTime,
        fwo.ZoneLocation
    FROM FactWarehouseOperations fwo
    WHERE fwo.OperationType = 'Picking'
        AND fwo.TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime) -- Last 7 days
    GROUP BY fwo.ProductKey, fwo.ZoneLocation
),
OptimalZones AS (
    SELECT 
        ProductKey,
        ZoneLocation,
        PickCount,
        AvgPickTime,
        ROW_NUMBER() OVER (PARTITION BY ProductKey ORDER BY PickCount DESC) AS ZoneRank
    FROM PickFrequency
)
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    oz.ZoneLocation AS CurrentZone,
    oz.PickCount,
    oz.AvgPickTime,
    CASE 
        WHEN p.GravityScore >= 75 AND oz.ZoneLocation NOT LIKE 'A-%' THEN 'MOVE TO HIGH-GRAVITY ZONE A'
        WHEN p.GravityScore < 50 AND oz.ZoneLocation LIKE 'A-%' THEN 'MOVE TO LOW-GRAVITY ZONE C/D'
        ELSE 'OPTIMAL PLACEMENT'
    END AS Recommendation
FROM OptimalZones oz
INNER JOIN DimProduct p ON oz.ProductKey = p.ProductKey
WHERE oz.ZoneRank = 1
ORDER BY p.GravityScore DESC, oz.PickCount DESC;
```

### Predictive Bottleneck Detection

```sql
-- Query: Identify potential congestion points using time-series velocity patterns

WITH HourlyWarehouseLoad AS (
    SELECT 
        t.[Hour],
        t.DayOfWeek,
        g.LocationName AS WarehouseName,
        COUNT(*) AS OperationCount,
        AVG(fwo.DurationMinutes) AS AvgDuration
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime t ON fwo.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON fwo.WarehouseKey = g.GeographyKey
    WHERE fwo.TimeKey >= (SELECT MAX(TimeKey) - 2016 FROM DimTime) -- Last 3 weeks
    GROUP BY t.[Hour], t.DayOfWeek, g.LocationName
),
Baseline AS (
    SELECT 
        [Hour],
        DayOfWeek,
        WarehouseName,
        AVG(OperationCount) AS BaselineCount,
        STDEV(OperationCount) AS StdDevCount
    FROM HourlyWarehouseLoad
    GROUP BY [Hour], DayOfWeek, WarehouseName
)
SELECT 
    hwl.WarehouseName,
    hwl.DayOfWeek,
    hwl.[Hour],
    hwl.OperationCount AS CurrentLoad,
    b.BaselineCount,
    b.StdDevCount,
    (hwl.OperationCount - b.BaselineCount) / NULLIF(b.StdDevCount, 0) AS ZScore,
    CASE 
        WHEN (hwl.OperationCount - b.BaselineCount) / NULLIF(b.StdDevCount, 0) > 2 THEN 'CRITICAL OVERLOAD'
        WHEN (hwl.OperationCount - b.BaselineCount) / NULLIF(b.StdDevCount, 0) > 1 THEN 'WARNING'
        ELSE 'NORMAL'
    END AS BottleneckRisk
FROM HourlyWarehouseLoad hwl
INNER JOIN Baseline b ON hwl.[Hour] = b.[Hour] 
    AND hwl.DayOfWeek = b.DayOfWeek 
    AND hwl.WarehouseName = b.WarehouseName
WHERE (hwl.OperationCount - b.BaselineCount) / NULLIF(b.StdDevCount, 0) > 1
ORDER BY ZScore DESC;
```

### Supplier Reliability Impact on Inventory Velocity

```sql
-- Query: Cross-reference supplier compliance with product turnover rates

WITH SupplierDeliveries AS (
    SELECT 
        fcd.InboundSupplierKey,
        fcd.ProductKey,
        COUNT(*) AS DeliveryCount,
        AVG(fcd.TransferTimeMinutes) AS AvgCrossDockTime
    FROM FactCrossDock fcd
    WHERE fcd.InboundTimeKey >= (SELECT MAX(TimeKey) - 4320 FROM DimTime) -- Last 45 days
    GROUP BY fcd.InboundSupplierKey, fcd.ProductKey
),
ProductVelocity AS (
    SELECT 
        fwo.ProductKey,
        COUNT(*) AS TurnoverCount,
        AVG(fwo.DwellTimeHours) AS AvgDwellHours
    FROM FactWarehouseOperations fwo
    WHERE fwo.OperationType = 'Shipping'
        AND fwo.TimeKey >= (SELECT MAX(TimeKey) - 4320 FROM DimTime)
    GROUP BY fwo.ProductKey
)
SELECT 
    s.SupplierName,
    s.ComplianceScore,
    p.SKU,
    p.ProductName,
    sd.DeliveryCount,
    pv.TurnoverCount,
    pv.AvgDwellHours,
    CASE 
        WHEN s.ComplianceScore < 70 AND pv.AvgDwellHours > 72 THEN 'HIGH RISK - CHANGE SUPPLIER'
        WHEN s.ComplianceScore < 80 AND pv.AvgDwellHours > 48 THEN 'MODERATE RISK'
        ELSE 'ACCEPTABLE'
    END AS RiskAssessment
FROM SupplierDeliveries sd
INNER JOIN DimSupplier s ON sd.InboundSupplierKey = s.SupplierKey
INNER JOIN DimProduct p ON sd.ProductKey = p.ProductKey
LEFT JOIN ProductVelocity pv ON sd.ProductKey = pv.ProductKey
ORDER BY s.ComplianceScore ASC, pv.AvgDwellHours DESC;
```

## Power BI Configuration

### Import the Template

1. **Open the template file:**

```bash
# File: powerbi/LogiFleet_Pulse_Master.pbit
# Double-click or open in Power BI Desktop
```

2. **Configure the data source connection:**

When prompted, enter your SQL Server connection details:

- **Server**: Your MS SQL Server instance (e.g., `SQLSERVER01.domain.com`)
- **Database**: `LogiFleetPulse`
- **Authentication**: Windows or SQL Server (use env vars for credentials)

3. **Key measures in the semantic model:**

```dax
// Measure: Warehouse Throughput per Hour
Warehouse Throughput = 
CALCULATE(
    SUM(FactWarehouseOperations[Quantity]),
    FILTER(
        FactWarehouseOperations,
        FactWarehouseOperations[OperationType] IN {"Picking", "Packing", "Shipping"}
    )
) / DISTINCTCOUNT(DimTime[Hour])

// Measure: Fleet Fuel Efficiency Index
Fleet Fuel Efficiency = 
DIVIDE(
    AVERAGE(FactFleetTrips[DistanceKM]),
    AVERAGE(FactFleetTrips[FuelConsumedLiters]),
    0
)

// Measure: Cross-Dock Velocity Score
Cross-Dock Velocity = 
VAR AvgTransferTime = AVERAGE(FactCrossDock[TransferTimeMinutes])
VAR Baseline = 120 -- 2 hours baseline
RETURN
IF(
    AvgTransferTime > 0,
    (Baseline / AvgTransferTime) * 100,
    0
)

// Measure: Gravity-Weighted Dwell Time
Gravity Dwell Index = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeHours] * RELATED(DimProduct[GravityScore])
) / SUMX(FactWarehouseOperations, RELATED(DimProduct[GravityScore]))

// Measure: Predictive Bottleneck Probability (simplified)
Bottleneck Risk % = 
VAR CurrentLoad = COUNT(FactWarehouseOperations[OperationKey])
VAR HistoricalAvg = CALCULATE(
    COUNT(FactWarehouseOperations[OperationKey]),
    DATEADD(DimTime[FullDateTime], -7, DAY)
)
RETURN
IF(
    HistoricalAvg > 0,
    ((CurrentLoad - HistoricalAvg) / HistoricalAvg) * 100,
    0
)
```

4. **Apply row-level security:**

```dax
// RLS Role: Regional Manager
// Table: DimGeography
// Filter: [Region] = USERNAME()

// RLS Role: Warehouse Supervisor
// Table: DimGeography
// Filter: [LocationName] IN {"Warehouse A", "Warehouse B"}
```

### Dashboard Best Practices

1. **Use bookmarks for scenario comparison:**
   - Create bookmarks for "Current State" vs. "Optimized Gravity Zones"
   - Enable users to toggle between views

2. **Enable drill-through for anomaly investigation:**
   - From bottleneck heatmap → detailed operation log
   - From fleet efficiency chart → individual trip telemetry

3. **Configure data refresh schedule:**
   - Set to refresh every 15 minutes during business hours
   - Use incremental refresh for fact tables (last 30 days active, 2 years archive)

## Common Patterns & Troubleshooting

### Pattern: Time-Phased Dimension Lookup

When joining fact tables to `DimTime`, match both date and time slot:

```sql
-- Correct approach
FROM FactWarehouseOperations fwo
INNER JOIN DimTime t ON fwo.TimeKey = t.TimeKey

-- For external data without TimeKey:
INNER JOIN DimTime t ON 
    CAST(ExternalData.Timestamp AS DATE) = CAST(t.FullDateTime AS DATE)
    AND DATEPART(HOUR, ExternalData.Timestamp) = t.[Hour]
    AND (DATEPART(MINUTE, ExternalData.Timestamp) / 15) = (t.[Minute] / 15)
```

### Pattern: Many-to-Many Bridge for Routes ↔ Products

```sql
-- Bridge table for handling shipments with multiple SKUs across multiple route segments
CREATE TABLE BridgeRouteProduct (
    BridgeKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TripKey BIGINT NOT NULL,
    ProductKey INT NOT NULL,
    Quantity INT NOT NULL,
    CONSTRAINT FK_Bridge_Trip FOREIGN KEY (TripKey) REFERENCES FactFleetTrips(TripKey),
    CONSTRAINT FK_Bridge_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Query using the bridge
SELECT 
    fft.VehicleID,
    p.SKU,
    SUM(brp.Quantity) AS TotalDelivered,
    AVG(fft.FuelConsumedLiters) AS AvgFuelPerTrip
FROM FactFleetTrips fft
INNER JOIN BridgeRouteProduct brp ON fft.TripKey = brp.TripKey
INNER JOIN DimProduct p ON brp.ProductKey = p.ProductKey
GROUP BY fft.VehicleID, p.SKU;
```

### Troubleshooting: Slow Cross-Fact Queries

**Symptom:** Queries joining multiple fact tables take >30 seconds

**Solution 1 - Use aggregate tables:**

```sql
CREATE TABLE AggWarehouseDaily (
    DateKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    TotalQuantityShipped INT,
    AvgDwellTimeHours DECIMAL(10,2),
    PRIMARY KEY (DateKey, WarehouseKey, ProductKey)
);

-- Populate via scheduled job
INSERT INTO AggWarehouseDaily
SELECT 
    t.DateKey,
    fwo.WarehouseKey,
    fwo.ProductKey,
    SUM(fwo.Quantity),
    AVG(fwo.DwellTimeHours)
FROM FactWarehouseOperations fwo
INNER JOIN DimTime t ON fwo.TimeKey = t.TimeKey
WHERE CAST(t.FullDateTime AS DATE) = CAST(GETDATE()-1 AS DATE)
GROUP BY t.DateKey, fwo.WarehouseKey, fwo.ProductKey;
```

**Solution 2 - Partition large fact tables:**

```sql
-- Partition FactFleetTrips by month
ALTER TABLE FactFleetTrips
SWITCH PARTITION 1 TO FactFleetTrips_Archive PARTITION 1;

-- Create monthly partitions
CREATE PARTITION FUNCTION PF_FleetTrips_Month (INT)
AS RANGE RIGHT FOR VALUES (202601, 202602, 202603, ...);

CREATE PARTITION SCHEME PS_FleetTrips_Month
AS PARTITION PF_FleetTrips_Month ALL TO ([PRIMARY]);
```

### Troubleshooting: Power BI Report Timeout

**Symptom:** Power BI dashboards fail to refresh with "timeout" error

**Solution - Use DirectQuery with aggregations:**

1. In Power BI Desktop → Model view → Right-click fact table → Storage mode → DirectQuery
2. Create aggregate tables in SQL Server (see above)
3. Set aggregate tables to Import mode
4. Power BI automatically routes simple queries to aggregations, complex to DirectQuery

### Troubleshooting: Missing TimeKey References

**Symptom:** ETL fails with foreign key violation on `DimTime`

**Solution - Pre-populate time dimension:**

```sql
-- Generate 5 years of 15-minute time slots
