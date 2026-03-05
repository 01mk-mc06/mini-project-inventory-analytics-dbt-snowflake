# Snowflake Validation Queries 
##  Quick Reference Guide

This document contains all Snowflake SQL commands used to validate and query the dbt project objects.

---

##  Part 1: Database & Schema Exploration

### Check Available Databases

```sql
-- Show all databases you have access to
SHOW DATABASES;

-- Show databases with pattern matching
SHOW DATABASES LIKE 'RAW%';

-- Get detailed database information
SELECT 
    DATABASE_NAME,
    CREATED,
    OWNER,
    COMMENT
FROM INFORMATION_SCHEMA.DATABASES
WHERE DATABASE_NAME = 'RAW_DATA_DB';
```

---

### Check All Schemas in Database

```sql
-- Switch to database
USE DATABASE RAW_DATA_DB;

-- Show all schemas
SHOW SCHEMAS;

-- Show schemas created by dbt
SHOW SCHEMAS LIKE 'dbt_dev%';

-- Detailed schema information
SELECT 
    SCHEMA_NAME,
    CREATED,
    LAST_ALTERED,
    COMMENT
FROM INFORMATION_SCHEMA.SCHEMATA
WHERE SCHEMA_NAME LIKE 'dbt_dev%'
ORDER BY SCHEMA_NAME;
```

---

### Check Current Context

```sql
-- What database am I using?
SELECT CURRENT_DATABASE();

-- What schema am I using?
SELECT CURRENT_SCHEMA();

-- What role am I using?
SELECT CURRENT_ROLE();

-- What warehouse am I using?
SELECT CURRENT_WAREHOUSE();

-- All context at once
SELECT 
    CURRENT_DATABASE() AS current_db,
    CURRENT_SCHEMA() AS current_schema,
    CURRENT_ROLE() AS current_role,
    CURRENT_WAREHOUSE() AS current_wh,
    CURRENT_USER() AS current_user;
```

---

##  Part 2: Seeds Layer Validation

### Navigate to Seeds Schema

```sql
USE DATABASE RAW_DATA_DB;
USE SCHEMA dbt_dev_raw_seeds;
```

---

### List All Seed Tables

```sql
-- Show all tables in seeds schema
SHOW TABLES IN SCHEMA dbt_dev_raw_seeds;

-- Detailed table information
SELECT 
    TABLE_NAME,
    TABLE_TYPE,
    ROW_COUNT,
    BYTES,
    CREATED,
    LAST_ALTERED
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'DBT_DEV_RAW_SEEDS'
ORDER BY TABLE_NAME;
```

---

### Validate Seeds Row Counts

```sql
-- Quick count of all seed tables
SELECT 'raw_products' AS table_name, COUNT(*) AS row_count 
FROM raw_products
UNION ALL
SELECT 'raw_warehouses', COUNT(*) 
FROM raw_warehouses
UNION ALL
SELECT 'raw_inventory_movements', COUNT(*) 
FROM raw_inventory_movements
UNION ALL
SELECT 'raw_inventory_levels', COUNT(*) 
FROM raw_inventory_levels;

-- Expected results:
-- raw_products: 10
-- raw_warehouses: 5
-- raw_inventory_movements: 20
-- raw_inventory_levels: 15
```

---

### Query Individual Seeds

```sql
-- View all products
SELECT * FROM raw_products;

-- Products with their costs
SELECT 
    product_id,
    product_name,
    category,
    unit_cost,
    reorder_point
FROM raw_products
ORDER BY category, product_name;

-- View all warehouses
SELECT * FROM raw_warehouses;

-- Active warehouses only
SELECT 
    warehouse_code,
    warehouse_name,
    location,
    capacity_sqm,
    is_active
FROM raw_warehouses
WHERE is_active = TRUE
ORDER BY warehouse_code;

-- View inventory movements
SELECT * FROM raw_inventory_movements
ORDER BY movement_date DESC;

-- Movement summary by type
SELECT 
    movement_type,
    COUNT(*) AS transaction_count,
    SUM(quantity) AS total_quantity
FROM raw_inventory_movements
GROUP BY movement_type;

-- View inventory levels
SELECT * FROM raw_inventory_levels;

-- Current inventory by product
SELECT 
    product_id,
    warehouse_location,
    quantity_on_hand,
    quantity_reserved,
    quantity_available
FROM raw_inventory_levels
WHERE snapshot_date = (SELECT MAX(snapshot_date) FROM raw_inventory_levels)
ORDER BY product_id, warehouse_location;
```

