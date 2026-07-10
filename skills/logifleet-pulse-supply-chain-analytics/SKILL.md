---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI data warehousing template for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - set up logifleet pulse logistics dashboard
  - configure supply chain analytics warehouse
  - implement multi-fact star schema for logistics
  - create power bi fleet management dashboard
  - deploy logifleet pulse sql schema
  - build warehouse gravity zone analytics
  - integrate fleet telemetry with inventory data
  - setup cross-modal supply chain orchestration
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill provides expertise in deploying and using **LogiFleet Pulse**, an advanced MS SQL Server and Power BI data warehousing template for logistics and supply chain analytics. The project implements a multi-fact star schema that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a cohesive analytics platform.

## What LogiFleet Pulse Does

LogiFleet Pulse is a **data modeling and visualization template** that provides:

- **Multi-fact star schema** linking warehouse operations, fleet trips, and cross-dock activities
- **Time-phased dimensions** with 15-minute granularity for real-time analytics
- **Cross-fact KPI harmonization** connecting inventory metrics with fleet performance
- **Power BI dashboards** for operational, tactical, and strategic insights
- **Predictive analytics** for bottleneck detection and fleet maintenance prioritization
- **Warehouse Gravity Zones™** for spatial optimization based on velocity and value
- **Role-based access control** with row-level security

## Installation and Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Access to data sources (WMS, TMS, telemetry APIs, ERP systems)

### Step 1: Deploy SQL Schema

Clone the repository and navigate to the SQL scripts directory:

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

Connect to your SQL Server instance and execute the schema creation script:

```sql
-- Execute the main schema deployment script
-- This creates all dimension tables, fact tables, views, and stored procedures
:r "./sql/01_create_schema.sql"

-- Verify schema creation
SELECT 
    SCHEMA_NAME(schema_id) AS SchemaName,
    name AS TableName,
    type_desc AS ObjectType
FROM sys.objects
WHERE type IN ('U', 'V', 'P')
    AND SCHEMA_NAME(schema_id) IN ('dbo', 'logistics')
ORDER BY SchemaName, ObjectType, TableName;
```

### Step 2: Configure Data Sources

Create a configuration file for your data source connections:

```json
{
  "connections": {
    "wms_api": {
      "type": "rest",
      "base_url": "${WMS_API_URL}",
      "auth_token": "${WMS_API_KEY}"
    },
    "fleet_telemetry": {
      "type": "sql",
      "server": "${FLEET_DB_SERVER}",
      "database": "${FLEET_DB_NAME}",
      "integrated_security": true
    },
    "erp_system": {
      "type": "odbc",
      "connection_string": "${ERP_CONNECTION_STRING}"
    }
  },
  "refresh_interval_minutes": 15,
  "alert_email": "${ALERT_EMAIL_ADDRESS}"
}
```

### Step 3: Set Up Data Loading

Create stored procedures for incremental data loading:

