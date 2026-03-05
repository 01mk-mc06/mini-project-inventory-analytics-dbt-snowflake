# Troubleshooting Guide

##  Issues Encountered & Solutions

This document captures all major issues encountered during the project and their solutions.

---

## Issue 1: Insufficient Privileges Error

### Error Message
```
003001 (42501): SQL access control error:
Insufficient privileges to operate on schema 'DBT_DEV'.
```

### Root Cause
User profile had `role: useradmin` instead of `ACCOUNTADMIN`

### Impact
- Could not create tables in target schemas
- dbt seed failed
- dbt run failed

### Solution
**Updated profiles.yml:**
```yaml
warehouse_inventory:
  outputs:
    dev:
      role: ACCOUNTADMIN  # Changed from useradmin
```

### Verification
```powershell
dbt debug
# Should show: Connection test: [OK connection ok]
```

### Lesson Learned
- Always verify role has CREATE TABLE privileges
- `ACCOUNTADMIN` required for full schema operations
- Test with `dbt debug` after profile changes
- Check permissions in Snowflake before running dbt

---

## Issue 2: Database Name Mismatch

### Error Message
```
Object does not exist, or operation cannot be performed.
```

### Root Cause
`profiles.yml` referenced `ANALYTICS` database but actual database was `RAW_DATA_DB`

### Impact
- dbt could not connect to database
- All commands failed

### Solution

**Step 1: Verified actual database in Snowflake**
```sql
SHOW DATABASES;
-- Found: RAW_DATA_DB (not ANALYTICS)
```

**Step 2: Updated profiles.yml**
```yaml
database: RAW_DATA_DB  # Changed from ANALYTICS
```

### Verification
```powershell
dbt debug
# Connection test should pass
```

### Lesson Learned
- Never assume database names
- Always verify in Snowflake first using `SHOW DATABASES`
- Document database names in setup guide
- Use consistent naming across environments

---

## Issue 3: Project Name Mismatch in Seeds Config

### Error Message
Seeds loading into wrong schema (`dbt_dev` instead of `raw_seeds`)

### Root Cause
**Mismatch in dbt_project.yml:**
```yaml
name: 'project_01_warehouse_inventory'
...
seeds:
  warehouse_inventory:  # MISMATCH!
    +schema: raw_seeds
```

### Impact
- Seeds loaded into default schema
- Could not find seeds in expected location
- Tests failed

### Solution
**Fix dbt_project.yml:**
```yaml
name: 'project_01_warehouse_inventory'
...
seeds:
  project_01_warehouse_inventory:  # Must match project name!
    +schema: raw_seeds
```

### Verification
```powershell
dbt seed --full-refresh

# Check schema in output:
# Should show: dbt_dev_raw_seeds.raw_products
```

### Lesson Learned
- Project name in config MUST exactly match `name:` at top of file
- dbt uses project name for namespacing
- Case-sensitive matching required
- Always use `dbt ls` to verify model discovery

---

## Issue 4: Custom Schema Naming Confusion

### Expected Behavior
Seeds in `raw_seeds` schema

### Actual Behavior  
Seeds in `dbt_dev_raw_seeds` schema

### Root Cause
dbt's custom schema naming convention:
- **Dev:** `{target_schema}_{custom_schema}` = `dbt_dev_raw_seeds`
- **Prod:** `{custom_schema}` only = `raw_seeds`

### Impact
- Initial confusion about schema names
- Queries failed using wrong schema name

### Solution
**This is expected behavior!** Update mental model and documentation.

**Workaround (if needed):**
Create `macros/generate_schema_name.sql`:
```sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- if target.name == 'prod' -%}
        {{ custom_schema_name | trim }}
    {%- else -%}
        {{ target.schema }}_{{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

### Verification
```sql
-- In Snowflake
USE DATABASE RAW_DATA_DB;
SHOW SCHEMAS LIKE 'dbt_dev%';

