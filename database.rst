Database
==============

Before you can start working with entities you have to create a ``Database`` object. This object manages database connections using a connection pool. The ``Database`` object is thread safe and can be shared between all threads in your application. The ``Database`` object allows you to work with the database directly using SQL, but most of the time you will work with entities and let Pony generate SQL statements in order to make the corresponding changes in the database. Pony allows you to work with several databases at the same time, but each entity belongs to one specific database.

Mapping entities to the database can be divided into four steps:

* Creating a database object
* Defining entities which are related to the database object
* Binding the database object to a specific database
* Mapping entities to the database tables

Creating a database object
-----------------------------------------

At this step we simply create an instance of the ``Database`` class::

    db = Database()

Although you can pass the database connection parameters right here, often it is more convenient to do it at a later stage using the ``db.bind()`` method. This way you can use different databases for testing and production.

The ``Database`` instance has an attribute ``Entity`` which represents a base class to be used for entities declaration.


Defining entities which are related to the database object
---------------------------------------------------------------------------------------

Entities should inherit from the base class of the ``Database`` object::

    class MyEntity(db.Entity):
        attr1 = Required(str)

We'll talk about entities definition in detail in the next chapter. Now, let's see the next step in mapping entities to a database.

Binding the database object to a specific database
--------------------------------------------------------------

Before we can map entities to a database, we need to connect to the database. At this step we should use the ``bind()`` method::

    db.bind('postgres', user='', password='', host='', database='')

The first parameter of this method is the name of the database provider. The database provider is a module which resides in the ``pony.orm.dbproviders`` package and which knows how to work with a particular database. After the database provider name you should specify parameters which will be passed to the ``connect()`` method of the corresponding DBAPI driver.


Database providers
------------------------------

Currently Pony can work with four database systems: SQLite, PostgreSQL, MySQL and Oracle, with the corresponding Pony provider names: ``'sqlite'``, ``'postgres'``, ``'mysql'`` and ``'oracle'``.  Pony can easily be extended to incorporate additional database providers.

During the ``bind()`` call, Pony tries to establish a test connection to the database. If the specified parameters are not correct or the database is not available, an exception will be raised. After the connection to the database was established, Pony retrieves the version of the database and returns the connection to the connection pool.


SQLite
~~~~~~~~~~~~~~~~~~~~~~

Using SQLite database is the easiest way to work with Pony because there is no need to install a database system separately - the SQLite database system is included in the Python distribution. It is a perfect choice for beginners who want to experiment with Pony in the interactive shell. In order to bind the ``Database`` object a SQLite database you can do the following::
   
    db.bind('sqlite', 'filename', create_db=True)

Here 'filename' is the name of the file where SQLite will store the data. The filename can be absolute or relative.

.. note:: If you specify a relative path, that path is appended to the directory path of the Python file where this database was created (and not to the current working directory). We did it this way because sometimes a programmer doesn’t have the control over the current working directory (e.g. in mod_wsgi application). This approach allows the programmer to create applications which consist of independent modules, where each module can work with a separate database.

When working in the interactive shell, Pony requires that you to always specify the absolute path of the storage file.

If the parameter ``create_db`` is set to ``True`` then Pony will try to create the database if such filename doesn’t exists.  Default value for ``create_db`` is ``False``.

.. note:: Normally SQLite database is stored in a file on disk, but it also can be stored entirely in memory. This is a convenient way to create a SQLite database when playing with Pony in the interactive shell, but you should remember, that the entire in-memory database will be lost on program exit. Also you should not work with the same in-memory SQLite database simultaneously from several threads because in this case all threads share the same connection due to SQLite limitation.
 
  In order to bind with an in-memory database you should specify ``:memory:`` instead of the filename::

      db.bind('sqlite', ':memory:')

  There is no need in the parameter ``create_db`` when creating an in-memory database. 

.. note:: By default SQLite doesn’t check foreign key constraints. Pony always enables the foreign key support by sending the command ``PRAGMA foreign_keys = ON;`` starting with the release 0.4.9.



PostgreSQL
~~~~~~~~~~~~~~~~~~~~~~~

Pony uses psycopg2 driver in order to work with PostgreSQL. In order to bind the ``Database`` object to PostgreSQL use the following line::

    db.bind('postgres', user='', password='', host='', database='')

All the parameters that follow the Pony database provider name will be passed to the ``psycopg2.connect()`` method. Check the `psycopg2.connect documentation <http://initd.org/psycopg/docs/module.html#psycopg2.connect>`_ in order to learn what other parameters you can pass to this method.