---

### Check Data Quality in Seeds

```sql
-- Check for NULL values in key columns
SELECT 'raw_products' AS table_name,
       SUM(CASE WHEN product_id IS NULL THEN 1 ELSE 0 END) AS null_product_id,
       SUM(CASE WHEN product_name IS NULL THEN 1 ELSE 0 END) AS null_product_name,
       SUM(CASE WHEN unit_cost IS NULL THEN 1 ELSE 0 END) AS null_unit_cost
FROM raw_products
UNION ALL
SELECT 'raw_warehouses',
       SUM(CASE WHEN warehouse_id IS NULL THEN 1 ELSE 0 END),
       SUM(CASE WHEN warehouse_code IS NULL THEN 1 ELSE 0 END),
       0
FROM raw_warehouses;

-- Check for duplicates
SELECT product_id, COUNT(*) AS duplicate_count
FROM raw_products
GROUP BY product_id
HAVING COUNT(*) > 1;

-- Should return 0 rows
```

---

##  Part 3: Staging Layer Validation

### Navigate to Staging Schema

```sql
USE DATABASE RAW_DATA_DB;
USE SCHEMA dbt_dev_staging;
```

---

### List All Staging Views

```sql
-- Show all views in staging schema
SHOW VIEWS IN SCHEMA dbt_dev_staging;

-- Detailed view information
SELECT 
    TABLE_NAME,
    TABLE_TYPE,
    CREATED,
    LAST_ALTERED,
    COMMENT
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = 'DBT_DEV_STAGING'
ORDER BY TABLE_NAME;
```

---

### Validate Staging Row Counts

```sql
-- Count rows in all staging views
SELECT 'stg_warehouse__products' AS view_name, COUNT(*) AS row_count 
FROM stg_warehouse__products
UNION ALL
SELECT 'stg_warehouse__warehouses', COUNT(*) 
FROM stg_warehouse__warehouses
UNION ALL
SELECT 'stg_warehouse__movements', COUNT(*) 
FROM stg_warehouse__movements
UNION ALL
SELECT 'stg_warehouse__levels', COUNT(*) 
FROM stg_warehouse__levels;

-- Should match seed counts:
-- stg_warehouse__products: 10
-- stg_warehouse__warehouses: 5
-- stg_warehouse__movements: 20
-- stg_warehouse__levels: 15
```

---

### Query Staging Views

```sql
-- View staging products
SELECT * FROM stg_warehouse__products;

-- Check audit columns were added
SELECT 
    product_id,
    product_name,
    dbt_loaded_at
FROM stg_warehouse__products
LIMIT 5;

-- View staging movements with derived columns
SELECT 
    movement_id,
    product_id,
    movement_type,
    quantity,
    quantity_inbound,
    quantity_outbound,
    dbt_loaded_at
FROM stg_warehouse__movements
LIMIT 10;

-- Check derived fields calculation
SELECT 
    movement_type,
    quantity,
    quantity_inbound,
    quantity_outbound,
    CASE 
        WHEN movement_type = 'INBOUND' AND quantity_inbound != quantity THEN 'ERROR'
        WHEN movement_type = 'OUTBOUND' AND quantity_outbound != ABS(quantity) THEN 'ERROR'
        ELSE 'OK'
    END AS validation
FROM stg_warehouse__movements;

-- View staging levels with calculated availability %
SELECT 
    product_id,
    warehouse_location,
    quantity_on_hand,
    quantity_available,
    availability_pct
FROM stg_warehouse__levels
ORDER BY availability_pct DESC;
```

---

### Compare Seeds vs Staging

