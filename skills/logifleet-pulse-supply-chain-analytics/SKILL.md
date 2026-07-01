---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform for unified warehouse, fleet, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure warehouse and fleet data model in SQL Server"
  - "create Power BI logistics dashboard"
  - "implement cross-fact KPI queries for supply chain"
  - "build multi-modal logistics intelligence platform"
  - "deploy warehouse gravity zone analytics"
  - "troubleshoot Power BI logistics template connection"
  - "query fleet telemetry and warehouse operations data"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a multi-modal logistics intelligence engine combining MS SQL Server data warehousing with Power BI visualization. It creates a unified semantic layer for warehouse operations, fleet telemetry, inventory management, and external supply chain signals using a multi-fact star schema architecture.

**Key Capabilities:**
- Multi-fact star schema with time-phased dimensions
- Cross-fact KPI harmonization (warehouse + fleet metrics)
- Real-time dashboard updates (15-minute granularity)
- Predictive bottleneck detection
- Warehouse Gravity Zones™ spatial optimization
- Role-based access control with row-level security

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, TMS, telematics APIs, ERP systems

### Step 1: Clone Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```bash
# Connect to your SQL Server instance
sqlcmd -S your_server_name -d your_database_name -i schema/deploy_schema.sql

# Or using SSMS, open and execute:
# - schema/01_create_tables.sql
# - schema/02_create_relationships.sql
# - schema/03_create_views.sql
# - schema/04_create_stored_procedures.sql
```

### Step 3: Configure Data Sources

```bash
# Copy and edit configuration
cp config_sample.json config.json
# Update connection strings with your credentials
```

Configuration structure:
```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "${SQL_DATABASE_NAME}",
    "user": "${SQL_USER}",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "fleet_telemetry": "${FLEET_API_ENDPOINT}",
    "weather_api": "${WEATHER_API_KEY}"
  }
}
```

### Step 4: Import Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter SQL Server connection details when prompted
3. Configure data refresh schedule (Edit Queries > Data source settings)

## Core Data Model

### Fact Tables

**FactWarehouseOperations**
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseZoneKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Putaway', 'Pick', 'Pack', 'Ship'
    DwellTimeMinutes INT,
    PickRateUnitsPerHour DECIMAL(10,2),
    PackingTimeSeconds INT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    FOREIGN KEY (WarehouseZoneKey) REFERENCES DimWarehouseZone(ZoneKey)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouse 
ON FactWarehouseOperations;
```

**FactFleetTrips**
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    DistanceMiles DECIMAL(10,2),
    FuelGallons DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadWeightPounds INT,
    MaintenanceScore DECIMAL(5,2),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleet 
ON FactFleetTrips;
```

### Key Dimension Tables

**DimTime** (15-minute granularity)
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    TimeOf15Min TIME NOT NULL,
    HourOfDay TINYINT,
    DayOfWeek VARCHAR(10),
    FiscalPeriod VARCHAR(20),
    IsBusinessHour BIT
);

-- Index for time-series queries
CREATE INDEX IX_DimTime_Date ON DimTime(Date);
CREATE INDEX IX_DimTime_HourOfDay ON DimTime(HourOfDay);
```

**DimProductGravity** (velocity-based scoring)
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Computed: velocity * value * (1/fragility)
    VelocityClass VARCHAR(20), -- 'Fast-Mover', 'Medium', 'Slow-Mover'
    UnitValue DECIMAL(10,2),
    FragilityIndex DECIMAL(3,2)
);

CREATE INDEX IX_ProductGravity_Score ON DimProductGravity(GravityScore DESC);
```

## Common Query Patterns

### Cross-Fact KPI: Dwell Time vs Fleet Idle Cost

```sql
-- Find correlation between warehouse dwell time and fleet delays
WITH WarehouseMetrics AS (
    SELECT 
        t.Date,
        p.Category,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE t.Date >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY t.Date, p.Category
),
FleetMetrics AS (
    SELECT 
        t.Date,
        SUM(f.IdleTimeMinutes) AS TotalIdleTime,
        SUM(f.FuelGallons * 3.50) AS IdleFuelCost, -- $3.50 per gallon
        COUNT(DISTINCT f.VehicleKey) AS ActiveVehicles
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY t.Date
)
SELECT 
    w.Date,
    w.Category,
    w.AvgDwellTime,
    f.TotalIdleTime,
    f.IdleFuelCost,
    CASE 
        WHEN w.AvgDwellTime > 72 THEN 'Critical'
        WHEN w.AvgDwellTime > 48 THEN 'Warning'
        ELSE 'Normal'
    END AS DwellStatus