```sql
-- Incremental load for warehouse operations fact table
CREATE OR ALTER PROCEDURE [logistics].[usp_LoadFactWarehouseOperations]
    @LoadDate DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to current date if not specified
    IF @LoadDate IS NULL
        SET @LoadDate = GETDATE();
    
    -- Stage data from WMS source
    MERGE INTO [logistics].[FactWarehouseOperations] AS target
    USING (
        SELECT 
            wo.OperationID,
            dt.TimeKey,
            dg.GeographyKey,
            dp.ProductKey,
            wo.OperationType,
            wo.QuantityHandled,
            wo.DurationMinutes,
            wo.DwellTimeHours,
            wo.PickingAccuracy,
            wo.PackingEfficiency
        FROM [staging].[WarehouseOperations_Raw] wo
        INNER JOIN [logistics].[DimTime] dt 
            ON CAST(wo.OperationTimestamp AS DATE) = dt.FullDate
            AND DATEPART(HOUR, wo.OperationTimestamp) = dt.Hour
            AND (DATEPART(MINUTE, wo.OperationTimestamp) / 15) = (dt.Minute / 15)
        INNER JOIN [logistics].[DimGeography] dg 
            ON wo.WarehouseCode = dg.LocationCode
        INNER JOIN [logistics].[DimProduct] dp 
            ON wo.SKU = dp.SKU
        WHERE wo.OperationTimestamp >= @LoadDate
    ) AS source
    ON target.OperationID = source.OperationID
    WHEN MATCHED THEN
        UPDATE SET 
            target.QuantityHandled = source.QuantityHandled,
            target.DurationMinutes = source.DurationMinutes,
            target.DwellTimeHours = source.DwellTimeHours,
            target.PickingAccuracy = source.PickingAccuracy,
            target.PackingEfficiency = source.PackingEfficiency
    WHEN NOT MATCHED THEN
        INSERT (
            OperationID, TimeKey, GeographyKey, ProductKey,
            OperationType, QuantityHandled, DurationMinutes,
            DwellTimeHours, PickingAccuracy, PackingEfficiency
        )
        VALUES (
            source.OperationID, source.TimeKey, source.GeographyKey, source.ProductKey,
            source.OperationType, source.QuantityHandled, source.DurationMinutes,
            source.DwellTimeHours, source.PickingAccuracy, source.PackingEfficiency
        );
END;
GO
```

Load fleet telemetry data:

```sql
-- Incremental load for fleet trips fact table
CREATE OR ALTER PROCEDURE [logistics].[usp_LoadFactFleetTrips]
    @LoadDate DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    IF @LoadDate IS NULL
        SET @LoadDate = GETDATE();
    
    MERGE INTO [logistics].[FactFleetTrips] AS target
    USING (
        SELECT 
            ft.TripID,
            dt.TimeKey,
            dg.GeographyKey AS OriginKey,
            dg2.GeographyKey AS DestinationKey,
            dv.VehicleKey,
            dd.DriverKey,
            ft.DistanceMiles,
            ft.DurationMinutes,
            ft.FuelConsumedGallons,
            ft.IdleTimeMinutes,
            ft.AverageSpeedMPH,
            ft.HarshBrakingEvents,
            ft.LoadWeightLbs,
            ft.OnTimeDelivery
        FROM [staging].[FleetTrips_Raw] ft
        INNER JOIN [logistics].[DimTime] dt 
            ON CAST(ft.DepartureTimestamp AS DATE) = dt.FullDate
            AND DATEPART(HOUR, ft.DepartureTimestamp) = dt.Hour
        INNER JOIN [logistics].[DimGeography] dg 
            ON ft.OriginLocationCode = dg.LocationCode
        INNER JOIN [logistics].[DimGeography] dg2 
            ON ft.DestinationLocationCode = dg2.LocationCode
        INNER JOIN [logistics].[DimVehicle] dv 
            ON ft.VehicleID = dv.VehicleID
        INNER JOIN [logistics].[DimDriver] dd 
            ON ft.DriverID = dd.DriverID
        WHERE ft.DepartureTimestamp >= @LoadDate
    ) AS source
    ON target.TripID = source.TripID
    WHEN MATCHED THEN
        UPDATE SET 
            target.DistanceMiles = source.DistanceMiles,
            target.DurationMinutes = source.DurationMinutes,
            target.FuelConsumedGallons = source.FuelConsumedGallons,
            target.IdleTimeMinutes = source.IdleTimeMinutes
    WHEN NOT MATCHED THEN
        INSERT (
            TripID, TimeKey, OriginKey, DestinationKey, VehicleKey, DriverKey,
            DistanceMiles, DurationMinutes, FuelConsumedGallons, IdleTimeMinutes,
            AverageSpeedMPH, HarshBrakingEvents, LoadWeightLbs, OnTimeDelivery
        )
        VALUES (
            source.TripID, source.TimeKey, source.OriginKey, source.DestinationKey,
            source.VehicleKey, source.DriverKey, source.DistanceMiles,
            source.DurationMinutes, source.FuelConsumedGallons, source.IdleTimeMinutes,
            source.AverageSpeedMPH, source.HarshBrakingEvents, source.LoadWeightLbs,
            source.OnTimeDelivery
        );
END;
GO
```

