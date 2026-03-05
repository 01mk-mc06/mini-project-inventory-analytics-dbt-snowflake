# Warehouse Inventory Analytics - Overview

##  Executive Summary

**Project Name:** Warehouse Inventory Analytics  
**Domain:** Operations & Supply Chain  
**Status:**  Complete  
**Duration:** ~6-8 hours (including troubleshooting)  
**Complexity Level:** Foundation → Intermediate  

---

##  Project Objectives

### Business Goals
1. Track inventory levels across 5 warehouse locations in the Philippines
2. Monitor product movement patterns (inbound, outbound, adjustments)
3. Identify products below reorder points
4. Calculate inventory value and turnover metrics
5. Enable cross-warehouse inventory comparison

### Technical Goals
1. Learn dbt Core fundamentals
2. Implement medallion architecture (staging → intermediate → marts)
3. Master seeds, sources, and models
4. Apply testing strategies (built-in, custom singular, custom generic)
5. Use Jinja for dynamic SQL generation
6. Leverage dbt packages (dbt_utils, dbt_expectations)
7. Build reusable macros
8. Generate lineage documentation

---

##  Data Model Summary

### Grain & Cardinality

| Layer | Model | Grain | Row Count |
|-------|-------|-------|-----------|
| **Seeds** | raw_products | One row per product | 10 |
| | raw_warehouses | One row per warehouse | 5 |
| | raw_inventory_movements | One row per transaction | 20 |
| | raw_inventory_levels | One row per product-warehouse-date | 15 |
| **Staging** | stg_warehouse__* | Same as seeds | 10/5/20/15 |
| **Intermediate** | int_warehouse__product_warehouse_grid | One row per product-warehouse | 40 |
| | int_warehouse__movement_summary | One row per product-warehouse with movements | ~10 |
| | int_warehouse__latest_levels | One row per product-warehouse (latest date) | ~10 |
| **Marts** | fct_inventory_summary | One row per product-warehouse (active wh only) | 40 |
| | fct_inventory_turnover | One row per product-warehouse (qty > 0) | ~10 |
| | fct_inventory_pivot | One row per product | 10 |

---

##  Success Metrics

### Project Metrics
- **Models Created:** 14 (4 staging, 3 intermediate, 3 marts, 4 seeds)
- **Tests Implemented:** 35+
- **Documentation Coverage:** 100%
- **Lines of SQL:** ~800
- **Build Time:** < 2 minutes (full refresh)
- **Test Pass Rate:** 100%

### Learning Outcomes Achieved
- ✅ Understand dbt project structure
- ✅ Build multi-layer data pipelines
- ✅ Implement comprehensive testing
- ✅ Use Jinja for dynamic SQL
- ✅ Create reusable macros
- ✅ Generate documentation
- ✅ Apply best practices

---

##  Project Structure

```
project_01_warehouse_inventory/
├── dbt_project.yml
├── packages.yml
├── models/
│   ├── staging/warehouse/          (4 models)
│   ├── intermediate/warehouse/     (3 models)
│   └── marts/                      (3 models)
├── tests/
│   ├── generic/                    (1 custom test)
│   └── *.sql                       (4 singular tests)
├── macros/                         (4 macros)
└── seeds/                          (4 CSV files)
```

---

## 🎓 Skills Mastered

### dbt Core Fundamentals
✅ Project initialization and configuration  
✅ Profile management (dev/prod)  
✅ Seeds for version-controlled data  
✅ Sources with freshness checks  
✅ Models (staging, intermediate, marts)  
✅ ref() and source() functions  
✅ Materialization strategies  
✅ Custom schema configuration  

### Advanced dbt Features
✅ Generic tests (custom and packaged)  
✅ Singular tests for business logic  
✅ Jinja templating (variables, loops, conditionals)  
✅ Macros for code reuse  
✅ dbt packages (dbt_utils, dbt_expectations)  
✅ Documentation generation  
✅ Lineage tracking  
✅ Selective runs and testing  

### SQL & Data Modeling
✅ CTE patterns  
✅ Window functions  
✅ CROSS JOIN for complete grids  
✅ LEFT JOIN for optional relationships  
✅ COALESCE for NULL handling  
✅ CASE statements for derived fields  
✅ Aggregations (SUM, COUNT, MAX)  

---

## 📚 Documentation Files

This project includes comprehensive documentation broken into focused files:

1. **01_OVERVIEW.md** (this file) - Executive summary
2. **02_ARCHITECTURE.md** - Data flow and model design
3. **03_SETUP_GUIDE.md** - Environment and configuration
4. **04_TROUBLESHOOTING.md** - Issues encountered and solutions
5. **05_CONCEPTS_AND_LEARNINGS.md** - Key concepts explained
6. **06_BEST_PRACTICES.md** - Standards and patterns
7. **07_WORKFLOWS.md** - Common development workflows
8. **08_BUSINESS_QUERIES.md** - Sample analytics queries
9. **SNOWFLAKE_VALIDATION_QUERIES.md** - Validation queries reference

---

## 🚀 Quick Start

```powershell
# 1. Activate environment
venv_warehouse\Scripts\activate

# 2. Navigate to project
cd project_01_warehouse_inventory

# 3. Install packages
dbt deps

# 4. Load seeds
dbt seed

# 5. Run all models
dbt run

# 6. Run tests
dbt test

# 7. Generate docs
dbt docs generate
dbt docs serve
```

---

##  Project Status

**Setup:**  Complete  
**Seeds:**  Loaded (50 rows)  
**Models:**  Built (14 objects)  
**Tests:**  Passing (35+ tests)  
**Documentation:**  Generated  

**Ready for:** Portfolio, interviews, next project

---

*For detailed information, refer to the specific documentation files listed above.*
