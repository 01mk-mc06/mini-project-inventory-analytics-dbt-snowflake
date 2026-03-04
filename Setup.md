# Setup Summary & Lessons Learned

##  What We Built

**Project:** Warehouse Inventory Analytics  
**Approach:** dbt Core + Snowflake + Seeds  
**Status:**  Seeds Successfully Loaded

---

##  Infrastructure Setup

### 1. Environment Setup

**Location:** `C:\Users\King\dbt_learning\`

```
C:\Users\King\
├── dbt_learning_old\              # Old messy projects (archived)
└── dbt_learning\                  # Fresh start
    ├── venv_warehouse\            # Python virtual environment
    └── project_01_warehouse_inventory\  # dbt project
        ├── seeds\
        │   ├── raw_products.csv (10 rows)
        │   ├── raw_warehouses.csv (5 rows)
        │   ├── raw_inventory_movements.csv (20 rows)
        │   └── raw_inventory_levels.csv (15 rows)
        ├── models\
        ├── tests\
        └── dbt_project.yml
```

### 2. Virtual Environment

```powershell
# Created fresh virtual environment
cd C:\Users\King\dbt_learning
python -m venv venv_warehouse

# Activated
venv_warehouse\Scripts\activate

# Installed dbt-snowflake
pip install dbt-snowflake

# Version: dbt=1.11.6, snowflake adapter=1.11.3
```

### 3. dbt Project Initialization

```powershell
# Initialized project
dbt init project_01_warehouse_inventory

# Configuration:
# - Profile name: warehouse_inventory (initially dbt_learning)
# - Database: Snowflake
# - Authentication: Password
```

---

##  Snowflake Configuration

### Database Structure

**Database:** `RAW_DATA_DB`

**Schemas Created:**
- `dbt_dev` - Default development schema
- `raw_seeds` - Target for seed data
- `staging` - For staging models
- `marts` - For mart models

**Actual Schema Used by Seeds:** `dbt_dev_raw_seeds` (custom schema)

### Key Settings

```yaml
# profiles.yml location: C:\Users\King\.dbt\profiles.yml
warehouse_inventory:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT_IDENTIFIER
      user: YOUR_USERNAME
      password: YOUR_PASSWORD
      role: ACCOUNTADMIN           # CRITICAL: Must be ACCOUNTADMIN
      database: RAW_DATA_DB
      warehouse: COMPUTE_WH
      schema: dbt_dev
      threads: 4
```

---

##  dbt_project.yml Configuration

### Project Settings

```yaml
name: 'project_01_warehouse_inventory'
version: '1.0.0'
config-version: 2
profile: 'warehouse_inventory'
```

### Seeds Configuration

```yaml
seeds:
  project_01_warehouse_inventory:  # Must match project name!
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
```

---

##  Troubleshooting & Issues Encountered

### Issue 1: "Insufficient privileges to operate on schema"

**Error:**
```
003001 (42501): SQL access control error:
Insufficient privileges to operate on schema 'DBT_DEV'.
```

**Root Cause:** User was assigned `useradmin` role instead of `ACCOUNTADMIN`

**Solution:**
```yaml
# profiles.yml - Changed role
role: ACCOUNTADMIN  # Was: useradmin
```

**Lesson:** Always verify role has CREATE TABLE privileges in target schemas

---

### Issue 2: Database/Schema Doesn't Exist

**Error:**
```
Object does not exist, or operation cannot be performed.
```

**Root Cause:** Referenced database name in profiles.yml didn't match actual Snowflake database

**Solution:**
1. Verified actual database name in Snowflake: `SHOW DATABASES;`
2. Updated profiles.yml to use `RAW_DATA_DB` instead of `ANALYTICS`

**Lesson:** Always verify database/schema names in Snowflake before configuring dbt

---

### Issue 3: Project Name Mismatch in dbt_project.yml

**Error:** Seeds loaded into wrong schema (dbt_dev instead of raw_seeds)

**Root Cause:** Seeds configuration used `warehouse_inventory` but project name was `project_01_warehouse_inventory`

**Wrong:**
```yaml
name: 'project_01_warehouse_inventory'
...
seeds:
  warehouse_inventory:  # Mismatch!
    +schema: raw_seeds