FROM WarehouseMetrics w
LEFT JOIN FleetMetrics f ON w.Date = f.Date
ORDER BY w.Date DESC, w.AvgDwellTime DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products in wrong gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    wz.ZoneName,
    wz.OptimalGravityRange,
    AVG(wo.PickRateUnitsPerHour) AS CurrentPickRate,
    COUNT(*) AS PickOperations,
    CASE 
        WHEN p.GravityScore NOT BETWEEN wz.MinGravity AND wz.MaxGravity 
        THEN 'Misaligned'
        ELSE 'Optimal'
    END AS ZoneAlignment
FROM DimProductGravity p
INNER JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
INNER JOIN DimWarehouseZone wz ON wo.WarehouseZoneKey = wz.ZoneKey
WHERE wo.TimeKey >= (SELECT MAX(TimeKey) FROM DimTime WHERE Date >= DATEADD(DAY, -30, GETDATE()))
GROUP BY 
    p.SKU, p.ProductName, p.GravityScore, 
    wz.ZoneName, wz.OptimalGravityRange, wz.MinGravity, wz.MaxGravity
HAVING COUNT(*) > 10
ORDER BY 
    CASE WHEN p.GravityScore NOT BETWEEN wz.MinGravity AND wz.MaxGravity 
    THEN 0 ELSE 1 END,
    p.GravityScore DESC;
```

### Predictive Bottleneck Detection

```sql
CREATE PROCEDURE sp_DetectBottlenecks
    @ThresholdHours INT = 2
AS
BEGIN
    -- Real-time bottleneck scoring
    SELECT TOP 20
        t.FullDateTime,
        wz.ZoneName,
        p.Category,
        AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes,
        COUNT(*) AS CongestionCount,
        CASE 
            WHEN AVG(wo.DwellTimeMinutes) > (@ThresholdHours * 60) 
                AND COUNT(*) > 5 
            THEN 'High Risk'
            WHEN AVG(wo.DwellTimeMinutes) > (@ThresholdHours * 60 * 0.7) 
            THEN 'Medium Risk'
            ELSE 'Low Risk'
        END AS BottleneckRisk
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    INNER JOIN DimWarehouseZone wz ON wo.WarehouseZoneKey = wz.ZoneKey
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE t.TimeKey >= (
        SELECT MAX(TimeKey) FROM DimTime 
        WHERE FullDateTime >= DATEADD(HOUR, -@ThresholdHours, GETDATE())
    )
    GROUP BY t.FullDateTime, wz.ZoneName, p.Category
    HAVING AVG(wo.DwellTimeMinutes) > (@ThresholdHours * 60 * 0.5)
    ORDER BY AvgDwellMinutes DESC, CongestionCount DESC;
END;
GO

-- Execute bottleneck detection
EXEC sp_DetectBottlenecks @ThresholdHours = 3;
```

## Data Ingestion Patterns

### Incremental Load from WMS

```sql
CREATE PROCEDURE sp_IncrementalLoadWarehouse
    @LastLoadTimeKey INT
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentTimeKey INT = (
        SELECT TimeKey FROM DimTime 
        WHERE FullDateTime = (
            SELECT MAX(FullDateTime) 
            FROM DimTime 
            WHERE FullDateTime <= GETDATE()
        )
    );
    
    -- Insert new warehouse operations
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseZoneKey, 
        OperationType, DwellTimeMinutes, PickRateUnitsPerHour
    )
    SELECT 
        dt.TimeKey,
        dp.ProductKey,
        dz.ZoneKey,
        ext.operation_type,
        DATEDIFF(MINUTE, ext.start_time, ext.end_time) AS DwellTimeMinutes,
        CASE 
            WHEN ext.operation_type = 'Pick' 
            THEN ext.units_picked * 60.0 / NULLIF(DATEDIFF(MINUTE, ext.start_time, ext.end_time), 0)
            ELSE NULL 
        END AS PickRateUnitsPerHour
    FROM ExternalWarehouseData ext
    INNER JOIN DimTime dt ON CAST(ext.operation_time AS DATETIME2) = dt.FullDateTime
    INNER JOIN DimProductGravity dp ON ext.sku = dp.SKU
    INNER JOIN DimWarehouseZone dz ON ext.zone_code = dz.ZoneCode
    WHERE dt.TimeKey > @LastLoadTimeKey
        AND dt.TimeKey <= @CurrentTimeKey;
    
    -- Return new high watermark
    SELECT @CurrentTimeKey AS NewLastLoadTimeKey;
END;
GO
```

### Fleet Telemetry Streaming Insert

```sql
CREATE PROCEDURE sp_StreamFleetTelemetry
    @VehicleID INT,
    @Timestamp DATETIME2,
    @DistanceMiles DECIMAL(10,2),
    @FuelGallons DECIMAL(8,2),
    @IdleMinutes INT,
    @LoadWeight INT