```sql
-- Verify staging has same data as seeds
SELECT 
    'Products' AS table_name,
    (SELECT COUNT(*) FROM RAW_DATA_DB.dbt_dev_raw_seeds.raw_products) AS seed_count,
    (SELECT COUNT(*) FROM RAW_DATA_DB.dbt_dev_staging.stg_warehouse__products) AS staging_count,
    CASE 
        WHEN seed_count = staging_count THEN 'MATCH' 
        ELSE 'MISMATCH' 
    END AS status
UNION ALL
SELECT 
    'Warehouses',
    (SELECT COUNT(*) FROM RAW_DATA_DB.dbt_dev_raw_seeds.raw_warehouses),
    (SELECT COUNT(*) FROM RAW_DATA_DB.dbt_dev_staging.stg_warehouse__warehouses),
    CASE 
        WHEN (SELECT COUNT(*) FROM RAW_DATA_DB.dbt_dev_raw_seeds.raw_warehouses) = 
             (SELECT COUNT(*) FROM RAW_DATA_DB.dbt_dev_staging.stg_warehouse__warehouses) 
        THEN 'MATCH' ELSE 'MISMATCH' 
    END;
```

---

##  Part 4: Intermediate Layer Validation

### Navigate to Intermediate Schema

```sql
USE DATABASE RAW_DATA_DB;
USE SCHEMA dbt_dev_intermediate;
```

---

### List All Intermediate Views

```sql
-- Show all intermediate views
SHOW VIEWS IN SCHEMA dbt_dev_intermediate;

-- Detailed information
SELECT 
    TABLE_NAME,
    TABLE_TYPE,
    CREATED,
    LAST_ALTERED
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = 'DBT_DEV_INTERMEDIATE'
ORDER BY TABLE_NAME;
```

---

### Validate Intermediate Views

```sql
-- Count rows in intermediate views
SELECT 'int_warehouse__product_warehouse_grid' AS view_name, COUNT(*) AS row_count 
FROM int_warehouse__product_warehouse_grid
UNION ALL
SELECT 'int_warehouse__movement_summary', COUNT(*) 
FROM int_warehouse__movement_summary
UNION ALL
SELECT 'int_warehouse__latest_levels', COUNT(*) 
FROM int_warehouse__latest_levels;

-- Expected:
-- grid: 40 (10 products × 4 active warehouses)
-- movement_summary: ~10-15 (only where movements exist)
-- latest_levels: ~10-15 (only where levels exist for latest date)
```

---

### Query Intermediate Views

```sql
-- View the complete product-warehouse grid
SELECT * FROM int_warehouse__product_warehouse_grid
ORDER BY product_id, warehouse_code;

-- Verify CROSS JOIN created all combinations
SELECT 
    COUNT(DISTINCT product_id) AS unique_products,
    COUNT(DISTINCT warehouse_code) AS unique_warehouses,
    COUNT(*) AS total_combinations,
    COUNT(DISTINCT product_id) * COUNT(DISTINCT warehouse_code) AS expected_combinations
FROM int_warehouse__product_warehouse_grid;
-- total_combinations should equal expected_combinations

-- View movement summary
SELECT * FROM int_warehouse__movement_summary
ORDER BY total_movements DESC;

-- Products with most movements
SELECT 
    product_id,
    warehouse_location,
    total_inbound,
    total_outbound,
    total_movements,
    last_movement_date
FROM int_warehouse__movement_summary
ORDER BY total_movements DESC
LIMIT 5;

-- View latest levels only
SELECT * FROM int_warehouse__latest_levels
ORDER BY product_id, warehouse_location;

-- Check we only have latest snapshot
SELECT 
    COUNT(DISTINCT snapshot_date) AS distinct_dates,
    MIN(snapshot_date) AS min_date,
    MAX(snapshot_date) AS max_date
FROM RAW_DATA_DB.dbt_dev_staging.stg_warehouse__levels;
-- Should show latest date only in intermediate view
```

---

##  Part 5: Marts Layer Validation

### Navigate to Marts Schema

```sql
USE DATABASE RAW_DATA_DB;
USE SCHEMA dbt_dev_marts;
```

---

### List All Mart Tables

