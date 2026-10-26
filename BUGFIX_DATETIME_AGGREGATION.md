# Bug Fix: Datetime64 Aggregation Error in Phase 4

## Problem
**Error**: `datetime64 type does not support operation 'sum'`
**Location**: Phase 4 (Gold layer materialization and reporting)
**Root Cause**: Aggregation functions were applied to columns without checking their data types, causing operations like `sum()` to fail on datetime64 columns.

## Root Cause Analysis
The code attempted to apply arithmetic aggregations (`sum`, `mean`, `average`) to all columns specified in the Gold STTM rules, regardless of their data type. Datetime columns cannot be summed or averaged in pandas, leading to the error.

## Files Fixed

### 1. `agents/gold_agent.py` (Lines 199-213)
**Fix**: Added dtype validation before assigning aggregation functions.

**Before**:
```python
if source_col and source_col in df.columns:
    if "sum" in logic:
        agg_map[source_col] = "sum"
    elif "average" in logic or "avg" in logic or "mean" in logic:
        agg_map[source_col] = "mean"
    # ...
```

**After**:
```python
if source_col and source_col in df.columns:
    dtype = df[source_col].dtype
    is_numeric = pd.api.types.is_numeric_dtype(dtype)
    is_datetime = pd.api.types.is_datetime64_any_dtype(dtype)
    
    if "sum" in logic and is_numeric:
        agg_map[source_col] = "sum"
    elif ("average" in logic or "avg" in logic or "mean" in logic) and is_numeric:
        agg_map[source_col] = "mean"
    elif "count" in logic:
        agg_map[source_col] = "count"
    elif "max" in logic:
        agg_map[source_col] = "max"
    elif "min" in logic:
        agg_map[source_col] = "min"
```

**Changes**:
- Check if column is numeric before applying `sum` or `mean`
- `count`, `max`, `min` can be applied to any data type
- Datetime columns skip numeric-only aggregations

### 2. `agents/reporter.py` (Lines 128-155)
**Fix**: Added dtype validation in chart generation for bar and pie charts.

**Changes in Bar Chart (Lines 128-140)**:
- Check if `y_col` is numeric
- Use `sum()` for numeric columns
- Use `count()` for non-numeric columns (datetime, string, etc.)

**Changes in Pie Chart (Lines 150-154)**:
- Check if `values_col` is numeric
- Use `sum()` for numeric columns
- Use `count()` for non-numeric columns

## Impact
- ✅ Phase 4 (Gold materialization) will no longer crash when processing datetime columns
- ✅ Charts will render correctly even with non-numeric columns
- ✅ Existing numeric column behavior unchanged
- ✅ More robust aggregation function selection

## Testing
To verify the fix:
1. Run orchestrator with data containing datetime columns
2. Ensure Phase 4 completes successfully
3. Check Gold layer output contains expected tables
4. Verify HTML report generates without errors

## Related Code
- `pd.api.types.is_numeric_dtype()`: Used to check if column contains numeric data
- `pd.api.types.is_datetime64_any_dtype()`: Used to check for datetime columns (noted for future use)
- Standard pandas aggregation functions: `sum()`, `mean()`, `count()`, `min()`, `max()`