```

**Correct:**
```yaml
name: 'project_01_warehouse_inventory'
...
seeds:
  project_01_warehouse_inventory:  # Must match!
    +schema: raw_seeds
```

**Lesson:** Project name in seeds config MUST exactly match `name:` at top of dbt_project.yml

---

### Issue 4: Virtual Environment Organization

**Challenge:** Multiple projects getting jumbled in one folder

**Solution:** 
1. Archived old work: `Rename-Item dbt_learning dbt_learning_old`
2. Started fresh with clean structure
3. One virtual environment, separate project folders

**Best Practice Structure:**
```
dbt_learning\
├── venv_warehouse\        # Shared virtual env
├── project_01\            # Project 1
├── project_02\            # Future projects
└── project_03\
```

---

### Issue 5: Schema Naming (Custom Schemas)

**Expected:** Seeds in `raw_seeds` schema  
**Actual:** Seeds in `dbt_dev_raw_seeds` schema

**Why:** dbt combines target schema (`dbt_dev`) + custom schema (`raw_seeds`) = `dbt_dev_raw_seeds`

**Solution:** This is expected dbt behavior for custom schemas in dev environment

**Production Behavior:** In prod target, would be just `raw_seeds` (no prefix)

**Lesson:** Understand dbt's custom schema naming convention

---

##  Final Validation

### Seeds Loaded Successfully

```powershell
dbt seed --full-refresh
```

**Results:**
```
1 of 4 OK loaded seed file dbt_dev_raw_seeds.raw_inventory_levels .... [CREATE 15 in 2.61s]
2 of 4 OK loaded seed file dbt_dev_raw_seeds.raw_inventory_movements . [CREATE 20 in 2.23s]
3 of 4 OK loaded seed file dbt_dev_raw_seeds.raw_products ............ [CREATE 10 in 1.29s]
4 of 4 OK loaded seed file dbt_dev_raw_seeds.raw_warehouses .......... [CREATE 5 in 1.31s]

Completed successfully
PASS=4 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=4
```

### Snowflake Verification

```sql
USE DATABASE RAW_DATA_DB;
USE SCHEMA dbt_dev_raw_seeds;

SHOW TABLES;
-- Results: 4 tables created

SELECT 
    'raw_products' as table_name, COUNT(*) as rows FROM raw_products
UNION ALL
SELECT 'raw_warehouses', COUNT(*) FROM raw_warehouses
UNION ALL
SELECT 'raw_inventory_movements', COUNT(*) FROM raw_inventory_movements
UNION ALL
SELECT 'raw_inventory_levels', COUNT(*) FROM raw_inventory_levels;