-- Should see:
-- dbt_dev_raw_seeds
-- dbt_dev_staging
-- dbt_dev_intermediate
-- dbt_dev_marts
```

### Lesson Learned
- Understand dbt's schema naming conventions
- Dev and prod have different schema prefixing
- Document expected schema names for team
- This is a feature, not a bug!

---

## Issue 5: Missing .sql Extension

### Error Message
```
The selection criterion 'stg_warehouse__levels' does not match any enabled nodes
```

### Root Cause
File named `stg_warehouse__levels` instead of `stg_warehouse__levels.sql`

### Impact
- dbt could not discover the model
- `dbt run` skipped the model
- Downstream models failed

### Solution
Renamed file to include `.sql` extension

### Verification
```powershell
# List all models
dbt ls --resource-type model

# Should now show:
# stg_warehouse__levels
```

### Lesson Learned
- dbt only discovers `.sql` files in model paths
- Always use proper file extensions
- Use `dbt ls` to verify model discovery
- Check file extensions in error messages

---

## Issue 6: ref() Function Not Compiling

### Error Message
```
Object 'RAW_DATA_DB.DBT_DEV_INTERMEDIATE.REFSTG_WAREHOUSE__MOVEMENTS' does not exist
```

### Root Cause
Missing folder structure in `dbt_project.yml` config

**Original (broken):**
```yaml
intermediate:
  +materialized: view
  +schema: intermediate
```

### Impact
- ref() function not compiling correctly
- Model tried to query literal "REF..." table
- All intermediate models failed

### Solution
**Add warehouse subfolder config:**
```yaml
intermediate:
  +materialized: view
  +schema: intermediate
  warehouse:              # Added this
    +materialized: view
    +schema: intermediate
```

### Verification
```powershell
# Compile to see SQL
dbt compile --select int_warehouse__movement_summary

# Check compiled SQL - should have proper table names
type target\compiled\...\int_warehouse__movement_summary.sql
```

### Lesson Learned
- Folder structure in `models/` must match `dbt_project.yml`
- Nested folders need explicit configuration
- Test with `dbt compile` before `dbt run`
- Check compiled SQL when refs don't work

---

## Issue 7: Column Name with Hyphens

### Error Message
```
SQL compilation error:
syntax error line 16 at position 87 unexpected '-'
```

### Root Cause
Generated column names like `WH-001_qty` contain hyphens (invalid in SQL)

**Problematic macro:**
```sql
{{ warehouse }}_qty  -- Results in WH-001_qty
```

### Impact
- fct_inventory_pivot model failed to compile
- All downstream uses failed

### Solution
**Use Jinja filter to replace hyphens:**
```sql
{{ warehouse | replace('-', '_') }}_qty
-- Results in: WH_001_qty
```

### Verification
```powershell
dbt run --select fct_inventory_pivot
# Should succeed now
```

```sql
-- In Snowflake
DESCRIBE TABLE fct_inventory_pivot;
-- Should show: WH_001_qty, WH_002_qty, etc.
```

### Lesson Learned
- SQL column names: alphanumeric + underscore only
- Use Jinja filters for string manipulation
- Quote column names if special chars required (not recommended)
- Test generated SQL with sample data

---

## Issue 8: Test Configuration Deprecation Warning

### Warning Message
```
[WARNING]: Arguments to generic tests should be nested under the `arguments` property
```

### Root Cause
**Old syntax (deprecated):**
```yaml
tests:
  - relationships:
      to: source('warehouse_seeds', 'raw_products')
      field: product_id
```

### Impact
- Tests work but use deprecated syntax
- Future dbt versions may break
- Warning clutter in output

### Solution
**Update to new syntax:**
```yaml
tests:
  - relationships:
      arguments:           # Added this wrapper
        to: source('warehouse_seeds', 'raw_products')
        field: product_id
