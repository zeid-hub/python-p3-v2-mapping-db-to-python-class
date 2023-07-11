# Mapping a database row to a Python class instance

## Learning Goals

- Map the attribute values stored in a database row to a Python class instance.

---

## Key Vocab

- **Object-Relational Mapping (ORM)**: a programming technique that provides a
  mapping between an object-oriented data model and a relational database model.

---

## Introduction

The previous lesson showed how to implement methods to persist a Python object
as a row in a database table.

In this lesson, we see how to map the opposite direction, namely we will map the
values stored in a database table row to a Python class instance. We will
implement the following methods:

| Method                      | Return     | Description                                                                                |
| --------------------------- | ---------- | ------------------------------------------------------------------------------------------ |
| instance_from_db (cls, row) | Department | Return a class instance having the attribute values in a table row                         |
| get_all (cls)               | List       | Return a list of class instances from all rows in a table.                                 |
| find_by_id(cls, id))        | Department | Return a class instance corresponding to the table row specified by the primary key `id`   |
| find_by_name(cls, name)     | Department | Return a class instance corresponding to the first table row matching the specified `name` |

---

## Code Along

This lesson is a code-along, so fork and clone the repo.

**NOTE: Remember to run `pipenv install` to install the dependencies and
`pipenv shell` to enter your virtual environment before running your code.**

```bash
pipenv install
pipenv shell
```

Let's continue building out the `Department` class and its object-relational
mapping methods from the previous lesson. The starter code for the `Department`
class is in `lib/department.py`.

We need to build a few methods to access one or more table rows and return
Python objects with attribute values that match the row data.

To start, review the code from the `Department` class. Then take a look at this
code in the `debug.py` file:

```py
#!/usr/bin/env python3

from __init__ import CONN, CURSOR
from department import Department

import ipdb


def reset_database():
    Department.drop_table()
    Department.create_table()

    Department.create("Payroll", "Building A, 5th Floor")
    Department.create("Human Resources", "Building C, East Wing")
    Department.create("Accounting", "Building B, 1st Floor")


reset_database()
ipdb.set_trace()

```

This file is set up so that you can explore the database using the `Department`
class from an `ipdb` session. The file recreates the `Department` table and
persists a few objects into the table.

Run `debug.py` to confirm the code is able to reset and initialize the table:

```bash
python lib/debug.py
```

Then execute the following query in the `ipdb` shell to confirm the table rows,
or use SQLITE EXPLORER to confirm the table contents:

```py
ipdb> departments = CURSOR.execute('SELECT * FROM departments')
ipdb> [row for row in departments]
# => [(1, 'Payroll', 'Building A, 5th Floor'), (2, 'Human Resources', 'Building C, East Wing'), (3, 'Accounting', 'Building B, 1st Floor')]
```

---

## `instance_from_db()`

| Method                      | Return     | Description                                                        |
| --------------------------- | ---------- | ------------------------------------------------------------------ |
| instance_from_db (cls, row) | Department | Return a class instance having the attribute values in a table row |

The first thing we need to do is map the data stored in a table row into a
Python object.

One thing to know is that the database, SQLite in our case, will return a list
of data for each row. For example, a row for the payroll department would look
like this: `[1, "Payroll", "Building A, 5th Floor"]`. We use `row[0]` to get the
department id `1` and `row[1]` to get the department name `"Payroll"`, etc.

Add the new class method `instance_from_db(cls, row)` to the end of the
`Department` class:

```py
class Department

    # ... rest of attributes and methods

    @classmethod
    def instance_from_db(cls, row):
        """Return a Department object having the attribute values from the table row."""

        # Check the dictionary for an existing class instance using the row's primary key
        department = Department.all.get(row[0])
        # If not in dictionary, create a new class instance using the row data and add to dictionary
        if department is None:
            department = cls(row[1], row[2])
            department.id = row[0]
            Department.all[department.id] = department
        return department
```

The `instance_from_db` method takes a reference to a Python class `cls` and a
list storing the column values from a table row `row`, and returns an instance
of the class using the attribute values from the table row data. The method
looks in the dictionary for an existing class instance using the primary key
`id` as the dictionary key. If there isn't an entry, the method will create a
new class instance and add it to the dictionary. In general the dictionary
**should** contain the corresponding class instance since we add to the
dictionary each time we persist an object using the `save()` method, and we
should not need to create a new instance and add to the dictionary. However, it
is possible that the table may contain a row inserted directly using an SQL
statement rather than the `save()` method, so we'll handle that case with the
conditional statement.

