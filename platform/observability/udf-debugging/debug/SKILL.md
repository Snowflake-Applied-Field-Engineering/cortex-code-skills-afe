---
name: udf-debugging-debug
description: "Query event table logs and debug UDF issues"
parent_skill: udf-debugging
---

# Debug UDF with Event Table Logs

This workflow helps query event tables for UDF logs, analyze errors, identify common patterns, and suggest fixes.

## When to Load

Main skill routes here when user wants to:
- Debug a failing UDF
- View logs from UDF executions
- Analyze UDF errors
- Understand why a UDF is misbehaving

---

## ⚠️ CRITICAL: Always Check Telemetry First

**Before attempting any debugging, always check for logs, metrics, and traces in the event table.**

Even if you can determine the source of an error from the code alone:
- **DO NOT skip telemetry collection** - runtime data often reveals additional bugs not visible in code
- **DO NOT proceed with code-only debugging** until telemetry is enabled and checked
- **ALWAYS prompt the user to enable collection** if no telemetry data is found

**Rationale:**
1. Runtime errors may differ from what code inspection suggests
2. There may be multiple bugs - fixing one visible in code may miss others only visible at runtime
3. Telemetry data improves future debugging by establishing baselines
4. Users benefit from learning the telemetry-first debugging workflow

---

## Workflow

### Step 1: Gather UDF Information

**Goal:** Identify the target UDF and execution context

**Ask user for:**
1. **Fully qualified UDF name**: `<database>.<schema>.<function_name>`
2. **Parameter types** (for overloaded functions): e.g., `(VARCHAR, NUMBER)`
3. **Time frame**: When did the issue occur? (last hour, specific time, etc.)
4. **Query ID** (if available): Specific execution to investigate

**Verify UDF exists:**
```sql
SHOW USER FUNCTIONS LIKE '<function_name>' IN SCHEMA <database>.<schema>;
```

---

### Step 2: Verify Event Table Configuration

**Goal:** Ensure logs are being collected

**Load reference:** [../references/observability-parameters.md](../references/observability-parameters.md)

**Check event table and levels:**
```sql
-- Check what event table is active
SHOW PARAMETERS LIKE 'EVENT_TABLE' IN ACCOUNT;
SHOW PARAMETERS LIKE 'EVENT_TABLE' IN DATABASE <database>;

-- Check telemetry levels at account
SHOW PARAMETERS LIKE 'LOG_LEVEL' IN ACCOUNT;
SHOW PARAMETERS LIKE 'METRIC_LEVEL' IN ACCOUNT;
SHOW PARAMETERS LIKE 'TRACE_LEVEL' IN ACCOUNT;

-- Check telemetry levels on the specific UDF
SHOW PARAMETERS LIKE 'LOG_LEVEL' IN FUNCTION <db.schema.fn>(<types>);
SHOW PARAMETERS LIKE 'METRIC_LEVEL' IN FUNCTION <db.schema.fn>(<types>);
SHOW PARAMETERS LIKE 'TRACE_LEVEL' IN FUNCTION <db.schema.fn>(<types>);
```

**If telemetry levels are disabled at account level (LOG_LEVEL = OFF, METRIC_LEVEL = NONE, TRACE_LEVEL = OFF):**

**⚠️ STOPPING POINT**: Present these options to the user:

---

**Telemetry collection is disabled at the account level.**

I recommend enabling at the **account level** as a best practice, but you can also enable on just this UDF.

**Option A: Enable at Account Level (Recommended)**
- Applies to all UDFs/procedures automatically
- No need to configure each object individually
- Requires ACCOUNTADMIN or delegated privileges

```sql
ALTER ACCOUNT SET LOG_LEVEL = 'INFO';
ALTER ACCOUNT SET METRIC_LEVEL = 'ALL';
ALTER ACCOUNT SET TRACE_LEVEL = 'ALWAYS';
```