-- Results: 10, 5, 20, 15 rows respectively 
```

---

##  Key Learnings

### 1. **Role Permissions Matter**
- `ACCOUNTADMIN` > `useradmin` for full privileges
- Always verify role can CREATE TABLE in target schemas
- Grant ownership on schemas when needed

### 2. **Configuration Consistency**
- Project name must match across dbt_project.yml
- Database/schema names must match Snowflake exactly
- Profile names connect dbt_project.yml to profiles.yml

### 3. **dbt Custom Schemas**
- Dev: `{target_schema}_{custom_schema}`
- Prod: `{custom_schema}` only
- This is expected behavior, not a bug

### 4. **Seeds Best Practices**
- Use for small reference data (< 1000 rows)
- Define column types explicitly to avoid inference issues
- Perfect for demo/learning projects
- Version controlled in git

### 5. **Troubleshooting Workflow**
1. Check `dbt debug` first
2. Verify Snowflake permissions manually
3. Check actual vs expected schema names
4. Review dbt_project.yml for name mismatches
5. Use `--full-refresh` to reset seeds

---

##  Workarounds Applied

### Workaround 1: Clean Slate Approach
**Problem:** Multiple failed attempts created confusing folder structure  
**Solution:** Archive old folder, start fresh
```powershell
Rename-Item dbt_learning dbt_learning_old
mkdir dbt_learning
```

### Workaround 2: Manual Schema Grants
**Problem:** Even ACCOUNTADMIN lacked permissions  
**Solution:** Explicit ownership and privilege grants
```sql
GRANT OWNERSHIP ON DATABASE RAW_DATA_DB TO ROLE ACCOUNTADMIN;
GRANT ALL PRIVILEGES ON SCHEMA dbt_dev TO ROLE ACCOUNTADMIN;
```

### Workaround 3: Accepted Custom Schema Naming
**Problem:** Expected `raw_seeds`, got `dbt_dev_raw_seeds`  
**Solution:** Understood this is dbt's design, updated mental model

---

##  Current State

### What's Working
 Virtual environment activated  
 dbt installed and configured  
 Snowflake connection successful (`dbt debug` passes)  
 4 seed files loaded into Snowflake  
 Data verified in `dbt_dev_raw_seeds` schema  

### What's Next
- [ ] Create source definitions YAML
- [ ] Build staging models (4 models)
- [ ] Build mart model (1 model)
- [ ] Add tests and documentation
- [ ] Generate dbt docs

---

##  Ready for Next Phase

**Current Location:** `C:\Users\King\dbt_learning\project_01_warehouse_inventory`

**Commands to Proceed:**
```powershell
# Create staging structure
mkdir models\staging
mkdir models\staging\warehouse

# Create marts structure
mkdir models\marts

# Ready to build models!
```

---

##  Commands Reference

### Essential Commands Used

```powershell
# Virtual environment
python -m venv venv_warehouse
venv_warehouse\Scripts\activate
pip install dbt-snowflake

# dbt operations
dbt init project_01_warehouse_inventory
dbt debug                    # Test connection
dbt seed                     # Load seeds
dbt seed --full-refresh      # Reload seeds from scratch
dbt ls --resource-type seed  # List seeds

# File operations
mkdir folder_name
dir
cd folder_name
```

### Snowflake Commands Used

```sql
-- Database/schema management
SHOW DATABASES;
SHOW SCHEMAS;
SHOW TABLES;
USE DATABASE database_name;
USE SCHEMA schema_name;

-- Permissions
GRANT OWNERSHIP ON DATABASE db TO ROLE role_name;
GRANT ALL PRIVILEGES ON SCHEMA schema TO ROLE role_name;
SHOW GRANTS ON DATABASE db_name;

-- Verification
SELECT CURRENT_ROLE();
SELECT CURRENT_DATABASE();
SELECT COUNT(*) FROM table_name;
```

---

##  Tips for Future Projects

1. **Start with correct role** - Use ACCOUNTADMIN from the beginning
2. **Verify Snowflake first** - Check databases/schemas exist before configuring dbt
3. **Match names exactly** - Project names, schema names must be consistent
4. **Use version control** - Keep old work in archive folders
5. **Document as you go** - This summary would have been harder to write later
6. **Test incrementally** - `dbt debug` after each config change
7. **Understand custom schemas** - Know how dbt names schemas in dev vs prod

---

##  Files Created

### Configuration Files
- `C:\Users\King\.dbt\profiles.yml` - Snowflake connection config
- `dbt_project.yml` - Project and seeds configuration

### Seed Files
- `seeds\raw_products.csv`
- `seeds\raw_warehouses.csv`
- `seeds\raw_inventory_movements.csv`
- `seeds\raw_inventory_levels.csv`

### Snowflake Objects Created
- Database: `RAW_DATA_DB` (pre-existing)
- Schema: `dbt_dev_raw_seeds` (created by dbt)
- Tables: 4 seed tables with 50 total rows

---

**Setup Complete** 
