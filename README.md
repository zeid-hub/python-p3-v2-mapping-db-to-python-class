# Mapping a database row to a Python class instance

## Learning Goals

- Instantiate a Python object using the attribute values stored in a database
  row.

---

## Key Vocab

- **Object-Relational Mapping (ORM)**: a programming technique that provides a
  mapping between an object-oriented data model and a relational database model.

---

## Introduction

<style>
table th:first-of-type {
    width: 30%;
}
table th:nth-of-type(2) {
    width: 10%;
}
table th:nth-of-type(3) {
    width: 40%;
}
</style>

The previous lesson showed how to implement methods to persist a Python object
as a row in a database table.

In this lesson, we see how to map the opposite direction, namely we will
instantiate a Python object using the values stored in a database table row. We
will implement the following methods:

| Method                  | Return | Description                                                                                |
| ----------------------- | ------ | ------------------------------------------------------------------------------------------ |
| new_from_db (cls, row)  | Object | Instantiate a class instance using the attribute values in a table row.                    |
| get_all (cls)           | List   | Create a list of class instances from all rows in a table.                                 |
| find_by_id(cls, id))    | Object | Create a class instance corresponding to the table row specified by the primary key `id`   |
| find_by_name(cls, name) | Object | Create a class instance corresponding to the first table row matching the specified `name` |

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

We need to build a few methods to access one or more table rows and create
Python objects using the row data.

To start, review the code from the `Department` class. Then take a look at this
code in the `debug.py` file:

```py
#!/usr/bin/env python3

from config import CONN, CURSOR
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

## `new_from_db()`

The first thing we need to do is convert the data stored in a table row into a
Python object. We will use this method to create all the Python objects needed
by the rest of the lesson methods.

One thing to know is that the database, SQLite in our case, will return an array
of data for each row. For example, a row for the payroll department would look
like this: `[1, "Payroll", "Building A, 5th Floor"]`.

```py
class Department

    # ... rest of methods

    @classmethod
    def new_from_db(cls, row):
        """Initialize a new Department object using the values from the table row."""
        department = cls(row[1], row[2])
        department.id = row[0]
        return department
```

The `new_from_db` method takes a reference to a Python class `cls` and a tuple
storing the column values from a table row `row`, and creates an instance of the
class using the attribute values from the table row data.

---

## `get_all()`

Recall that in previous lessons with Python classes, we used the `all` class
attribute to represent all instances of our class. In those examples, `all` was
the **single source of truth** for instances in a particular class.

That approach showed some limitations, however. Using that attribute meant that
our Python objects were only persisted in memory as long as our Python program
was running. If we exited the program and re-ran our code, we'd lose access to
that data.

Now that we have a SQL database, our classes have a new way to persist data:
using the database!

To return all the departments in the database, we need to do the following:

1. define a SQL query statement to select all rows from the table
2. use the CURSOR to execute the query, and then call the `fetchall()` method on
   the query result to return the rows sequentially in a tuple.
3. iterate over each row and call `new_from_db()` with each row in the query
   result, creating a new Python object from the row data:

```py
class Department

    # ... rest of methods

    @classmethod
    def get_all(cls):
        sql = """
            SELECT *
            FROM departments
        """

        rows = CURSOR.execute(sql).fetchall()

        cls.all = [cls.new_from_db(row) for row in rows]
        return cls.all
```

With this method in place, let's try calling the `get_all()` method to access
all the departments in the database.

Run `python lib/debug.py`, and then follow along in the `ipdb` shell:

```py
ipdb> Department.get_all()
[<Department 1: Payroll, Building A, 5th Floor>, <Department 2: Human Resources, Building C, East Wing>, <Department 3: Accounting, Building B, 1st Floor>]
ipdb>
```

To see a bit more detail, we can use the `.__dict__` attribute that Python
assigns to new objects:

```py
ipdb> [department.__dict__ for department in Department.all]
[{'id': 1, 'name': 'Payroll', 'location': 'Building A, 5th Floor'}, {'id': 2, 'name': 'Human Resources', 'location': 'Building C, East Wing'}, {'id': 3, 'name': 'Accounting', 'location': 'Building B, 1st Floor'}]
```

Success! We can see all three departments in the database as an array of
`Department` instances. We can interact with them just like any other Python
objects:

```py
ipdb> Department.all[0].__dict__
# => {'id': 1, 'name': 'Payroll', 'location': 'Building A, 5th Floor'}
ipbd> Department.all[0].name
# => 'Payroll'
```

## `find_by_id()`

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
      """Return Department object corresponding to the table row matching the specified primary key"""
      sql = """
          SELECT *
          FROM departments
          WHERE id = ?
      """

      row = CURSOR.execute(sql, (id,)).fetchone()
      return cls.new_from_db(row) if row else None

```

