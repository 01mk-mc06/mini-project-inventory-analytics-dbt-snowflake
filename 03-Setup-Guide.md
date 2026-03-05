# Environment Setup Guide

##  Prerequisites

### Hardware Requirements
- Windows 10/11
- 8GB RAM minimum
- Internet connection for Snowflake

### Software Requirements
- Python 3.8 - 3.11
- Git
- VS Code (recommended)
- Snowflake account (trial version sufficient)

---

##  Part 1: Virtual Environment Setup

### Location
```
C:\Users\King\dbt_learning\
```

### Create Virtual Environment

```powershell
# Navigate to project directory
cd C:\Users\King\dbt_learning

# Create virtual environment
python -m venv venv_warehouse

# Activate (Windows)
venv_warehouse\Scripts\activate

# You should see: (venv_warehouse) in your prompt
```

### Install dbt-snowflake

```powershell
# Install dbt with Snowflake adapter
pip install dbt-snowflake

# Verify installation
dbt --version
```

**Expected Output:**
```
Core:
  - installed: 1.11.6
  - latest:    1.11.6

Plugins:
  - snowflake: 1.11.3 - Up to date!
```

---

## ❄️ Part 2: Snowflake Configuration

### Create Snowflake Account

1. Go to: https://signup.snowflake.com/
2. Choose **Standard Edition**
3. Cloud Provider: **Azure**
4. Region: **Southeast Asia (Singapore)**
5. Complete registration

**Save These Credentials:**
- Account URL (e.g., `https://abc12345.southeast-asia.azure.snowflakecomputing.com`)
- Username
- Password

### Finding Account Identifier

**From URL:** `https://ABC12345.southeast-asia.azure.snowflakecomputing.com`

**Account Identifier:** `ABC12345.southeast-asia.azure`

---

### Initial Snowflake Setup

Login to Snowflake web UI and run:

```sql
-- Switch to ACCOUNTADMIN role
USE ROLE ACCOUNTADMIN;

-- Create database
CREATE DATABASE IF NOT EXISTS RAW_DATA_DB
  COMMENT = 'Database for dbt learning projects';

-- Create schemas
CREATE SCHEMA IF NOT EXISTS RAW_DATA_DB.dbt_dev
  COMMENT = 'Default development schema';

CREATE SCHEMA IF NOT EXISTS RAW_DATA_DB.raw_seeds
  COMMENT = 'Landing zone for seed data';

CREATE SCHEMA IF NOT EXISTS RAW_DATA_DB.staging
  COMMENT = 'Staging layer for cleaned data';

CREATE SCHEMA IF NOT EXISTS RAW_DATA_DB.intermediate
  COMMENT = 'Intermediate transformations';

CREATE SCHEMA IF NOT EXISTS RAW_DATA_DB.marts
  COMMENT = 'Analytics-ready fact tables';

-- Verify schemas created
SHOW SCHEMAS IN DATABASE RAW_DATA_DB;

-- Grant permissions to ACCOUNTADMIN
GRANT ALL PRIVILEGES ON DATABASE RAW_DATA_DB TO ROLE ACCOUNTADMIN;
GRANT ALL PRIVILEGES ON ALL SCHEMAS IN DATABASE RAW_DATA_DB TO ROLE ACCOUNTADMIN;

-- Verify warehouse exists
SHOW WAREHOUSES;
```

---

##  Part 3: dbt Profile Configuration

### Create profiles.yml

**Location:** `C:\Users\King\.dbt\profiles.yml`

**Content:**

```yaml
warehouse_inventory:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT_IDENTIFIER  # e.g., abc12345.southeast-asia.azure
      user: YOUR_USERNAME
      password: YOUR_PASSWORD
      role: ACCOUNTADMIN              # CRITICAL: Must be ACCOUNTADMIN
      database: RAW_DATA_DB
      warehouse: COMPUTE_WH
      schema: dbt_dev                 # Default schema
      threads: 4
      client_session_keep_alive: False
      query_tag: dbt_warehouse_inventory
```

**Replace:**
- `YOUR_ACCOUNT_IDENTIFIER` with your Snowflake account
- `YOUR_USERNAME` with your Snowflake username
- `YOUR_PASSWORD` with your Snowflake password

---

##  Part 4: Initialize dbt Project

### Create Project

```powershell
# Navigate to project directory
cd C:\Users\King\dbt_learning

# Initialize dbt project
dbt init project_01_warehouse_inventory

# When prompted:
# Profile name: warehouse_inventory
# Database: [1] snowflake

# Navigate into project
cd project_01_warehouse_inventory

# Test connection
dbt debug
```

**Expected Output:**
```
Configuration:
  profiles.yml file [OK found and valid]
  dbt_project.yml file [OK found and valid]

Connection:
  account: OK
  user: OK
  ...
  Connection test: [OK connection ok]
```

---

##  Part 5: Configure dbt_project.yml

**File:** `dbt_project.yml`

Replace contents with:

```yaml
name: 'project_01_warehouse_inventory'
version: '1.0.0'
config-version: 2

profile: 'warehouse_inventory'

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

clean-targets:
  - "target"
  - "dbt_packages"

# Seed configuration
seeds:
  project_01_warehouse_inventory:
    +schema: raw_seeds
    +quote_columns: false
    
    raw_products:
      +column_types:
        product_id: number
        unit_cost: number(10,2)
        supplier_id: number
        reorder_point: number
        created_at: timestamp
        updated_at: timestamp
        
    raw_warehouses:
      +column_types:
        warehouse_id: number
        capacity_sqm: number(10,2)
        is_active: boolean
        opened_date: date
        
    raw_inventory_movements:
      +column_types:
        movement_id: number
        product_id: number
        quantity: number
        movement_date: timestamp
        
    raw_inventory_levels:
      +column_types:
        snapshot_date: date
        product_id: number
        quantity_on_hand: number
        quantity_reserved: number
        quantity_available: number

# Model configuration
models:
  project_01_warehouse_inventory:
    +query_tag: dbt_warehouse_inventory
    
    staging:
      +materialized: view
      +schema: staging
      +tags: ['staging']
      warehouse:
        +materialized: view
        +schema: staging
        
    intermediate:
      +materialized: view
      +schema: intermediate
      +tags: ['intermediate']
      warehouse:
        +materialized: view
        +schema: intermediate
        
    marts:
      +materialized: table
      +schema: marts
      +tags: ['marts']
```