**Option B: Enable on This UDF Only**
- If you don't want to change account settings or lack permissions
- You'll need to repeat this for each UDF you want to debug

```sql
ALTER FUNCTION <db.schema.fn>(<types>) SET LOG_LEVEL = 'DEBUG';
ALTER FUNCTION <db.schema.fn>(<types>) SET METRIC_LEVEL = 'ALL';
ALTER FUNCTION <db.schema.fn>(<types>) SET TRACE_LEVEL = 'ALWAYS';
```

**Option C: Request Admin to Enable Account-Level**
- I can generate a message to send to your admin

---

**Based on user's choice:**
- **Option A** → Execute account-level SQL (with confirmation), then continue to Step 3
- **Option B** → Execute object-level SQL (with confirmation), then continue to Step 3
- **Option C** → Load [../setup/SKILL.md](../setup/SKILL.md) Step 4D for admin request template

**If event table not configured:**
> Load the full setup workflow: **-> Load**: [../setup/SKILL.md](../setup/SKILL.md)

---

### Step 3: Check for Existing Telemetry Data (MANDATORY)

**Goal:** Verify telemetry data exists before proceeding with debugging

**⚠️ This step is MANDATORY** - do not skip even if you think you can identify the issue from code.

**Query for logs, traces, and metrics:**
```sql
-- Check for any telemetry data from this UDF
SELECT 
  RECORD_TYPE,
  COUNT(*) as count,
  MIN(TIMESTAMP) as earliest,
  MAX(TIMESTAMP) as latest
FROM SNOWFLAKE.TELEMETRY.EVENTS
WHERE RESOURCE_ATTRIBUTES['snow.executable.name'] ILIKE '%<udf_name>%'
  AND TIMESTAMP > DATEADD('hour', -24, CURRENT_TIMESTAMP())
GROUP BY RECORD_TYPE
ORDER BY RECORD_TYPE;
```

**Expected RECORD_TYPE values:**
- `LOG` - Log messages from the UDF
- `SPAN` - Trace spans (execution timing)
- `SPAN_EVENT` - Custom trace events
- `METRIC` - Metrics data

**If NO telemetry data is found:**

**⚠️ MANDATORY STOPPING POINT** - Do not proceed with code analysis.

Present to user:
> **No telemetry data found for this UDF.**
>
> Before I can effectively debug this issue, we need to enable telemetry collection. Even if the error seems obvious from the code, runtime telemetry often reveals:
> - Additional bugs not visible in static code analysis
> - The actual execution path and timing
> - Unexpected input values or edge cases
> - Performance issues
>
> **Please choose an option to enable telemetry, then re-run the UDF so we can collect data.**

Return to Step 2 to enable telemetry, then ask user to:
1. Re-execute the UDF to generate telemetry data
2. Wait a few seconds for data to appear
3. Return to this step to verify data exists

**If telemetry data IS found:**
- Summarize what data types are available (logs, spans, events, metrics)
- Note the time range covered
- Proceed to Step 4

---

### Step 4: Query Event Table for Logs

**Goal:** Retrieve logs from the target UDF

**Load reference:** [../references/event-table-columns.md](../references/event-table-columns.md)

**Base query for logs:**
```sql
SELECT 
  TIMESTAMP,
  RECORD_TYPE,
  RECORD['severity_text']::STRING AS SEVERITY,
  VALUE::STRING AS MESSAGE,
  RESOURCE_ATTRIBUTES['snow.executable.name']::STRING AS EXECUTABLE_NAME,
  RESOURCE_ATTRIBUTES['snow.executable.type']::STRING AS EXECUTABLE_TYPE,
  RESOURCE_ATTRIBUTES['snow.query.id']::STRING AS QUERY_ID,
  RESOURCE_ATTRIBUTES['snow.database.name']::STRING AS DATABASE_NAME,
  RESOURCE_ATTRIBUTES['snow.schema.name']::STRING AS SCHEMA_NAME,
  RECORD_ATTRIBUTES['code.lineno']::STRING AS LINE_NUMBER,
  RECORD_ATTRIBUTES['exception.type']::STRING AS EXCEPTION_TYPE,
  RECORD_ATTRIBUTES['exception.message']::STRING AS EXCEPTION_MESSAGE
FROM SNOWFLAKE.TELEMETRY.EVENTS
WHERE RESOURCE_ATTRIBUTES['snow.executable.name'] ILIKE '%<udf_name>%'
  AND TIMESTAMP > DATEADD('hour', -<N>, CURRENT_TIMESTAMP())
ORDER BY TIMESTAMP DESC
LIMIT 100;
```