```sql
-- Show all tables in marts schema
SHOW TABLES IN SCHEMA dbt_dev_marts;

-- Detailed table information with row counts
SELECT 
    TABLE_NAME,
    TABLE_TYPE,
    ROW_COUNT,
    BYTES,
    CREATED,
    LAST_ALTERED
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'DBT_DEV_MARTS'
ORDER BY TABLE_NAME;
```

---

### Validate Marts Row Counts

```sql
-- Count rows in all marts
SELECT 'fct_inventory_summary' AS table_name, COUNT(*) AS row_count 
FROM fct_inventory_summary
UNION ALL
SELECT 'fct_inventory_turnover', COUNT(*) 
FROM fct_inventory_turnover
UNION ALL
SELECT 'fct_inventory_pivot', COUNT(*) 
FROM fct_inventory_pivot;

-- Expected:
-- fct_inventory_summary: 40 (10 products × 4 active warehouses)
-- fct_inventory_turnover: ~10-15 (only products with qty > 0)
-- fct_inventory_pivot: 10 (one row per product)
```

---

### Query Fact Tables

```sql
-- View complete inventory summary
SELECT * FROM fct_inventory_summary
ORDER BY product_id, warehouse_code;

-- Summary statistics
SELECT 
    COUNT(*) AS total_rows,
    COUNT(DISTINCT product_id) AS unique_products,
    COUNT(DISTINCT warehouse_code) AS unique_warehouses,
    SUM(current_quantity) AS total_inventory_units,
    SUM(inventory_value_usd) AS total_inventory_value
FROM fct_inventory_summary;

-- Products below reorder point
SELECT 
    product_name,
    warehouse_name,
    current_quantity,
    reorder_point,
    (reorder_point - current_quantity) AS shortage_qty
FROM fct_inventory_summary
WHERE is_below_reorder_point = TRUE
ORDER BY shortage_qty DESC;

-- Inventory value by warehouse
SELECT 
    warehouse_code,
    warehouse_name,
    SUM(inventory_value_usd) AS total_value,
    SUM(current_quantity) AS total_units,
    COUNT(DISTINCT product_id) AS product_count
FROM fct_inventory_summary
GROUP BY warehouse_code, warehouse_name
ORDER BY total_value DESC;

-- View inventory turnover
SELECT * FROM fct_inventory_turnover
ORDER BY turnover_ratio DESC NULLS LAST;

-- Slow-moving inventory (turnover < 1.0)
SELECT 
    product_name,
    warehouse_name,
    current_quantity,
    turnover_ratio,
    inventory_value_usd
FROM fct_inventory_turnover
WHERE turnover_ratio < 1.0
ORDER BY inventory_value_usd DESC;

-- View pivoted inventory
SELECT * FROM fct_inventory_pivot
ORDER BY product_name;

-- Total inventory across all warehouses
SELECT 
    product_name,
    WH_001_qty AS Manila,
    WH_002_qty AS Cebu,
    WH_003_qty AS Davao,
    WH_004_qty AS Laguna,
    (WH_001_qty + WH_002_qty + WH_003_qty + WH_004_qty) AS Total
FROM fct_inventory_pivot
ORDER BY Total DESC;
```

---

##  Part 6: Data Quality Validation

### Check for Data Anomalies

```sql
-- Check for negative inventory
SELECT 
    'Negative Current Quantity' AS check_name,
    COUNT(*) AS failed_count
FROM fct_inventory_summary
WHERE current_quantity < 0
UNION ALL
SELECT 
    'Negative Reserved Quantity',
    COUNT(*)
FROM fct_inventory_summary
WHERE reserved_quantity < 0
UNION ALL
SELECT 
    'Negative Available Quantity',
    COUNT(*)
FROM fct_inventory_summary
WHERE available_quantity < 0;
-- All should return 0

-- Verify reserved doesn't exceed on-hand
SELECT 
    product_name,
    warehouse_name,
    current_quantity,
    reserved_quantity
FROM fct_inventory_summary
WHERE reserved_quantity > current_quantity;
-- Should return 0 rows

-- Verify available calculation
SELECT 
    product_name,
    warehouse_name,
    current_quantity,
    reserved_quantity,
    available_quantity,
    (current_quantity - reserved_quantity) AS calculated_available,
    CASE 
        WHEN available_quantity = (current_quantity - reserved_quantity) 
        THEN 'OK' 
        ELSE 'ERROR' 
    END AS validation
FROM fct_inventory_summary
WHERE available_quantity != (current_quantity - reserved_quantity);
-- Should return 0 rows

-- Verify inventory value calculation
SELECT 
    product_name,
    current_quantity,
    unit_cost,
    inventory_value_usd,
    (current_quantity * unit_cost) AS calculated_value,
    ABS(inventory_value_usd - (current_quantity * unit_cost)) AS difference
FROM fct_inventory_summary
WHERE ABS(inventory_value_usd - (current_quantity * unit_cost)) > 0.01;
-- Should return 0 rows (allowing tiny rounding differences)
```

