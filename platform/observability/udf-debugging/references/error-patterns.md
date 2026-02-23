# Common Snowpark UDF Error Patterns

This reference documents common errors encountered when running Snowpark UDFs and how to fix them.

---

## Memory Errors

### Pattern
```
MemoryError: Unable to allocate...
Python worker process was killed
Out of memory
```

### Causes
- Loading too much data into memory at once
- Large intermediate results
- Inefficient data structures

### Fixes

**1. Process data in batches:**
```python
# Instead of
df = session.table("large_table").to_pandas()  # Loads all into memory

# Use batching
for batch in session.table("large_table").to_pandas_batches():
    process_batch(batch)
```

**2. Use larger warehouse:**
```sql
-- More memory available
ALTER WAREHOUSE my_wh SET WAREHOUSE_SIZE = 'MEDIUM';
```

**3. Optimize data types:**
```python
# Use efficient types
df['id'] = df['id'].astype('int32')  # Instead of int64
df['category'] = df['category'].astype('category')  # For repeated strings
```

**4. Stream results instead of collecting:**
```python
# Instead of collecting all results
results = [process(row) for row in big_list]

# Use generators
def process_stream(items):
    for item in items:
        yield process(item)
```

---

## Import and Module Errors

### Pattern
```
ModuleNotFoundError: No module named 'xyz'
ImportError: cannot import name 'abc' from 'xyz'
```

### Causes
- Package not available in Snowflake
- Wrong package version
- Missing from UDF definition

### Fixes

**1. Check available packages:**
```sql
SELECT * FROM INFORMATION_SCHEMA.PACKAGES 
WHERE LANGUAGE = 'python' 
  AND PACKAGE_NAME ILIKE '%<name>%';
```

**2. Add package to UDF:**
```sql
CREATE OR REPLACE FUNCTION my_udf(...)
RETURNS ...
LANGUAGE PYTHON
PACKAGES = ('numpy', 'pandas==2.0.0', 'scikit-learn')  -- Add needed packages
HANDLER = 'handler'
AS $$ ... $$;
```

**3. For custom modules, use stage imports:**
```sql
CREATE OR REPLACE FUNCTION my_udf(...)
RETURNS ...
LANGUAGE PYTHON
IMPORTS = ('@my_stage/my_module.py')  -- Import from stage
HANDLER = 'my_module.handler'
AS $$ ... $$;
```

---

## Type Errors

### Pattern
```
TypeError: unsupported operand type(s)
TypeError: 'NoneType' object is not subscriptable
TypeError: cannot unpack non-iterable NoneType object
```

### Causes
- NULL values not handled
- Wrong input types
- Type mismatches in operations

### Fixes

**1. Handle NULL values:**
```python
def handler(input_val):
    if input_val is None:
        return None  # Or appropriate default
    
    # Safe to process now
    return process(input_val)
```

**2. Validate and convert types:**
```python
def handler(value):
    # Explicit type conversion
    if isinstance(value, str):
        value = int(value)
    
    return value * 2
```

**3. Use defensive access:**
```python
def handler(data):
    # Safe dictionary access
    result = data.get('key', 'default')
    
    # Safe list access
    if len(items) > 0:
        first = items[0]
```

---

## Timeout Errors

### Pattern
```
TimeoutError
Statement reached its statement or warehouse timeout
Query execution was canceled
```

### Causes
- Long-running computation
- Inefficient algorithms
- External service timeouts

### Fixes

**1. Increase timeout:**
```sql
ALTER SESSION SET STATEMENT_TIMEOUT_IN_SECONDS = 3600;
```

**2. Optimize algorithm:**
```python
# Instead of O(n²)
for i in items:
    for j in items:
        compare(i, j)

# Use O(n) approach
seen = set()
for item in items:
    if item in seen:
        handle_duplicate(item)
    seen.add(item)
```

**3. Use vectorized operations:**
```python
# Instead of row-by-row
results = []
for row in df.iterrows():
    results.append(process(row))

# Use vectorized pandas
results = df.apply(process_vectorized, axis=1)
```

