# dbt Advanced Features Workshop - Complete Guide
## 45-60 Minute Hands-On Workshop

---

## Table of Contents

1. [Workshop Overview](#workshop-overview)
2. [Prerequisites](#prerequisites)
3. [Workshop Structure](#workshop-structure)
4. [Session 1: Model Contracts](#session-1-model-contracts)
5. [Session 2: State-aware Runs](#session-2-state-aware-runs)
6. [Session 3: Python Models](#session-3-python-models)
7. [Session 4: Exposures](#session-4-exposures)
8. [Troubleshooting](#troubleshooting)
9. [Additional Resources](#additional-resources)

---

## Workshop Overview

### What You'll Learn

This workshop covers four advanced dbt features:

1. **Model Contracts** - Enforce schema expectations (15 min)
2. **State-aware Runs** - Run only changed models (10 min)
3. **Python Models** - Transform data with Python (15 min)
4. **Exposures** - Document downstream dependencies (5 min)

### Workshop Models

All workshop models are stored in `models/mini-workshop/` and use the `mini-workshop` tag.

**Schema:** `DEV.MINI_WORKSHOP` (isolated from production)

**Model Structure:**
```
models/mini-workshop/
├── 01_base/                    # Base models (read from sources)
├── 02_dim/                     # Dimension models
├── 03_fact/                    # Fact models
├── 04_mart_contracts/          # Model Contracts demo
├── 05_python_models/           # Python Models demo
├── 06_exposures/               # Exposures demo
└── 07_state_aware/             # State-aware demo
```

---

## Prerequisites

- dbt version 1.5+ installed
- Access to Snowflake (DEV database)
- Basic SQL knowledge
- Familiarity with dbt project structure
- Environment variables configured

### Verify Your Setup

```bash
# Check dbt version
dbt --version  # Should be 1.5 or higher

# Test connection
dbt debug

# Parse the project
dbt parse
```

---

## Workshop Structure

### Quick Start Commands

```bash
# Build all workshop models
dbt build --select tag:mini-workshop

# Build specific feature
dbt build --select mart_proofhub_task_ws_contract     # Contracts
dbt build --select mart_proofhub_task_ws_simple       # State-aware
dbt build --select mart_proofhub_task_ws_python         # Python models

# View documentation
dbt docs generate && dbt docs serve
```

⚠️ **IMPORTANT:** Always use `tag:mini-workshop`, NOT `path:models/mini-workshop`  
The path selector includes exposures and other non-model files that will cause errors.

---

## Session 1: Model Contracts (15 minutes)

### What Are Model Contracts?

Model contracts enforce schema expectations between your models and downstream consumers (other models, BI tools, etc.).

**Why use them?**
- ✅ Fail fast on schema mismatches
- ✅ Explicity declare expected output
- ✅ Protect downstream consumers from breaking changes

### How It Works

A model contract defines:
- Column names
- Data types
- Required fields
- Constraints

If the actual output doesn't match, dbt fails immediately.

### Hands-On Exercise

#### Step 1: Examine the Contract Model

Look at `models/mini-workshop/04_mart_contracts/mart_proofhub_task_ws_contract.sql`:

```sql
{{
    config(
        materialized = 'table',
        tags = ['mini-workshop'],
        owner = ['iqbal.effendy@evermos.com'],
        contract={
            "enforced": false,  # Set to true for strict enforcement
            "columns": [
                {"name": "ticket_number", "data_type": "VARCHAR"},
                {"name": "task_id", "data_type": "BIGINT"},
                {"name": "is_completed", "data_type": "BOOLEAN"},
                # ... etc
            ]
        }
    )
}}
```

Notice the `contract` block defining expected columns and types.

#### Step 2: Build the Model

```bash
# Build the model with contract
dbt build --select mart_proofhub_task_ws_contract
```

**Expected Result:** Model builds successfully (with `enforced: false`).  
If you set `enforced: true`, dbt will validate the schema matches exactly.

#### Step 3: Compare with Non-Contract Model

```bash
# Build the simple version without contract
dbt build --select mart_proofhub_task_ws_simple
```

**Key Difference:** The contract model has explicit schema definition, the simple model doesn't.

#### Step 4: Try Introducing a Schema Violation (Optional)

1. Edit `mart_proofhub_task_ws_contract.sql`
2. Change the contract to expect `ticket_number` as `BIGINT` instead of `VARCHAR`
3. Set `enforced: true`
4. Rebuild and see dbt catch the mismatch!

### When to Use Model Contracts

- ✅ Mart layer models consumed by BI tools
- ✅ Public APIs or external consumers
- ✅ Critical data products
- ⚠️ Experimental or frequently-changing models

### Key Takeaways

- Contracts enforce schema expectations
- They prevent breaking changes to downstream consumers
- Use `enforced: false` during development, `true` in production
- dbt fails early if schema doesn't match

---

## Session 2: State-aware Runs (10 minutes)

### What Are State-aware Runs?

State-aware runs compare the current code state with previous run state to determine which models changed.

**Why use them?**
- ⚡ Run only what changed - faster iterations
- 💰 Reduce compute costs
- 🚀 Faster CI/CD pipelines
- 📊 Selective refresh

### How It Works

```
Current State          Previous State
  ↓                       ↓
code changes          last run manifest
  ↓                       ↓
State Comparison
  ↓
Only changed models run
```

### Hands-On Exercise

#### Step 1: Initial Build

```bash
# Build everything first to establish baseline state
dbt build --select tag:mini-workshop
```

**Expected Result:** All models run (they all have changed from baseline).

#### Step 2: Modify a Model

Edit `models/mini-workshop/07_state_aware/mart_proofhub_task_ws_simple.sql`:

```sql
-- Add a comment or minor change
-- This is a test modification
SELECT
    task.ticket_number,
    task.task_id,
    -- ... rest of query
```

#### Step 3: Run State-Aware

```bash
# Only run what changed
dbt run --select state:modified
```

**Expected Result:** Only the modified model (and its downstream dependencies) runs!

#### Step 4: Verify

```bash
# See what would run with state comparison
dbt list --select state:modified
```

You should see only the changed model(s).

### Available State Selectors

```bash
state:modified    # Models that changed
state:new         # Models that don't exist in previous state
state:deleted     # Models that existed but no longer do
state:modified+   # Modified models + their downstream
```

### When to Use State-aware Runs

- ✅ Development: Quick iteration on changed models
- ✅ CI/CD: Test only what changed
- ✅ Large projects: Avoid unnecessary runs
- ✅ Cost optimization: Reduce compute usage

### Key Takeaways

- State-aware runs save time by running only what changed
- Essential for large projects and CI/CD
- Use `state:modified` for development, `state:modified+` for testing downstream impact

---

## Session 3: Python Models (15 minutes)

### What Are Python Models?

Python models allow you to transform data using Python instead of SQL.

**Why use them?**
- 🐍 Complex logic easier in Python
- 📊 Statistical operations
- 🔬 Machine learning features
- 🧮 Data quality checks

### How It Works

```python
def model(dbt, session):
    import pandas as pd
    
    # Get upstream data
    upstream = dbt.ref("my_model")
    df = upstream.to_pandas()  # Convert to pandas DataFrame
    
    # Python transformations
    df['new_col'] = df['col1'] + df['col2']
    
    # Return result
    return df
```

### Hands-On Exercise

#### Step 1: Examine Python Models

Look at `models/mini-workshop/05_python_models/mart_proofhub_task_ws_python_simple.py`:

```python
def model(dbt, session):
    import pandas as pd
    
    # Get reference - dbt.ref() returns a Snowpark DataFrame
    fact_task = dbt.ref("fact_proofhub_task_ws")
    
    # Convert to pandas
    df = fact_task.to_pandas()
    
    # Python transformations
    if not df.empty:
        df['has_high_priority_keyword'] = df['task_title'].str.contains(
            'Urgent|Critical|Important', case=False, na=False
        )
        df['analysis_timestamp'] = pd.Timestamp.now()
    
    return df
```

#### Step 2: Configure Python Models

Check `models/mini-workshop/05_python_models/_05_python_models_models.yml`:

```yaml
models:
  - name: mart_proofhub_task_ws_python_simple
    config:
      tags: ['mini-workshop']
      materialized: table
      packages:
        - pandas
```

**Important:** Python models configure via YAML, not in the `.py` file.

#### Step 3: Build the Python Model

```bash
# Build the Python model
dbt build --select mart_proofhub_task_ws_python_simple
```

**Expected Result:** 
- Model runs successfully
- Creates table in `DEV.MINI_WORKSHOP`
- Adds `has_high_priority_keyword` and `analysis_timestamp` columns

#### Step 4: Examine the Output

```bash
# Query the result
dbt run-operation query \
  --args '{sql: "SELECT * FROM DEV.MINI_WORKSHOP.MART_PROOFHUB_TASK_WS_PYTHON_SIMPLE LIMIT 5"}'
```

Or use your SQL client to query:
```sql
SELECT * FROM DEV.MINI_WORKSHOP.MART_PROOFHUB_TASK_WS_PYTHON_SIMPLE 
LIMIT 10;
```

You should see the new columns added by Python transformations.

### Python Models Limitations

- ⚠️ Not all adapters support Python models (Snowflake, Databricks, BigQuery do)
- ⚠️ Performance considerations - SQL is often faster
- ⚠️ More complex to debug
- ⚠️ Requires pandas/numpy libraries

### When to Use Python Models

- ✅ Complex statistical operations
- ✅ Data quality checks
- ✅ Feature engineering for ML
- ✅ Transformations awkward in SQL

### When NOT to Use Python

- ❌ Simple aggregations (SQL is better)
- ❌ Standard data transformations
- ❌ When performance is critical
- ❌ Basic joins and filters

### Key Takeaways

- Python models extend dbt's capabilities beyond SQL
- Useful for complex logic and data science workflows
- But SQL is usually simpler and faster for most transformations

---

## Session 4: Exposures (5 minutes)

### What Are Exposures?

Exposures document external dependencies like BI dashboards, notebooks, and APIs that use your dbt models.

**Why use them?**
- 📊 Know what uses your data
- 🔍 Impact analysis: What breaks if you change a model?
- 👥 Communication: Notify affected teams
- 📝 Complete lineage graph

### How It Works

Exposures link your dbt models to:
- BI dashboards (Tableau, Looker, Metabase)
- Notebooks (Jupyter, Hex)
- APIs
- Analysis reports

### Hands-On Exercise

#### Step 1: View Exposures

Look at `models/mini-workshop/06_exposures/sources/_workshop_exposures.yml`:

```yaml
exposures:
  - name: proofhub_task_dashboard
    type: dashboard
    description: "Main ProofHub Task Dashboard"
    url: "https://your-bi-tool.com/dashboard/proofhub_tasks"
    maturity: high
    owner:
      name: "Data Team"
      email: "data@evermos.com"
    depends_on:
      - ref('mart_proofhub_task_ws_contract')
```

Notice it documents:
- What type of exposure (dashboard, notebook, API)
- Who owns it
- What models it depends on

#### Step 2: Generate Documentation with Exposures

```bash
# Generate docs
dbt docs generate

# Serve docs (opens in browser)
dbt docs serve
```

#### Step 3: View in Lineage Graph

In the dbt docs UI:
1. Navigate to the lineage graph
2. Find your exposure nodes
3. See the complete dependency chain from models → exposures

#### Step 4: Check Impact

In the docs UI, click on `mart_proofhub_task_ws_contract` and see:
- Which exposures use it
- Who owns those exposures
- Impact of making changes

### Exposure Types

```yaml
type: dashboard   # BI dashboards (Tableau, Looker, etc.)
type: notebook   # Jupyter notebooks, Hex
type: application # Applications using the data
type: analysis   # Analysis reports
type: ml         # ML experiments
```

### When to Use Exposures

- ✅ Any model used by BI tools
- ✅ Mart layer models
- ✅ Critical data products
- ✅ Models with external consumers

### Key Takeaways

- Exposures complete the dependency picture
- Help with impact analysis and change management
- Essential for data governance

---

## Troubleshooting

### Issue: "Path selector includes exposures"

**Error:** Building with `path:models/mini-workshop` includes non-model files

**Solution:** Use tag selector instead
```bash
dbt build --select tag:mini-workshop  # ✅ Correct
dbt build -s "path:models/mini-workshop"  # ❌ Wrong
```

### Issue: Python model errors

**Error:** `KeyError: 'column_name'` or similar

**Solution:** Column names might be uppercase. Use defensive programming:
```python
if 'column_name' in df.columns:
    # use column
```

### Issue: State-aware not working

**Error:** No previous state to compare

**Solution:** First run without state-aware:
```bash
dbt build --select tag:mini-workshop  # Establish state
# Then modify and use:
dbt run --select state:modified
```

### Issue: Contract violations

**Error:** Schema doesn't match contract

**Solution:** 
- Check data types match
- Verify column names match exactly
- Start with `enforced: false` during development

---

## Additional Resources

### Documentation
- [Model Contracts](https://docs.getdbt.com/docs/collaborate/govern/model-contracts)
- [State Comparison](https://docs.getdbt.com/reference/commands/cmd-state)
- [Python Models](https://docs.getdbt.com/docs/build/python-models)
- [Exposures](https://docs.getdbt.com/docs/build/exposures)

### Workshop Files
- Location: `models/mini-workshop/`
- Schema: `DEV.MINI_WORKSHOP`
- Tag: `mini-workshop`

### Useful Commands Reference

```bash
# Build everything
dbt build --select tag:mini-workshop

# Build specific feature
dbt build --select mart_proofhub_task_ws_contract
dbt build --select mart_proofhub_task_ws_python_simple

# State-aware
dbt run --select state:modified

# Documentation
dbt docs generate && dbt docs serve

# Test models
dbt test --select tag:mini-workshop

# List models
dbt list --select tag:mini-workshop
```

---

## Workshop Completion Checklist

- [ ] Session 1: Built and understood model contracts
- [ ] Session 2: Used state-aware runs successfully
- [ ] Session 3: Created and ran Python models
- [ ] Session 4: Viewed exposures in documentation
- [ ] Generated docs and explored lineage graph
- [ ] Understand when to use each feature
- [ ] Ready to apply these in production

---

## Key Takeaways Summary

| Feature | Use Case | Command |
|---------|----------|---------|
| **Contracts** | Schema enforcement | `dbt build --select contract` |
| **State-aware** | Run only changed | `dbt run --select state:modified` |
| **Python** | Complex transforms | `dbt build --select python_model` |
| **Exposures** | Documentation | `dbt docs generate` |

### Best Practices

1. **Model Contracts:** Use for mart layer and BI-consumed models
2. **State-aware:** Always use in CI/CD, optionally in development
3. **Python Models:** Use sparingly, prefer SQL when possible
4. **Exposures:** Document all external dependencies

---

**Congratulations!** 🎉 You've completed the dbt Advanced Features Workshop!