---

### Validate Referential Integrity

```sql
-- Check all products in fct exist in staging
SELECT 
    f.product_id,
    f.product_name
FROM fct_inventory_summary f
LEFT JOIN RAW_DATA_DB.dbt_dev_staging.stg_warehouse__products p
    ON f.product_id = p.product_id
WHERE p.product_id IS NULL;
-- Should return 0 rows

-- Check all warehouses in fct exist in staging
SELECT 
    f.warehouse_code,
    f.warehouse_name
FROM fct_inventory_summary f
LEFT JOIN RAW_DATA_DB.dbt_dev_staging.stg_warehouse__warehouses w
    ON f.warehouse_code = w.warehouse_code
WHERE w.warehouse_code IS NULL;
-- Should return 0 rows
```

---

##  Part 7: Lineage & Dependencies

### Check View Definitions

```sql
-- Get DDL for a staging view
SELECT GET_DDL('VIEW', 'RAW_DATA_DB.dbt_dev_staging.stg_warehouse__products');

-- Get DDL for intermediate view
SELECT GET_DDL('VIEW', 'RAW_DATA_DB.dbt_dev_intermediate.int_warehouse__product_warehouse_grid');

-- Get DDL for mart table
SELECT GET_DDL('TABLE', 'RAW_DATA_DB.dbt_dev_marts.fct_inventory_summary');
```

---

### Check Object Dependencies

```sql
-- What objects does fct_inventory_summary depend on?
SELECT 
    REFERENCED_OBJECT_NAME,
    REFERENCED_OBJECT_DOMAIN
FROM INFORMATION_SCHEMA.OBJECT_DEPENDENCIES
WHERE REFERENCING_OBJECT_NAME = 'FCT_INVENTORY_SUMMARY'
  AND REFERENCING_SCHEMA = 'DBT_DEV_MARTS'
ORDER BY REFERENCED_OBJECT_NAME;

-- What objects depend on stg_warehouse__products?
SELECT 
    REFERENCING_OBJECT_NAME,
    REFERENCING_OBJECT_DOMAIN
FROM INFORMATION_SCHEMA.OBJECT_DEPENDENCIES
WHERE REFERENCED_OBJECT_NAME = 'STG_WAREHOUSE__PRODUCTS'
  AND REFERENCED_SCHEMA = 'DBT_DEV_STAGING'
ORDER BY REFERENCING_OBJECT_NAME;
```

---

##  Part 8: Performance & Metadata

### Check Table Sizes

```sql
-- Storage used by all schemas
SELECT 
    TABLE_SCHEMA,
    COUNT(*) AS object_count,
    SUM(ROW_COUNT) AS total_rows,
    SUM(BYTES) AS total_bytes,
    ROUND(SUM(BYTES) / 1024 / 1024, 2) AS total_mb
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA LIKE 'dbt_dev%'
GROUP BY TABLE_SCHEMA
ORDER BY total_bytes DESC;

-- Detailed table/view listing with sizes
SELECT 
    TABLE_SCHEMA,
    TABLE_NAME,
    TABLE_TYPE,
    ROW_COUNT,
    BYTES,
    ROUND(BYTES / 1024 / 1024, 2) AS size_mb,
    CREATED,
    LAST_ALTERED
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA LIKE 'dbt_dev%'
ORDER BY TABLE_SCHEMA, TABLE_NAME;
```