### Step 4: Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI template (.pbit)
3. Select `LogiFleet_Pulse_Master.pbit` from the repository
4. Enter your SQL Server connection parameters when prompted
5. Click Load to build the semantic model

## Key SQL Views and Queries

### Cross-Fact KPI Query: Warehouse Dwell Time vs Fleet Idle Time

```sql
-- Analyze correlation between warehouse dwell time and fleet idle time
-- This cross-fact query reveals if warehouse bottlenecks cascade to fleet delays
SELECT 
    dt.FiscalYear,
    dt.FiscalQuarter,
    dt.MonthName,
    dg.WarehouseName,
    dp.ProductCategory,
    
    -- Warehouse metrics
    AVG(fwo.DwellTimeHours) AS AvgDwellTimeHours,
    SUM(fwo.QuantityHandled) AS TotalUnitsHandled,
    AVG(fwo.PickingAccuracy) AS AvgPickingAccuracy,
    
    -- Fleet metrics
    AVG(fft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    AVG(fft.FuelConsumedGallons) AS AvgFuelPerTrip,
    AVG(CAST(fft.OnTimeDelivery AS FLOAT)) AS OnTimeDeliveryRate,
    
    -- Cross-fact KPI: Dwell-to-Idle correlation coefficient
    -- High dwell time should correlate with high idle time
    CASE 
        WHEN AVG(fwo.DwellTimeHours) > 48 AND AVG(fft.IdleTimeMinutes) > 30 
        THEN 'High Risk Zone'
        WHEN AVG(fwo.DwellTimeHours) > 24 AND AVG(fft.IdleTimeMinutes) > 15 
        THEN 'Moderate Risk'
        ELSE 'Normal Operations'
    END AS OperationalRiskLevel

FROM [logistics].[FactWarehouseOperations] fwo
INNER JOIN [logistics].[DimTime] dt ON fwo.TimeKey = dt.TimeKey
INNER JOIN [logistics].[DimGeography] dg ON fwo.GeographyKey = dg.GeographyKey
INNER JOIN [logistics].[DimProduct] dp ON fwo.ProductKey = dp.ProductKey
LEFT JOIN [logistics].[FactFleetTrips] fft 
    ON fft.OriginKey = fwo.GeographyKey 
    AND fft.TimeKey = fwo.TimeKey

WHERE dt.FullDate >= DATEADD(MONTH, -3, GETDATE())
    AND dg.WarehouseActive = 1

GROUP BY 
    dt.FiscalYear,
    dt.FiscalQuarter,
    dt.MonthName,
    dg.WarehouseName,
    dp.ProductCategory

ORDER BY AvgDwellTimeHours DESC, AvgFleetIdleMinutes DESC;
```

### Warehouse Gravity Zone Analysis

