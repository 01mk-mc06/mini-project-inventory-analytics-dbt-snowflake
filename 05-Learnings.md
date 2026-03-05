# Key Concepts & Learnings

##  Core dbt Concepts

### 1. Medallion Architecture

**Concept:** Layered data transformation approach

```
Bronze (Raw) → Silver (Cleaned) → Gold (Analytics-Ready)
```

**In our project:**
- **Bronze:** Seeds (CSV files)
- **Silver:** Staging + Intermediate layers
- **Gold:** Marts layer

**Why it matters:**
- Clear separation of concerns
- Easier debugging
- Reusable transformations
- Self-documenting lineage

---

### 2. CTE Pattern

**Standard structure in all models:**

```sql
with source as (
    select * from {{ source('schema', 'table') }}
),

renamed as (
    select
        -- Keys first
        id,
        -- Attributes
        name,
        -- Metrics
        amount,
        -- Audit
        current_timestamp() as dbt_loaded_at
    from source
),

final as (
    select * from renamed
)

select * from final
```

**Benefits:**
- Easy to debug (query each CTE)
- Consistent structure
- Clear data lineage
- Readable and maintainable

---

### 3. ref() vs source()

**source()** - Raw data from external systems:
```sql
select * from {{ source('warehouse_seeds', 'raw_products') }}
```
- Points to YAML-defined sources
- Triggers source freshness checks
- Tracks external dependencies

**ref()** - dbt-managed models:
```sql
select * from {{ ref('stg_warehouse__products') }}
```
- Points to other dbt models
- Builds dependency graph (DAG)
- Enables lineage tracking

**Never hardcode table names!**

---

### 4. Testing Strategies

#### Schema Tests (YAML):
```yaml
columns:
  - name: product_id
    tests:
      - unique
      - not_null
      - relationships:
          arguments:
            to: ref('stg_warehouse__products')
            field: product_id
```

#### Singular Tests (SQL):
```sql
-- tests/assert_no_negative_inventory.sql
select *
from {{ ref('fct_inventory_summary') }}
where current_quantity < 0
```

#### Custom Generic Tests:
```sql
-- tests/generic/test_is_positive.sql
{% test is_positive(model, column_name) %}
  select {{ column_name }}
  from {{ model }}
  where {{ column_name }} < 0
{% endtest %}
```

---

### 5. Jinja for Dynamic SQL

**Variables:**
```sql
{% set warehouses = ['WH-001', 'WH-002'] %}
```

**Loops:**
```sql
{% for warehouse in warehouses %}
  sum(case when wh = '{{ warehouse }}' then qty else 0 end) as {{ warehouse }}_qty
  {%- if not loop.last -%},{%- endif %}
{% endfor %}
```

**Conditionals:**
```sql
{%- if target.name == 'prod' -%}
  {{ custom_schema }}
{%- else -%}
  {{ target.schema }}_{{ custom_schema }}
{%- endif -%}
```

---

### 6. Macros for Code Reuse

**Creating:**
```sql
-- macros/safe_divide.sql
{% macro safe_divide(num, denom) %}
  case when {{ denom }} = 0 then null
  else {{ num }}::float / {{ denom }} end
{% endmacro %}
```

**Using:**
```sql
select {{ safe_divide('outbound', 'inventory') }} as turnover
from table
```

---

### 7. Seeds vs External Sources

**Use Seeds For:**
- Small reference data (< 1000 rows)
- Lookup tables
- Demo/test data
- Version-controlled data

**Use External Sources For:**
- Large datasets (> 1000 rows)
- Production data
- Frequently changing data
- External systems

---

### 8. Materialization Strategies

**View:**
```yaml
+materialized: view
```
- Fast to build
- Always fresh
- Query on demand

**Table:**
```yaml
+materialized: table
```
- Slower to build
- Snapshot at build time
- Fast to query

**Incremental:**
```yaml
+materialized: incremental
```
- Only processes new records
- Best for large tables

---

##  SQL Patterns Learned

### Pattern 1: CROSS JOIN for Complete Grids

```sql
-- Create ALL product-warehouse combinations
select *
from products p
cross join warehouses w
where w.is_active = true

-- Result: 10 × 4 = 40 rows
```

**Why:** Show gaps (zero inventory), not just what exists

---

### Pattern 2: COALESCE for NULL Handling

```sql
select
    coalesce(l.quantity_on_hand, 0) as current_quantity
from grid g
left join levels l on ...
```

**Why:** Convert NULLs to meaningful defaults

---

### Pattern 3: Derived Flags

```sql
case 
    when quantity <= reorder_point then true
    else false
end as is_below_reorder_point
```

**Why:** Simplify business logic in downstream queries

---

### Pattern 4: Audit Columns

```sql
current_timestamp() as dbt_loaded_at,
current_timestamp() as dbt_updated_at
```

**Why:** Track when data was transformed

---

##  Best Practices Applied

### 1. Naming Conventions

**Models:**
- Staging: `stg_<source>__<entity>`
- Intermediate: `int_<source>__<description>`
- Marts: `fct_<description>` or `dim_<description>`

**Files:**
- SQL: `model_name.sql`
- YAML: `_<source>__models.yml`
- Tests: `assert_<description>.sql`

---

### 2. Column Ordering

Consistent order in all models:
1. Primary keys
2. Foreign keys
3. Descriptive attributes
4. Numeric measures
5. Booleans/flags
6. Dates/timestamps
7. Audit fields (last)

---

### 3. Documentation Standards

```yaml
models:
  - name: fct_inventory_summary
    description: >
      Business context here...
      
      Grain: One row per product-warehouse
```

---

### 4. Testing Coverage

**Minimum tests:**
- Primary keys: `unique` + `not_null`
- Foreign keys: `relationships`
- Enums: `accepted_values`
- Ranges: `expression_is_true`
- Business logic: Custom singular tests

---

##  Key Takeaways

### 1. Modular Design
Break complex logic into smaller, testable pieces

### 2. Consistent Patterns
Use same structure across all models (CTE pattern)

### 3. Test Everything
Catch data quality issues early

### 4. Document as You Go
Don't wait until the end

### 5. Version Control
Track all code and configuration changes

---

*See 06_BEST_PRACTICES.md for detailed standards*
