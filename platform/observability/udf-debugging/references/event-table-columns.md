# Event Table Columns Reference

Snowflake event tables have a predefined schema. You cannot modify these columns.

## Core Columns

| Column | Type | Description |
|--------|------|-------------|
| `TIMESTAMP` | TIMESTAMP_NTZ | When the event was recorded |
| `START_TIMESTAMP` | TIMESTAMP_NTZ | Start time (for spans) |
| `OBSERVED_TIMESTAMP` | TIMESTAMP_NTZ | When the event was observed |
| `TRACE_ID` | VARCHAR | Trace identifier for distributed tracing |
| `SPAN_ID` | VARCHAR | Span identifier within a trace |
| `PARENT_SPAN_ID` | VARCHAR | Parent span (for nested spans) |
| `RECORD_TYPE` | VARCHAR | Type: LOG, SPAN, SPAN_EVENT, or METRIC |
| `RECORD` | VARIANT | Event-specific data (structure varies by type) |
| `RECORD_ATTRIBUTES` | VARIANT | Additional attributes about the event |
| `RESOURCE_ATTRIBUTES` | VARIANT | Information about the source (UDF, query, etc.) |
| `VALUE` | VARCHAR | Log message text (for LOG records) |
| `EXEMPLARS` | VARCHAR | Metric exemplars |
| `SCOPE` | VARIANT | Instrumentation scope info |

## RESOURCE_ATTRIBUTES Keys

These identify where the telemetry came from:

| Key | Description | Example |
|-----|-------------|---------|
| `snow.account.name` | Account identifier | `"MYACCOUNT"` |
| `snow.database.id` | Database ID | `12345` |
| `snow.database.name` | Database name | `"MY_DATABASE"` |
| `snow.schema.id` | Schema ID | `67890` |
| `snow.schema.name` | Schema name | `"MY_SCHEMA"` |
| `snow.executable.id` | UDF/procedure ID | `111213` |
| `snow.executable.name` | UDF/procedure name | `"MY_UDF"` |
| `snow.executable.type` | Object type | `"FUNCTION"` or `"PROCEDURE"` |
| `snow.query.id` | Query ID that invoked the UDF | `"01b..."` |
| `snow.session.id` | Session ID | `141516` |
| `snow.session.role.primary.id` | Role ID | `171819` |
| `snow.session.role.primary.name` | Role name | `"MY_ROLE"` |
| `snow.user.id` | User ID | `202122` |
| `snow.user.name` | User name | `"MY_USER"` |
| `snow.warehouse.id` | Warehouse ID | `232425` |
| `snow.warehouse.name` | Warehouse name | `"COMPUTE_WH"` |

## RECORD Structure by Record Type

### LOG Records

```json
{
  "severity_text": "INFO",    // DEBUG, INFO, WARN, ERROR, FATAL
  "severity_number": 9        // Numeric severity
}
```

### SPAN Records (Traces)

```json
{
  "name": "span_name",
  "kind": "SPAN_KIND_INTERNAL",  // INTERNAL, SERVER, CLIENT, PRODUCER, CONSUMER
  "status": {
    "code": "STATUS_CODE_OK"     // OK, ERROR, UNSET
  }
}
```

### METRIC Records

```json
{
  "name": "metric_name",
  "metric": {
    "sum": {...},
    "gauge": {...},
    "histogram": {...}
  }
}
```

## RECORD_ATTRIBUTES Keys

Additional context about the event:

| Key | Description | Present In |
|-----|-------------|------------|
| `code.filepath` | Source file path | LOG, SPAN |
| `code.function` | Function name | LOG, SPAN |
| `code.lineno` | Line number | LOG |
| `code.namespace` | Module/namespace | LOG |
| `exception.type` | Exception class name | LOG (errors) |
| `exception.message` | Exception message | LOG (errors) |
| `exception.stacktrace` | Full stack trace | LOG (errors) |
| `thread.id` | Thread ID | LOG |
| `thread.name` | Thread name | LOG |

## Common Query Patterns

**Get logs with severity and message:**
```sql
SELECT 
  TIMESTAMP,
  RECORD['severity_text']::STRING AS SEVERITY,
  VALUE AS MESSAGE,
  RESOURCE_ATTRIBUTES['snow.executable.name']::STRING AS UDF_NAME
FROM <event_table>
WHERE RECORD_TYPE = 'LOG'
ORDER BY TIMESTAMP DESC;
```

**Get error details:**
```sql
SELECT 
  TIMESTAMP,
  VALUE AS MESSAGE,
  RECORD_ATTRIBUTES['exception.type']::STRING AS ERROR_TYPE,
  RECORD_ATTRIBUTES['exception.message']::STRING AS ERROR_MSG,
  RECORD_ATTRIBUTES['exception.stacktrace']::STRING AS STACKTRACE
FROM <event_table>
WHERE RECORD_TYPE = 'LOG'
  AND RECORD['severity_text'] IN ('ERROR', 'FATAL')
ORDER BY TIMESTAMP DESC;
```

**Get trace spans:**
```sql
SELECT 
  TIMESTAMP,
  START_TIMESTAMP,
  RECORD['name']::STRING AS SPAN_NAME,
  RECORD['status']['code']::STRING AS STATUS,
  TRACE_ID,
  SPAN_ID
FROM <event_table>
WHERE RECORD_TYPE = 'SPAN'
ORDER BY START_TIMESTAMP DESC;
```