---

### Check Query History (dbt runs)

```sql
-- Recent queries with dbt query tag
SELECT 
    QUERY_ID,
    QUERY_TEXT,
    QUERY_TAG,
    DATABASE_NAME,
    SCHEMA_NAME,
    START_TIME,
    END_TIME,
    EXECUTION_TIME,
    ROWS_PRODUCED,
    BYTES_SCANNED
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE QUERY_TAG = 'dbt_warehouse_inventory'
ORDER BY START_TIME DESC
LIMIT 20;

-- Count queries by query tag
SELECT 
    QUERY_TAG,
    COUNT(*) AS query_count,
    SUM(EXECUTION_TIME) AS total_execution_ms,
    AVG(EXECUTION_TIME) AS avg_execution_ms
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE START_TIME > DATEADD(hour, -24, CURRENT_TIMESTAMP())
  AND QUERY_TAG LIKE 'dbt%'
GROUP BY QUERY_TAG
ORDER BY query_count DESC;
```

---

##  Part 9: Business Intelligence Queries

### Inventory Analysis

```sql
-- Current inventory position summary
SELECT 
    p.category,
    COUNT(DISTINCT f.product_id) AS product_count,
    SUM(f.current_quantity) AS total_units,
    SUM(f.inventory_value_usd) AS total_value,
    ROUND(AVG(f.availability_pct), 2) AS avg_availability_pct
FROM fct_inventory_summary f
JOIN RAW_DATA_DB.dbt_dev_staging.stg_warehouse__products p
    ON f.product_id = p.product_id
GROUP BY p.category
ORDER BY total_value DESC;

-- Warehouse utilization
SELECT 
    warehouse_name,
    capacity_sqm,
    SUM(current_quantity) AS total_units_stored,
    SUM(inventory_value_usd) AS total_value_stored,
    COUNT(DISTINCT product_id) AS product_variety
FROM fct_inventory_summary
GROUP BY warehouse_name, capacity_sqm
ORDER BY total_value_stored DESC;

-- Reorder alerts
SELECT 
    product_name,
    category,
    warehouse_name,
    current_quantity,
    reorder_point,
    (reorder_point - current_quantity) AS order_quantity_needed,
    (reorder_point - current_quantity) * unit_cost AS order_value_needed
FROM fct_inventory_summary
WHERE is_below_reorder_point = TRUE
ORDER BY order_value_needed DESC;

-- Movement velocity
SELECT 
    p.product_name,
    p.category,
    SUM(m.total_inbound) AS total_received,
    SUM(m.total_outbound) AS total_shipped,
    SUM(m.total_movements) AS total_transactions,
    MAX(m.last_movement_date) AS last_activity
FROM RAW_DATA_DB.dbt_dev_intermediate.int_warehouse__movement_summary m
JOIN RAW_DATA_DB.dbt_dev_staging.stg_warehouse__products p
    ON m.product_id = p.product_id
GROUP BY p.product_name, p.category
ORDER BY total_transactions DESC;
```

---

### Comparative Analysis

```sql
-- Product inventory distribution across warehouses
SELECT 
    product_name,
    SUM(CASE WHEN warehouse_code = 'WH-001' THEN current_quantity ELSE 0 END) AS Manila,
    SUM(CASE WHEN warehouse_code = 'WH-002' THEN current_quantity ELSE 0 END) AS Cebu,
    SUM(CASE WHEN warehouse_code = 'WH-003' THEN current_quantity ELSE 0 END) AS Davao,
    SUM(CASE WHEN warehouse_code = 'WH-004' THEN current_quantity ELSE 0 END) AS Laguna,
    SUM(current_quantity) AS Total
FROM fct_inventory_summary
GROUP BY product_name
ORDER BY Total DESC;

-- Category performance
SELECT 
    p.category,
    COUNT(DISTINCT f.product_id) AS product_count,
    SUM(f.lifetime_inbound) AS total_inbound,
    SUM(f.lifetime_outbound) AS total_outbound,
    ROUND(SUM(f.lifetime_outbound)::FLOAT / NULLIF(SUM(f.lifetime_inbound), 0), 2) AS outbound_ratio,
    SUM(f.current_quantity) AS current_stock,
    SUM(f.inventory_value_usd) AS current_value
FROM fct_inventory_summary f
JOIN RAW_DATA_DB.dbt_dev_staging.stg_warehouse__products p
    ON f.product_id = p.product_id
GROUP BY p.category
ORDER BY current_value DESC;
```