```sql
-- Calculate and assign gravity scores to products
-- Higher gravity = higher priority placement near shipping docks
CREATE OR ALTER VIEW [logistics].[vw_ProductGravityScore] AS
SELECT 
    dp.ProductKey,
    dp.SKU,
    dp.ProductName,
    dp.ProductCategory,
    
    -- Velocity component (pick frequency)
    COUNT(DISTINCT fwo.OperationID) AS PickFrequency,
    
    -- Value component (revenue per unit)
    AVG(dp.UnitPrice) AS AvgUnitPrice,
    
    -- Fragility component (handling care required)
    dp.FragilityScore,
    
    -- Lead time component (replenishment urgency)
    AVG(ds.LeadTimeVarianceDays) AS AvgLeadTimeVariance,
    
    -- Composite gravity score (weighted formula)
    (
        (COUNT(DISTINCT fwo.OperationID) * 0.40) +  -- 40% weight on velocity
        (AVG(dp.UnitPrice) / 10 * 0.30) +            -- 30% weight on value
        (dp.FragilityScore * 0.20) +                 -- 20% weight on fragility
        ((30 - AVG(ds.LeadTimeVarianceDays)) * 0.10) -- 10% weight on lead time stability
    ) AS GravityScore,
    
    -- Zone assignment
    CASE 
        WHEN (
            (COUNT(DISTINCT fwo.OperationID) * 0.40) +
            (AVG(dp.UnitPrice) / 10 * 0.30) +
            (dp.FragilityScore * 0.20) +
            ((30 - AVG(ds.LeadTimeVarianceDays)) * 0.10)
        ) >= 50 THEN 'Zone A - High Gravity'
        WHEN (
            (COUNT(DISTINCT fwo.OperationID) * 0.40) +
            (AVG(dp.UnitPrice) / 10 * 0.30) +
            (dp.FragilityScore * 0.20) +
            ((30 - AVG(ds.LeadTimeVarianceDays)) * 0.10)
        ) >= 30 THEN 'Zone B - Medium Gravity'
        ELSE 'Zone C - Low Gravity'
    END AS RecommendedZone

FROM [logistics].[DimProduct] dp
LEFT JOIN [logistics].[FactWarehouseOperations] fwo 
    ON dp.ProductKey = fwo.ProductKey
    AND fwo.OperationType = 'PICK'
LEFT JOIN [logistics].[DimSupplier] ds 
    ON dp.PrimarySupplierKey = ds.SupplierKey

WHERE dp.ProductActive = 1

GROUP BY 
    dp.ProductKey,
    dp.SKU,
    dp.ProductName,
    dp.ProductCategory,
    dp.FragilityScore;
GO
```

### Predictive Bottleneck Detection

```sql
-- Identify operational bottlenecks using time-series trend analysis
WITH HourlyTrends AS (
    SELECT 
        dt.FullDate,
        dt.Hour,
        dg.WarehouseName,
        AVG(fwo.DurationMinutes) AS AvgOperationDuration,
        COUNT(*) AS OperationVolume,
        
        -- Moving average over previous 4 hours
        AVG(AVG(fwo.DurationMinutes)) OVER (
            PARTITION BY dg.GeographyKey 
            ORDER BY dt.FullDate, dt.Hour 
            ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
        ) AS MovingAvgDuration
        
    FROM [logistics].[FactWarehouseOperations] fwo
    INNER JOIN [logistics].[DimTime] dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN [logistics].[DimGeography] dg ON fwo.GeographyKey = dg.GeographyKey
    
    WHERE dt.FullDate >= DATEADD(DAY, -7, GETDATE())
    
    GROUP BY dt.FullDate, dt.Hour, dg.WarehouseName, dg.GeographyKey
)
SELECT 
    FullDate,
    Hour,
    WarehouseName,
    AvgOperationDuration,
    OperationVolume,
    MovingAvgDuration,
    
    -- Detect when current duration exceeds moving average by >30%
    CASE 
        WHEN AvgOperationDuration > (MovingAvgDuration * 1.30) 
        THEN 'Bottleneck Detected'
        WHEN AvgOperationDuration > (MovingAvgDuration * 1.15) 
        THEN 'Warning - Slowing Down'
        ELSE 'Normal'
    END AS BottleneckStatus,
    
    -- Calculate severity score (0-100)
    CASE 
        WHEN MovingAvgDuration > 0 
        THEN ROUND(
            ((AvgOperationDuration - MovingAvgDuration) / MovingAvgDuration) * 100, 
            2
        )
        ELSE 0 
    END AS SeverityScore

FROM HourlyTrends

WHERE AvgOperationDuration > (MovingAvgDuration * 1.15)

ORDER BY FullDate DESC, Hour DESC, SeverityScore DESC;
```

## Power BI DAX Measures

### Fleet Utilization Rate

```dax
Fleet Utilization Rate = 
VAR TotalTripMinutes = SUM(FactFleetTrips[DurationMinutes])
VAR TotalIdleMinutes = SUM(FactFleetTrips[IdleTimeMinutes])
VAR TotalAvailableMinutes = TotalTripMinutes + TotalIdleMinutes
RETURN
    DIVIDE(
        TotalTripMinutes - TotalIdleMinutes,
        TotalAvailableMinutes,
        0
    )
```

### Warehouse Throughput Efficiency

