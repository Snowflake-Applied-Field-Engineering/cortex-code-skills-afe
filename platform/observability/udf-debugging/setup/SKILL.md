---
name: udf-debugging-setup
description: "Set up event table and telemetry levels for UDF debugging"
parent_skill: udf-debugging
---

# Event Table Setup for UDF Debugging

This workflow helps detect existing event table configuration, set up telemetry collection if needed, and ensure UDFs are properly configured for logging.

## When to Load

Main skill routes here when user wants to:
- Enable event table logging
- Configure telemetry levels
- Check if logging is set up correctly
- Troubleshoot missing logs

---

## Workflow

### Step 1: Detect Current Event Table Configuration

**Goal:** Understand what event table configuration exists

**Actions:**

1. **Check account-level event table**:
   ```sql
   SHOW PARAMETERS LIKE 'EVENT_TABLE' IN ACCOUNT;
   ```

2. **If user has a specific database in mind, check database-level**:
   ```sql
   SHOW PARAMETERS LIKE 'EVENT_TABLE' IN DATABASE <database_name>;
   ```

3. **Explain precedence to user**:
   > Event table precedence: **Database > Account**
   >
   > If a database has its own event table, objects in that database write there.
   > Otherwise, they write to the account-level event table.

**Present findings**:
- Account event table: `<value or "Not configured">`
- Database event table (if checked): `<value or "Not configured">`
- Effective event table for target UDF: `<determined value>`

---

### Step 2: Check Telemetry Levels

**Goal:** Verify that telemetry collection is enabled at appropriate levels

**Load reference:** [../references/observability-parameters.md](../references/observability-parameters.md)

**Actions:**

1. **Check account-level telemetry settings**:
   ```sql
   SHOW PARAMETERS LIKE 'LOG_LEVEL' IN ACCOUNT;
   SHOW PARAMETERS LIKE 'METRIC_LEVEL' IN ACCOUNT;
   SHOW PARAMETERS LIKE 'TRACE_LEVEL' IN ACCOUNT;
   ```

2. **If user has a target UDF, check its specific settings**:
   ```sql
   -- For a function (include parameter types)
   SHOW PARAMETERS LIKE 'LOG_LEVEL' IN FUNCTION <db.schema.fn_name>(<param_types>);
   SHOW PARAMETERS LIKE 'TRACE_LEVEL' IN FUNCTION <db.schema.fn_name>(<param_types>);
   ```

3. **Summarize effective levels**:

   | Level Type | Account | Object (if set) | Effective |
   |------------|---------|-----------------|-----------|
   | LOG_LEVEL | `<val>` | `<val>` | `<most verbose>` |
   | METRIC_LEVEL | `<val>` | `<val>` | `<effective>` |
   | TRACE_LEVEL | `<val>` | `<val>` | `<effective>` |

**Understanding LOG_LEVEL values** (most to least verbose):
- `TRACE` - All messages
- `DEBUG` - Debug and above
- `INFO` - Info and above (recommended default)
- `WARN` - Warnings and errors only
- `ERROR` - Errors only
- `FATAL` - Fatal errors only
- `OFF` - No logging

**Understanding TRACE_LEVEL values:**
- `ALWAYS` - Always capture traces (recommended)
- `ON_EVENT` - Capture only when events are added
- `OFF` - No tracing (default)

---

### Step 3: Determine Setup Needs and Present Options

**Goal:** Decide what configuration changes are needed and present options to user

Based on Steps 1-2, determine the situation:

| Situation | Action |
|-----------|--------|
| No event table configured | Go to Step 4A |
| Event table exists but telemetry levels disabled at account | Go to Step 4B (present options) |
| Account levels enabled but UDF needs more verbose level | Go to Step 4C |
| Everything configured correctly | Go to Step 5 (verification) |

---

### Step 4A: Enable Account-Level Event Collection

**Goal:** Set up telemetry collection using the default event table

**Recommendation:**
> The `SNOWFLAKE.TELEMETRY.EVENTS` event table exists by default in all accounts.
> For most use cases, enabling account-level telemetry with this table is the simplest approach.

**Proposed SQL:**
```sql
-- Enable logging at INFO level (captures INFO, WARN, ERROR, FATAL)
ALTER ACCOUNT SET LOG_LEVEL = 'INFO';

-- Enable all metrics collection
ALTER ACCOUNT SET METRIC_LEVEL = 'ALL';

-- Enable tracing for all executions
ALTER ACCOUNT SET TRACE_LEVEL = 'ALWAYS';
```