---

## Permission Errors

### Pattern
```
PermissionError
Access denied
Insufficient privileges
```

### Causes
- UDF owner lacks access to underlying objects
- Caller lacks USAGE on UDF
- Network access not allowed

### Fixes

**1. Check UDF owner privileges:**
```sql
-- UDF runs with owner's privileges by default
SHOW GRANTS TO ROLE <udf_owner_role>;
```

**2. Grant necessary access:**
```sql
GRANT USAGE ON DATABASE db TO ROLE udf_owner_role;
GRANT USAGE ON SCHEMA db.schema TO ROLE udf_owner_role;
GRANT SELECT ON TABLE db.schema.table TO ROLE udf_owner_role;
```

**3. For external access, create network rule:**
```sql
CREATE OR REPLACE NETWORK RULE my_rule
  TYPE = HOST_PORT
  MODE = EGRESS
  VALUE_LIST = ('api.example.com:443');

CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION my_integration
  ALLOWED_NETWORK_RULES = (my_rule)
  ENABLED = TRUE;

-- Use in UDF
CREATE OR REPLACE FUNCTION call_api(...)
RETURNS ...
LANGUAGE PYTHON
EXTERNAL_ACCESS_INTEGRATIONS = (my_integration)
...
```

---

## Key/Index Errors

### Pattern
```
KeyError: 'column_name'
IndexError: list index out of range
```

### Causes
- Missing expected data
- Schema changes
- Empty inputs

### Fixes

**1. Check before access:**
```python
def handler(data):
    if 'expected_key' not in data:
        logger.warning("Missing expected_key")
        return None
    
    return data['expected_key']
```

**2. Use safe access methods:**
```python
def handler(data):
    # Dictionary: use .get() with default
    value = data.get('key', 'default')
    
    # List: check length first
    if len(items) > 0:
        first = items[0]
    else:
        first = None
```

---

## Serialization Errors

### Pattern
```
Object of type X is not JSON serializable
Cannot serialize object
```

### Causes
- Returning non-serializable types
- Complex objects (datetime, numpy types)

### Fixes

**1. Convert to serializable types:**
```python
import json
from datetime import datetime
import numpy as np

def handler(data):
    result = process(data)
    
    # Convert datetime
    if isinstance(result, datetime):
        return result.isoformat()
    
    # Convert numpy types
    if isinstance(result, np.integer):
        return int(result)
    if isinstance(result, np.floating):
        return float(result)
    if isinstance(result, np.ndarray):
        return result.tolist()
    
    return result
```

---

## Query Pattern to Find Errors

```sql
-- Find all errors for a UDF in last 24 hours
SELECT 
  TIMESTAMP,
  RECORD_ATTRIBUTES['exception.type']::STRING AS ERROR_TYPE,
  RECORD_ATTRIBUTES['exception.message']::STRING AS ERROR_MESSAGE,
  VALUE AS LOG_MESSAGE,
  RESOURCE_ATTRIBUTES['snow.query.id']::STRING AS QUERY_ID
FROM SNOWFLAKE.TELEMETRY.EVENTS
WHERE RESOURCE_ATTRIBUTES['snow.executable.name'] = '<UDF_NAME>'
  AND RECORD['severity_text'] IN ('ERROR', 'FATAL')
  AND TIMESTAMP > DATEADD('hour', -24, CURRENT_TIMESTAMP())
ORDER BY TIMESTAMP DESC;
```

```sql
-- Group errors by type
SELECT 
  RECORD_ATTRIBUTES['exception.type']::STRING AS ERROR_TYPE,
  COUNT(*) AS COUNT,
  MAX(TIMESTAMP) AS LAST_OCCURRENCE
FROM SNOWFLAKE.TELEMETRY.EVENTS
WHERE RESOURCE_ATTRIBUTES['snow.executable.name'] = '<UDF_NAME>'
  AND RECORD_ATTRIBUTES['exception.type'] IS NOT NULL
  AND TIMESTAMP > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY 1
ORDER BY COUNT DESC;
```