```dax
Warehouse Throughput Efficiency = 
VAR TotalUnitsHandled = SUM(FactWarehouseOperations[QuantityHandled])
VAR TotalLaborHours = SUM(FactWarehouseOperations[DurationMinutes]) / 60
VAR BaselineUnitsPerHour = 45  // Industry benchmark
RETURN
    DIVIDE(
        DIVIDE(TotalUnitsHandled, TotalLaborHours, 0),
        BaselineUnitsPerHour,
        0
    ) * 100
```

### Cross-Modal Delivery Performance

```dax
Cross-Modal Delivery Performance = 
VAR WarehousePickAccuracy = AVERAGE(FactWarehouseOperations[PickingAccuracy])
VAR FleetOnTimeRate = AVERAGE(FactFleetTrips[OnTimeDelivery])
VAR CrossDockEfficiency = 
    DIVIDE(
        COUNTROWS(
            FILTER(FactCrossDock, FactCrossDock[DwellTimeMinutes] < 120)
        ),
        COUNTROWS(FactCrossDock),
        0
    )
RETURN
    (WarehousePickAccuracy * 0.35) + 
    (FleetOnTimeRate * 0.40) + 
    (CrossDockEfficiency * 0.25)
```

### Fuel Cost per Delivery

```dax
Fuel Cost per Delivery = 
VAR FuelPrice = 3.85  // Use parameter table in production
VAR TotalFuelCost = SUM(FactFleetTrips[FuelConsumedGallons]) * FuelPrice
VAR TotalDeliveries = COUNTROWS(FactFleetTrips)
RETURN
    DIVIDE(TotalFuelCost, TotalDeliveries, 0)
```

## Automated Alerting Setup

Create a stored procedure for threshold-based alerts:

```sql
-- Alert system for critical KPI breaches
CREATE OR ALTER PROCEDURE [logistics].[usp_GenerateOperationalAlerts]
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertRecipient VARCHAR(255) = '${ALERT_EMAIL_ADDRESS}';
    DECLARE @AlertSubject VARCHAR(500);
    DECLARE @AlertBody VARCHAR(MAX);
    
    -- Alert 1: Fleet idle time exceeds 20% of trip duration
    IF EXISTS (
        SELECT 1 
        FROM [logistics].[FactFleetTrips] fft
        INNER JOIN [logistics].[DimTime] dt ON fft.TimeKey = dt.TimeKey
        WHERE dt.FullDate = CAST(GETDATE() AS DATE)
            AND (CAST(fft.IdleTimeMinutes AS FLOAT) / fft.DurationMinutes) > 0.20
    )
    BEGIN
        SET @AlertSubject = 'ALERT: Fleet Idle Time Threshold Exceeded';
        SET @AlertBody = 'Multiple fleet trips today have idle time exceeding 20% of total trip duration. Review fleet efficiency dashboard.';
        
        -- Integration point for email/SMS/Teams notification
        INSERT INTO [logistics].[AlertLog] (AlertType, Severity, Message, CreatedAt)
        VALUES ('Fleet Efficiency', 'High', @AlertBody, GETDATE());
    END;
    
    -- Alert 2: Warehouse dwell time exceeds 72 hours for perishables
    IF EXISTS (
        SELECT 1 
        FROM [logistics].[FactWarehouseOperations] fwo
        INNER JOIN [logistics].[DimProduct] dp ON fwo.ProductKey = dp.ProductKey
        INNER JOIN [logistics].[DimTime] dt ON fwo.TimeKey = dt.TimeKey
        WHERE dt.FullDate >= DATEADD(DAY, -1, GETDATE())
            AND dp.ProductCategory = 'Perishables'
            AND fwo.DwellTimeHours > 72
    )
    BEGIN
        SET @AlertSubject = 'CRITICAL: Perishable Goods Dwell Time Exceeded';
        SET @AlertBody = 'Perishable products have exceeded 72-hour dwell time threshold. Immediate action required to prevent spoilage.';
        
        INSERT INTO [logistics].[AlertLog] (AlertType, Severity, Message, CreatedAt)
        VALUES ('Inventory Risk', 'Critical', @AlertBody, GETDATE());
    END;
    
    -- Alert 3: On-time delivery rate drops below 90%
    DECLARE @CurrentOTD FLOAT;
    SELECT @CurrentOTD = AVG(CAST(OnTimeDelivery AS FLOAT))
    FROM [logistics].[FactFleetTrips] fft
    INNER JOIN [logistics].[DimTime] dt ON fft.TimeKey = dt.TimeKey
    WHERE dt.FullDate >= DATEADD(DAY, -7, GETDATE());
    
    IF @CurrentOTD < 0.90
    BEGIN
        SET @AlertSubject = 'WARNING: On-Time Delivery Rate Below Target';
        SET @AlertBody = CONCAT(
            'Weekly on-time delivery rate has dropped to ',
            CAST(ROUND(@CurrentOTD * 100, 2) AS VARCHAR),
            '%, below the 90% SLA threshold.'
        );
        
        INSERT INTO [logistics].[AlertLog] (AlertType, Severity, Message, CreatedAt)
        VALUES ('Service Level', 'Medium', @AlertBody, GETDATE());
    END;
END;
GO

-- Schedule this procedure to run every 15 minutes via SQL Server Agent
```

