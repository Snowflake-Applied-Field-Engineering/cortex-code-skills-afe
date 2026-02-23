# Adding Logging and Telemetry to Python UDFs

This reference shows how to add logging statements and OpenTelemetry-based tracing to Python UDFs that will be captured in Snowflake event tables.

## Telemetry Package and Auto-Instrumentation

**Always include** the `snowflake-telemetry-python` package and import to enable **auto-instrumentation**:

```sql
CREATE OR REPLACE FUNCTION my_udf(input VARCHAR)
RETURNS VARCHAR
LANGUAGE PYTHON
PACKAGES = ('snowflake-telemetry-python')  -- Enables auto-instrumentation
RUNTIME_VERSION = 3.13
HANDLER = 'run'
AS $$
from snowflake import telemetry  # ALWAYS include this import

def run(input):
    # Your code here - Snowflake auto-instruments execution
    return input.upper()
$$;
```

**What auto-instrumentation provides:**
- Snowflake automatically creates a span (`snow.auto_instrumented`) for each execution
- Captures start/end timestamps, execution status, and query context
- No additional code needed beyond the package and import

> **Important**: If `snowflake-telemetry-python` is not in PACKAGES, the `from snowflake import telemetry` import will fail with `ModuleNotFoundError`.

> **Note**: By default, `snowflake-telemetry-python` is included automatically. However, if you have a package policy that explicitly allows/disallows packages, you must add it to the PACKAGES clause.

## Basic Logging Setup

Python UDFs use the standard `logging` module. Messages are automatically captured when telemetry is enabled.

```python
import logging

# Create a logger with a descriptive name
logger = logging.getLogger('my_udf_name')

def handler(input_value):
    logger.info(f"Starting processing for input: {input_value}")
    
    # Your logic here
    result = process(input_value)
    
    logger.info(f"Completed processing, result: {result}")
    return result
```

## Log Levels

Use appropriate log levels for different messages:

| Level | Method | Use For |
|-------|--------|---------|
| DEBUG | `logger.debug()` | Detailed debugging info, variable values |
| INFO | `logger.info()` | Normal operation milestones |
| WARNING | `logger.warning()` | Unexpected but handled situations |
| ERROR | `logger.error()` | Errors that don't crash the function |
| CRITICAL | `logger.critical()` | Severe errors, about to fail |

**Example with multiple levels:**
```python
import logging

logger = logging.getLogger('data_processor')

def handler(data):
    logger.debug(f"Received data type: {type(data)}")
    
    if data is None:
        logger.warning("Received NULL input, returning default")
        return "default"
    
    try:
        result = transform(data)
        logger.info(f"Successfully transformed {len(data)} items")
        return result
    except ValueError as e:
        logger.error(f"Validation failed: {e}")
        raise
    except Exception as e:
        logger.critical(f"Unexpected error: {e}")
        raise
```

## Complete UDF Example

```sql
CREATE OR REPLACE FUNCTION process_json(input_json VARIANT)
RETURNS VARIANT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.10'
PACKAGES = ('snowflake-snowpark-python')
HANDLER = 'main'
AS
$$
import logging
import json

logger = logging.getLogger('process_json_udf')

def main(input_json):
    logger.info("Starting JSON processing")
    logger.debug(f"Input keys: {list(input_json.keys()) if input_json else 'None'}")
    
    if not input_json:
        logger.warning("Empty input received")
        return {}
    
    try:
        # Process the JSON
        result = {
            "processed": True,
            "item_count": len(input_json),
            "keys": list(input_json.keys())
        }
        
        logger.info(f"Processed {len(input_json)} items successfully")
        return result
        
    except Exception as e:
        logger.error(f"Processing failed: {str(e)}")
        raise
$$;

-- Set log level to capture DEBUG messages
ALTER FUNCTION process_json(VARIANT) SET LOG_LEVEL = 'DEBUG';
```

## Logging with Context

Add structured context to your logs:

```python
import logging

logger = logging.getLogger('batch_processor')

def handler(batch_id, items):
    # Include context in messages
    logger.info(f"[batch={batch_id}] Starting processing of {len(items)} items")
    
    for i, item in enumerate(items):
        logger.debug(f"[batch={batch_id}][item={i}] Processing: {item}")
        process_item(item)
    
    logger.info(f"[batch={batch_id}] Completed")
    return len(items)
```

## Logging Exceptions with Stack Traces