**Present initial results** before applying filters.

---

### Step 5: Apply Filters

**Goal:** Narrow down to relevant logs

**Filter by severity:**
```sql
-- Errors only
WHERE RECORD['severity_text'] IN ('ERROR', 'FATAL')

-- Warnings and above
WHERE RECORD['severity_text'] IN ('WARN', 'ERROR', 'FATAL')

-- All logs (INFO and above)
WHERE RECORD['severity_text'] IN ('INFO', 'WARN', 'ERROR', 'FATAL')
```

**Filter by time range:**
```sql
-- Last N minutes
AND TIMESTAMP > DATEADD('minute', -30, CURRENT_TIMESTAMP())

-- Last N hours
AND TIMESTAMP > DATEADD('hour', -2, CURRENT_TIMESTAMP())

-- Specific time window
AND TIMESTAMP BETWEEN '2026-01-15 10:00:00' AND '2026-01-15 11:00:00'
```

**Filter by query ID (specific execution):**
```sql
AND RESOURCE_ATTRIBUTES['snow.query.id'] = '<query_id>'
```

**Filter by record type:**
```sql
-- Logs only
AND RECORD_TYPE = 'LOG'

-- Traces only (spans)
AND RECORD_TYPE = 'SPAN'

-- Span events
AND RECORD_TYPE = 'SPAN_EVENT'
```

---

### Step 6: Query for Traces (Execution Flow)

**Goal:** Understand execution flow and timing

```sql
SELECT 
  TIMESTAMP,
  RECORD['name']::STRING AS SPAN_NAME,
  RECORD['status']['code']::STRING AS STATUS_CODE,
  RESOURCE_ATTRIBUTES['snow.executable.name']::STRING AS EXECUTABLE_NAME,
  RESOURCE_ATTRIBUTES['snow.query.id']::STRING AS QUERY_ID,
  RECORD_ATTRIBUTES
FROM SNOWFLAKE.TELEMETRY.EVENTS
WHERE RECORD_TYPE = 'SPAN'
  AND RESOURCE_ATTRIBUTES['snow.executable.name'] ILIKE '%<udf_name>%'
  AND TIMESTAMP > DATEADD('hour', -<N>, CURRENT_TIMESTAMP())
ORDER BY TIMESTAMP DESC
LIMIT 50;
```

---

### Step 7: Analyze Error Patterns

**Goal:** Identify root cause from log patterns

**Load reference:** [../references/error-patterns.md](../references/error-patterns.md)

**Common patterns to look for:**

| Pattern | Indicator | Likely Cause |
|---------|-----------|--------------|
| `MemoryError` | `exception.type` contains "Memory" | Data too large for worker |
| `ModuleNotFoundError` | `exception.type` contains "ModuleNotFound" | Missing package |
| `ImportError` | `exception.type` contains "Import" | Package version issue |
| `TimeoutError` | `exception.type` contains "Timeout" | Long-running operation |
| `TypeError` | `exception.type` contains "Type" | Input/output type mismatch |
| `KeyError` / `IndexError` | `exception.type` contains "Key" or "Index" | Data access issue |
| `PermissionError` | `exception.message` contains "permission" | Access control issue |

