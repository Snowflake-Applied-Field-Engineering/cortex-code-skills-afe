---
name: udf-debugging
description: "Debug Snowpark UDFs using Snowflake Event Tables. Use for: UDF errors, adding logging to UDFs, querying UDF logs, debugging Python/Java/Scala UDFs, event table setup, custom traces, span attributes, trace events, OpenTelemetry, snowflake-telemetry-python. Triggers: debug UDF, UDF logs, event table, UDF error, snowpark logging, function not working, UDF not returning, trace UDF, add telemetry, custom span, trace event."
---

# UDF Debugging with Event Tables

Expert guidance for debugging Snowpark User-Defined Functions (UDFs) using Snowflake Event Tables. This skill helps you set up telemetry collection, query logs from UDFs, identify common error patterns, and add logging to your code.

## When to Use

Use this skill when users need to:
- Debug a failing or misbehaving UDF
- Set up event table logging for UDFs
- Query logs, metrics, or traces from UDF executions
- Add logging statements to Python/Java/Scala UDFs
- Add custom span attributes or trace events (OpenTelemetry)
- Understand why a UDF is returning unexpected results
- Troubleshoot Snowpark dependency or memory issues

## Prerequisites

- Access to the Snowflake account where the UDF runs
- Appropriate privileges to query event tables or set telemetry levels

---

## Intent Detection

When a user makes a request, detect their intent and route to the appropriate sub-skill:

### SETUP Intent

**Trigger phrases**: "enable event table", "set up logging", "configure telemetry", "enable tracing", "no logs showing", "event table not working", "how to enable logging"

**-> Load**: [setup/SKILL.md](setup/SKILL.md)

### DEBUG Intent

**Trigger phrases**: "debug UDF", "check logs", "UDF failing", "view errors", "query event table", "UDF logs", "why is my UDF", "trace UDF execution", "UDF not working"

**-> Load**: [debug/SKILL.md](debug/SKILL.md)

---

## Workflow Decision Tree

```
User Request
    |
    v
+-------------------+
| Detect Intent     |
+-------------------+
    |
    +---> SETUP (enable logging, configure telemetry)
    |         |
    |         v
    |     Load setup/SKILL.md
    |         - Detect event table config
    |         - Check telemetry levels
    |         - Guide setup if needed
    |
    +---> DEBUG (query logs, troubleshoot UDF)
              |
              v
          Load debug/SKILL.md
              - Verify event table exists
              - Query logs for UDF
              - Analyze errors
              - Suggest fixes
```

---

## Quick Diagnostic Queries

For immediate assessment before routing:

```sql
-- Check if account has event table configured
SHOW PARAMETERS LIKE 'EVENT_TABLE' IN ACCOUNT;

-- Check current telemetry levels
SHOW PARAMETERS LIKE 'LOG_LEVEL' IN ACCOUNT;
SHOW PARAMETERS LIKE 'TRACE_LEVEL' IN ACCOUNT;
SHOW PARAMETERS LIKE 'METRIC_LEVEL' IN ACCOUNT;

-- Quick check for recent UDF logs (if event table exists)
SELECT COUNT(*) as log_count
FROM SNOWFLAKE.TELEMETRY.EVENTS
WHERE RESOURCE_ATTRIBUTES['snow.executable.type'] IN ('FUNCTION', 'PROCEDURE')
  AND TIMESTAMP > DATEADD('hour', -1, CURRENT_TIMESTAMP());
```

---

## ⚠️ Telemetry-First Debugging Principle

**Always check for telemetry data before proceeding with any debugging.**

Even if the error source seems obvious from the code:
- **DO NOT skip telemetry collection** - runtime data reveals bugs not visible in code
- **ALWAYS prompt the user to enable collection** if no data is found
- **Require re-execution** of the UDF to generate telemetry before analyzing

This ensures:
1. Runtime errors are captured (may differ from code inspection)
2. Multiple bugs are caught (not just the obvious one)
3. Users learn effective debugging workflows
4. Future debugging is easier with telemetry baselines

---

## Sub-Skills

| Sub-Skill | Purpose | When to Load |
|-----------|---------|--------------|
| [setup/SKILL.md](setup/SKILL.md) | Event table detection & telemetry setup | SETUP intent |
| [debug/SKILL.md](debug/SKILL.md) | Query logs, analyze errors, suggest fixes | DEBUG intent |

---

## Reference Documents

| Reference | Content |
|-----------|---------|
| [references/observability-parameters.md](references/observability-parameters.md) | LOG_LEVEL, METRIC_LEVEL, TRACE_LEVEL, SQL_TRACE_QUERY_TEXT |
| [references/event-table-columns.md](references/event-table-columns.md) | Event table schema and column descriptions |
| [references/python-logging.md](references/python-logging.md) | Logging, custom traces, span attributes, and events |
| [references/error-patterns.md](references/error-patterns.md) | Common Snowpark errors and fixes |

---

## Key Concepts

### Event Table Precedence

An order of precedence determines which event table collects telemetry:

```
Database Event Table > Account Event Table
```

- If a database has an associated event table, objects in that database write to the database's event table
- Other databases without their own event table write to the account-level event table
- **Recommendation**: Use account-level event tables unless you have specific governance requirements

### Telemetry Level Hierarchy

Levels can be set at multiple scopes and override each other:

**Session parameters**: Account > User > Session
**Object parameters**: Account > Database > Schema > Object (UDF/procedure)

When set at multiple levels, the **more verbose** level wins.

See [references/observability-parameters.md](references/observability-parameters.md) for detailed parameter documentation.

### Default Event Table

All Snowflake accounts include `SNOWFLAKE.TELEMETRY.EVENTS` by default. This is recommended for most use cases.

### Telemetry Package for Auto-Instrumentation

**Always prompt users to add** the `snowflake-telemetry-python` package and import. This enables **auto-instrumentation**:

```sql
CREATE OR REPLACE FUNCTION my_udf(...)
  RETURNS ...
  LANGUAGE PYTHON
  PACKAGES = ('snowflake-telemetry-python')  -- Enables auto-instrumentation
  ...
AS $$
from snowflake import telemetry  # ALWAYS include this import
...
$$;
```

**What auto-instrumentation provides (no extra code needed):**
- Snowflake creates a span (`snow.auto_instrumented`) for each execution
- Captures start/end timestamps, execution status, and query context
- Links traces to query IDs for correlation

### Custom Span Attributes and Events (Only When Needed)

**Only add custom traces when auto-instrumentation isn't sufficient:**
- `telemetry.set_span_attribute(key, value)` - Add searchable metadata
- `telemetry.add_event(name, attributes)` - Add timestamped milestones

**Use cases for custom traces:**
- Breaking down execution into smaller portions for analysis
- Adding searchable metadata (customer ID, order ID) to query later
- Marking specific milestones in complex operations
- Isolating computation-heavy actions (e.g., ML model training)

See [references/python-logging.md](references/python-logging.md) for complete examples.

---

## Stopping Points Summary

All sub-skills follow this philosophy: **NO changes without explicit user approval.**

- **READ-ONLY queries**: Can run freely (diagnostics, log queries)
- **ANY mutation**: Requires stopping point and user approval
  - Enabling telemetry collection
  - Modifying LOG_LEVEL/METRIC_LEVEL/TRACE_LEVEL
  - Altering UDF parameters

---

## Output

- Clear diagnosis of event table configuration
- Actionable debugging suggestions based on log analysis
- Code snippets for adding logging to UDFs
- SQL scripts for enabling telemetry (with user approval)