```python
import logging
import traceback

logger = logging.getLogger('safe_processor')

def handler(value):
    try:
        return risky_operation(value)
    except Exception as e:
        # Log with full stack trace
        logger.error(f"Operation failed: {e}\n{traceback.format_exc()}")
        raise
```

## Setting LOG_LEVEL on UDF

> 📖 **See [observability-parameters.md](observability-parameters.md)** for comprehensive documentation on LOG_LEVEL, METRIC_LEVEL, TRACE_LEVEL, and SQL_TRACE_QUERY_TEXT parameters including values, scopes, and privileges.

The UDF's LOG_LEVEL determines what messages are captured:

```sql
-- Capture all messages (DEBUG and above)
ALTER FUNCTION my_udf(VARCHAR) SET LOG_LEVEL = 'DEBUG';

-- Capture INFO and above (recommended for production)
ALTER FUNCTION my_udf(VARCHAR) SET LOG_LEVEL = 'INFO';

-- Only capture warnings and errors
ALTER FUNCTION my_udf(VARCHAR) SET LOG_LEVEL = 'WARN';

-- Check current setting
SHOW PARAMETERS LIKE 'LOG_LEVEL' IN FUNCTION my_udf(VARCHAR);
```

## Important Notes

1. **Logger name appears in event table** - Use descriptive names to identify your UDF
2. **Don't log sensitive data** - Avoid logging PII, passwords, or secrets
3. **Performance impact** - Excessive DEBUG logging can impact performance
4. **Level hierarchy** - If account LOG_LEVEL is ERROR but UDF is DEBUG, the more verbose (DEBUG) wins
5. **Telemetry latency** - Logs may take a few seconds to appear in the event table

## Querying Your Logs

After adding logging, query the event table:

```sql
SELECT 
  TIMESTAMP,
  RECORD['severity_text']::STRING AS LEVEL,
  VALUE AS MESSAGE,
  RESOURCE_ATTRIBUTES['snow.query.id']::STRING AS QUERY_ID
FROM SNOWFLAKE.TELEMETRY.EVENTS
WHERE RESOURCE_ATTRIBUTES['snow.executable.name'] = 'PROCESS_JSON'
  AND TIMESTAMP > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY TIMESTAMP DESC;
```

---

## Custom Traces and Events (When Auto-Instrumentation Isn't Enough)

Snowflake's auto-instrumentation (enabled by the telemetry package) provides basic span tracking. **Add custom span attributes and events only when you need to:**
- Break down the auto-instrumented span into smaller, more specific portions
- Add searchable metadata (span attributes) for later analysis
- Mark specific milestones within a complex operation
- Isolate trace data for computation-heavy actions (e.g., ML model training)

> **Rule of thumb**: Start with auto-instrumentation only. Add custom traces when debugging shows you need more granular visibility into specific parts of execution.

### Adding Span Attributes

Span attributes are key-value pairs attached to the current execution span. Use them to add searchable metadata:

```python
from snowflake import telemetry

def handler(customer_id, action):
    # Add attributes that will be searchable in the event table
    telemetry.set_span_attribute("customer.id", customer_id)
    telemetry.set_span_attribute("action.type", action)
    telemetry.set_span_attribute("processing.stage", "validation")
    
    # Your logic here
    result = process(customer_id, action)
    
    telemetry.set_span_attribute("processing.stage", "complete")
    telemetry.set_span_attribute("result.status", "success")
    
    return result
```

**Attribute naming conventions:**
- Use dot notation for namespacing: `example.key`, `customer.id`
- Keep names descriptive but concise
- Use consistent prefixes across your UDFs

### Adding Trace Events

Trace events are timestamped markers within a span. They can have their own attributes:

```python
from snowflake import telemetry

def handler(data):
    telemetry.add_event("processing_started", {
        "input.size": len(data),
        "input.type": type(data).__name__
    })
    
    # Validation step
    if not validate(data):
        telemetry.add_event("validation_failed", {
            "reason": "invalid_format"
        })
        raise ValueError("Validation failed")
    
    telemetry.add_event("validation_passed")
    
    # Processing step
    result = transform(data)
    
    telemetry.add_event("processing_completed", {
        "output.size": len(result),
        "records.processed": count
    })
    
    return result
```

### Complete Example with Logging, Spans, and Events