**Required privileges:**
- `MODIFY LOG LEVEL ON ACCOUNT`
- `MODIFY METRIC LEVEL ON ACCOUNT`
- `MODIFY TRACE LEVEL ON ACCOUNT`

(Typically requires ACCOUNTADMIN or delegated privileges)

**⚠️ MANDATORY STOPPING POINT**: Present the SQL and ask user to confirm before executing.

**If user lacks permissions or prefers not to change account settings**, go to Step 4B-Fallback.

---

### Step 4B: Telemetry Levels Disabled - Present Options

**Goal:** When telemetry collection is disabled at account level, offer the user a choice

**⚠️ MANDATORY STOPPING POINT**: Present these options to the user:

---

**Telemetry collection is currently disabled at the account level.**

I recommend enabling telemetry at the **account level** as this is the best practice:
- Applies to all UDFs/procedures automatically
- No need to configure each object individually
- Centralized management of telemetry settings

**Option A: Enable at Account Level (Recommended)**

```sql
ALTER ACCOUNT SET LOG_LEVEL = 'INFO';
ALTER ACCOUNT SET METRIC_LEVEL = 'ALL';
ALTER ACCOUNT SET TRACE_LEVEL = 'ALWAYS';
```

Required privileges: `MODIFY LOG LEVEL ON ACCOUNT`, `MODIFY METRIC LEVEL ON ACCOUNT`, `MODIFY TRACE LEVEL ON ACCOUNT` (typically ACCOUNTADMIN)

**Option B: Enable at Object Level Only**

If you don't want to change account-level settings or lack permissions, I can enable telemetry on the specific UDF only:

```sql
ALTER FUNCTION <db.schema.fn_name>(<param_types>) SET LOG_LEVEL = 'DEBUG';
ALTER FUNCTION <db.schema.fn_name>(<param_types>) SET TRACE_LEVEL = 'ALWAYS';
```

Required privileges: `MODIFY` on the function, `USAGE` on database and schema

**Option C: Request Admin to Enable Account-Level**

I can generate a message you can send to your admin requesting account-level telemetry.

---

**Based on user's choice:**
- **Option A** → Execute account-level SQL (with confirmation)
- **Option B** → Go to Step 4B-Fallback
- **Option C** → Go to Step 4D

---

### Step 4B-Fallback: Enable Telemetry at Object Level

**Goal:** Enable telemetry on the specific UDF when account-level is not an option

**When to use:**
- User chose Option B (prefers object-level)
- User lacks account-level permissions
- User wants to limit telemetry to specific objects

**Proposed SQL:**
```sql
-- Enable logging on specific function (DEBUG captures all messages)
ALTER FUNCTION <db.schema.fn_name>(<param_types>) SET LOG_LEVEL = 'DEBUG';

-- Enable tracing on specific function
ALTER FUNCTION <db.schema.fn_name>(<param_types>) SET TRACE_LEVEL = 'ALWAYS';
```

**Required privileges:**
- `MODIFY` on the function
- `USAGE` on the database and schema

**Note:** Object-level settings take effect even when account-level is OFF. When both are set, the more verbose level wins.

**⚠️ MANDATORY STOPPING POINT**: Ask user to confirm before executing.

**Inform user of limitation:**
> With object-level only, you'll need to set these parameters on each UDF you want to debug. Consider requesting account-level enablement from an admin for easier management.

---

### Step 4C: Adjust UDF-Specific Log Level

**Goal:** Ensure the target UDF has an appropriate log level

If the account/database level is restrictive (e.g., ERROR) but user wants more verbose logging for a specific UDF:

**Proposed SQL:**
```sql
-- Set log level on specific function
ALTER FUNCTION <db.schema.fn_name>(<param_types>) SET LOG_LEVEL = 'DEBUG';
```

**Required privileges:**
- `MODIFY` on the function
- `USAGE` on the database and schema

**Note:** Object-level LOG_LEVEL takes precedence over account level. When both session and object levels apply, the more verbose wins.

**⚠️ MANDATORY STOPPING POINT**: Ask user to confirm before executing.

---

### Step 4D: Generate Admin Permission Request

**Goal:** Create a message the user can send to an admin

If user lacks privileges to enable event collection:

**Generate this message:**

---

**Subject: Request to Enable UDF Telemetry Collection**

Hi,

I need to debug UDFs in our Snowflake account and require event table telemetry to be enabled. This will allow me to view logs, traces, and metrics from UDF executions.

**Why this is needed:**
- Debug failing or misbehaving UDFs
- View execution logs and error messages
- Monitor UDF performance with traces and metrics