Let's run `python lib/debug.py` to create the table with initial values. We'll
execute a query to get a row of data, then use that row to get a Python
Department object:

```py
ipdb> row = CURSOR.execute("select * from departments").fetchone()
ipdb> row
(1, 'Payroll', 'Building A, 5th Floor')
ipdb> department = Department.instance_from_db(row)
ipdb> department
<Department 1: Payroll, Building A, 5th Floor>
ipdb>
```

---

## `get_all()`

| Method        | Return | Description                                               |
| ------------- | ------ | --------------------------------------------------------- |
| get_all (cls) | List   | Return a list of class instances from all rows in a table |

To return all the departments in the database, we need to do the following:

1. define a SQL query statement to select all rows from the table
2. use the CURSOR to execute the query, and then call the `fetchall()` method on
   the query result to return the rows sequentially in a tuple.
3. iterate over each row and call `instance_from_db()` with each row in the
   query result to retrieve a Python object from the row data:

```py
class Department

    # ... rest of methods

    @classmethod
    def get_all(cls):
        """Return a list of class instances corresponding to each row in the table"""
        sql = """
            SELECT *
            FROM departments
        """

        rows = CURSOR.execute(sql).fetchall()

        return [cls.instance_from_db(row) for row in rows]
```

With this method in place, let's try calling the `get_all()` method to access
all the departments in the database.

Run `python lib/debug.py`, and then follow along in the `ipdb` session:

```py
ipdb> Department.get_all()
[<Department 1: Payroll, Building A, 5th Floor>, <Department 2: Human Resources, Building C, East Wing>, <Department 3: Accounting, Building B, 1st Floor>]
ipdb>
```

Success! We can see all three departments in the database as a list of
`Department` instances. We can interact with them just like any other Python
objects:

```py
ipdb> departments = Department.get_all()
ipdb> departments[0]
<Department 1: Payroll, Building A, 5th Floor>
ipdb> departments[0].name
'Payroll'
ipdb>
```

## `find_by_id()`

| Method               | Return     | Description                                                                              |
| -------------------- | ---------- | ---------------------------------------------------------------------------------------- |
| find_by_id(cls, id)) | Department | Return a class instance corresponding to the table row specified by the primary key `id` |

This one is similar to `get_all()`, with the small exception being that we have
a `WHERE` clause to test the `id` in our SQL statement. To do this, we use a
bound parameter (i.e. question mark) where we want the `id` parameter to be
passed in, and we include `id` in a tuple as the second argument to the
`execute()` method:

```py
class Department:

    # ... rest of methods

    @classmethod
    def find_by_id(cls, id):
        """Return a Department object corresponding to the table row matching the specified primary key"""
        sql = """
            SELECT *
            FROM departments
            WHERE id = ?
        """

        row = CURSOR.execute(sql, (id,)).fetchone()
        return cls.instance_from_db(row) if row else None

```

There are a couple important things to note here:

- Bound parameters must be passed to the `execute` statement as a sequence data
  type. This is typically performed with tuples to match the format that results
  are returned in. A tuple containing only one element must have a comma after
  that element, otherwise it is interpreted as a grouped statement (think
  [PEMDAS](https://en.wikipedia.org/wiki/Order_of_operations)).
- The `fetchone()` method returns the first element from `fetchall()`.

Let's try out this new method. Exit `ipdb` and run `python lib/debug.py` again:

```py
ipdb> department = Department.find_by_id(1)
ipdb> department
<Department 1: Payroll, Building A, 5th Floor>
ipdb> department.name
'Payroll'
ipdb> department.location
'Building A, 5th Floor'
ipdb>
```

## `find_by_name()`

| Method                  | Return     | Description                                                                                |
| ----------------------- | ---------- | ------------------------------------------------------------------------------------------ |
| find_by_name(cls, name) | Department | Return a class instance corresponding to the first table row matching the specified `name` |

The `find_by_name()` method is similar to `find_by_id()`, but we will limit the
result to the first row matching the specified name.

```py
class Department:

    # ... rest of methods

    @classmethod
    def find_by_name(cls, name):
        """Return a Department object corresponding to first table row matching specified name"""
        sql = """
            SELECT *
            FROM departments
            WHERE name is ?
        """

        row = CURSOR.execute(sql, (name,)).fetchone()
        return cls.instance_from_db(row) if row else None

```

Let's try out this new method. Exit `ipdb` and run `python lib/debug.py` again:

```py
ipdb> department = Department.find_by_name("Payroll")
ipdb> department
<Department 1: Payroll, Building A, 5th Floor>

```

Success!

### Testing the ORM

The `testing` folder contains the file `department_orm_test.py`, which has been
updated to test all of the ORM methods.

Run `pytest -x` to confirm your code passes the tests, then submit this
code-along assignment using `git`.

> **Note: You may have to delete your existing database `company.db` for all of
> the tests to pass- SQLite sometimes "locks" databases that have been accessed
> by multiple modules.**

---

## Conclusion

In the previous lesson, we have created a database and persisted Python objects
to the database. In this lesson, we saw how to map data stored in the database
into Python objects.

We haven't covered every use case, but we have a functioning ORM for a single
Python class!

In the next lesson, we will see how to persist relationships between Python
class instances.

---

## Solution Code

```py
from __init__ import CURSOR, CONN

class Department:

    # Define a dictionary to store class instances for subsequent lookup when mapping a table row to a class instance.
    all = {}

    def __init__(self, name, location, id=None):
        self.id = id
        self.name = name
        self.location = location

    def __repr__(self):
        return f"<Department {self.id}: {self.name}, {self.location}>"

    @classmethod
    def create_table(cls):
        """ Create a new table to persist the attributes of Department class instances """
        sql = """
            CREATE TABLE IF NOT EXISTS departments (
            id INTEGER PRIMARY KEY,
            name TEXT,
            location TEXT)
        """
        CURSOR.execute(sql)
        CONN.commit()

    @classmethod
    def drop_table(cls):
        """ Drop the table that persists Department class instances """
        sql = """
            DROP TABLE IF EXISTS departments;
        """
        CURSOR.execute(sql)
        CONN.commit()

    def save(self):
        """ Insert a new row with the name and location values of the current Department object.
        Update object id attribute using the primary key value of new row.
        Save the object in local dictionary using table row's PK as dictionary key"""
        sql = """
            INSERT INTO departments (name, location)
            VALUES (?, ?)
        """

        CURSOR.execute(sql, (self.name, self.location))
        CONN.commit()

        self.id = CURSOR.lastrowid
        Department.all[self.id] = self

    @classmethod
    def create(cls, name, location):
        """ Initialize a new Department object and save the object to the database """
        department = Department(name, location)
        department.save()
        return department

    def update(self):
        """Update the table row corresponding to the current Department object."""
        sql = """
            UPDATE departments
            SET name = ?, location = ?
            WHERE id = ?
        """
        CURSOR.execute(sql, (self.name, self.location, self.id))
        CONN.commit()

    def delete(self):
        """Delete the table row corresponding to the current Department class instance.
        Remove the object from local dictionary."""
        sql = """
            DELETE FROM departments
            WHERE id = ?
        """

        CURSOR.execute(sql, (self.id,))
        CONN.commit()

        del Department.all[self.id]

    @classmethod
    def instance_from_db(cls, row):
        """Return a Department object having the attribute values from the table row."""

        # Check the dictionary for an existing class instance using the row's primary key
        department = Department.all.get(row[0])
        # If not in dictionary, create a new class instance using the row data and add to dictionary
        if department is None:
            department = cls(row[1], row[2])
            department.id = row[0]
            Department.all[department.id] = department
        return department

    @classmethod
    def get_all(cls):
        """Return a list containing a Department object corresponding to each row in the table"""
        sql = """
            SELECT *
            FROM departments
        """

        rows = CURSOR.execute(sql).fetchall()

        return [cls.instance_from_db(row) for row in rows]

    @classmethod
    def find_by_id(cls, id):
        """Return a Department object corresponding to the table row matching the specified primary key"""
        sql = """
            SELECT *
            FROM departments
            WHERE id = ?
        """

        row = CURSOR.execute(sql, (id,)).fetchone()
        return cls.instance_from_db(row) if row else None

    @classmethod
    def find_by_name(cls, name):
        """Return a Department object corresponding to first table row matching specified name"""
        sql = """
            SELECT *
            FROM departments
            WHERE name is ?
        """

        row = CURSOR.execute(sql, (name,)).fetchone()
        return cls.instance_from_db(row) if row else None
```

## Resources

- [sqlite3 - DB-API 2.0 interface for SQLite databases - Python](https://docs.python.org/3/library/sqlite3.html)