There are a couple important things to note here:

- Bound parameters must be passed to the `execute` statement as a sequence data
  type. This is typically performed with tuples to match the format that results
  are returned in. A tuple containing only one element must have a comma after
  that element, otherwise it is interpreted as a grouped statement (think
  [PEMDAS](https://en.wikipedia.org/wiki/Order_of_operations)).
- The `fetchone()` method returns the first element from `fetchall()`.

Let's try out this new method. Exit `ipdb` and run `python debug.py` again:

```py
ipydb> Department.find_by_id(1).__dict__
# => {'id': 1, 'name': 'Payroll', 'location': 'Building A, 5th Floor'}
```

## `find_by_name()`

The `find_by_name()` method is similar to `find_by_id()`, but we will limit the
result to the first row matching the specified name.

```py
class Department:

    # ... rest of methods

    @classmethod
    def find_by_name(cls, name):
        """Return Department object corresponding to first table row matching specified name"""
        sql = """
            SELECT *
            FROM departments
            WHERE name is ?
        """

        row = CURSOR.execute(sql, (name,)).fetchone()
        return cls.new_from_db(row) if row else None

```

Let's try out this new method. Exit `ipdb` and run `python debug.py` again:

```py
Department.find_by_name("Payroll").__dict__
# => {'id': 1, 'name': 'Payroll', 'location': 'Building A, 5th Floor'}
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
to the database using the following methods:

| Method             | Return | Description                                   |
| ------------------ | ------ | --------------------------------------------- |
| create_table (cls) | None   | Create a table to store class instances.      |
| drop_table (cls)   | None   | Drop the table.                               |
| save (self)        | None   | Save an object as a new table row.            |
| update(self)       | None   | Update an object's corresponding table row    |
| delete (self)      | None   | Delete the table row for the specified object |

In this lesson, we created new Python objects from the data in the database
using the following methods:

| Method                  | Return | Description                                                                                |
| ----------------------- | ------ | ------------------------------------------------------------------------------------------ |
| new_from_db (cls, row)  | Object | Instantiate a class instance using the attribute values in a table row.                    |
| get_all (cls)           | List   | Create a list of class instances from all rows in a table.                                 |
| find_by_id(cls, id))    | Object | Create a class instance corresponding to the table row specified by the primary key `id`   |
| find_by_name(cls, name) | Object | Create a class instance corresponding to the first table row matching the specified `name` |

We haven't covered every use case, but we have a functioning ORM for a single
Python class!

In the next lesson, we will see how to persist relationships between Python
class instances.

---

## Solution Code

```py
from config import CURSOR, CONN


class Department:

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
        Update object id attribute using the primary key value of new row"""
        sql = """
            INSERT INTO departments (name, location)
            VALUES (?, ?)
        """

        CURSOR.execute(sql, (self.name, self.location))
        CONN.commit()

        self.id = CURSOR.lastrowid

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
        """Delete the table row corresponding to the current Department class instance"""
        sql = """
            DELETE FROM departments
            WHERE id = ?
        """

        CURSOR.execute(sql, (self.id,))
        CONN.commit()

    @classmethod
    def new_from_db(cls, row):
        """Return a new Department object using the values from the table row."""
        department = cls(row[1], row[2])
        department.id = row[0]
        return department

    @classmethod
    def get_all(cls):
        """Return a list containing a new Department object for each row in the table"""
        sql = """
            SELECT *
            FROM departments
        """

        rows = CURSOR.execute(sql).fetchall()

        cls.all = [cls.new_from_db(row) for row in rows]
        return cls.all

    @classmethod
    def find_by_id(cls, id):
        """Return a new Department object corresponding to the table row matching the specified primary key"""
        sql = """
            SELECT *
            FROM departments
            WHERE id = ?
        """

        row = CURSOR.execute(sql, (id,)).fetchone()
        return cls.new_from_db(row) if row else None

    @classmethod
    def find_by_name(cls, name):
        """Return a new Department object corresponding to first table row matching specified name"""
        sql = """
            SELECT *
            FROM departments
            WHERE name is ?
        """

        row = CURSOR.execute(sql, (name,)).fetchone()
        return cls.new_from_db(row) if row else None


```

## Resources

- [sqlite3 - DB-API 2.0 interface for SQLite databases - Python](https://docs.python.org/3/library/sqlite3.html)