## Role-Based Security Implementation

```sql
-- Create role-based views with row-level security
CREATE OR ALTER VIEW [logistics].[vw_WarehouseOperations_Regional] AS
SELECT 
    fwo.*,
    dt.FullDate,
    dg.WarehouseName,
    dg.RegionName,
    dp.ProductCategory
FROM [logistics].[FactWarehouseOperations] fwo
INNER JOIN [logistics].[DimTime] dt ON fwo.TimeKey = dt.TimeKey
INNER JOIN [logistics].[DimGeography] dg ON fwo.GeographyKey = dg.GeographyKey
INNER JOIN [logistics].[DimProduct] dp ON fwo.ProductKey = dp.ProductKey
WHERE dg.RegionName = (
    SELECT RegionName 
    FROM [logistics].[UserRegionMapping] 
    WHERE UserName = SYSTEM_USER
);
GO

-- Apply row-level security in Power BI
-- In Power BI, create a role filter on DimGeography:
-- [RegionName] = USERNAME()
-- Then map Active Directory users to roles
```

## Common Patterns

### Pattern 1: Time-Intelligence Comparisons

```dax
// Year-over-Year warehouse throughput comparison
YoY Throughput Change = 
VAR CurrentYearThroughput = 
    CALCULATE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        DATESYTD(DimTime[FullDate])
    )
VAR PriorYearThroughput = 
    CALCULATE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        SAMEPERIODLASTYEAR(DimTime[FullDate])
    )
RETURN
    DIVIDE(
        CurrentYearThroughput - PriorYearThroughput,
        PriorYearThroughput,
        0
    )
```

### Pattern 2: Dynamic Segmentation

```dax
// ABC Classification based on revenue contribution
Product ABC Class = 
VAR TotalRevenue = 
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[QuantityHandled] * RELATED(DimProduct[UnitPrice])
    )
VAR ProductRevenue = 
    SUMX(
        FILTER(
            FactWarehouseOperations,
            FactWarehouseOperations[ProductKey] = EARLIER(DimProduct[ProductKey])
        ),
        FactWarehouseOperations[QuantityHandled] * RELATED(DimProduct[UnitPrice])
    )
VAR ContributionPct = DIVIDE(ProductRevenue, TotalRevenue, 0)
RETURN
    SWITCH(
        TRUE(),
        ContributionPct >= 0.70, "A - High Value",
        ContributionPct >= 0.20, "B - Medium Value",
        "C - Low Value"
    )
```

### Pattern 3: Conditional Formatting Logic

```dax
// KPI status indicator for dashboard tiles
Fleet KPI Status = 
VAR UtilizationRate = [Fleet Utilization Rate]
VAR Target = 0.85
RETURN
    SWITCH(
        TRUE(),
        UtilizationRate >= Target, "Green",
        UtilizationRate >= (Target * 0.90), "Yellow",
        "Red"
    )
```

