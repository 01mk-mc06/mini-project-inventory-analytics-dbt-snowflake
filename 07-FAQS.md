# Frequently Asked Questions (FAQ)

##  About This Document

This FAQ explains the "why" behind key decisions made during the Warehouse Inventory Analytics project. Perfect for interviews, code reviews, or understanding architectural choices.

---

##  Architecture Decisions

### Q1: Why did you use medallion architecture (Bronze → Silver → Gold)?

**A:** Three main reasons:

1. **Separation of concerns** - Each layer has ONE job:
   - Bronze (Seeds): Store raw data as-is
   - Silver (Staging/Intermediate): Clean and transform
   - Gold (Marts): Serve analytics

2. **Easier debugging** - When something breaks, I know exactly which layer to check. If `fct_inventory_summary` has wrong numbers, I can trace back through intermediate → staging → seeds.

3. **Reusability** - Intermediate models like `int_warehouse__movement_summary` can be used by multiple marts. Write once, use everywhere.

**Alternative considered:** Flat structure (all models in one folder)  
**Why rejected:** Would work for 14 models, but doesn't scale. With 100+ models, you need clear organization.

---

### Q2: Why did you create an intermediate layer? Why not just staging → marts?

**A:** The intermediate layer solved a specific problem: `fct_inventory_summary` was becoming a 200+ line monster.

**Before intermediate layer:**
```sql
-- fct_inventory_summary.sql (200+ lines)
with products as (...),
     warehouses as (...),
     movements as (
         -- 50 lines of aggregation logic here
     ),
     levels as (
         -- 30 lines of filtering logic here
     ),
     grid as (
         -- 40 lines of cross join logic here
     )
-- ... 80 more lines of joins and calculations
```

**After intermediate layer:**
```sql
-- Now it's just simple joins!
with grid as (
    select * from {{ ref('int_warehouse__product_warehouse_grid') }}
),
movements as (
    select * from {{ ref('int_warehouse__movement_summary') }}
),
levels as (
    select * from {{ ref('int_warehouse__latest_levels') }}
)
-- Much cleaner!
```

**Benefits:**
- Each intermediate model is testable independently
- Reusable by other marts (e.g., `fct_inventory_turnover` also uses movement_summary)
- Easier to understand (read 3 files of 30 lines vs 1 file of 200 lines)

---

### Q3: Why CROSS JOIN instead of LEFT JOIN for the product-warehouse grid?

**A:** This was a critical business insight!

