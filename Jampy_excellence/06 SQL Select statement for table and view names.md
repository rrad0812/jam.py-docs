
# SQL Select expression for db table and view names

Previews SELECT statements for table and view names in the most common databases

| DBMS | Inquiry |
| ---- | ------- |
| SQLite | sqlite_master |
| MySQL | information_schema.tables |
| PostgreSQL | information_schema.tables ili pg_catalog |
| SQL Server | information_schema.tables ili sys.objects |
| Oracle | ALL_OBJECTS / USER_OBJECTS |

## 1 SQLite

**All tables and views**:

```sql
SELECT
    name,
    type
FROM
    sqlite_master
WHERE
    type IN (
        'table',
        'view'
    )
ORDER BY
    type,
    name;
```

**If you don't want internal SQLite tables**:

```sql
SELECT 
    name, 
    type
FROM 
    sqlite_master
WHERE 
    type IN (
        'table', 
        'view'
    ) AND 
    name NOT LIKE 
        'sqlite_%'
ORDER BY 
    type, 
    name;
```

## 2 MySQL

```sql
SELECT 
    table_schema,
    table_name,
    table_type
FROM 
    information_schema.tables
WHERE 
    table_schema NOT IN (
        'information_schema',      
        'mysql',                     
        'performance_schema',
        'sys'
    ) AND 
    table_type IN (
        'BASE TABLE', 
        'VIEW'
    )
ORDER BY 
    table_schema, 
    table_name;
```

## 3 PostgreSQL

**Via information_schema**:

```sql
SELECT 
    table_schema,
    table_name,
    table_type
FROM 
    information_schema.tables
WHERE 
    table_schema NOT IN (
        'pg_catalog', 
        'information_schema'
    ) AND 
    table_type IN (
        'BASE TABLE', 
        'VIEW'
    )
ORDER BY 
    table_schema, 
    table_name;
```

**Via pg_catalog**:

```sql
SELECT 
    n.nspname AS schema_name,
    c.relname AS object_name,
    CASE c.relkind
        WHEN 'r' THEN 'TABLE'
        WHEN 'v' THEN 'VIEW'
        WHEN 'm' THEN 'MATERIALIZED VIEW'
    END AS object_type
FROM 
    pg_class c
JOIN 
    pg_namespace n ON 
        n.oid = c.relnamespace
WHERE 
    c.relkind IN (
        'r',
        'v',
        'm'
    ) AND 
    n.nspname NOT IN (
        'pg_catalog',
        'information_schema'
    )
ORDER BY 
    n.nspname, 
    c.relname;
```

## 4 Microsoft SQL Server

**Via INFORMATION_SCHEMA**:

```sql
SELECT 
    TABLE_SCHEMA,
    TABLE_NAME,
    TABLE_TYPE
FROM 
    INFORMATION_SCHEMA.TABLES
WHERE 
    TABLE_TYPE IN (
        'BASE TABLE', 
        'VIEW'
    )
ORDER BY 
    TABLE_SCHEMA, 
    TABLE_NAME;
```

**Via sys objects and sys schemas**:

```sql
SELECT 
    s.name AS schema_name,
    o.name AS object_name,
    o.type_desc AS object_type
FROM 
    sys.objects o
JOIN 
    sys.schemas s ON 
        o.schema_id = s.schema_id
WHERE 
    o.type IN (
        'U', 
        'V'
    )
ORDER BY 
    s.name, 
    o.name;
```

Where is it:

```sql
U = User Table
V = View
```

## 5 Oracle

**Only objects of the current user**:

```sql
SELECT 
    object_name,
    object_type
FROM 
    user_objects
WHERE 
    object_type IN (
        'TABLE', 
        'VIEW'
    )
ORDER BY 
    object_type, 
    object_name;
```

**All objects to which the user has access**:

```sql
SELECT 
    owner,
    object_name,
    object_type
FROM 
    all_objects
WHERE 
    object_type IN (
        'TABLE', 
        'VIEW'
    )
ORDER BY 
    owner, 
    object_name;
```

**If you have DBA privileges**:

```sql
SELECT 
    owner,
    object_name,
    object_type
FROM 
    dba_objects
WHERE 
    object_type IN (
        'TABLE', 
        'VIEW'
    )
ORDER BY 
    owner, 
    object_name;
```

## The most portable solution

If you want SQL to be as similar as possible between databases, then:

| DBMS | System view |
| ---- | ----------- |
| MySQL | information_schema.tables |
| PostgreSQL | information_schema.tables |
| SQL Server | information_schema.tables |
| Oracle | all_objects or user_objects |
| SQLite | sqlite_master |

Practically, SQLite is the only "exception".

For MySQL, PostgreSQL, and SQL Server, you can use an almost identical query over `information_schema.tables`.