---

##  Part 6: Package Dependencies

### Create packages.yml

**File:** `packages.yml` (in project root)

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1
    
  - package: calogica/dbt_expectations
    version: 0.10.1
    
  - package: dbt-labs/audit_helper
    version: 0.9.0
```

### Install Packages

```powershell
dbt deps
```

**Expected Output:**
```
Installing dbt-labs/dbt_utils
Installing calogica/dbt_expectations
Installing dbt-labs/audit_helper
Installed 3 packages
```

---

##  Part 7: Create Project Structure

```powershell
# Create folder structure
mkdir models\staging
mkdir models\staging\warehouse
mkdir models\intermediate
mkdir models\intermediate\warehouse
mkdir models\marts
mkdir tests
mkdir tests\generic
mkdir macros

# Verify structure
tree /F
```

---

##  Part 8: Verification Steps

### Test Connection

```powershell
dbt debug
```

Should show: `Connection test: [OK connection ok]`

### Verify Snowflake Access

```sql
-- In Snowflake UI
SELECT CURRENT_ROLE();        -- Should be ACCOUNTADMIN
SELECT CURRENT_DATABASE();    -- Should be RAW_DATA_DB
SELECT CURRENT_WAREHOUSE();   -- Should be COMPUTE_WH

SHOW SCHEMAS IN DATABASE RAW_DATA_DB;
-- Should show: dbt_dev, raw_seeds, staging, intermediate, marts
```

---

##  Part 9: Initial Data Load

### Download Seed CSV Files

Place these 4 CSV files in `seeds/` folder:
1. `raw_products.csv` (10 rows)
2. `raw_warehouses.csv` (5 rows)
3. `raw_inventory_movements.csv` (20 rows)
4. `raw_inventory_levels.csv` (15 rows)

### Load Seeds

```powershell
dbt seed
```

**Expected Output:**
```
Running with dbt=1.11.6
Found 4 seeds

1 of 4 OK loaded seed file dbt_dev_raw_seeds.raw_products ............ [CREATE 10 in 1.2s]
2 of 4 OK loaded seed file dbt_dev_raw_seeds.raw_warehouses .......... [CREATE 5 in 1.1s]
3 of 4 OK loaded seed file dbt_dev_raw_seeds.raw_inventory_movements . [CREATE 20 in 1.3s]
4 of 4 OK loaded seed file dbt_dev_raw_seeds.raw_inventory_levels ..... [CREATE 15 in 1.2s]

Completed successfully
```

### Verify in Snowflake

```sql
USE DATABASE RAW_DATA_DB;
USE SCHEMA dbt_dev_raw_seeds;

SHOW TABLES;

SELECT 'raw_products' AS table_name, COUNT(*) AS rows FROM raw_products
UNION ALL
SELECT 'raw_warehouses', COUNT(*) FROM raw_warehouses
UNION ALL
SELECT 'raw_inventory_movements', COUNT(*) FROM raw_inventory_movements
UNION ALL
SELECT 'raw_inventory_levels', COUNT(*) FROM raw_inventory_levels;

-- Expected: 10, 5, 20, 15
```

---

##  Setup Complete Checklist

- [ ] Virtual environment created and activated
- [ ] dbt-snowflake installed
- [ ] Snowflake account created
- [ ] Database and schemas created in Snowflake
- [ ] profiles.yml configured
- [ ] dbt_project.yml configured
- [ ] packages.yml created
- [ ] dbt packages installed
- [ ] Folder structure created
- [ ] `dbt debug` passes
- [ ] Seed files placed in seeds/ folder
- [ ] `dbt seed` completes successfully
- [ ] Seeds verified in Snowflake

---

##  Common Setup Issues

### Issue: "Profile not found"
```powershell
# Check profile file location
type C:\Users\King\.dbt\profiles.yml

# Verify profile name matches dbt_project.yml
type dbt_project.yml | findstr profile
```

### Issue: "Invalid account identifier"
- Remove `https://` and `.snowflakecomputing.com`
- Format: `account_locator.region.cloud`
- Example: `abc12345.southeast-asia.azure`

### Issue: "Insufficient privileges"
- Use `ACCOUNTADMIN` role in profiles.yml
- Grant permissions in Snowflake (see Part 2)

### Issue: "Database does not exist"
- Run Snowflake setup SQL (Part 2)
- Verify database name in profiles.yml matches

---

##  Security Best Practices

### For Development
- Keep profiles.yml with passwords (local only)
- Add `.dbt/*.yml` to .gitignore
- Never commit credentials

### For Production
Use environment variables:

```yaml
warehouse_inventory:
  target: prod
  outputs:
    prod:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      # ... rest of config
```

---

##  Next Steps

After setup is complete:

1. ✅ Build staging models → See model SQL files
2. ✅ Build intermediate models → See model SQL files
3. ✅ Build marts models → See model SQL files
4. ✅ Add tests → See testing documentation
5. ✅ Generate docs → Run `dbt docs generate`

---

*For troubleshooting, see 04_TROUBLESHOOTING.md*  
*For architecture details, see 02_ARCHITECTURE.md*