```

### Verification
```powershell
dbt test
# Should run without warnings
```

### Lesson Learned
- dbt syntax evolves; update to latest conventions
- Deprecation warnings work but should be fixed
- Check dbt docs for current best practices
- Update all YAML files at once for consistency

---

##  General Troubleshooting Workflow

### Step 1: Read the Error Message
```powershell
# Note the exact error
# Note the file/model mentioned
# Note the line number if provided
```

### Step 2: Check Configuration
```powershell
# Verify connection
dbt debug

# Verify model discovery
dbt ls --resource-type model

# Check compiled SQL
dbt compile --select model_name
```

### Step 3: Isolate the Issue
```powershell
# Test upstream models
dbt run --select +failing_model --exclude failing_model

# Test the failing model alone
dbt run --select failing_model
```

### Step 4: Check Snowflake
```sql
-- Verify objects exist
SHOW SCHEMAS;
SHOW TABLES;
SHOW VIEWS;

-- Test SQL manually
SELECT * FROM RAW_DATA_DB.dbt_dev_staging.stg_warehouse__products LIMIT 1;
```

### Step 5: Review Recent Changes
```powershell
# What changed?
git diff

# Revert if needed
git checkout -- file_name
```

---

##  Quick Diagnostic Commands

### Check dbt Installation
```powershell
dbt --version
pip list | findstr dbt
```

### Check Configuration Files
```powershell
# Check profiles.yml
type C:\Users\King\.dbt\profiles.yml

# Check dbt_project.yml
type dbt_project.yml

# Check packages installed
dir dbt_packages
```

### Check Snowflake Connection
```powershell
# Test connection
dbt debug

# View connection details
dbt debug --config-dir
```

### Check Model Compilation
```powershell
# Compile specific model
dbt compile --select model_name

# View compiled SQL
type target\compiled\project_01_warehouse_inventory\models\...
```

---

##  Common Error Patterns

### Pattern 1: "Object does not exist"
**Likely causes:**
- Schema name mismatch
- Database name incorrect
- Object not created yet
- Permissions issue

**Debug:**
```sql
SHOW SCHEMAS IN DATABASE RAW_DATA_DB;
SHOW TABLES IN SCHEMA dbt_dev_staging;
```

### Pattern 2: "Insufficient privileges"
**Likely causes:**
- Wrong role in profiles.yml
- Missing grants in Snowflake
- Schema ownership issue

**Debug:**
```sql
SELECT CURRENT_ROLE();
SHOW GRANTS TO ROLE ACCOUNTADMIN;
```

### Pattern 3: "Compilation error"
**Likely causes:**
- SQL syntax error
- Missing ref() or source()
- Jinja templating issue

**Debug:**
```powershell
dbt compile --select model_name
type target\compiled\...\model_name.sql
```

### Pattern 4: "Test failed"
**Likely causes:**
- Data quality issue
- Test logic incorrect
- Schema mismatch

**Debug:**
```powershell
dbt test --select model_name --store-failures
# Then query failed rows in Snowflake
```

---

##  Getting Help

### Resources
1. **dbt Slack:** https://www.getdbt.com/community
2. **dbt Discourse:** https://discourse.getdbt.com
3. **dbt Docs:** https://docs.getdbt.com
4. **Snowflake Docs:** https://docs.snowflake.com

### When Asking for Help
Include:
- Exact error message
- dbt version (`dbt --version`)
- Relevant configuration (profiles.yml, dbt_project.yml)
- Steps to reproduce
- What you've already tried

---

##  Prevention Checklist

Before running dbt commands:
- [ ] Virtual environment activated
- [ ] `dbt debug` passes
- [ ] Snowflake connection verified
- [ ] Configuration files updated
- [ ] Model files have .sql extension
- [ ] YAML syntax validated
- [ ] Compiled SQL checked for errors

---

*For setup instructions, see 03_SETUP_GUIDE.md*  
*For common workflows, see 07_WORKFLOWS.md*