**Recommended SQL to run (requires ACCOUNTADMIN or delegated privileges):**

```sql
-- Enable telemetry collection at account level
-- This uses the default SNOWFLAKE.TELEMETRY.EVENTS table
ALTER ACCOUNT SET LOG_LEVEL = 'INFO';
ALTER ACCOUNT SET METRIC_LEVEL = 'ALL';
ALTER ACCOUNT SET TRACE_LEVEL = 'ALWAYS';
```

**Alternative: Grant me privileges to manage telemetry levels:**
```sql
GRANT MODIFY LOG LEVEL ON ACCOUNT TO ROLE <my_role>;
GRANT MODIFY METRIC LEVEL ON ACCOUNT TO ROLE <my_role>;
GRANT MODIFY TRACE LEVEL ON ACCOUNT TO ROLE <my_role>;
```

**Documentation:**
- [Event Table Overview](https://docs.snowflake.com/en/developer-guide/logging-tracing/event-table-setting-up)
- [Setting Telemetry Levels](https://docs.snowflake.com/en/developer-guide/logging-tracing/telemetry-levels)

Thank you!

---

**⚠️ STOPPING POINT**: Present this message template to user.

---

### Step 4E: Database-Level Event Table (Only If Explicitly Requested)

**⚠️ WARNING**: Only proceed here if user explicitly requests a database-level event table.

**Explain limitations:**

> **Database-level event tables have important limitations:**
>
> 1. **Replication not supported** - Event tables cannot be replicated. Databases with event tables are skipped during replication.
>
> 2. **Non-database telemetry lost** - Telemetry from actions not owned by a database (e.g., anonymous SQL blocks) can only go to account-level event tables. Without an account-level table, this data is lost.
>
> 3. **Enterprise Edition required** - Database-level event tables require Snowflake Enterprise Edition.
>
> 4. **Increased management overhead** - Must manage event tables per database.
>
> **Recommendation:** Use a single account-level event table unless you have specific governance, isolation, or compliance requirements that cannot be met through views or row-level access policies.

**If user still wants to proceed:**

```sql
-- Create event table in specific database
CREATE EVENT TABLE <db>.<schema>.<event_table_name>;

-- Associate with database
ALTER DATABASE <db> SET EVENT_TABLE = '<db>.<schema>.<event_table_name>';

-- Set telemetry levels for that database
ALTER DATABASE <db> SET LOG_LEVEL = 'INFO';
```

**⚠️ MANDATORY STOPPING POINT**: Confirm user understands limitations before executing.

---

### Step 5: Verify Configuration

**Goal:** Confirm that telemetry collection is working

**Actions:**

1. **Verify settings applied:**
   ```sql
   SHOW PARAMETERS LIKE 'LOG_LEVEL' IN ACCOUNT;
   SHOW PARAMETERS LIKE 'METRIC_LEVEL' IN ACCOUNT;
   SHOW PARAMETERS LIKE 'TRACE_LEVEL' IN ACCOUNT;
   ```

2. **Test with a simple UDF call:**
   ```sql
   -- Execute a UDF that should produce logs
   SELECT <udf_name>(<test_params>);
   ```

3. **Check for logs (may take a few seconds):**
   ```sql
   SELECT *
   FROM SNOWFLAKE.TELEMETRY.EVENTS
   WHERE RESOURCE_ATTRIBUTES['snow.executable.name'] ILIKE '%<udf_name>%'
     AND TIMESTAMP > DATEADD('minute', -5, CURRENT_TIMESTAMP())
   ORDER BY TIMESTAMP DESC
   LIMIT 10;
   ```

**If no logs appear:**
- Verify the UDF contains logging statements (see [python-logging.md](../references/python-logging.md))
- Check that UDF's LOG_LEVEL is not more restrictive than collection level
- Wait a few more seconds (telemetry has slight latency)

---

## Stopping Points Summary

1. ✋ Before executing any `ALTER ACCOUNT SET` command
2. ✋ When telemetry is disabled at account level - present options (account vs object level)
3. ✋ Before executing any `ALTER FUNCTION SET` command
4. ✋ Before executing any `ALTER DATABASE SET` command
5. ✋ When generating admin permission request
6. ✋ When user requests database-level event table (warn about limitations)

---

## Output

- Clear summary of current event table configuration
- Telemetry level status at account and object levels
- SQL scripts for enabling telemetry (with user approval)
- Admin request message if permissions are lacking
- Verification that configuration is working