MySQL
~~~~~~~~~~~~~~~
::

    db.bind('mysql', host='', user='', passwd='', db='')

Pony tries to use the MySQLdb driver for working with MySQL. If this module cannot be imported, Pony tries to use pymysql. See the `MySQLdb <http://mysql-python.sourceforge.net/MySQLdb.html#functions-and-attributes>`_ and `pymysql <https://pypi.python.org/pypi/PyMySQL>`_ documentation for more information regarding these drivers.

Oracle
~~~~~~~~~~~~~~~~
::

    db.bind('oracle', 'user/password@dsn')

Pony uses cx_Oracle driver for connecting to Oracle databases. More information about the parameters which you can use for creating a connection to Oracle database can be found `here <http://cx-oracle.sourceforge.net/html/module.html>`_. 


Mapping entities to the database tables
----------------------------------------------------------

After the ``Database`` object is created, entities are defined, and a database is bound, the next step is to map entities to the database tables::

    db.generate_mapping(check_tables=True, create_tables=False)



Early database binding
----------------------------------------------------------------------------

You can combine the steps 'Creating a database object' and 'Binding the database object to a specific database' into one step by passing the database parameters during the database object creation::

    db = Database('sqlite', 'filename', create_db=True)

    db = Database('postgres', user='', password='', host='', database='')

    db = Database('mysql', host='', user='', passwd='', db='')

    db = Database('oracle', 'user/password@dsn')

It is the same set of parameters which you can pass to the ``bind()`` method. If you pass the parameters during the creation of the ``Database`` object, then there is no need in calling the ``bind()`` method later - the database will be already bound to the instance.


Methods and attributes of the Database object
---------------------------------------------------------


Methods for working with transactions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


Database object attributes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


Methods for raw SQL access
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



Database statistics
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. class:: Database

   The ``Database`` object keeps statistics on executed queries. You can check which queries were executed more often and how long it took to execute them as well as many other parameters. Pony keeps all statistics separately for each thread. If you want to see the aggregated statistics for all threads then you need to call the ``merge_local_stats()`` method.




Using Database object for raw SQL queries
----------------------------------------------------

Typically you will work with entities and let Pony interact with the database, but Pony also allows you to work with the database using SQL, or even combine both ways. Of course you can work with the database directly using the DBAPI interface, but using the ``Database`` object gives you the following advantages:

* Automatic transaction management using the :ref:`db_session <db_session_ref>` decorator or context manager. All data will be stored to the database after the transaction is finished, or rolled back if an exception happened.
* Connection pool. There is no need to keep track of database connections. You have the connection when you need it and when you have finished your transaction the connection will be returned to the pool.
* Unified database exceptions. Each DBAPI module defines its own exceptions. Pony allows you to work with the same set of exceptions when working with any database. This helps you to create applications which can be ported from one database to another.
* Unified way of passing parameters to SQL queries with the protection from injection attacks. Different database drivers use different paramstyles - the DBAPI specification offers 5 different ways of passing parameters to SQL queries. Using the ``Database`` object you can use one way of passing parameters for all databases and eliminate the risk of SQL injection.
* Automatic unpacking of single column results when using ``get`` or ``select`` methods of the ``Database`` object. If the ``select`` method returns just one column, Pony returns a list of values, not a list of tuples each of which has just one item, as it does DBAPI. If the ``get`` method returns a single column it returns just value, not a tuple consisting of one item. It’s just convenient.
* When the methods ``select`` or ``get`` return more than one column, Pony uses smart tuples which allow accessing items as tuple attributes using column names, not just tuple indices.

In other words the ``Database`` object helps you save time completing routine tasks and provides convenience and uniformity.


Using parameters in raw SQL queries
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

With Pony you can easily pass parameters into SQL queries. In order to specify a parameter you need to put the `$` sign before the variable name::

    x = "John"
    data = db.select("* from Person where name = $x")

When Pony encounters such a parameter within the SQL query it gets the variable value from the current frame (from globals and locals) or from the dictionary which is passed as the second parameter. In the example above Pony will try to get the value for ``$x`` from the variable ``x`` and will pass this value as a parameter to the SQL query which eliminates the risk of SQL injection. Below you can see how to pass a dictionary with the parameters::

    data = db.select("* from Person where name = $x", {"x" : "Susan"})

This method of passing parameters to the SQL queries is very flexible and allows using not only single variables, but any Python expression. In order to specify an expression you need to put it in parentheses after the $ sign::

    data = db.select("* from Person where name = $(x.lower()) and age > $(y + 2)")