**Query to find exception types:**
```sql
SELECT 
  RECORD_ATTRIBUTES['exception.type']::STRING AS EXCEPTION_TYPE,
  RECORD_ATTRIBUTES['exception.message']::STRING AS EXCEPTION_MESSAGE,
  COUNT(*) AS OCCURRENCE_COUNT
FROM SNOWFLAKE.TELEMETRY.EVENTS
WHERE RESOURCE_ATTRIBUTES['snow.executable.name'] ILIKE '%<udf_name>%'
  AND RECORD_ATTRIBUTES['exception.type'] IS NOT NULL
  AND TIMESTAMP > DATEADD('hour', -24, CURRENT_TIMESTAMP())
GROUP BY 1, 2
ORDER BY OCCURRENCE_COUNT DESC
LIMIT 20;
```

---

### Step 8: Suggest Fixes Based on Patterns

**Goal:** Provide actionable recommendations

Based on error analysis, suggest fixes:

#### Memory Errors

**Symptoms:** `MemoryError`, `Out of memory`, process killed

**Suggestions:**
1. **Reduce batch size** - Process data in smaller chunks
2. **Use larger warehouse** - More memory available
3. **Optimize data types** - Use efficient types (INT vs FLOAT)
4. **Stream processing** - Process rows individually vs loading all

```python
# Instead of loading all data at once
# df = session.table("large_table").to_pandas()

# Process in batches
for batch in session.table("large_table").to_pandas_batches():
    process_batch(batch)
```

#### Import/Module Errors

**Symptoms:** `ModuleNotFoundError`, `ImportError`

**Suggestions:**
1. **Check package availability:**
   ```sql
   -- List available packages
   SELECT * FROM INFORMATION_SCHEMA.PACKAGES WHERE LANGUAGE = 'python';
   ```
2. **Add package to UDF definition:**
   ```sql
   CREATE OR REPLACE FUNCTION my_udf(...)
     RETURNS ...
     LANGUAGE PYTHON
     PACKAGES = ('numpy', 'pandas', 'scikit-learn==1.2.0')
     ...
   ```
3. **Use stage imports for custom modules**

#### Timeout Errors

**Symptoms:** `TimeoutError`, execution exceeds limit

**Suggestions:**
1. **Optimize algorithm** - Reduce complexity
2. **Use vectorized operations** - Avoid row-by-row processing
3. **Increase statement timeout:**
   ```sql
   ALTER SESSION SET STATEMENT_TIMEOUT_IN_SECONDS = 3600;
   ```

#### Type Errors

**Symptoms:** `TypeError`, unexpected type

**Suggestions:**
1. **Check input types** match function signature
2. **Handle NULL values explicitly**
3. **Validate data before processing**

```python
def my_handler(input_val):
    if input_val is None:
        return None  # Handle NULL
    # Process non-null value
```

---

### Step 9: Help Add Logging and Telemetry (If Insufficient Data)

**Goal:** Guide user to add logging and telemetry to their UDF

If no logs appear for the UDF, it likely doesn't have logging statements or the telemetry package.

**Load reference:** [../references/python-logging.md](../references/python-logging.md)

#### Check for Missing Telemetry Package

**First, check if the UDF includes the telemetry package:**
```sql
-- Show UDF definition to check PACKAGES
SHOW USER FUNCTIONS LIKE '<function_name>' IN SCHEMA <database>.<schema>;

-- Or describe for full details
DESCRIBE FUNCTION <db.schema.fn>(<types>);
```

**Look for `snowflake-telemetry-python` in the PACKAGES list and `from snowflake import telemetry` in the code.**

#### Always Prompt: Add Telemetry Package for Auto-Instrumentation

If the package or import is missing, **always prompt the user to add them**. This enables auto-instrumentation:

