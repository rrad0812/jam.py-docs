# `Jam.py V5` - PostgreSQL Boolean support

## Overview

This document describes changes to three files of the Jam.py V5 framework

- sql.py,
- server_classes.py and
- dataset.py

that resolve a compatibility issue with the PostgreSQL database regarding the `boolean` data type.

### Symptoms

PostgreSQL strictly checks data types and throws an error when attempting to assign a boolean value ( `False` ) to an `Integer` or `Text` column and vice versa:

```sh
ERROR - column "mesta_troska_id" is of type integer but expression is of type boolean
LINE 1: ...E "masine" SET "aktivna"=false, "mesta_troska_id"=false, "na...
                                                             ^
HINT:  You will need to rewrite or cast the expression.
```

### Causes

- **`Value reuse bug`**: The variable `value` was not initialized for each field, which allowed the boolean value from one field to "leak" into the next

- **`False as a placeholder`**: Application code uses Python `False` as a placeholder for an empty value

- **`Inconsistent boolean conversion`**: Different value types (`int`, `float`, `string`) were not being consistently converted to boolean for PostgreSQL

### Code changes

- **jam/sql.py**

  - **Adding the `@staticmethod` decorator to the `_to_bool()` method**
  
    ```py
    @staticmethod
    def _to_bool(value):
        if value is None:
            return None
        if isinstance(value, bool):
            return value
        if isinstance(value, (int, float)):
            return False if value == 0 else True
        if isinstance(value, str):
            v = value.strip().lower()
            if v in ('1', 'true', 't', 'yes', 'y', 'on'):
                return True
            if v in ('0', 'false', 'f', 'no', 'n', 'off', ''):
                return False
            try:
                return float(v) != 0.0
            except Exception:
                return True
        return bool(value)
    ```

    Reason:

    - The method is marked as `@staticmethod` because it does not depend on the state of the instance.
    - Can be used as a helper function outside the context of an instance (e.g. in server_classes.py)
    - Normalizes different value types (`int`, `float`, `string`, `bool`) to Python `boolean`

  - **`insert_sql()` method**
  
    ```py
    # if a non-boolean field got a Python False convert it to None 
    # so we don't assign a boolean to an integer/text column
    
    _val = field.data
    if _val is False and field.data_type != consts.BOOLEAN:
        _val = None
    value = (_val, field.data_type)
                    
    if field.data is False and field.data_type != consts.BOOLEAN:
        value = (None, field.data_type)
    
    if field.data_type == consts.BOOLEAN:
        if db_module.DATABASE == 'POSTGRESQL':
            normalized = self._to_bool(field.data)
            value = (normalized, field.data_type)
        else:
            value = (field.data, field.data_type)
    ```

    Key points:

    - **Per-field initialization**: **value** initializes for each field separately
    - **False → None conversion**: If it is `field.data` Python `False` but the field type is not `consts.BOOLEAN`, it is converted to `None` (SQL `NULL`)
    - **PostgreSQL boolean normalization**: For boolean fields in PostgreSQL, use `_to_bool()` the correct value
    - **deleted_flag override**: `deleted_flag` field MUST be 0 (`INTEGER`), not `None` and not `False`

  - **`update_sql()` method**

    **Implementation**: Identical structure as in `insert_sql()` — ensures consistency between INSERT and  UPDATE operations.

- **jam/server_classes.py**

  - **Boolean conversion in a `copy_database()` method**

    Location:

    Inside the function `copy_rows()` (line ~955)

    ```py
    elif field.data_type == consts.BOOLEAN:
        # Patch for PostgreSQL and boolean type of fields
        if db_module.DATABASE == 'POSTGRESQL':
            r[j] = SQL._to_bool(r[j])
        elif r[j]:
            r[j] = 1
        else:
            r[j] = 0
    ```

    Reason:

    When copying a database between different DB systems (e.g. MySQL → PostgreSQL) used SQL `_to_bool()` for PostgreSQL to ensure correct boolean value. For other databases (MySQL, SQLite) converts to integer (1/0).

    Context:

    This function is called when the administrator copies data from one database to another (different DB types).

- **jam/dataset.py**

  **Smart boolean conversion in a `set_value()` method**

  ```py
  if self.data_type == consts.BOOLEAN:
      
      # Use Python booleans for PostgreSQL, integers for other DBs
      is_postgres = False
      try:
          if self.owner and hasattr(self.owner, 'task') and self.owner.task:
              db_module = self.owner.task.db_module
              if db_module and hasattr(db_module, 'DATABASE'):
                  is_postgres = db_module.DATABASE == 'POSTGRESQL'
      except Exception:
          pass
      if is_postgres:
          self.new_value = True if bool(value) else False
      else:
          self.new_value = 1 if bool(value) else 0
  ```

  Reason:

  - Dynamically detects database type
    - For PostgreSQL: use Python boolean ( `True/ False` )
    - For other bases: use integer ( `1/ 0` )
  - Safe fallback with `try-except` block if db_module not available
  
  Context:
  
  This method is called every time the value of a boolean field is set through application code.

