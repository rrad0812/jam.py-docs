
# Update views on databases

Shortly: No, views are not updatable.

All of these DBMSs support updating views to some extent, but the rules differ.

| DBMS | UPDATE/INSERT/DELETE on view |
| ---- | ---------------------------- |
| SQLite | Yes, but practically only with INSTEAD OF trigger |
| MySQL | Yes, if the view is simple enough |
| PostgreSQL | Yes, if the view is automatically update-balanced or with INSTEAD OF trigger/rules |
| SQL Server | Yes, if the view is simple; stacked via INSTEAD OF trigger |
| Oracle | Yes, if the view is simple; stacked via INSTEAD OF trigger |

## 1 SQLite

This is the biggest exception.

Normal view:

```sql
CREATE VIEW v_emp AS
  SELECT id, name
  FROM emp;

UPDATE v_emp 
  SET name='Pera' 
  WHERE id=1;
```

```sql
→ error:
cannot modify v_emp because it is a view
```

To perform an update, you need to do:

```sql
CREATE TRIGGER ...
  INSTEAD OF 
  UPDATE ON v_emp
```

So, without a trigger, the view is not update-ready.

## 2 MySQL

Works for simple views:

```sql
CREATE VIEW v_emp AS
  SELECT id, name
  FROM emp;
```

This works:

```sql
UPDATE v_emp
  SET name='Pera'
  WHERE id=1;
```

But if you have:

```sql
JOIN
GROUP BY
aggregate functions ( SUM, COUNT, …)
DISTINCT
UNION
```

then the view is generally not update-aware.

## 3 PostgreSQL

PostgreSQL has automatically updatable views since version 9.3.

- uses one table
- there is no GROUP BY
- no aggregates
- there is no DISTINCT
- there is no UNION
- there is no JOIN complicated

First:

```sql
CREATE VIEW v_emp AS
  SELECT id, name
  FROM emp;

UPDATE v_emp
SET name='Pera'
WHERE id=1;
```

works.

For more complex views you can use:

- `INSTEAD OF` trigger
- or older mechanism `RULE`.

## 4 SQL Server

This simple view:

```sql
CREATE VIEW v_emp AS
  SELECT id, name
  FROM emp;

UPDATE v_emp
  SET name='Pera'
  WHERE id=1;
```

works.

There are limitations for views with joins. If you want full control, use:

```sql
CREATE TRIGGER ...
  ON v_emp
  INSTEAD OF UPDATE
```

## 5 Oracle

Oracle has supported updates over simple views for a very long time.

First:

```sql
CREATE VIEW v_emp AS
  SELECT id, name
  FROM emp;

UPDATE v_emp
  SET name='Pera'
  WHERE id=1;
```

works.

If the view contains:

- AGGREGATES
- GROUP BY
- DISTINCT
- UNION
- complex joins

then it is usually not update-able, unless you define:

```sql
CREATE TRIGGER ...
  INSTEAD OF UPDATE ON v_emp
```

A practical rule that applies almost everywhere:

- If a view is a projection of a single table without aggregations or groupings, it is probably update-balanced.
- If it contains JOIN, GROUP BY, DISTINCT, aggregates or UNION, assume that it is not automatically updateable and check the documentation or use INSTEAD OF a trigger.
- Of all these databases, SQLite is the only one where a regular view is never directly updatable – you always have to use INSTEAD OFa trigger.
- INSTEAD OF trigger, you practically solve everything in the code, both the table that is being changed and the primary keys of the rows that are being changed...?

The INSTEAD OF trigger practically points to the database:

- you see which operation was called ( INSERT, UPDATE, DELETE );
- you have access to the old and new values ​​of the row:
- in PostgreSQL: OLD i NEW
- in SQLite: OLD i NEW
- in SQL Server: pseudo-tables deleted and inserted
- in Oracle: :OLD i :NEW

You decide for yourself:

- which table or tables are you changing,
- which lines are you changing,
- how do you use primary keys,

Will you even allow the operation?

For example, imagine a view:

```sql
CREATE VIEW v_emp_dept AS
  SELECT e.id,
    e.name,
    d.name AS dept_name
  FROM
    emp e
  JOIN
    dept d ON d.id = e.dept_id;
```

The user writes:

```sql
UPDATE v_emp_dept
  SET name='Pera',
    dept_name='IT'
  WHERE id=10;
```

The database itself does not know:

- should dept_name it change dept.name,
- does it need to find an existing one department,
- does he need to make a new one department,
- what if multiple employees share the same department.

Therefore, the INSTEAD OF UPDATE trigger gets:

```sql
OLD.id = 10
OLD.name = 'Marko'
OLD.dept_name = 'HR'
NEW.name = 'Pera'
NEW.dept_name = 'IT'
```

and you write:

```sql
UPDATE emp
  SET name = NEW.name
  WHERE id = OLD.id;

UPDATE dept
  SET name = NEW.dept_name
  WHERE id = (
    SELECT dept_id
    FROM emp
    WHERE id = OLD.id
);
```

or whatever you think is right.

That's why it's often said that an INSTEAD OF trigger is a kind of mini ORM based on:

- the application sees one view,
- love  him INSERT/UPDATE/DELETE,
- and a trigger translates those operations to one or more physical tables.

This is widely used in Oracle and SQL Server for creating “updatable join views.”
In PostgreSQL it is less common today because simple views are automatically updatable,
but for complex views an INSTEAD OF trigger is still the standard solution.
