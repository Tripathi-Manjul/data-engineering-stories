# STAR Story: Fixing Critical Snowflake Pipeline Under Pressure

> **Scenario**: Production pipeline failure 4 hours before Leadership meeting
> **Technologies**: Snowflake, Airflow, Python, SQL
> **Role**: Data Engineer (3 YOE)

---

## The Story

### **Situation**

Our company was preparing for a quarterly board meeting where leadership would present manufacturing inventory forecasts. At 10 AM, I received an urgent Slack alert: our core forecasting pipeline in Airflow had failed with a cryptic Snowflake error.

The pipeline had been running smoothly for 3 months, but the data team had pushed a schema change the previous night—adding new columns to track material categories. The forecasting logic itself hadn't changed, but suddenly nothing worked.

**Timeline pressure**: Board meeting at 2 PM. Business needed the forecasts by 1 PM for final review.

---

### **Task**

My immediate responsibilities:
1. **Diagnose the root cause** of the failure quickly
2. **Fix the pipeline** to generate forecasts in time
3. **Ensure data accuracy**—couldn't risk wrong numbers in a board presentation
4. **Prevent recurrence** across our other pipelines

The challenge: I had roughly 3 hours of actual working time (accounting for testing and validation).

---

### **Action**

#### Phase 1: Rapid Diagnosis (20 minutes)

**First instinct—check the error:**

```
Error: SQL compilation error: 
Invalid identifier 'MATERIAL-CATEGORY'
Object 'PROD.INVENTORY.MATERIAL-CATEGORY' does not exist
```

**Initial panic thought**: "Did we lose data? Is the table corrupted?"

**Systematic approach:**

1. **Pulled up Snowflake Query History** via the web UI
   - Searched for failed queries from our service account in the last hour
   - Found the exact SQL statement that broke

2. **Examined the failing query:**
```sql
SELECT 
    material_id,
    "MATERIAL-CATEGORY",  -- This line was breaking
    consumption_rate
FROM PROD.INVENTORY.MATERIALS
WHERE ...
```

3. **Checked Snowflake documentation** (frantic Google search):
   - Found: Snowflake treats identifiers with special characters (hyphens, spaces) as case-sensitive
   - Requires exact quoting: `"MATERIAL-CATEGORY"` vs `MATERIAL-CATEGORY`

4. **Verified in Snowflake console:**
```sql
-- This worked:
SELECT "MATERIAL-CATEGORY" FROM PROD.INVENTORY.MATERIALS LIMIT 1;

-- This failed:
SELECT MATERIAL-CATEGORY FROM PROD.INVENTORY.MATERIALS LIMIT 1;
```

**Root cause identified**: Our Python connector was auto-quoting identifiers, but inconsistently. The schema change introduced a hyphenated column name that triggered Snowflake's case-sensitivity rules.

#### Phase 2: Quick Fix Implementation (30 minutes)

**Decision point**: Proper fix (rename column) would require schema migration + full regression testing. **Time: don't have it.**

**Short-term solution: Disable identifier quoting in connector**

1. **Modified Airflow DAG code:**
```python
# OLD CODE (in snowflake_hook.py wrapper)
conn = snowflake.connector.connect(
    user=user,
    password=password,
    account=account,
    warehouse=warehouse,
    database=database,
    schema=schema
)

# NEW CODE - Added this parameter
conn = snowflake.connector.connect(
    user=user,
    password=password,
    account=account,
    warehouse=warehouse,
    database=database,
    schema=schema,
    quote_identifiers=False  # CRITICAL FIX
)
```

2. **Pushed code to dev branch** (no time for full CI/CD)

3. **Triggered manual DAG run in Airflow** on dev environment

#### Phase 3: Validation Under Pressure (40 minutes)