```sql
CREATE OR REPLACE FUNCTION process_order(order_id VARCHAR, items VARIANT)
RETURNS VARCHAR
LANGUAGE PYTHON
PACKAGES = ('snowflake-telemetry-python')
RUNTIME_VERSION = 3.13
HANDLER = 'run'
AS $$
import logging
from snowflake import telemetry

logger = logging.getLogger("process_order")

def run(order_id, items):
    # Start with span attributes for context
    telemetry.set_span_attribute("order.id", order_id)
    telemetry.set_span_attribute("order.item_count", len(items) if items else 0)
    
    logger.info(f"Processing order {order_id}")
    telemetry.add_event("order_processing_started", {
        "order.id": order_id,
        "item.count": len(items) if items else 0
    })
    
    if not items:
        logger.warning("Empty order received")
        telemetry.add_event("empty_order_received")
        return "EMPTY"
    
    try:
        # Process each item
        total = 0
        for idx, item in enumerate(items):
            telemetry.add_event("item_processing", {
                "item.index": idx,
                "item.sku": item.get("sku", "unknown")
            })
            total += item.get("price", 0) * item.get("quantity", 1)
        
        telemetry.set_span_attribute("order.total", total)
        telemetry.add_event("order_completed", {
            "order.total": total,
            "status": "success"
        })
        
        logger.info(f"Order {order_id} completed, total: {total}")
        return "SUCCESS"
        
    except Exception as e:
        logger.error(f"Order processing failed: {e}")
        telemetry.add_event("order_failed", {
            "error.type": type(e).__name__,
            "error.message": str(e)
        })
        raise
$$;

-- Enable tracing to capture spans and events
ALTER FUNCTION process_order(VARCHAR, VARIANT) SET TRACE_LEVEL = 'ALWAYS';
ALTER FUNCTION process_order(VARCHAR, VARIANT) SET LOG_LEVEL = 'DEBUG';
```

### Querying Span Attributes and Events

**Query span attributes:**
```sql
SELECT 
  TIMESTAMP,
  RECORD['name']::STRING AS SPAN_NAME,
  RECORD_ATTRIBUTES AS SPAN_ATTRIBUTES,
  RESOURCE_ATTRIBUTES['snow.query.id']::STRING AS QUERY_ID
FROM SNOWFLAKE.TELEMETRY.EVENTS
WHERE RECORD_TYPE = 'SPAN'
  AND RESOURCE_ATTRIBUTES['snow.executable.name'] = 'PROCESS_ORDER'
  AND TIMESTAMP > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY TIMESTAMP DESC;
```

**Query trace events:**
```sql
SELECT 
  TIMESTAMP,
  RECORD['name']::STRING AS EVENT_NAME,
  RECORD_ATTRIBUTES AS EVENT_ATTRIBUTES,
  RESOURCE_ATTRIBUTES['snow.query.id']::STRING AS QUERY_ID
FROM SNOWFLAKE.TELEMETRY.EVENTS
WHERE RECORD_TYPE = 'SPAN_EVENT'
  AND RESOURCE_ATTRIBUTES['snow.executable.name'] = 'PROCESS_ORDER'
  AND TIMESTAMP > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY TIMESTAMP DESC;
```

**Find spans with specific attribute values:**
```sql
SELECT *
FROM SNOWFLAKE.TELEMETRY.EVENTS
WHERE RECORD_TYPE = 'SPAN'
  AND RECORD_ATTRIBUTES['order.id']::STRING = 'ORD-12345'
  AND TIMESTAMP > DATEADD('day', -1, CURRENT_TIMESTAMP());
```

### When to Use Each Approach

| Approach | Use For | When to Add |
|----------|---------|-------------|
| Package + import only | Auto-instrumentation (spans, timing) | **Always** - enables basic tracing |
| `logging` | Human-readable progress/error messages | When you need debug/info messages |
| `set_span_attribute()` | Searchable metadata about execution | When you need to query by specific values |
| `add_event()` | Timestamped milestones | When you need to see timing of specific steps |

### Quick Reference: Telemetry Checklist

**Always do (enables auto-instrumentation):**
1. Add package: `PACKAGES = ('snowflake-telemetry-python')`
2. Add import: `from snowflake import telemetry`
3. Enable TRACE_LEVEL: `ALTER FUNCTION ... SET TRACE_LEVEL = 'ALWAYS'`

**Add only when needed (custom instrumentation):**
4. Add span attributes for searchable context → when you need to query traces by specific values
5. Add events for key milestones → when you need to see timing of specific operations within the auto-instrumented span