## Troubleshooting

### Issue: Power BI refresh fails with timeout

**Solution:** Implement incremental refresh on large fact tables.

In Power BI Desktop:
1. Create RangeStart and RangeEnd parameters (DateTime type)
2. Filter fact table: `DimTime[FullDate] >= RangeStart and DimTime[FullDate] < RangeEnd`
3. Publish to Power BI Service
4. Configure incremental refresh policy (e.g., refresh last 7 days, archive 2 years)

### Issue: Cross-fact queries are slow

**Solution:** Add covering indexes on join columns.

```sql
-- Create composite index on FactFleetTrips for common joins
CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Coverage
ON [logistics].[FactFleetTrips] (TimeKey, OriginKey, DestinationKey)
INCLUDE (DistanceMiles, FuelConsumedGallons, IdleTimeMinutes, OnTimeDelivery);

-- Create filtered index for recent data queries
CREATE NONCLUSTERED INDEX IX_FactWarehouseOperations_Recent
ON [logistics].[FactWarehouseOperations] (TimeKey, GeographyKey, ProductKey)
INCLUDE (QuantityHandled, DwellTimeHours)
WHERE TimeKey >= 20240101;  -- Adjust based on your DimTime key format
```

### Issue: Gravity zone assignments don't reflect current reality

**Solution:** Schedule regular recalculation of gravity scores.

```sql
-- Create a SQL Server Agent job to refresh gravity scores weekly
CREATE OR ALTER PROCEDURE [logistics].[usp_RefreshProductGravityScores]
AS
BEGIN
    -- Truncate and reload the gravity score dimension
    TRUNCATE TABLE [logistics].[DimProductGravity];
    
    INSERT INTO [logistics].[DimProductGravity] (
        ProductKey, SKU, GravityScore, RecommendedZone, LastCalculated
    )
    SELECT 
        ProductKey,
        SKU,
        GravityScore,
        RecommendedZone,
        GETDATE()
    FROM [logistics].[vw_ProductGravityScore];
END;
GO
```

### Issue: DAX measures return blank values

**Solution:** Check relationship cardinality and filter direction.

```dax
// Debug measure to identify missing relationships
Relationship Check = 
VAR FactCount = COUNTROWS(FactWarehouseOperations)
VAR DimCount = COUNTROWS(DimProduct)
VAR JoinedCount = 
    COUNTROWS(
        FILTER(
            FactWarehouseOperations,
            NOT(ISBLANK(RELATED(DimProduct[ProductName])))
        )
    )
RETURN
    "Fact rows: " & FactCount & 
    " | Dim rows: " & DimCount & 
    " | Joined: " & JoinedCount
```

### Issue: Real-time dashboard doesn't update

**Solution:** Verify DirectQuery mode or scheduled refresh configuration.

For near-real-time data:
- Set fact tables to DirectQuery mode
- Keep dimension tables in Import mode (Composite model)
- Configure automatic page refresh in Power BI Service (Premium required)

```powerquery
// Power Query M code for DirectQuery source
let
    Source = Sql.Database(
        "${SQL_SERVER_NAME}", 
        "${DATABASE_NAME}",
        [Query="SELECT * FROM logistics.FactWarehouseOperations WHERE TimeKey >= DATEADD(day, -1, GETDATE())"]
    )
in
    Source
```

## Best Practices

1. **Partition large fact tables** by date range (monthly or quarterly) for faster queries
2. **Use columnstore indexes** on fact tables with > 1M rows
3. **Implement data retention policies** to archive historical data beyond 2 years
4. **Test DAX measures with SUMMARIZE** to verify grain alignment
5. **Document custom calculations** in measure descriptions for team collaboration
6. **Use variables in DAX** to improve readability and performance
7. **Create a semantic layer** with business-friendly column names in Power BI model
8. **Version control your .pbix files** using Git LFS or Power BI deployment pipelines

## Additional Resources

- SQL schema scripts: `./sql/` directory in repository
- Power 