**Testing strategy** (couldn't skip this even with time pressure):

1. **Smoke test on small data subset:**
   - Filtered to single plant: `WHERE plant_id = 'PLANT_001'`
   - Ran pipeline end-to-end
   - Verified output matched previous day's logic

2. **Spot-checked critical metrics:**
   - Row counts: 91,000 records (expected ~90K)
   - No null values in forecast columns
   - Date ranges correct

3. **Parallel communication:**
   - **10:50 AM**: Slacked business stakeholder: "Found issue, testing fix, need 90 more minutes"
   - Bought myself buffer time and managed expectations

#### Phase 4: Production Deployment (15 minutes)

1. **Merged dev branch to main** (with quick peer review over Slack)
2. **Cleared Airflow DAG cache**: `airflow dags reserialize`
3. **Triggered production run**: 11:15 AM
4. **Monitored actively:**
   - Watched Snowflake query history in real-time
   - Set up Slack notifications for task status
   - Kept terminal open with Airflow logs streaming

**Pipeline completed successfully at 12:30 PM.**

#### Phase 5: Post-Crisis Actions (30 minutes after delivery)

**Immediate documentation** (while details were fresh):

1. **Created runbook in Confluence:**
   - Title: "Snowflake Identifier Quoting Issues - Troubleshooting Guide"
   - Included error patterns, root cause, solution steps
   - Added prevention checklist

2. **Audited other pipelines:**
   - Searched codebase for similar Snowflake connections
   - Found 4 other pipelines potentially affected
   - Created backlog tickets with priority labels

3. **Quick team huddle:**
   - Explained what happened (learning opportunity)
   - Proposed database naming standards

#### Phase 6: Long-term Prevention (Next sprint)

1. **Established naming conventions:**
   - No hyphens in column names (use underscores)
   - No special characters in identifiers
   - Added pre-merge checks in CI/CD

2. **Created reusable Snowflake connection helper:**
```python
def get_snowflake_connection(quote_identifiers=False):
    """
    Standardized Snowflake connector with safe defaults.
    
    Args:
        quote_identifiers: Set to False to avoid case-sensitivity issues.
                          Default: False (safer for most use cases)
    """
    return snowflake.connector.connect(
        quote_identifiers=quote_identifiers,
        # ... other params
    )
```

3. **Updated team onboarding docs** with lessons learned

---

### **Result**

**Immediate outcomes:**
- ✅ **Fixed pipeline in 2.5 hours** (12:30 PM completion)
- ✅ **Forecasts delivered by 12:45 PM** (75 minutes before deadline)
- ✅ **100% data accuracy** (validated against previous runs)
- ✅ **Board meeting proceeded smoothly** (no visibility into the crisis)

**Long-term impact:**
- ✅ **Prevented 8 future pipeline failures** by auditing similar code
- ✅ **Created reusable runbook** referenced 12+ times by team in next 6 months
- ✅ **Established company-wide database naming standards** (adopted by 3 teams)
- ✅ **Reduced mean time to resolution (MTTR)** for Snowflake issues from ~4 hours to ~30 minutes

**Personal growth:**
- Learned to balance quick fixes with proper solutions
- Improved crisis communication with non-technical stakeholders
- Built confidence in debugging under pressure

---

## Follow-Up Questions & Model Answers

### Q1: Why did you choose `quote_identifiers=False` instead of fixing the column name?

**Model Answer:**

"Great question—this was a deliberate trade-off decision.

**Context**: I had 3 hours total. Renaming the column properly would have required:
1. Schema migration script (20 min)
2. Updating all downstream SQL queries (unknown scope—could be dozens)
3. Full regression testing (2+ hours minimum)
4. Coordinating with the data team who pushed the change

**Risk assessment**: Renaming had high risk of introducing new breaks.

**The `quote_identifiers=False` approach:**
- **Low risk**: Only changes how the connector handles quoting, doesn't touch data
- **Quick validation**: Could test on subset in 30 minutes
- **Reversible**: If it broke something else, I could roll back instantly

**Post-crisis**: I created a ticket to properly rename the column in the next sprint. We did it during a maintenance window with full testing. The quick fix gave us time to do the right fix properly.

**Key principle**: In a crisis, **contain the damage first, then fix the root cause properly.** Don't introduce new risks when you're already under pressure."

---

### Q2: How did you ensure data accuracy when you were rushing?

**Model Answer:**

"Even under pressure, I couldn't skip validation—wrong data in a board meeting would be career-limiting.

**My testing strategy was streamlined but comprehensive:**

1. **Subset testing first** (10 minutes):
   - Ran on single plant instead of all 15 plants
   - This gave me 90% confidence the logic worked
   - Trade-off: Speed vs. full coverage

2. **Comparison against baseline** (15 minutes):
   - Pulled previous day's forecast output
   - Compared row counts, column schemas, null rates
   - Spot-checked 20 random material IDs manually

3. **Business logic sanity checks** (10 minutes):
   - Forecasted consumption rates within expected ranges?
   - No negative numbers (red flag for data issues)
   - Date ranges correct (not forecasting for past dates)

4. **Parallel validation** (during production run):
   - While main pipeline ran, I queried output in real-time
   - Watched row counts accumulate
   - Set up alerts for any NULL values in critical columns

**What I didn't do**: Full end-to-end integration testing (would take 2 hours). I accepted calculated risk—my fix was isolated to the connection layer, not business logic.

**Result**: Data was accurate. But I documented assumptions and limitations in my runbook, so future engineers wouldn't blindly trust this approach."

---

### Q3: What would you have done differently if you had this situation again?

**Model Answer:**

"Two key improvements:

**1. Better monitoring and alerting (prevention):**

If we'd had proper schema change alerts, I would've caught this issue *before* production. After this incident, I implemented:
```python
# Daily schema drift check in Airflow
@dag(schedule_interval='@daily')
def schema_validation_dag():
    check_for_special_characters_in_columns()
    alert_if_new_problematic_columns()
```

Now we get Slack alerts when someone adds columns with hyphens/spaces.

**2. Earlier stakeholder communication:**

I waited 20 minutes before messaging the business team. In hindsight, I should've sent an immediate heads-up:
- **10:00 AM**: "Pipeline failed, investigating, will update in 30 min"

This would've given them time to prepare contingency plans if I couldn't fix it. **Managing expectations early reduces panic on both sides.**

**What I did right:**
- Systematic debugging (didn't just try random fixes)
- Documentation during the crisis (captured details while fresh)
- Post-mortem without blame (focused on process improvements)

**Philosophy shift**: After this, I started thinking about **blast radius**—how can I design systems where one schema change doesn't break everything? Led me to explore better abstractions like dbt semantic layer."

---

### Q4: How did this incident change your approach to database design or team processes?

**Model Answer:**

"This incident had ripple effects beyond the immediate fix. It exposed gaps in our processes.

**1. Database naming standards (immediate impact):**

We formalized rules that didn't exist before:
```yaml
# database_standards.yaml (added to team wiki)
column_naming:
  allowed: [a-z, 0-9, underscore]
  forbidden: [hyphen, space, special chars]
  case: lowercase_with_underscores
  
table_naming:
  pattern: "{source}_{entity}_{granularity}"
  example: "raw_inventory_materials_daily"
```

**2. Schema change process (process improvement):**

Before: Anyone could add columns ad-hoc.  
After: Schema changes require:
- Pull request with before/after schema diff
- Impact analysis (which pipelines read this table?)
- Automated tests checking for problematic column names

**3. Dependency mapping (visibility):**

Created a **data lineage graph** showing which pipelines depend on which tables. Now when someone changes a schema, they can immediately see: "This table is read by 12 downstream processes."

Used tools like:
- dbt for lineage within transformations
- Custom Airflow plugin to track source → DAG dependencies

**4. Crisis runbooks (knowledge sharing):**

This incident became our template for documentation. Now every production incident gets:
- Root cause analysis doc
- Runbook with troubleshooting steps
- Jira epic tracking long-term fixes

**Cultural shift**: We went from "move fast and break things" to "move fast with safety nets." This incident was the catalyst for adopting dbt in our next quarter—better abstractions prevent these issues at the design level.

**The meta-lesson**: Sometimes the best outcome of a crisis isn't the immediate fix—it's the systemic improvements that prevent entire classes of problems."

---