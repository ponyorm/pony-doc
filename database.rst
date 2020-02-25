Connecting to the Database
==========================

Before you can start working with entities you have to create the ``Database`` object. The entities, that you declare in your Python code, will be mapped to the database through this object.

Mapping entities to the database can be divided into four steps:

* Creating the :py:class:`Database` object
* Defining entities which are related to this :py:class:`Database` object
* Binding the :py:class:`Database` object to a specific database
* Mapping entities to the database tables

Now we'll describe the main workflow of working with the :py:class:`Database` object and its methods. When you'll need more details on this, you can find them in the :ref:`API Reference <api_reference>`.

Creating the Database object
----------------------------

At this step we simply create an instance of the :py:class:`Database` class:

.. code-block:: python

    db = Database()

The :py:class:`Database` class instance has an attribute :py:attr:`~Database.Entity` which represents a base class to be used for entities declaration.


Defining entities which are related to the Database object
----------------------------------------------------------

Entities should inherit from the base class of the :py:class:`Database` object:

.. code-block:: python

    class MyEntity(db.Entity):
        attr1 = Required(str)

We'll talk about entities definition in detail in the next chapter. Now, let's see the next step in mapping entities to a database.


Binding the database object to a specific database
--------------------------------------------------

Before we can map entities to the database, we need to connect to establish connection to it. It can be done using the :py:meth:`~Database.bind` method:

.. code-block:: python

    db.bind(provider='postgres', user='', password='', host='', database='')

The first parameter of this method is the name of the database provider. The database provider is a module which resides in the ``pony.orm.dbproviders`` package and which knows how to work with a particular database. After the database provider name you should specify parameters which will be passed to the ``connect()`` method of the corresponding DBAPI driver.

Currently Pony can work with the following database systems: SQLite, PostgreSQL, MySQL, Oracle, CockroachDB, with the corresponding Pony provider names: ``'sqlite'``, ``'postgres'``, ``'mysql'``, ``'oracle'`` and ``'cockroach'``.  Pony can easily be extended to incorporate additional database providers.

When you just start working with Pony, you can use the SQLite database. This database is included into Python distribution and you don't need to install anything separately. Using SQLite you can create the database either in a file or in memory. For creating the database in the file use the following command:

.. code-block:: python

    db.bind(provider='sqlite', filename='database.sqlite', create_db=True)

When ``create_db=True``, Pony will create the database file if it doesn't exist. If it already exists, Pony will use it.

For in-memory database use this:

.. code-block:: python

  db.bind(provider='sqlite', filename=':memory:')

There is no need in the parameter ``create_db`` when creating an in-memory database. This is a convenient way to create a SQLite database when playing with Pony in the interactive shell, but you should remember, that the entire in-memory database will be lost on program exit.

Here are the examples of binding to other databases:

.. code-block:: python

    db.bind(provider='sqlite', filename=':memory:')
    db.bind(provider='sqlite', filename='filename', create_db=True)
    db.bind(provider='mysql', host='', user='', passwd='', db='')
    db.bind(provider='oracle', user='', password='', dsn='')
    db.bind(provider='cockroach', user='', password='', host='',
            database='', sslmode='disable')

You can find more details on working with each database in the API Reference:

* :ref:`SQLite <sqlite>`
* :ref:`PostgreSQL <postgresql>`
* :ref:`MySQL <mysql>`
* :ref:`Oracle <oracle>`
* :ref:`CockroachDB <cockroachdb>`



Mapping entities to the database tables
---------------------------------------

After the :py:class:`Database` object is created, entities are defined, and a database is bound, the next step is to map entities to the database tables using the :py:meth:`~Database.generate_mapping` method:

.. code-block:: python

    db.generate_mapping(create_tables=True)

This method creates tables, foreign key references and indexes if they don't exist. After entities are mapped, you can start working with them in your Python code - select, create, modify objects and save them in the database.

Methods and attributes of the Database object
---------------------------------------------

The :py:class:`Database` object has a set of methods, which you can exampne in the :ref:`API Reference <api_reference>`.