AS
BEGIN
    DECLARE @TimeKey INT = (
        SELECT TimeKey FROM DimTime 
        WHERE FullDateTime = @Timestamp
    );
    
    IF @TimeKey IS NULL
    BEGIN
        RAISERROR('Time dimension not populated for timestamp', 16, 1);
        RETURN;
    END;
    
    INSERT INTO FactFleetTrips (
        TimeKey, VehicleKey, RouteKey, GeographyKey,
        DistanceMiles, FuelGallons, IdleTimeMinutes, LoadWeightPounds
    )
    SELECT 
        @TimeKey,
        v.VehicleKey,
        r.RouteKey,
        g.GeographyKey,
        @DistanceMiles,
        @FuelGallons,
        @IdleMinutes,
        @LoadWeight
    FROM DimVehicle v
    LEFT JOIN DimRoute r ON v.DefaultRouteKey = r.RouteKey
    LEFT JOIN DimGeography g ON r.OriginGeographyKey = g.GeographyKey
    WHERE v.VehicleID = @VehicleID;
END;
GO
```

## Power BI Integration

### DAX Measures for Cross-Fact Analysis

```dax
// Composite KPI: Warehouse Efficiency Index
WarehouseEfficiencyIndex = 
VAR AvgPickRate = AVERAGE(FactWarehouseOperations[PickRateUnitsPerHour])
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR NormalizedPickRate = DIVIDE(AvgPickRate, 100, 0) // Assuming 100 units/hr is optimal
VAR NormalizedDwellTime = DIVIDE(48, AvgDwellTime, 0) // 48 hours is target max
RETURN
    (NormalizedPickRate * 0.6 + NormalizedDwellTime * 0.4) * 100

// Fleet Cost Per Mile
FleetCostPerMile = 
VAR TotalFuelCost = SUMX(FactFleetTrips, [FuelGallons] * 3.50)
VAR TotalDistance = SUM(FactFleetTrips[DistanceMiles])
RETURN
    DIVIDE(TotalFuelCost, TotalDistance, 0)

// Cross-Fact: Shipment Profitability
ShipmentProfitability = 
VAR WarehouseCost = 
    SUMX(
        FactWarehouseOperations,
        [DwellTimeMinutes] * 0.08 // $0.08 per minute storage cost
    )
VAR FleetCost = 
    SUMX(
        FactFleetTrips,
        ([FuelGallons] * 3.50) + ([IdleTimeMinutes] * 0.50)
    )
VAR Revenue = SUM(FactOrders[OrderValue])
RETURN
    Revenue - WarehouseCost - FleetCost
```

### Row-Level Security Configuration

```dax
// RLS Table: UserSecurity
[RegionAccess] = 
    VAR UserEmail = USERPRINCIPALNAME()
    VAR UserRegions = 
        FILTER(
            UserSecurity,
            UserSecurity[UserEmail] = UserEmail
        )
    RETURN
        DimGeography[Region] IN VALUES(UserRegions[AllowedRegion])
```

Apply in Power BI:
1. Modeling tab > Manage Roles
2. Create role "RegionalManager"
3. Add DAX filter to DimGeography: `[RegionAccess] = TRUE()`

## Troubleshooting

### Issue: Power BI Template Connection Fails

**Symptom:** "Cannot connect to SQL Server" when opening `.pbit` file

**Solution:**
```powershell
# Verify SQL Server allows remote connections
sqlcmd -S your_server_name -U your_username -P your_password -Q "SELECT @@VERSION"

# Check firewall rules
Test-NetConnection -ComputerName your_server_name -Port 1433

# In Power BI, use this connection string format:
# Server=your_server_name;Database=your_database_name;Trusted_Connection=False;User ID=your_user;Password=your_password;
```

### Issue: Slow Query Performance on Large Fact Tables

**Symptom:** Cross-fact queries take >30 seconds

**Solution:**
```sql
-- Add partition function for time-based partitioning
CREATE PARTITION FUNCTION PF_MonthlyPartition (INT)
AS RANGE RIGHT FOR VALUES (
    20260101, 20260201, 20260301, 20260401 -- TimeKey values for month boundaries
);

CREATE PARTITION SCHEME PS_MonthlyPartition
AS PARTITION PF_MonthlyPartition
ALL TO ([PRIMARY]);

-- Recreate fact table with partitioning
CREATE TABLE FactWarehouseOperations_Partitioned (
    -- Same columns as before
    TimeKey INT NOT NULL,
    -- other columns...
) ON PS_MonthlyPartition(TimeKey);

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations_Partitioned WITH FULLSCAN;
```

### Issue: Dimension Time Table Missing Future Dates

**Symptom:** New data not appearing in dashboards

**Solution:**
```sql
CREATE PROCEDURE sp_PopulateFutureDates
    @DaysForward INT = 90