```sql
CREATE OR REPLACE FUNCTION my_udf(input_val VARCHAR)
RETURNS VARCHAR
LANGUAGE PYTHON
PACKAGES = ('snowflake-telemetry-python')  -- ADD THIS for auto-instrumentation
RUNTIME_VERSION = 3.13
HANDLER = 'handler'
AS $$
from snowflake import telemetry  # ADD THIS import

def handler(input_val):
    # Snowflake auto-instruments this execution
    return input_val.upper() if input_val else None
$$;
```

**What auto-instrumentation provides (no extra code needed):**
- Snowflake creates a span (`snow.auto_instrumented`) for each execution
- Captures start/end timestamps, execution status, and query context
- Links traces to query IDs for correlation

#### Adding Logging for Debug Messages

For human-readable log messages, add the logging module:

```python
import logging
from snowflake import telemetry

logger = logging.getLogger('my_udf_logger')

def handler(input_val):
    logger.info(f"Processing input: {input_val}")
    
    try:
        result = process(input_val)
        logger.debug(f"Result: {result}")
        return result
    except Exception as e:
        logger.error(f"Error processing: {e}")
        raise
```

#### Custom Span Attributes and Events (Only When Needed)

**Only suggest custom traces when auto-instrumentation isn't sufficient.** Use cases:
- Need to break down the execution into smaller portions for analysis
- Need searchable metadata (e.g., customer ID, order ID) in traces
- Need to mark specific milestones in complex operations
- Isolating computation-heavy actions (e.g., ML model training)

**Ask user:** "Do you need more granular visibility into specific parts of the execution, or is basic auto-instrumentation sufficient?"

**If custom traces are needed:**
```python
from snowflake import telemetry

def handler(input_val):
    # Add searchable attributes
    telemetry.set_span_attribute("input.type", type(input_val).__name__)
    
    # Mark milestones
    telemetry.add_event("processing_started", {"input": str(input_val)})
    
    result = complex_operation(input_val)
    
    telemetry.add_event("processing_completed", {"status": "success"})
    return result
```

> **Note**: There is a limit of 128 events and 128 span attributes per execution unit.

#### Enable Telemetry Levels

**Ensure TRACE_LEVEL is enabled to capture spans:**
```sql
-- Enable tracing for spans (required for auto-instrumentation to appear in event table)
ALTER FUNCTION <db.schema.fn>(<types>) SET TRACE_LEVEL = 'ALWAYS';

-- Enable logging for log messages
ALTER FUNCTION <db.schema.fn>(<types>) SET LOG_LEVEL = 'DEBUG';
```

**⚠️ STOPPING POINT**: Ask user before modifying UDF TRACE_LEVEL or LOG_LEVEL.

---

### Step 10: Correlate with Query History

**Goal:** Link logs to specific query executions

```sql
-- Get query details for a specific query_id from logs
SELECT 
  query_id,
  query_text,
  start_time,
  end_time,
  total_elapsed_time,
  error_code,
  error_message,
  warehouse_name,
  warehouse_size
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE query_id = '<query_id_from_logs>'
LIMIT 1;
```

**Find queries that executed the UDF:**
```sql
SELECT 
  query_id,
  start_time,
  total_elapsed_time,
  error_message,
  query_text
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE query_text ILIKE '%<udf_name>%'
  AND start_time > DATEADD('hour', -24, CURRENT_TIMESTAMP())
ORDER BY start_time DESC
LIMIT 20;
```

---

## Stopping Points Summary

1. ✋ When telemetry is disabled at account level - present options (account vs object level)
2. ✋ **MANDATORY**: When no telemetry data found - must enable collection before proceeding
3. ✋ After presenting initial log query results (before filtering)
4. ✋ Before suggesting fixes that require code changes
5. ✋ Before modifying UDF LOG_LEVEL or TRACE_LEVEL
6. ✋ If full setup is needed, before routing to setup workflow

---

## Output

- Log entries from UDF executions with timestamps and severity
- Error pattern analysis with identified root causes
- Actionable fix suggestions based on error types
- Code examples for adding logging if needed
- Correlation with query execution history