### Technical details of the solution

#### Problem 1: Value Reuse Bug

Before the code change value of field wasn't initialized every time:

```py
if field.data_type == consts.BOOLEAN:
    value = (...)  # value set up only for boolean
```

If the field is not boolean, the value remains with the value from the previous iteration!

```py
row.append(value)  # BUG: old value is re-use
```

After the code change value setup initialized on begin allways

```py
_val = field.data
if _val is False and field.data_type != consts.BOOLEAN:
    _val = None
value = (_val, field.data_type)  # default value for all fields

if field.data_type == consts.BOOLEAN:
    # override samo za boolean
    ...
row.append(value)  # Now, any field got value
```

#### Problem 2: `False` as a marker for an `empty` value

Situation:

Application code often sets `False` as a marker for an `empty` field:

```py
item.field_integer.data = False  # "tag for none of value"
```

Solution:

Explicit conversion `False → None` for non-boolean fields:

```py
if _val is False and field.data_type != consts.BOOLEAN:
    _val = None  # SQL NULL
```

#### Problem 3: PostgreSQL strict typing

PostgreSQL expectations:

- INTEGER column: accepts only `integer` or `NULL`
- BOOLEAN column: accepts only `True / False / NULL`

PostgreSQL will not accept `boolean` for `integer` column.

Solution:

- Boolean fields in PostgreSQL get Python `True`/`False`
- Boolean fields in other databases get `1`/`0`
- Non-boolean fields never get `False`, but `None` (`NULL`)

### Testing

**Test 1**: INSERT with an empty integer field

```py
item.insert()
item.field_integer.data = False  # marker za "prazno"
item.field_boolean.data = False   # stvarni boolean
item.post()
item.apply()

# Expected:
# field_integer → NULL in DB
# field_boolean → False (0 or false depedent of DB type)
```

**Test 2**: UPDATE an existing record

```py
item.edit()
item.field_integer.data = False
item.post()
item.apply()

# Expected:
# - field_integer → NULL in DB
```

**Test 3**: `deleted_flag` polje

```py
item.edit()

# deleted_flag internally set up to 0
item.post()
item.apply()

# Expected:
# - deleted_flag → 0 in DB (but not NULL)
```

**Test 4**: `copy_database` (MySQL → PostgreSQL)

```py
task.copy_database('POSTGRESQL', database='target_db', ...)


# Expected:
# Boolean values ​​from MySQL (0/1) converted to PostgreSQL (false/true)
# No type mismatch errors
```

## Compatibility

| Database | Status | Remark |
| -------- | ------ | ------ |
| PostgreSQL | Full support | All changes are designed for PostgreSQL strict typing |
| SQLite | Compatible | Use integer (0/1) for boolean |
| MySQL | Compatible | Use integer (0/1) for boolean |
| MSSQL | Compatible | Use integer (0/1) for boolean |
| Firebird | Testing required | Theoretically compatible |
| Oracle | Testing required | Theoretically compatible |

## Recommendations for further work

1. **Unit tests**  
   Create automated tests that check:
   - False → NULL conversion for non-boolean fields
   - Boolean normalization for PostgreSQL
   - deleted_flag = 0 logic
   - Copy database between different DB types

2. **Code review**
   - Check if there are similar situations with value reuse in other methods
   - Audit of all places where it is used `field.data = False`

3. **Documentation for developers**
   Add to developer guide:
   - Don't set up `False` for non-boolean fields

       ```py
       item.field_integer.data = False  
       ```

     - Use `None` for empty value

       ```py
       item.field_integer.data = None
       ```

4. **Regression tests**

Test on a production copy of the database

- **Check** all INSERT/UPDATE/DELETE operations
- **Validate** `copy_database` functionality

## Conclusion

These changes fix a critical bug that prevented Jam.py from being used with a PostgreSQL database boolean fields.

The solution is:

- **Backward compatible** - existing code for other databases continues to work
- **Forward compatible** - allows the use of PostgreSQL
- **Minimal changes** - only three files, clearly marked sections
- **Safe fallback** - try-except blocks prevent runtime errors

| Author | Date | Version |
| ------ | ---- | :-----: |
| Radosav | November 12, 2025 | 1.0 |