---

##  Part 10: Cleanup & Maintenance

### Drop All Project Objects (CAUTION!)

```sql
-- DROP ENTIRE SCHEMAS (CAUTION - THIS DELETES EVERYTHING!)
-- Only run if you want to completely remove the project

USE DATABASE RAW_DATA_DB;

-- Drop marts schema
DROP SCHEMA IF EXISTS dbt_dev_marts CASCADE;

-- Drop intermediate schema
DROP SCHEMA IF EXISTS dbt_dev_intermediate CASCADE;

-- Drop staging schema
DROP SCHEMA IF EXISTS dbt_dev_staging CASCADE;

-- Drop seeds schema
DROP SCHEMA IF EXISTS dbt_dev_raw_seeds CASCADE;

-- Verify schemas are gone
SHOW SCHEMAS LIKE 'dbt_dev%';
-- Should return 0 rows
```

---

### Truncate Specific Tables (Keep structure)

```sql
-- Truncate a specific table (keeps table structure, removes data)
TRUNCATE TABLE IF EXISTS RAW_DATA_DB.dbt_dev_raw_seeds.raw_products;

-- You would then re-run: dbt seed
```

---

### Drop Specific Objects

```sql
-- Drop a specific table
DROP TABLE IF EXISTS RAW_DATA_DB.dbt_dev_marts.fct_inventory_summary;

-- Drop a specific view
DROP VIEW IF EXISTS RAW_DATA_DB.dbt_dev_staging.stg_warehouse__products;

-- You would then re-run: dbt run --select <model_name>
```

---

##  Part 11: Quick Validation Checklist

### Run This Full Validation Query

```sql
-- Complete validation in one query
WITH schema_counts AS (
    SELECT 
        'Seeds' AS layer,
        COUNT(*) AS object_count,
        SUM(ROW_COUNT) AS total_rows
    FROM INFORMATION_SCHEMA.TABLES
    WHERE TABLE_SCHEMA = 'DBT_DEV_RAW_SEEDS'
    
    UNION ALL
    
    SELECT 
        'Staging',
        COUNT(*),
        0  -- Views don't have row_count
    FROM INFORMATION_SCHEMA.VIEWS
    WHERE TABLE_SCHEMA = 'DBT_DEV_STAGING'
    
    UNION ALL
    
    SELECT 
        'Intermediate',
        COUNT(*),
        0
    FROM INFORMATION_SCHEMA.VIEWS
    WHERE TABLE_SCHEMA = 'DBT_DEV_INTERMEDIATE'
    
    UNION ALL
    
    SELECT 
        'Marts',
        COUNT(*),
        SUM(ROW_COUNT)
    FROM INFORMATION_SCHEMA.TABLES
    WHERE TABLE_SCHEMA = 'DBT_DEV_MARTS'
)
SELECT 
    layer,
    object_count,
    total_rows,
    CASE 
        WHEN layer = 'Seeds' AND object_count = 4 THEN '✓'
        WHEN layer = 'Staging' AND object_count = 4 THEN '✓'
        WHEN layer = 'Intermediate' AND object_count = 3 THEN '✓'
        WHEN layer = 'Marts' AND object_count = 3 THEN '✓'
        ELSE '✗'
    END AS status
FROM schema_counts
ORDER BY 
    CASE layer
        WHEN 'Seeds' THEN 1
        WHEN 'Staging' THEN 2
        WHEN 'Intermediate' THEN 3
        WHEN 'Marts' THEN 4
    END;

-- Expected output:
-- Seeds        | 4  | 50  | ✓
-- Staging      | 4  | 0   | ✓
-- Intermediate | 3  | 0   | ✓
-- Marts        | 3  | 60  | ✓
```