**LEFT JOIN approach (what we DON'T do):**
```sql
select p.product_id, l.warehouse_location, l.quantity
from products p
left join levels l on p.product_id = l.product_id
```
**Problem:** Only shows warehouses WHERE inventory exists.

**CROSS JOIN approach (what we DO):**
```sql
select p.product_id, w.warehouse_code, coalesce(l.quantity, 0)
from products p
cross join warehouses w
left join levels l on p.product_id = l.product_id and w.warehouse_code = l.warehouse_location
```
**Benefit:** Shows ALL product-warehouse combinations, even with zero inventory.

**Why this matters:**
- "Product X is missing from Warehouse Y" is as important as "Product X has 25 units in Warehouse Z"
- Business needs to see gaps to make transfer decisions
- Zero inventory = action needed, not "no data"

**Result:** 10 products × 4 active warehouses = 40 rows (complete picture)

---

##  Tool & Technology Choices

### Q4: Why dbt Core instead of dbt Cloud?

**A:** Honest answer: Learning and cost.

**dbt Core pros:**
- ✅ Free and open source
- ✅ Full control over environment
- ✅ Learn the fundamentals without GUI abstractions
- ✅ Works with any git workflow

**dbt Cloud pros:**
- ✅ IDE with autocomplete
- ✅ Built-in job scheduling
- ✅ Easier collaboration
- ✅ Integrated documentation hosting

**What I'd do in production:** Use dbt Cloud. The IDE features alone would have saved 2+ hours of debugging. But for learning? Core forces you to understand what's happening under the hood.

---

### Q5: Why Snowflake instead of BigQuery or Redshift?

**A:** Three reasons:

1. **Separate storage and compute** - Only pay for what you use
2. **Zero maintenance** - No clusters to manage, auto-scaling
3. **Trial credits** - Free $400 credit for learning

**Could this work on BigQuery?** Absolutely! The dbt code would be 95% identical.  
**Could this work on Postgres?** Yes, but you'd lose some Snowflake-specific features (like COPY INTO for seeds).

**Key insight:** The value is in the dbt patterns, not the warehouse. These patterns transfer across platforms.

---

### Q6: Why seeds instead of connecting to a real data source?

**A:** For a learning project, seeds were perfect because:

1. **Version controlled** - Data lives in git alongside code
2. **Reproducible** - Anyone can clone and run immediately
3. **No infrastructure** - No need to set up source databases
4. **Fast iteration** - Change CSV, run `dbt seed`, done

**In production:** You'd use sources pointing to staging tables loaded via Fivetran, Airbyte, or custom ETL.

**When to use seeds in production:**
- Small lookup tables (< 1000 rows)
- Reference data that changes rarely
- Mapping tables

---

##  Configuration Decisions

### Q7: Why did you configure custom schemas? Why not use the default schema?

**A:** Organization and permissions.

**Without custom schemas:**
```
RAW_DATA_DB.dbt_dev
  ├── raw_products (seed)
  ├── stg_warehouse__products (staging)
  ├── int_warehouse__movement_summary (intermediate)
  └── fct_inventory_summary (mart)
```
Everything in one schema = messy.

**With custom schemas:**
```
RAW_DATA_DB.dbt_dev_raw_seeds     (seeds)
RAW_DATA_DB.dbt_dev_staging       (staging)
RAW_DATA_DB.dbt_dev_intermediate  (intermediate)
RAW_DATA_DB.dbt_dev_marts         (marts)
```

**Benefits:**
- **Clear organization** - Know where each layer lives
- **Different permissions** - Analysts get read on marts, engineers get write on staging
- **Easier querying** - `USE SCHEMA dbt_dev_marts;` instead of filtering by name prefix

**The gotcha:** Dev schemas have prefixes (`dbt_dev_staging`) but prod doesn't (`staging`). This is by design!

---

### Q8: Why materialized as views for staging/intermediate but tables for marts?

**A:** Performance vs freshness trade-off.

**Views (staging/intermediate):**
- ✅ Always fresh (query runs on demand)
- ✅ No storage cost
- ✅ Fast to build
- ❌ Slower to query (recalculated each time)

**Tables (marts):**
- ✅ Fast to query (pre-calculated)
- ✅ Optimized for BI tools
- ❌ Snapshot in time (stale until rebuilt)
- ❌ Storage cost

**Strategy:**
- Lightweight transformations (staging) → Views
- Heavy aggregations (marts) → Tables

**Exception:** If an intermediate model is expensive (joins millions of rows), materialize as table.

---

##  Testing Decisions

### Q9: Why 35+ tests? Isn't that overkill for a small project?

**A:** Testing isn't about project size—it's about building good habits.

**What the tests caught:**
1. **Schema test:** Found duplicate product_id in seeds
2. **Singular test:** Caught reserved quantity > on_hand quantity
3. **Custom generic test:** Identified negative inventory values
4. **Relationship test:** Discovered orphaned movement records

**In production:** One bad number in an executive dashboard = lost credibility. Tests are insurance.

**The 80/20 rule:** 20% of tests (primary keys, not null, relationships) catch 80% of issues.

---

### Q10: Why create custom generic tests? Why not just use built-in tests?

**A:** Reusability across projects.

**Built-in test (not reusable):**
```yaml
# In EVERY model, repeat this
columns:
  - name: quantity
    tests:
      - dbt_utils.expression_is_true:
          expression: ">= 0"
```

**Custom generic test (reusable):**
```sql
-- tests/generic/test_is_positive.sql
{% test is_positive(model, column_name) %}
  select {{ column_name }}
  from {{ model }}
  where {{ column_name }} < 0
{% endtest %}
```

```yaml
# Now in ANY model, just
columns:
  - name: quantity
    tests:
      - is_positive
```

**Benefit:** Write once, use everywhere. In Project 2, I can use `is_positive` test immediately.

---

##  Code Patterns

### Q11: Why use the CTE pattern (with source, renamed, final)?

**A:** Debugging and consistency.

**Without CTEs:**
```sql
select
    p.product_id,
    coalesce(l.quantity, 0) as qty
from {{ source('seeds', 'products') }} p
left join {{ source('seeds', 'levels') }} l on p.id = l.product_id
where l.date = (select max(date) from {{ source('seeds', 'levels') }})
```
**Problem:** If this breaks, where do you look?

**With CTEs:**
```sql
with source_products as (
    select * from {{ source('seeds', 'products') }}
),
source_levels as (
    select * from {{ source('seeds', 'levels') }}
),
latest_levels as (
    select * from source_levels
    where date = (select max(date) from source_levels)
),
final as (
    select
        p.product_id,
        coalesce(l.quantity, 0) as qty
    from source_products p
    left join latest_levels l on p.id = l.product_id
)
select * from final
```

**Debugging:**
```sql
-- Test just the filter
select * from latest_levels;

-- Test just the join
select * from final where product_id = 1;
```

**Consistency:** Every model follows the same pattern. New team members know what to expect.

---

### Q12: Why use Jinja macros instead of just copying SQL?

**A:** DRY principle (Don't Repeat Yourself).

**Without macro (repetitive):**
```sql
-- In model 1
case when denominator = 0 then null else numerator / denominator end

-- In model 2
case when denominator = 0 then null else numerator / denominator end

-- In model 3... (repeat 10 times)
```

**With macro (write once):**
```sql
-- macros/safe_divide.sql
{% macro safe_divide(num, denom) %}
  case when {{ denom }} = 0 then null else {{ num }} / {{ denom }} end
{% endmacro %}

-- In any model
{{ safe_divide('outbound', 'inventory') }}
```

**Benefit:** Fix bug once, fixed everywhere. Change logic once, updated everywhere.

---

##  Business Logic Decisions

### Q13: Why calculate turnover as outbound/inventory instead of a standard formula?

**A:** Data availability.

**Standard turnover formula:**
```
Turnover = Cost of Goods Sold / Average Inventory Value
```

**Our formula:**
```
Turnover = Lifetime Outbound / Current Inventory
```

**Why the difference:**
- We don't have COGS (cost of goods sold) data
- We don't have historical inventory snapshots for "average"
- We DO have total outbound quantity and current quantity

**Key insight:** Use the data you HAVE, not the formula you WANT. Document assumptions clearly.

---

### Q14: Why flag "below reorder point" in the mart instead of calculating in BI tool?

**A:** Single source of truth.

**If calculated in BI tool:**
- Tableau calculates it one way
- Looker calculates it differently
- Excel export has yet another version
- Everyone argues about "whose number is right"

**If calculated in dbt:**
```sql
case 
    when quantity <= reorder_point then true 
    else false 
end as is_below_reorder_point
```
- ONE definition
- ONE calculation
- Tested and documented
- Everyone uses the same number

**Rule:** Business logic lives in data models, not BI tools.

---

##  Scalability Decisions

### Q15: You have 50 rows. Why design for "50 million tomorrow"?

**A:** Architectural decisions are hard to change later.

**Easy to change:**
- Add a column
- Add a test
- Change materialization

**Hard to change:**
- Folder structure
- Naming conventions
- Layer architecture

**Example:** If I used flat structure now, migrating 100 models to medallion later = weeks of work.

**Philosophy:** Make the structure right from the start, even if you're small. You can always scale up, but refactoring is painful.

---

### Q16: Why didn't you use incremental models?

**A:** Incremental adds complexity that isn't needed yet.

**Current approach (full refresh):**
```sql
-- Simple!
select * from source
```
**Build time:** 10 seconds  
**Complexity:** Low

**Incremental approach:**
```sql
-- Complex!
{{
    config(
        materialized='incremental',
        unique_key='id'
    )
}}
select * from source
{% if is_incremental() %}
    where updated_at > (select max(updated_at) from {{ this }})
{% endif %}
```
**Build time:** 5 seconds (only 50% faster)  
**Complexity:** High (need to handle edge cases)

**Rule:** Optimize for simplicity first. When builds take 30+ minutes, THEN go incremental.

---

##  Workflow Decisions

### Q17: Why document as you go instead of at the end?

**A:** Because future-you won't remember the details.

**What I documented immediately:**
- Why I used CROSS JOIN (while debugging)
- The custom schema prefix issue (right after hitting it)
- Column naming gotcha with hyphens (immediately after error)

**If I documented at the end:**
- "I used CROSS JOIN because... um... it worked?"
- Lost 50% of the valuable context

**Rule:** Document the "why" when you make the decision, not when you finish the project.

---

### Q18: Why break documentation into multiple files instead of one big file?

**A:** Discoverability and focus.

**One 40-page file:**
- Hard to navigate
- Everything mixed together
- Share the whole thing or nothing

**Multiple focused files:**
- `SETUP_GUIDE.md` - Send to someone setting up
- `TROUBLESHOOTING.md` - Link when someone hits an error
- `ARCHITECTURE.md` - Share with architects reviewing design

**Benefit:** Right information, right person, right time.

---

##  Learning Decisions

### Q19: What would you do differently if you rebuilt this?

**A:** Five things:

1. **Use dbt Cloud** - IDE features worth the cost
2. **Write tests first** - TDD for data catches issues earlier
3. **Start with smaller seed files** - 5 rows per file, expand later
4. **Use dbt's slim CI** - Test only changed models from day 1
5. **Set up pre-commit hooks** - Catch YAML syntax errors before push

**Biggest lesson:** The gap between "it works" and "it's production-ready" is testing, documentation, and modularity.

---

### Q20: Why did you choose this project for your portfolio?

**A:** It demonstrates the full stack:

✅ **Architecture** - Medallion design  
✅ **SQL skills** - Complex joins, CTEs, window functions  
✅ **dbt fundamentals** - Seeds, sources, models, tests, macros  
✅ **Data quality** - Comprehensive testing strategy  
✅ **Documentation** - Clear explanations of decisions  
✅ **Business value** - Real problem with measurable impact  

**Bonus:** It's impressive but explainable in a 30-minute interview. Complex enough to show skills, simple enough to walk through.

---

### Q21: What was the hardest technical challenge?

**A:** The CROSS JOIN logic for showing missing inventory.

**Why it was hard:**
- Counterintuitive (usually avoid CROSS JOINs)
- Had to explain to team why we need 40 rows when only 10 have data
- Performance concern (CROSS JOIN with millions of rows = scary)

**How I solved it:**
1. Documented the business need (show gaps)
2. Tested with small dataset first
3. Proved it's efficient with proper WHERE clause (only active warehouses)
4. Added clear comments in code explaining the pattern

**Learning:** Sometimes the "wrong" SQL pattern is the right business solution.

---

### Q22: How do you know your data is accurate?

**A:** Four layers of validation:

1. **Schema tests** - Structure (unique keys, not nulls, relationships)
2. **Singular tests** - Business logic (reserved < on_hand)
3. **Custom tests** - Reusable checks (no negatives)
4. **Manual verification** - Spot-check against source spreadsheets

**Plus:** Every model has tests. 100% test coverage = every model validated.

**In production:** Add data profiling (dbt-expectations) for statistical anomalies.

---

