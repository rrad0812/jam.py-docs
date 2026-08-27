
# DuckDb

It's a very fast, embedded analytical SQL database engine, especially handy for working with CSV/Parquet files and Python. You can think of it as "SQLite for data analysis", but with a stronger focus on large queries, columns, and analytics.

## Quick course

### **1. Installation**

```py
pip install duckdb pandas
```

### **2. First query**

```py
import duckdb

result = duckdb.sql("""
    SELECT 1 + 2 AS rezultat
""")

print(result)
```

DuckDB can run without a dedicated server. The database is a local library in the process of your Python application.

### **3. Saving the database to a file**

```py
import duckdb

connection = duckdb.connect("podaci.duckdb")

connection.execute("""
    CREATE TABLE IF NOT EXISTS korisnici (
        id INTEGER,
        ime VARCHAR,
        godine INTEGER
    )
""")

connection.execute("""
    INSERT INTO korisnici VALUES
        (1, 'Ana', 31),
        (2, 'Marko', 25)
""")

rows = connection.execute("""
    SELECT *
    FROM korisnici
    WHERE godine >= 30
""").fetchall()

print(rows)

connection.close()
```

If you use:

```py
duckdb.connect(":memory:")
```

the database only exists while the program is running.

### **4. Rad in Pandas DataFrame-om**

```py
import duckdb
import pandas as pd

users = pd.DataFrame({
    "name": ["Ana", "Marko", "Jelena"],
    "age": [31, 25, 42]
})

result = duckdb.sql("""
    SELECT name, age
    FROM users
    WHERE age >= 30
    ORDER BY age DESC
""").df()

print(result)
```

DuckDB can directly use Python variables in SQL queries.

### **5. Reading a CSV file**

```py
import duckdb

result = duckdb.sql("""
    SELECT *
    FROM read_csv_auto('data/users.csv')
    LIMIT 10
""")

print(result)
```

Grouping:

```py
duckdb.sql("""
    SELECT city, COUNT(*) AS broj_korisnika
    FROM read_csv_auto('data/users.csv')
    GROUP BY city
    ORDER BY broj_korisnika DESC
""").show()
```

### **6. Reading a Parquet file**

```py
import duckdb

duckdb.sql("""
    SELECT year, category, SUM(amount) AS total
    FROM 'data/sales.parquet'
    GROUP BY year, category
    ORDER BY year, total DESC
""").show()
```

DuckDB can often use the Parquet file name directly as a table.

### **7. Entering results into Parquet**

```py
duckdb.sql("""
    COPY (
        SELECT *
        FROM read_csv_auto('data/users.csv')
        WHERE active = true
    )
    TO 'data/active-users.parquet'
    (FORMAT PARQUET)
""")
```

### **8. Using with Flask/server code**

A typical form for an application is:

```py
import duckdb

connection = duckdb.connect("app.duckdb")

def get_users():
    return connection.execute("""
        SELECT id, name
        FROM users
        ORDER BY name
    """).fetchall()
```

It's better to write queries with parameters, rather than string concatenation:

```py
user_id = 5

row = connection.execute(
    "SELECT * FROM users WHERE id = ?",
    [user_id]
).fetchone()
```

This prevents SQL injection and makes the query more reliable.

## When to use DuckDB

DuckDB is great for:

- CSV and Parquet data analysis;
- reports and statistics;
- local ETL/data-processing tasks;
- Python scripts and small to medium applications;
- replacement for complex Pandas transformations;
- An API that reads and aggregates data.

It is not the first choice when you have many concurrent users who are constantly changing data. For such an OLTP case PostgreSQL is usually better. DuckDB is primarily an analytical database.