---

##  Part 12: Export Results

### Export Query Results to CSV (Snowflake UI)

```sql
-- Run any query, then click "Download" button in Snowflake UI
SELECT * FROM fct_inventory_summary;

-- Or use COPY INTO for programmatic export
COPY INTO @my_stage/inventory_summary.csv
FROM fct_inventory_summary
FILE_FORMAT = (TYPE = CSV HEADER = TRUE)
OVERWRITE = TRUE;
```

---

##  Part 13: Useful System Queries

### Check Permissions

```sql
-- What privileges does my role have?
SHOW GRANTS TO ROLE ACCOUNTADMIN;

-- What roles do I have?
SHOW GRANTS TO USER CURRENT_USER();

-- Can I create tables in this schema?
SELECT 
    PRIVILEGE_TYPE,
    IS_GRANTABLE
FROM INFORMATION_SCHEMA.TABLE_PRIVILEGES
WHERE TABLE_SCHEMA = 'DBT_DEV_MARTS'
  AND GRANTEE = CURRENT_ROLE();
```

---

### Monitor Warehouse Usage

```sql
-- Warehouse usage in last 24 hours
SELECT 
    WAREHOUSE_NAME,
    SUM(CREDITS_USED) AS total_credits,
    COUNT(*) AS query_count
FROM TABLE(INFORMATION_SCHEMA.WAREHOUSE_METERING_HISTORY(
    DATE_RANGE_START => DATEADD(hour, -24, CURRENT_TIMESTAMP())
))
GROUP BY WAREHOUSE_NAME
ORDER BY total_credits DESC;
```

---

##  Quick Reference Commands

```sql
-- Essential Context Commands
USE DATABASE RAW_DATA_DB;
USE SCHEMA dbt_dev_marts;
SHOW DATABASES;
SHOW SCHEMAS;
SHOW TABLES;
SHOW VIEWS;

-- Essential Query Commands
SELECT * FROM fct_inventory_summary LIMIT 10;
SELECT COUNT(*) FROM fct_inventory_summary;
DESCRIBE TABLE fct_inventory_summary;

-- Essential Metadata Commands
SELECT CURRENT_DATABASE();
SELECT CURRENT_SCHEMA();
SELECT CURRENT_ROLE();
SELECT GET_DDL('TABLE', 'fct_inventory_summary');
```

---

##  Common Patterns

### Pattern: Check if Object Exists Before Querying

```sql
-- Safe way to check if table exists
SELECT COUNT(*) 
FROM INFORMATION_SCHEMA.TABLES 
WHERE TABLE_SCHEMA = 'DBT_DEV_MARTS' 
  AND TABLE_NAME = 'FCT_INVENTORY_SUMMARY';
-- Returns 1 if exists, 0 if not

-- Then conditionally query
SELECT * FROM RAW_DATA_DB.dbt_dev_marts.fct_inventory_summary
WHERE EXISTS (
    SELECT 1 
    FROM INFORMATION_SCHEMA.TABLES 
    WHERE TABLE_SCHEMA = 'DBT_DEV_MARTS' 
      AND TABLE_NAME = 'FCT_INVENTORY_SUMMARY'
);
```

---

### Pattern: Compare Before/After Counts

```sql
-- Store before count
SET before_count = (SELECT COUNT(*) FROM fct_inventory_summary);

-- Run dbt transformations...

-- Check after count
SELECT 
    $before_count AS before_count,
    COUNT(*) AS after_count,
    COUNT(*) - $before_count AS difference
FROM fct_inventory_summary;
```

---

##  End of Snowflake Validation Guide

This guide covers all essential Snowflake queries needed to validate and explore your dbt project!

**Quick Navigation:**
- Part 1-2: Database & Seeds exploration
- Part 3-5: Staging, Intermediate, Marts validation
- Part 6: Data quality checks
- Part 7-8: Lineage & performance
- Part 9: Business queries
- Part 10-13: Maintenance & utilities

**Remember:** Always run `dbt run` and `dbt test` before validating in Snowflake!
