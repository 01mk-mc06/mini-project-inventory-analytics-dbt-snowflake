# Architecture & Data Flow

##  Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     RAW DATA (Seeds)                        │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │  Products    │  │  Warehouses  │  │  Movements   │    │
│  │  (10 rows)   │  │  (5 rows)    │  │  (20 rows)   │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
│                                                             │
│  ┌──────────────┐                                          │
│  │   Levels     │                                          │
│  │  (15 rows)   │                                          │
│  └──────────────┘                                          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    STAGING LAYER (Views)                    │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │stg_warehouse │  │stg_warehouse │  │stg_warehouse │    │
│  │__products    │  │__warehouses  │  │__movements   │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
│                                                             │
│  ┌──────────────┐                                          │
│  │stg_warehouse │                                          │
│  │__levels      │                                          │
│  └──────────────┘                                          │
│                                                             │
│  Purpose: Clean, standardize, type-cast, add audit fields  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                 INTERMEDIATE LAYER (Views)                  │
│                                                             │
│  ┌────────────────────────────────────────┐                │
│  │ int_warehouse__product_warehouse_grid  │                │
│  │ (CROSS JOIN - All combinations)        │                │
│  │ 10 products × 4 active warehouses = 40 │                │
│  └────────────────────────────────────────┘                │
│                                                             │
│  ┌────────────────────────────────────────┐                │
│  │ int_warehouse__movement_summary        │                │
│  │ (Aggregated movements by product/wh)   │                │
│  └────────────────────────────────────────┘                │
│                                                             │
│  ┌────────────────────────────────────────┐                │
│  │ int_warehouse__latest_levels           │                │
│  │ (Most recent snapshot only)            │                │
│  └────────────────────────────────────────┘                │
│                                                             │
│  Purpose: Reusable transformations, modular building blocks│
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    MARTS LAYER (Tables)                     │
│                                                             │
│  ┌────────────────────────────────────────┐                │
│  │ fct_inventory_summary                  │                │
│  │ (Main fact table - 40 rows)            │                │
│  │ Complete product-warehouse grid with   │                │
│  │ metrics, movements, derived flags      │                │
│  └────────────────────────────────────────┘                │
│                                                             │
│  ┌────────────────────────────────────────┐                │
│  │ fct_inventory_turnover                 │                │
│  │ (Turnover analysis)                    │                │
│  └────────────────────────────────────────┘                │
│                                                             │
│  ┌────────────────────────────────────────┐                │
│  │ fct_inventory_pivot                    │                │
│  │ (Pivoted view - warehouses as columns) │                │
│  └────────────────────────────────────────┘                │
│                                                             │
│  Purpose: Analytics-ready, optimized for BI tools          │
└─────────────────────────────────────────────────────────────┘
```

---

##  Medallion Architecture Explained

### Why This Structure?

```
Seeds (Bronze)
  ↓ Clean & Standardize
Staging (Silver)
  ↓ Transform & Aggregate
Intermediate (Silver+)
  ↓ Join & Enrich
Marts (Gold)
```

**Benefits:**
- ✅ Clear separation of concerns
- ✅ Reusable intermediate transformations
- ✅ Easier debugging (test each layer)
- ✅ Better performance (incremental builds)
- ✅ Self-documenting lineage

---

## 📊 Snowflake Schema Layout

```
RAW_DATA_DB (Database)
│
├── dbt_dev_raw_seeds (Schema)
│   ├── raw_products (10 rows)
│   ├── raw_warehouses (5 rows)
│   ├── raw_inventory_movements (20 rows)
│   └── raw_inventory_levels (15 rows)
│
├── dbt_dev_staging (Schema)
│   ├── stg_warehouse__products (view)
│   ├── stg_warehouse__warehouses (view)
│   ├── stg_warehouse__movements (view)
│   └── stg_warehouse__levels (view)
│
├── dbt_dev_intermediate (Schema)
│   ├── int_warehouse__product_warehouse_grid (view)
│   ├── int_warehouse__movement_summary (view)
│   └── int_warehouse__latest_levels (view)
│
└── dbt_dev_marts (Schema)
    ├── fct_inventory_summary (table - 40 rows)
    ├── fct_inventory_turnover (table - ~10 rows)
    └── fct_inventory_pivot (table - 10 rows)
```

---

##  Layer Responsibilities

### Seeds Layer (Bronze)
**Purpose:** Version-controlled raw data

**What it contains:**
- CSV files in git
- Small reference data (< 1000 rows)
- Lookup tables

**Transformations:** None (raw data)

**Materialization:** Tables (loaded via `dbt seed`)

---

### Staging Layer (Silver)
**Purpose:** Clean and standardize

**What it does:**
- Rename columns to consistent standards
- Type casting
- Add audit fields (dbt_loaded_at)
- Basic derived fields
- Light transformations only

**Transformations:**
```sql
-- Example: stg_warehouse__movements
case 
    when movement_type = 'INBOUND' then quantity
    else 0