AS
BEGIN
    DECLARE @MaxDate DATE = (SELECT MAX(Date) FROM DimTime);
    DECLARE @EndDate DATE = DATEADD(DAY, @DaysForward, @MaxDate);
    
    WITH DateSequence AS (
        SELECT DATEADD(DAY, 1, @MaxDate) AS Date
        UNION ALL
        SELECT DATEADD(DAY, 1, Date)
        FROM DateSequence
        WHERE Date < @EndDate
    ),
    TimeIntervals AS (
        SELECT 
            d.Date,
            t.TimeOfDay
        FROM DateSequence d
        CROSS JOIN (
            VALUES 
                ('00:00:00'), ('00:15:00'), ('00:30:00'), ('00:45:00'),
                -- ... all 15-minute intervals
                ('23:00:00'), ('23:15:00'), ('23:30:00'), ('23:45:00')
        ) t(TimeOfDay)
    )
    INSERT INTO DimTime (TimeKey, FullDateTime, Date, TimeOf15Min, HourOfDay, DayOfWeek)
    SELECT 
        CAST(FORMAT(CAST(Date AS DATETIME) + CAST(TimeOfDay AS TIME), 'yyyyMMddHHmm') AS INT),
        CAST(Date AS DATETIME) + CAST(TimeOfDay AS TIME),
        Date,
        CAST(TimeOfDay AS TIME),
        DATEPART(HOUR, TimeOfDay),
        DATENAME(WEEKDAY, Date)
    FROM TimeIntervals
    OPTION (MAXRECURSION 365);
END;
GO

-- Execute to populate next 90 days
EXEC sp_PopulateFutureDates @DaysForward = 90;
```

## Alert Configuration

### Set Up Automated Alerts

```sql
CREATE PROCEDURE sp_ConfigureAlert
    @AlertName VARCHAR(100),
    @MetricType VARCHAR(50), -- 'DwellTime', 'FleetIdle', 'BottleneckRisk'
    @ThresholdValue DECIMAL(10,2),
    @RecipientEmail VARCHAR(255)
AS
BEGIN
    INSERT INTO AlertConfiguration (
        AlertName, MetricType, ThresholdValue, 
        RecipientEmail, IsActive, CreatedDate
    )
    VALUES (
        @AlertName, @MetricType, @ThresholdValue, 
        @RecipientEmail, 1, GETDATE()
    );
END;
GO

-- Example: Alert when dwell time exceeds 72 hours
EXEC sp_ConfigureAlert 
    @AlertName = 'Critical Dwell Time',
    @MetricType = 'DwellTime',
    @ThresholdValue = 4320, -- 72 hours in minutes
    @RecipientEmail = '${ALERT_EMAIL}';
```

## Best Practices

1. **Incremental Loading:** Always use TimeKey-based watermarks to load only new data
2. **Indexing Strategy:** Create clustered columnstore indexes on fact tables, B-tree indexes on dimension keys
3. **Data Refresh:** Schedule Power BI refresh every 15 minutes during business hours, hourly off-hours
4. **Security:** Implement row-level security based on user roles and geography
5. **Monitoring:** Set up SQL Server Extended Events to track long-running queries
6. **Backup:** Daily full backup of dimension tables, transaction log backups every 15 minutes for fact tables

## Advanced: Temporal Elasticity Modeling

```sql
-- Simulate warehouse capacity impact on fleet utilization
CREATE PROCEDURE sp_SimulateCapacityChange
    @CapacityIncreasePct DECIMAL(5,2) -- e.g., 15.0 for 15% increase
AS
BEGIN
    WITH BaselineMetrics AS (
        SELECT 
            AVG(DwellTimeMinutes) AS AvgDwell,
            AVG(PickRateUnitsPerHour) AS AvgPickRate
        FROM FactWarehouseOperations
        WHERE TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime) -- Last 30 days
    ),
    ProjectedMetrics AS (
        SELECT 
            AvgDwell * (1 - (@CapacityIncreasePct / 200)) AS ProjectedDwell,
            AvgPickRate * (1 + (@CapacityIncreasePct / 300)) AS ProjectedPickRate
        FROM BaselineMetrics
    )
    SELECT 
        b.AvgDwell AS CurrentDwellTime,
        p.ProjectedDwell AS ProjectedDwellTime,
        b.AvgPickRate AS CurrentPickRate,
        p.ProjectedPickRate AS ProjectedPickRate,
        ((b.AvgDwell - p.ProjectedDwell) / b.AvgDwell) * 100 AS DwellReductionPct
    FROM BaselineMetrics b, ProjectedMetrics p;
END;
GO

-- Run simulation for 20% capacity increase
EXEC sp_SimulateCapacityChange @CapacityIncreasePct = 20.0;
```

This skill provides comprehensive implementation guidance for deploying and utilizing LogiFleet Pulse in production logistics environments.