.. _raw_sql:

Using Database object for raw SQL queries
-----------------------------------------

Typically you will work with entities and let Pony interact with the database, but Pony also allows you to work with the database using SQL, or even combine both ways. Of course you can work with the database directly using the DBAPI interface, but using the ``Database`` object gives you the following advantages:

* Automatic transaction management using the :py:func:`db_session` decorator or context manager. All data will be stored to the database after the transaction is finished, or rolled back if an exception happened.
* Connection pool. There is no need to keep track of database connections. You have the connection when you need it and when you have finished your transaction the connection will be returned to the pool.
* Unified database exceptions. Each DBAPI module defines its own exceptions. Pony allows you to work with the same set of exceptions when working with any database. This helps you to create applications which can be ported from one database to another.
* Unified way of passing parameters to SQL queries with the protection from injection attacks. Different database drivers use different paramstyles - the DBAPI specification offers 5 different ways of passing parameters to SQL queries. Using the :py:class:`Database` object you can use one way of passing parameters for all databases and eliminate the risk of SQL injection.
* Automatic unpacking of single column results when using :py:meth:`~Database.get` or :py:meth:`~Database.select` methods of the :py:class:`Database` object. If the :py:meth:`~Database.select` method returns just one column, Pony returns a list of values, not a list of tuples each of which has just one item, as it does DBAPI. If the :py:meth:`~Database.get` method returns a single column it returns just value, not a tuple consisting of one item. It’s just convenient.
* When the methods :py:meth:`~Database.select` or :py:meth:`~Database.get` return more than one column, Pony uses smart tuples which allow accessing items as tuple attributes using column names, not just tuple indices.

In other words the :py:class:`Database` object helps you save time completing routine tasks and provides convenience and uniformity.


Using parameters in raw SQL queries
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

With Pony you can easily pass parameters into SQL queries. In order to specify a parameter you need to put the `$` sign before the variable name:

.. code-block:: python

    x = "John"
    data = db.select("select * from Person where name = $x")

When Pony encounters such a parameter within the SQL query it gets the variable value from the current frame (from globals and locals) or from the dictionary which is passed as the second parameter. In the example above Pony will try to get the value for ``$x`` from the variable ``x`` and will pass this value as a parameter to the SQL query which eliminates the risk of SQL injection. Below you can see how to pass a dictionary with the parameters:

.. code-block:: python

    data = db.select("select * from Person where name = $x", {"x" : "Susan"})

This method of passing parameters to the SQL queries is very flexible and allows using not only single variables, but any Python expression. In order to specify an expression you need to put it in parentheses after the $ sign:

.. code-block:: python

    data = db.select("select * from Person where name = $(x.lower()) and age > $(y + 2)")


All the parameters can be passed into the query using the Pony unified way, independently of the DBAPI provider, using the ``$`` sign. In the example above we pass ``name`` and ``age`` parameters into the query.

It is possible to have a Python expressions inside the query text, for example:

.. code-block:: python

    x = 10
    a = 20
    b = 30
    db.execute("SELECT * FROM Table1 WHERE column1 = $x and column2 = $(a + b)")

If you need to use the $ sign as a string literal inside the query, you need to escape it using another $ (put two $ signs in succession: $$).

Customizing connection behavior
-------------------------------

You can execute some queries to specify your connection (i.e. pragmas) using :py:func:`db.on_connect` decorator.

.. code-block:: python

    db = Database()

    # entities declaration

    @db.on_connect(provider='sqlite')
    def sqlite_case_sensitivity(db, connection):
        cursor = connection.cursor()
        cursor.execute('PRAGMA case_sensitive_like = OFF')

    db.bind(**options)
    db.generate_mapping(create_tables=True)

With following code each new sqlite connection will call this function. This example shows how to restore old case insensitive `like` for sqlite.

Database statistics
-------------------

The ``Database`` object keeps statistics on executed queries. You can check which queries were executed more often and how long it took to execute them as well as some other parameters. Check the :py:class:`QueryStat` class in the API Reference for more details.