end as quantity_inbound
```

**Materialization:** Views (always fresh)

---

### Intermediate Layer (Silver+)
**Purpose:** Reusable building blocks

**What it does:**
- Aggregations
- Complex transformations
- Grid generation (CROSS JOIN)
- Filter to latest records

**Transformations:**
```sql
-- Example: int_warehouse__movement_summary
select
    product_id,
    warehouse_location,
    sum(quantity_inbound) as total_inbound,
    sum(quantity_outbound) as total_outbound
from movements
group by 1, 2
```

**Materialization:** Views (or tables if expensive)

---

### Marts Layer (Gold)
**Purpose:** Analytics-ready fact tables

**What it does:**
- Join multiple sources
- Business logic
- Derived metrics
- Complete datasets for reporting

**Transformations:**
```sql
-- Example: fct_inventory_summary
select
    g.product_id,
    coalesce(l.quantity_on_hand, 0) as current_quantity,
    case 
        when quantity <= reorder_point 
        then true 
        else false 
    end as is_below_reorder_point
from grid g
left join levels l on ...
```

**Materialization:** Tables (cached for performance)

---

##  Key Architectural Decisions

### Decision 1: CROSS JOIN for Complete Grid

**Problem:** Need to show ALL product-warehouse combinations, even where no inventory exists

**Solution:**
```sql
-- int_warehouse__product_warehouse_grid
select *
from products p
cross join warehouses w
where w.is_active = true
```

**Result:** 10 products × 4 active warehouses = 40 rows

**Why:** Business needs to see gaps (zero inventory) not just what exists

---

### Decision 2: Separate Intermediate Layer

**Problem:** fct_inventory_summary was 200+ lines of complex SQL

**Solution:** Break into 3 intermediate models:
1. `int_warehouse__product_warehouse_grid` - Create combinations
2. `int_warehouse__movement_summary` - Aggregate movements
3. `int_warehouse__latest_levels` - Filter to latest snapshot

**Result:** Final mart is simple joins, each intermediate model testable

---

### Decision 3: Materialization Strategy

| Layer | Materialization | Why |
|-------|----------------|-----|
| Seeds | Table | Data loaded from CSV |
| Staging | View | Lightweight, always fresh |
| Intermediate | View | Reusable, no storage cost |
| Marts | Table | Heavy aggregations, fast queries |

---

##  Data Model Patterns

### Pattern 1: Star Schema

```
        Dimension: Products
                ↓
    Fact: fct_inventory_summary
                ↓
        Dimension: Warehouses
```

**Fact Table:** `fct_inventory_summary`
- Grain: One row per product-warehouse
- Measures: quantities, values, counts
- Flags: is_below_reorder_point

**Dimensions:** Embedded (denormalized for simplicity)
- Product attributes in fact table
- Warehouse attributes in fact table

---

### Pattern 2: Type 0 SCD (No History)

**Current Implementation:** Latest snapshot only

```sql
-- int_warehouse__latest_levels
where snapshot_date = (select max(snapshot_date) from levels)
```

**Future Enhancement:** Implement Type 2 SCD with snapshots

---

##  Lineage Graph

```
raw_products.csv
    ↓
stg_warehouse__products
    ↓
int_warehouse__product_warehouse_grid
    ↓
fct_inventory_summary
    ↓
fct_inventory_turnover

raw_movements.csv
    ↓
stg_warehouse__movements
    ↓
int_warehouse__movement_summary
    ↓
fct_inventory_summary

raw_levels.csv
    ↓
stg_warehouse__levels
    ↓
int_warehouse__latest_levels
    ↓
fct_inventory_summary
    ↓
fct_inventory_pivot
```

**Total Depth:** 4 layers  
**Total Width:** 14 models  
**Dependencies:** 24 ref() calls, 4 source() calls  

---

##  Architectural Principles Applied

### 1. Single Responsibility
Each model does ONE thing well:
- Staging: Clean
- Intermediate: Transform
- Marts: Join & serve

### 2. DRY (Don't Repeat Yourself)
- Reuse intermediate models
- Macros for common logic
- Packages for utilities

### 3. Separation of Concerns
- Data quality: Tests
- Business logic: Models
- Documentation: YAML

### 4. Modularity
- Small, focused models
- Clear dependencies
- Easy to test individually

---

##  Performance Considerations

### Current Performance
- Full refresh: < 2 minutes
- 14 objects, 110 total rows
- Suitable for real-time BI

### Future Optimizations
1. **Incremental Models**
   - For large fact tables
   - Only process new/changed records

2. **Clustering Keys**
   - On product_id for marts
   - On snapshot_date for levels

3. **Materialization Strategy**
   - Expensive intermediates → tables
   - Simple intermediates → views

---

##  Storage & Compute

### Storage Used
- Seeds: ~10 KB
- Views: 0 KB (virtual)
- Marts: ~5 KB
- **Total: ~15 KB**

### Compute Pattern
- Development: COMPUTE_WH (X-SMALL)
- Auto-suspend: 60 seconds
- Auto-resume: Enabled

---

*For implementation details, see 03_SETUP_GUIDE.md*  
*For business queries, see 08_BUSINESS_QUERIES.md*
