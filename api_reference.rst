.. _api_reference:

API Reference
=============

.. contents::
    :backlinks: none

Database
--------

Database class
~~~~~~~~~~~~~~

.. py:class:: Database

    The ``Database`` object manages database connections using a connection pool. It is thread safe and can be shared between all threads in your application. The ``Database`` object allows working with the database directly using SQL, but most of the time you will work with entities and let Pony generate SQL statements for makeing the corresponding changes in the database. You can work with several databases at the same time, having a separate ``Database`` object for each database, but each entity always belongs to one database.



    .. py:method:: bind(provider, *args, **kwargs)
    .. py:method:: bind(*args, **kwargs)

        Bind entities to a database.

        :param str provider: the name of the database provider. The database provider is a module which resides in the ``pony.orm.dbproviders`` package. It knows how to work with a particular database. After the database provider name you should specify parameters which will be passed to the ``connect()`` method of the corresponding DBAPI driver. Pony comes with the following providers: "sqlite", "postgres", "mysql", "oracle". This parameter can be used as a keyword argument as well.
        :param args: parameters required by the database driver.
        :param kwargs: parameters required by the database driver.

        During the ``bind()`` call, Pony tries to establish a test connection to the database. If the specified parameters are not correct or the database is not available, an exception will be raised. After the connection to the database was established, Pony retrieves the version of the database and returns the connection to the connection pool.

        The method can be called only once for a database object. All consequent calls of this method on the same database will raise the ``TypeError('Database object was already bound to ... provider')`` exception.

        .. code-block:: python

            db.bind('sqlite', ':memory:')
            db.bind('sqlite', 'filename', create_db=True)
            db.bind('postgres', user='', password='', host='', database='')
            db.bind('mysql', host='', user='', passwd='', db='')
            db.bind('oracle', 'user/password@dsn')

        Also you can use keyword arguments for passing the parameters:

        .. code-block:: python

            db.bind(provider='sqlite', filename=':memory:')
            db.bind(provider='sqlite', filename='db.sqlite', create_db=True)
            db.bind(provider='postgres', user='', password='', host='', database='')
            db.bind(provider='mysql', host='', user='', passwd='', db='')
            db.bind(provider='oracle', user='', password='', dsn='')

        This allows keeping these parameters in a dict:

        .. code-block:: python

            db_params = dict(provider='postgres', host='...', port=...,
                             user='...', password='...')
            db.bind(**db_params)


    .. py:method:: commit()

        Save all changes made within the current :py:func:`db_session` using the :py:meth:`~Database.flush` method and commits the transaction to the database.

        You can call ``commit()`` more than once within the same :py:func:`db_session`. In this case the :py:func:`db_session` cache keeps the cached objects after commits. The cache will be cleaned up when the :py:func:`db_session` is over or if the transaction will be rolled back.

    .. py:method:: create_tables()

        Check the existing mapping and create tables for entities if they don’t exist. Also, Pony checks if foreign keys and indexes exist and create them if they are missing.

        This method can be useful if you need to create tables after they were deleted using the :py:meth:`~Database.drop_all_tables` method. If you don't delete tables, you probably don't need this method, because Pony checks and creates tables during :py:meth:`~Database.generate_mapping` call.



    .. py:method:: disconnect()

        Close the database connection for the current thread if it was opened.



    .. py:method:: drop_all_tables(with_all_data=False)

        Drop all tables which are related to the current mapping.

        :param bool with_all_data: ``False`` means Pony drops tables only if none of them contain any data. In case at least one of them is not empty, the method will raise the ``TableIsNotEmpty`` exception without dropping any table. In order to drop tables with data you should set ``with_all_data=True``.



    .. py:method:: drop_table(table_name, if_exists=False, with_all_data=False)

        Drop the ``table_name`` table.

        If you need to delete a table which is mapped to an entity, you can use the class method :py:meth:`~Entity.drop_table` of an entity.

        :param str table_name: the name of the table to be deleted, case sensitive.
        :param bool if_exists: when ``True``, it will not raise the ``TableDoesNotExist`` exception if there is no such table in the database.
        :param bool with_all_data: if the table is not empty the method will raise the ``TableIsNotEmpty`` exception.



    .. py:attribute:: Entity

        This attribute represents the base class which should be inherited by all entities which are mapped to the particular database.

        Example:

        .. code-block:: python

            db = Database()

            class Person(db.Entity):
                name = Required(str)
                age = Required(int)



    .. py:method:: execute(sql, globals=None, locals=None)

        Execute SQL statement.

        Before executing the provided SQL, Pony flushes all changes made within the current :py:func:`db_session` using the :py:meth:`~Database.flush` method.

        :param str sql: the SQL statement text.
        :param dict globals:
        :param dict locals: optional parameters which can contain dicts with variables and its values, used within the query.
        :return: a DBAPI cursor.

        Example:

        .. code-block:: python

            cursor = db.execute("""create table Person (
                         id integer primary key autoincrement,
                         name text,
                         age integer
                  )""")

            name, age = "Ben", 33
            cursor = db.execute("insert into Person (name, age) values ($name, $age)")

        See :ref:`Raw SQL <raw_sql>` section for more info.



    .. py:method:: exists(sql, globals=None, locals=None)

        Check if the database has at least one row which satisfies the query.

        Before executing the provided SQL, Pony flushes all changes made within the current :py:func:`db_session` using the :py:meth:`~Database.flush` method.

        :param str sql: the SQL statement text.
        :param dict globals:
        :param dict locals: optional parameters which can contain dicts with variables and its values, used within the query.
        :rtype: bool

        Example:

        .. code-block:: python

            name = 'John'
            if db.exists("select * from Person where name = $name"):
                print "Person exists in the database"



    .. py:method:: flush()

        Save the changes accumulated in the :py:func:`db_session` cache to the database. You may never have a need to call this method manually, because it will be done on leaving the :py:func:`db_session` automatically.

        Pony always saves the changes accumulated in the cache automatically before executing the following methods: :py:meth:`~Database.get`, :py:meth:`~Database.exists`, :py:meth:`~Database.execute`, :py:meth:`~Database.commit`, :py:meth:`~Database.select`.



    .. py:method:: generate_mapping(check_tables=True, create_tables=False)

        Map declared entities to the corresponding tables in the database. Creates tables, foreign key references and indexes if necessary.

        :param bool check_tables: when ``True``, Pony makes a simple check that the table names and attribute names in the database correspond to entities declaration. It doesn’t catch situations when the table has extra columns or when the type of a particular column doesn’t match. Set it to ``False`` if you want to generate mapping and create tables for your entities later, using the method :py:meth:`~Database.create_tables`.
        :param bool create_tables: create tables, foreign key references and indexes if they don’t exist. Pony generates the names of the database tables and columns automatically, but you can override this behavior if you want. See more details in the :ref:`Mapping customization <mapping_customization>` section.



    .. py:method:: get(sql, globals=None, locals=None)

        Select one row or just one value from the database.

        The ``get()`` method assumes that the query returns exactly one row. If the query returns nothing then Pony raises ``RowNotFound`` exception. If the query returns more than one row, the exception ``MultipleRowsFound`` will be raised.

        Before executing the provided SQL, Pony flushes all changes made within the current :py:func:`db_session` using the :py:meth:`~Database.flush` method.

        :param str sql: the SQL statement text.
        :param dict globals:
        :param dict locals: optional parameters which can contain dicts with variables and its values, used within the query.
        :return: a tuple or a value. If your request returns a lot of columns then you can assign the resulting tuple of the ``get()`` method to a variable and work with it the same way as it is described in :py:meth:`~Database.select` method.

        Example:

        .. code-block:: python

            id = 1
            age = db.get("select age from Person where id = $id")

            name, age = db.get("select name, age from Person where id = $id")



    .. py:method:: get_connection()

        Return the active database connection. It can be useful if you want to work with the DBAPI interface directly. This is the same connection which is used by the ORM itself. The connection will be reset and returned to the connection pool on leaving the :py:func:`db_session` context or when the database transaction rolls back. This connection can be used only within the :py:func:`db_session` scope where the connection was obtained.

        :return: a DBAPI connection.



    .. py:attribute:: global_stats

        This attribute keeps the dictionary where the statistics for executed SQL queries is aggregated from all threads. The key of this dictionary is the SQL statement and the value is an object of the :py:class:`QueryStat` class.



    .. py:method:: insert(table_name|entity, returning=None, **kwargs)

        Insert new rows into a table. This command bypasses the identity map cache and can be used in order to increase the performance when you need to create lots of objects and not going to read them in the same transaction. Also you can use the :py:meth:`~Database.execute` method for this purpose. If you need to work with those objects in the same transaction it is better to create instances of entities and have Pony to save them in the database.

        :param str table_name|entity: the name of the table where the data will be inserted. The name is case sensitive. Instead of the ``table_name`` you can use the ``entity`` class. In this case Pony will insert into the table associated with the ``entity``.
        :param str returning: the name of the column that holds the automatically generated primary key. If you want the ``insert()`` method to return the value which is generated by the database, you should specify the name of the primary key column.
        :param dict kwargs: named parameters used within the query.

        Example:

        .. code-block:: python

            new_id = db.insert("Person", name="Ben", age=33, returning='id')



    .. py:attribute:: last_sql

        Read-only attribute which keeps the text of the last SQL statement. It can be useful for debugging.



    .. py:attribute:: local_stats

        This is a dictionary which keeps the SQL query statistics for the current thread. The key of this dictionary is the SQL statement and the value is an object of the :py:class:`QueryStat` class.



    .. py:method:: merge_local_stats()

        Merge the statistics from the current thread into the global statistics. You can call this method at the end of the HTTP request processing.

        When you call this method, the value of :py:attr:`local_stats` will be merged to :py:attr:`global_stats`, and :py:attr:`local_stats` will be cleared.

        In a web application, you can call this method on finishing processing an HTTP request. This way the :py:attr:`global_stats` attribute will contain the statistics for the whole application.



    .. py:method:: rollback()

        Rolls back the current transaction and clears the :py:func:`db_session` cache.



    .. py:method:: select(sql, globals=None, locals=None)

        Execute the SQL statement in the database and returns a list of tuples.

        :param str sql: the SQL statement text.
        :param dict globals:
        :param dict locals: optional parameters which can contain dicts with variables and its values, used within the query.
        :return: a list of tuples.

        Example:

        .. code-block:: python

            result = select("select * from Person")

        If a query returns more than one column and the names of table columns are valid Python identifiers, then you can access them as attributes:

        .. code-block:: python

            for row in db.select("name, age from Person"):
                print row.name, row.age



Supported databases
-------------------

.. _sqlite:

SQLite
~~~~~~

Using SQLite database is the easiest way to work with Pony because there is no need to install a database system separately - the SQLite database system is included in the Python distribution. It is a perfect choice for beginners who want to experiment with Pony in the interactive shell. In order to bind the :py:class:`Database` object a SQLite database you can do the following:

.. code-block:: python

    db.bind(provider='sqlite', filename='db.sqlite', create_db=False)

.. py:method:: db.bind(provider, filename, create_db=False, timeout=5.0)

    :param str provider: Should be 'sqlite' for the SQLite database.
    :param str filename: The name of the file where SQLite will store the data. The filename can be absolute or relative. If you specify a relative path, that path is appended to the directory path of the Python file where this database was created (and not to the current working directory). This is because sometimes a programmer doesn’t have the control over the current working directory (e.g. in mod_wsgi application). This approach allows the programmer to create applications which consist of independent modules, where each module can work with a separate database. When working in the interactive shell, Pony requires that you to always specify the absolute path of the storage file.
    :param bool create_db: ``True`` means that Pony will try to create the database if such filename doesn’t exists.  If such filename exists, Pony will use this file.
    :param float timeout: The ``timeout`` parameter specifies how long the connection should wait for the lock to go away until raising an exception. The default is 5.0 (five seconds). *(New in version 0.7.3)*

Normally SQLite database is stored in a file on disk, but it also can be stored entirely in memory. This is a convenient way to create a SQLite database when playing with Pony in the interactive shell, but you should remember, that the entire in-memory database will be lost on program exit. Also you should not work with the same in-memory SQLite database simultaneously from several threads because in this case all threads share the same connection due to SQLite limitation.

  In order to bind with an in-memory database you should specify ``:memory:`` instead of the filename:

.. code-block:: python

      db.bind(provider='sqlite', filename=':memory:')

There is no need in the parameter ``create_db`` when creating an in-memory database.

.. note:: By default SQLite doesn’t check foreign key constraints. Pony always enables the foreign key support by sending the command ``PRAGMA foreign_keys = ON;`` starting with the release 0.4.9.

.. _postgresql:

PostgreSQL
~~~~~~~~~~

Pony uses psycopg2 driver in order to work with PostgreSQL. In order to bind the ``Database`` object to PostgreSQL use the following line:

.. code-block:: python

    db.bind(provider='postgres', user='', password='', host='', database='')

All the parameters that follow the Pony database provider name will be passed to the ``psycopg2.connect()`` method. Check the `psycopg2.connect documentation <http://initd.org/psycopg/docs/module.html#psycopg2.connect>`_ in order to learn what other parameters you can pass to this method.

.. _mysql:

MySQL
~~~~~

.. code-block:: python

    db.bind(provider='mysql', host='', user='', passwd='', db='')

Pony tries to use the MySQLdb driver for working with MySQL. If this module cannot be imported, Pony tries to use pymysql. See the `MySQLdb <http://mysql-python.sourceforge.net/MySQLdb.html#functions-and-attributes>`_ and `pymysql <https://pypi.python.org/pypi/PyMySQL>`_ documentation for more information regarding these drivers.

.. _oracle:

Oracle
~~~~~~

.. code-block:: python

    db.bind(provider='oracle', user='', password='', dsn='')

Pony uses the **cx_Oracle** driver for connecting to Oracle databases. More information about the parameters which you can use for creating a connection to Oracle database can be found `here <http://cx-oracle.sourceforge.net>`_.


Transactions & db_session
-------------------------

.. py:decorator:: db_session(allowed_exceptions=[], immediate=False, optimistic=True, retry=0, retry_exceptions=[TransactionError], serializable=False, strict=False, sql_debug=None, show_values=None)

    Used for establishing a database session.

    :param list allowed_exceptions: a list of exceptions which when occurred do not cause the transaction rollback. Can be useful with some web frameworks which trigger HTTP redirect with the help of an exception.
    :param bool immediate: tells Pony when start a transaction with the database. Some databases (e.g. SQLite, Postgres) start a transaction only when a modifying query is sent to the database(UPDATE, INSERT, DELETE) and don’t start it for SELECTs. If you need to start a transaction on SELECT, then you should set ``immediate=True``. Usually there is no need to change this parameter.
    :param bool optimistic: ``True`` by default. When ``optimistic=False``, no optimistic checks will be added to queries within this db_session *(new in version 0.7.3)*
    :param int retry: specifies the number of attempts for committing the current transaction. This parameter can be used with the ``@db_session`` decorator only. The decorated function should not call ``commit()`` or ``rollback()`` functions explicitly. When this parameter is specified, Pony catches the ``TransactionError`` exception (and all its descendants) and restarts the current transaction. By default Pony catches the ``TransactionError`` exception only, but this list can be modified using the ``retry_exceptions`` parameter.
    :param list|callable retry_exceptions: a list of exceptions which will cause the transaction restart. By default this parameter is equal to ``[TransactionError]``. Another option is using a callable which returns a boolean value. This callable receives the only parameter - an exception object. If this callable returns ``True`` then the transaction will be restarted.
    :param bool serializable: allows setting the SERIALIZABLE isolation level for a transaction.
    :param bool strict: when ``True`` the cache will be cleared on exiting the ``db_session``. If you'll try to access an object after the session is over, you'll get the ``pony.orm.core.DatabaseSessionIsOver`` exception. Normally Pony strongly advises that you work with entity objects only within the ``db_session``. But some Pony users want to access extracted objects in read-only mode even after the ``db_session`` is over. In order to provide this feature, by default, Pony doesn't purge cache on exiting from the ``db_session``. This might be handy, but in the same time, this can require more memory for keeping all objects extracted from the database in cache.
    :param bool sql_debug: when ``sql_debug=True`` - log SQL statements to the console or to a log file. When ``sql_debug=False`` - suppress logging, if it was set globally by :py:func:`set_sql_debug`. The default value ``None`` means it doesn't change the global debug mode. *(new in version 0.7.3)*
    :param bool show_values: when ``True``, query parameters will be logged in addition to the SQL text. *(new in version 0.7.3)*


    Can be used as a decorator or a context manager. When the session ends it performs the following actions:

    * Commits transaction if data was changed and no exceptions occurred otherwise it rolls back transaction.
    * Returns the database connection to the connection pool.
    * Clears the Identity Map cache.

    If you forget to specify the ``db_session`` where necessary, Pony will raise the ``TransactionError: db_session is required when working with the database`` exception.

    When you work with Python’s interactive shell you don’t need to worry about the database session, because it is maintained by Pony automatically.

    If you'll try to access instance's attributes which were not loaded from the database outside of the ``db_session`` scope, you'll get the ``DatabaseSessionIsOver`` exception. This happens because by this moment the connection to the database is already returned to the connection pool, transaction is closed and we cannot send any queries to the database.

    When Pony reads objects from the database it puts those objects to the Identity Map. Later, when you update an object’s attributes, create or delete an object, the changes will be accumulated in the Identity Map first. The changes will be saved in the database on transaction commit or before calling the following functions: :py:func:`get`, :py:func:`exists`, :py:func:`commit`, :py:func:`select`.

    Example of usage as a decorator:

    .. code-block:: python

        @db_session
        def check_user(username):
            return User.exists(username=username)

    As a context manager:

    .. code-block:: python

        def process_request():
            ...
            with db_session:
                u = User.get(username=username)
                ...


.. _transaction_isolation_levels:

Transaction isolation levels and database peculiarities
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Isolation is a property that defines when the changes made by one transaction become visible to other concurrent transactions `Isolation levels <http://en.wikipedia.org/wiki/Isolation_(database_systems)>`_.
The ANSI SQL standard defines four isolation levels:

* READ UNCOMMITTED - the most unsafe level
* READ COMMITTED
* REPEATABLE READ
* SERIALIZABLE     - the most safe level


When using the SERIALIZABLE level, each transaction sees the database as a snapshot made at the beginning of a transaction. This level provides the highest isolation, but it requires more resources than other levels.

This is the reason why most databases use a lower isolation level by default which allow greater concurrency. By default Oracle and PostgreSQL use READ COMMITTED, MySQL - REPEATABLE READ. SQLite supports the SERIALIZABLE level only, but Pony emulates the READ COMMITTED level for allowing greater concurrency.

If you want Pony to work with transactions using the SERIALIZABLE isolation level, you can do that by specifying the ``serializable=True`` parameter to the :py:func:`db_session` decorator or :py:func:`db_session` context manager:

.. code-block:: python

    @db_session(serializable=True)
    def your_function():
        ...

READ COMMITTED vs. SERIALIZABLE mode
````````````````````````````````````

In SERIALIZABLE mode, you always have a chance to get a “Can’t serialize access due to concurrent update” error, and would have to retry the transaction until it succeeded. You always need to code a retry loop in your application when you are using SERIALIZABLE mode for a writing transaction.

In READ COMMITTED mode, if you want to avoid changing the same data by a concurrent transaction, you should use SELECT FOR UPDATE. But this way there is a chance to have a `database deadlock <http://en.wikipedia.org/wiki/Deadlock>`_ - the situation where one transaction is waiting for a resource which is locked by another transaction. If your transaction got a deadlock, your application needs to restart the transaction. So you end up needing a retry loop either way. Pony can restart a transaction automatically if you specify the ``retry`` parameter to the :py:func:`db_session` decorator (but not the :py:func:`db_session` context manager):

.. code-block:: python

    @db_session(retry=3)
    def your_function():
        ...


SQLite
``````

When using SQLite, Pony’s behavior is similar as with PostgreSQL: when a transaction is started, selects will be executed in the autocommit mode. The isolation level of this mode is equivalent of READ COMMITTED. This way the concurrent transactions can be executed simultaneously with no risk of having a deadlock (the ``sqlite3.OperationalError: database is locked`` is not arising with Pony ORM). When your code issues non-select statement, Pony begins a transaction and all following SQL statements will be executed within this transaction. The transaction will have the SERIALIZABLE isolation level.


PostgreSQL
``````````

PostgreSQL uses the READ COMMITTED isolation level by default. PostgreSQL also supports the autocommit mode. In this mode each SQL statement is executed in a separate transaction. When your application just selects data from the database, the autocommit mode can be more effective because there is no need to send commands for beginning and ending a transaction, the database does it automatically for you. From the isolation point of view, the autocommit mode is nothing different from the READ COMMITTED isolation level. In both cases your application sees the data which have been committed by this moment.

Pony automatically switches from the autocommit mode and begins an explicit transaction when your application needs to modify data by several INSERT, UPDATE or DELETE SQL statements in order to provide atomicity of data update.


MySQL
`````

MySQL uses the REPEATABLE READ isolation level by default. Pony doesn’t use the autocommit mode with MySQL because there is no benefit of using it here. The transaction begins with the first SQL statement sent to the database even if this is a SELECT statement.


Oracle
``````

Oracle uses the READ COMMITTED isolation level by default. Oracle doesn’t have the autocommit mode. The transaction begins with the first SQL statement sent to the database even if this is a SELECT statement.





.. _entity_definition:

Entity definition
-----------------

An entity is a Python class which stores an object’s state in the database. Each instance of an entity corresponds to a row in the database table. Often entities represent objects from the real world (e.g. Customer, Product).

Entity attributes
~~~~~~~~~~~~~~~~~

Entity attributes are specified as class attributes inside the entity class using the syntax:

.. code-block:: python

    class EntityName(inherits_from)
        attr_name = attr_kind(attr_type, attr_options)

For example:

.. code-block:: python

    class Person(db.Entity):
        id = PrimaryKey(int, auto=True)
        name = Required(str)
        age = Optional(int)


Attribute kinds
~~~~~~~~~~~~~~~

Each entity attribute can be one of the following kinds:

* ``Required`` - must have a value at all times
* ``Optional`` - the value is optional
* ``PrimaryKey`` - defines a primary key attribute
* ``Set`` - represents a collection, used for 'to-many' relationships
* ``Discriminator`` - used for entity inheritance


Optional string attributes
``````````````````````````

For most data types ``None`` is used when no value is assigned to the attribute. But when a string attribute is not assigned a value, Pony uses an empty string instead of ``None``. This is more practical than storing empty string as ``NULL`` in the database. Most frameworks behave this way. Also, empty strings can be indexed for faster search, unlike NULLs. If you will try to assign ``None`` to such an optional string attribute, you’ll get the ``ConstraintError`` exception.

You can change this behavior using the ``nullable=True`` option. In this case it will be possible to store both empty strings and ``NULL`` values in the same column, but this is rarely needed.

Oracle database treats empty strings as ``NULL`` values. Because of this all ``Optional`` attributes in Oracle have ``nullable`` set to ``True`` automatically.

If an optional string attribute is used as a unique key or as a part of a unique composite key, it will always have ``nullable`` set to ``True`` automatically.


.. _composite_keys:

Composite primary and secondary keys
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Pony fully supports composite keys. In order to declare a composite primary key you need to specify all the parts of the key as ``Required`` and then combine them into a composite primary key:

.. code-block:: python

    class Example(db.Entity):
        a = Required(int)
        b = Required(str)
        PrimaryKey(a, b)

In order to declare a secondary composite key you need to declare attributes as usual and then combine them using the ``composite_key`` directive:

.. code-block:: python

    class Example(db.Entity):
        a = Required(str)
        b = Optional(int)
        composite_key(a, b)

In the database ``composite_key(a, b)`` will be represented as the ``UNIQUE ("a", "b")`` constraint.


.. _composite_indexes:

Composite indexes
~~~~~~~~~~~~~~~~~

Using the ``composite_index()`` directive you can create a composite index for speeding up data retrieval. It can combine two or more attributes:

.. code-block:: python

    class Example(db.Entity):
        a = Required(str)
        b = Optional(int)
        composite_index(a, b)

The composite index can include a discriminator attribute used for inheritance.

Using the ``composite_index()`` you can create a non-unique index. In order to define an unique index, use the ``composite_key()`` function described above.



.. _attribute_types:

Attribute data types
~~~~~~~~~~~~~~~~~~~~

Pony supports the following attribute types:

* str
* unicode
* int
* float
* Decimal
* datetime
* date
* time
* timedelta
* bool
* buffer - used for binary data in Python 2 and 3
* bytes - used for binary data in Python 3
* LongStr - used for large strings
* LongUnicode - used for large strings
* UUID
* Json - used for mapping to native database JSON type

Also you can specify another entity as the attribute type for defining a relationship between two entities.


Strings in Python 2 and 3
`````````````````````````

As you know, Python 3 has some differences from Python 2 when it comes to strings. Python 2 provides two string types – ``str`` (byte string) and ``unicode`` (unicode string), whereas in Python 3 the ``str`` type represents unicode strings and the ``unicode`` was just removed.

Before the release 0.6, Pony stored ``str`` and ``unicode`` attributes as unicode in the database, but for ``str`` attributes it had to convert unicode to byte string on reading from the database. Starting with the Pony Release 0.6 the attributes of ``str`` type in Python 2 behave as if they were declared as ``unicode`` attributes. There is no difference now if you specify ``str`` or ``unicode`` as the attribute type – you will have unicode string in Python and in the database.

Starting with the Pony Release 0.6, where the support for Python 3 was added, instead of ``unicode`` and ``LongUnicode`` we recommend to use ``str`` and ``LongStr`` types respectively. ``LongStr`` and ``LongUnicode``  are stored as CLOB in the database.

The same thing is with the ``LongUnicode`` and ``LongStr``. ``LongStr`` now is an alias to ``LongUnicode``. This type uses unicode in Python and in the database.

.. code-block:: python

    attr1 = Required(str)
    # is the same as
    attr2 = Required(unicode)

    attr3 = Required(LongStr)
    # is the same as
    attr4 = Required(LongUnicode)


Buffer and bytes types in Python 2 and 3
````````````````````````````````````````

If you need to represent byte sequence in Python 2, you can use the ``buffer`` type. In Python 3 you should use the ``bytes`` type for this purpose. ``buffer`` and ``bytes`` types are stored as binary (BLOB) types in the database.

In Python 3 the ``buffer`` type has gone, and Pony uses the ``bytes`` type which was added in Python 3 to represent binary data. But for the sake of backward compatibility we still keep ``buffer`` as an alias to the ``bytes`` type in Python 3. If you're importing ``*`` from ``pony.orm`` you will get this alias too.

If you want to write code which can run both on Python 2 and Python 3, you should use the ``buffer`` type for binary attributes. If your code is for Python 3 only, you can use ``bytes`` instead:

.. code-block:: python

    attr1 = Required(buffer) # Python 2 and 3

    attr2 = Required(bytes) # Python 3 only

It would be cool if we could use the ``bytes`` type as an alias to ``buffer`` in Python 2, but unfortunately it is impossible, because `Python 2.6 adds bytes as a synonym for the str type`_.

.. _Python 2.6 adds bytes as a synonym for the str type: https://docs.python.org/2/whatsnew/2.6.html#pep-3112-byte-literals>



.. _attribute_options:

Attribute options
~~~~~~~~~~~~~~~~~

Attribute options can be specified as positional and as keyword arguments during an attribute definition.


Max string length
`````````````````

String types can accept a positional argument which specifies the max length of this column in the database:

.. code-block:: python

    class Person(db.Entity):
        name = Required(str, 40)   #  VARCHAR(40)

Also you can use the ``max_len`` option:

.. code-block:: python

    class Person(db.Entity):
        name = Required(str, max_len=40)   #  VARCHAR(40)


Decimal scale and precision
```````````````````````````

For the ``Decimal`` type you can specify precision and scale:

.. code-block:: python

    class Product(db.Entity):
        price = Required(Decimal, 10, 2)   #  DECIMAL(10, 2)

Also you can use ``precision`` and ``scale`` options:

.. code-block:: python

    class Product(db.Entity):
        price = Required(Decimal, precision=10, scale=2)   #  DECIMAL(10, 2)


Datetime, time and timedelta precision
``````````````````````````````````````

The ``datetime`` and ``time`` types accept a positional argument which specifies the column's precision. By default it is equal to 6 for most databases.

For MySQL database the default value is 0. Before the MySQL version 5.6.4, the ``DATETIME`` and ``TIME`` columns were `unable to store fractional seconds at all`_. Starting with the version 5.6.4, you can store fractional seconds if you set the precision equal to 6 during the attribute definition:

.. _unable to store fractional seconds at all: http://dev.mysql.com/doc/refman/5.6/en/fractional-seconds.html

.. code-block:: python

    class Action(db.Entity):
        dt = Required(datetime, 6)

The same, using the ``precision`` option:

.. code-block:: python

    class Action(db.Entity):
        dt = Required(datetime, precision=6)


Keyword argument options
````````````````````````

Additional attribute options can be set as keyword arguments. For example:

.. code-block:: python

    class Customer(db.Entity):
        email = Required(str, unique=True)


Below you can find the list of available options:


.. option:: auto

    (*bool*) Can be used for a PrimaryKey attribute only. If ``auto=True`` then the value for this attribute will be assigned automatically using the database’s incremental counter or sequence.


.. option:: autostrip

    (*bool*) Automatically removes leading and trailing whitespace characters in a string attribute. Similar to Python ``string.strip()`` function. By default is ``True``.


.. option:: cascade_delete

    (*bool*) Controls the cascade deletion of related objects. ``True`` means that Pony always does cascade delete even if the other side is defined as ``Optional``. ``False`` means that Pony never does cascade delete for this relationship. If the relationship is defined as ``Required`` at the other end and ``cascade_delete=False`` then Pony raises the ``ConstraintError`` exception on deletion attempt. :ref:`See also <cascade_delete>`.


.. option:: column

    (*str*) Specifies the name of the column in the database table which is used for mapping. By default Pony uses the attribute name as the column name in the database.


.. option:: columns

    (*list*) Specifies the column names in the database table which are used for mapping a composite attribute.


.. option:: default

    (*numeric|str|function*) Allows specifying a default value for the attribute. Pony processes default values in Python, it doesn't add SQL DEFAULT clause to the column definition. This is because the default expression can be not only a constant, but any arbitrary Python function. For example:

    .. code-block:: python

        import uuid
        from pony.orm import *

        db = Database()

        class MyEntity(db.Entity):
            code = Required(uuid.UUID, default=uuid.uuid4)

    If you need to set a default value in the database, you should use the ``sql_default`` option.


.. option:: fk_name

    (*str*) Applies for ``Required`` and ``Optional`` relationship attributes, allows to specify the name of the foreign key in the database.

.. option:: index

    (*bool|str*) Allows to control index creation for this column. ``index=True`` - the index will be created with the default name. ``index='index_name'`` - create index with the specified name. ``index=False`` – skip index creation. If no 'index' option is specified then Pony still creates index for foreign keys using the default name.


.. option:: lazy

    (*bool*) When ``True``, then Pony defers loading the attribute value when loading the object. The value will not be loaded until you try to access this attribute directly. By default ``lazy`` is set to ``True`` for ``LongStr`` and ``LongUnicode`` and to ``False`` for all other types.


.. option:: max

    (*numeric*) Allows specifying the maximum allowed value for numeric attributes (int, float, Decimal). If you will try to assign the value that is greater than the specified max value, you'll get the ``ValueError`` exception.


.. option:: max_len

    (*int*) Sets the maximum length for string attributes.


.. option:: min

    (*numeric*) Allows specifying the minimum allowed value for numeric attributes (int, float, Decimal). If you will try to assign the value that is less than the specified min value, you'll get the ``ValueError`` exception.


.. option:: nplus1_threshold

    (*int*) This parameter is used for fine tuning the threshold used for the N+1 problem solution.


.. option:: nullable

    (*bool*) ``True`` allows the column to be ``NULL`` in the database. Most likely you don't need to specify this option because Pony sets it to the most appropriate value by default.

.. _optimistic_option:

.. option:: optimistic

    (*bool*) ``True`` means this attribute will be used for automatic optimistic checks, :ref:`see Optimistic concurrency control <optimistic_control>` section. By default, this option is set to ``True`` for all attributes except attributes of ``float`` type - for ``float`` type attributes it is set to ``False`` by default.

    See also :ref:`volatile option <volatile_option>`.


.. option:: precision

    (*int*) Sets the precision for ``Decimal``, ``time``, ``timedelta``, ``datetime`` attribute types.

.. option:: py_check

    (*function*) Allows to specify a function which will be used for checking the value before it is assigned to the attribute. The function should return ``True`` or ``False``. Also it can raise the ``ValueError`` exception if the check failed.

    .. code-block:: python

        class Student(db.Entity):
            name = Required(str)
            gpa = Required(float, py_check=lambda val: val >= 0 and val <= 5)


.. option:: reverse

    (*str*) Specifies the attribute name at the other end which should be used for the relationship. It might be needed if there are more than one relationship between two entities.


.. option:: reverse_column

    (*str*) Used for a symmetric relationship in order to specify the name of the database column for the intermediate table.


.. option:: reverse_columns

    (*str*) Used for a symmetric relationship if the entity has a composite primary key. Allows you to specify the name of the database columns for the intermediate table.


.. option:: scale

    (*int*) Sets the scale for ``Decimal`` attribute types.

.. option:: size

    (*int*) For the ``int`` type you can specify the size of integer type that should be used in the database using the ``size`` keyword. This parameter receives the number of bits that should be used for representing an integer in the database. Allowed values are 8, 16, 24, 32 and 64:

    .. code-block:: python

        attr1 = Required(int, size=8)   # 8 bit - TINYINT in MySQL
        attr2 = Required(int, size=16)  # 16 bit - SMALLINT in MySQL
        attr3 = Required(int, size=24)  # 24 bit - MEDIUMINT in MySQL
        attr4 = Required(int, size=32)  # 32 bit - INTEGER in MySQL
        attr5 = Required(int, size=64)  # 64 bit - BIGINT in MySQL

    You can use the ``unsigned`` parameter to specify that the attribute is unsigned:

    .. code-block:: python

        attr1 = Required(int, size=8, unsigned=True) # TINYINT UNSIGNED in MySQL

    The default value of the ``unsigned`` parameter is ``False``. If ``unsigned`` is set to ``True``, but ``size`` is not provided, ``size`` assumed to be 32 bits.

    If current database does not support specified attribute size, the next bigger size is used. For example, PostgreSQL does not have ``MEDIUMINT`` numeric type, so ``INTEGER`` type will be used for an attribute with size 24.

    Only MySQL actually supports unsigned types. For other databases the column will use signed numeric type which can hold all valid values for the specified unsigned type. For example, in PostgreSQL an unsigned attribute with size 16 will use ``INTEGER`` type. An unsigned attribute with size 64 can be represented only in MySQL and Oracle.

    When the size is specified, Pony automatically assigns ``min`` and ``max`` values for this attribute. For example, a signed attribute with size 8 will receive ``min`` value -128 and ``max`` value 127, while unsigned attribute with the same size will receive ``min`` value 0 and ``max`` value 255. You can override ``min`` and ``max`` with your own values if necessary, but these values should not exceed the range implied by the size.

    Starting with the Pony release 0.6 the ``long`` type is deprecated and if you want to store 64 bit integers in the database, you need to use ``int`` instead with ``size=64``. If you don't specify the ``size`` parameter, Pony will use the default integer type for the specific database.


.. option:: sequence_name

    (*str*) Allows to specify the sequence name used for ``PrimaryKey`` attributes. *Oracle database only.*


.. option:: sql_default

    (*str*) This option allows specifying the default SQL text which will be included to the CREATE TABLE SQL command. For example:

    .. code-block:: python

        class MyEntity(db.Entity):
            created_at = Required(datetime, sql_default='CURRENT_TIMESTAMP')
            closed = Required(bool, default=True, sql_default='1')

    Specifying ``sql_default=True`` can be convenient when you have a ``Required`` attribute and the value for it is going to be calculated in the database during the INSERT command (e.g. by a trigger). ``None`` by default.


.. option:: sql_type

    (*str*) Sets a specific SQL type for the column.


.. option:: unique

    (*bool*) If ``True``, then the database will check that the value of this attribute is unique.


.. option:: unsigned

    (*bool*) Allows creating unsigned types in the database. Also checks that the assigned value is positive.


.. option:: table

    (*str*) Used for many-to-many relationship only in order to specify the name of the intermediate table.

.. _volatile_option:

.. option:: volatile

    (*bool*) Usually you specify the value of the attribute in Python and Pony stores this value in the database. But sometimes you might want to have some logic in the database which changes the value for a column. For example, you can have a trigger in the database which updates the timestamp of the last object's modification. In this case you want to have Pony to forget the value of the attribute on object's update sent to the database and read it from the database at the next access attempt. Set ``volatile=True`` in order to let Pony know that this attribute can be changed in the database.

    The ``volatile=True`` option can be combined with the ``sql_default`` option if the value for this attribute is going to be both created and updated by the database.

    You can get the exception ``UnrepeatableReadError: Value ... was updated outside of current transaction`` if another transaction changes the value of the attribute which is used in the current transaction. Pony notifies about it because this situation can break the business logic of the application. If you don't want Pony to protect you from such concurrent modifications you can set ``volatile=True`` for the attribute. This will turn the optimistic concurrency control off.

    See also :ref:`optimistic option <optimistic_option>`.


.. _collection_attribute_methods:

Collection attribute methods
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To-many attributes have methods that provide a convenient way of querying data. You can treat a to-many relationship attribute as a regular Python collection and use standard operations like ``in``, ``not in``, ``len``. Also Pony provides the following methods:

.. class:: Set

    .. py:method:: __len__

        Return the number of objects in the collection. If the collection is not loaded into cache, this methods loads all the collection instances into the cache first, and then returns the number of objects. Use this method if you are going to iterate over the objects and you need them loaded into the cache. If you don't need the collection to be loaded into the memory, you can use the :py:meth:`~Set.count` method.

        .. code-block:: python

            >>> p1 = Person[1]
            >>> Car[1] in p1.cars
            True
            >>> len(p1.cars)
            2


    .. py:method:: add(item|iter)

        Add instances to a collection and establish a two-way relationship between entity instances:

        .. code-block:: python

            photo = Photo[123]
            photo.tags.add(Tag['Outdoors'])

        Now the instance of the ``Photo`` entity with the primary key 123 has a relationship with the ``Tag['Outdoors']`` instance. The attribute ``photos`` of the ``Tag['Outdoors']`` instance contains the reference to the ``Photo[123]`` as well.

        You can also establish several relationships at once passing the list of tags to the ``add()`` method:

        .. code-block:: python

            photo.tags.add([Tag['Party'], Tag['New Year']])


    .. py:method:: clear()

        Remove all items from the collection which means breaking relationships between entity instances.


    .. py:method:: copy()

        Return a Python ``set`` object which contains the same items as the given collection.


    .. py:method:: count()

        Return the number of objects in the collection. This method doesn't load the collection instances into the cache, but generates an SQL query which returns the number of objects from the database. If you are going to work with the collection objects (iterate over the collection or change the object attributes), you might want to use the :py:meth:`~Set.__len__` method.


    .. py:method:: create(**kwargs)

        Create an return an instance of the related entity and establishes a relationship with it:

        .. code-block:: python

            new_tag = Photo[123].tags.create(name='New tag')

        is an equivalent of the following:

        .. code-block:: python

          new_tag = Tag(name='New tag')
          Photo[123].tags.add(new_tag)


    .. py:method:: drop_table(with_all_data=False)

        Drop the intermediate table which is created for establishing many-to-many relationship. If the table is not empty and ``with_all_data=False``, the method raises the ``TableIsNotEmpty`` exception and doesn't delete anything. Setting the ``with_all_data=True`` allows you to delete the table even if it is not empty.

        .. code-block:: python

            class Product(db.Entity):
                tags = Set('Tag')

            class Tag(db.Entity):
                products = Set(Product)

            Product.tags.drop_table(with_all_data=True) # removes the intermediate table


    .. py:method:: is_empty()

        Check if the collection is empty. Returns ``False`` if there is at lease one relationship and ``True`` if this attribute has no relationships.


    .. py:method:: filter()

        Select objects from a collection. The method names :py:meth:`~Set.select` and :py:meth:`~Set.filter` are synonyms. Example:

        .. code-block:: python

            g = Group[101]
            g.students.filter(lambda student: student.gpa > 3)


    .. py:method:: load()

        Load all related objects from the database.


    .. py:method:: order_by(attr|lambda)

        Return an ordered collection.

        .. code-block:: python

            g.students.order_by(Student.name).page(2, pagesize=3)
            g.students.order_by(lambda s: s.name).limit(3, offset=3)


    .. py:method:: sort_by(attr|lambda)

        Return an ordered collection. For a collection, the ``sort_by`` method works the same way as :py:func:`order_by`.

        .. code-block:: python

            g.students.sort_by(Student.name).page(2, pagesize=3)
            g.students.sort_by(lambda s: s.name).limit(3, offset=3)


    .. py:method:: page(pagenum, pagesize=10)

        This query can be used for displaying the second page of group 101 student's list ordered by the ``name`` attribute:

        .. code-block:: python

            g.students.order_by(Student.name).page(2, pagesize=3)
            g.students.order_by(lambda s: s.name).limit(3, offset=3)


    .. py:method:: random(limit)

        Return a number of random objects from a collection.

        .. code-block:: python

            g = Group[101]
            g.students.random(2)


    .. py:method:: remove(item|iter)

        Remove an item or items from the collection and thus break the relationship between entity instances.


    .. py:method:: select()

        Select objects from a collection. The method names :py:meth:`~Set.select` and :py:meth:`~Set.filter` are synonyms. Example:

        .. code-block:: python

            g = Group[101]
            g.students.select(lambda student: student.gpa > 3)

.. _entity_options:

Entity options
~~~~~~~~~~~~~~

.. py:function:: composite_index(attrs)

    Combine an index from multiple attributes. :ref:`Link <composite_indexes>`.


.. py:function:: composite_key(attrs)

    Combine a secondary key from multiple attributes. :ref:`Link <composite_keys>`.


.. py:attribute:: _discriminator_

    Specify the discriminator value for an entity. See more information in the :ref:`Entity inheritance <entity_inheritance>` section.


.. py:function:: PrimaryKey(attrs)

    Combine a primary key from multiple attributes. :ref:`Link <composite_keys>`.


.. py:attribute:: _table_

    Specify the name of mapped table in the database. See more information in the :ref:`Mapping customization <mapping_customization>` section.


.. py:attribute:: _table_options_

    All parameters specified here will be added as plain text at the end of the `CREATE TABLE` command. Example:

    .. code-block:: python

        class MyEntity(db.Entity):
            id = PrimaryKey(int)
            foo = Required(str)
            bar = Optional(int)

            _table_options_ = {
                'ENGINE': 'InnoDB',
                'TABLESPACE': 'my_tablespace',
                'ENCRYPTION': "'N'",
                'AUTO_INCREMENT': 10
            }


.. _entity_hooks:

Entity hooks
~~~~~~~~~~~~

Sometimes you might need to perform an action before or after your entity instance is going to be created, updated or deleted in the database. For this purpose you can use entity hooks.

Here is the list of available hooks:

.. py:method:: after_delete()

    Called after the entity instance is deleted in the database.

.. py:method:: after_insert()

    Called after the row is inserted into the database.

.. py:method:: after_update()

    Called after the instance updated in the database.

.. py:method:: before_delete()

    Called before deletion the entity instance in the database.

.. py:method:: before_insert()

    Called only for newly created objects before it is inserted into the database.

.. py:method:: before_update()

    Called for entity instances before updating the instance in the database.

In order to use a hook, you need to define an entity method with the hook name:

    .. code-block:: python

        class Message(db.Entity):
            title = Required(str)
            content = Required(str)

            def before_insert(self):
                print("Before insert! title=%s" % self.title)

Each hook method receives the instance of the object to be modified. You can check how it works in the interactive mode:

    .. code-block:: python

        >>> m = Message(title='First message', content='Hello, world!')
        >>> commit()
        Before insert! title=First message

        INSERT INTO "Message" ("title", "content") VALUES (?, ?)
        [u'First message', u'Hello, world!']


.. _entity_methods:

Entity methods
--------------

.. class:: Entity

    .. py:classmethod:: __getitem__

        Return an entity instance selected by its primary key. Raises the ``ObjectNotFound`` exception if there is no such object. Example:

        .. code-block:: python

            p = Product[123]

        For entities with a composite primary key, use a comma between the primary key values:

        .. code-block:: python

            item = OrderItem[123, 456]

        If object with the specified primary key was already loaded into the :py:func:`db_session` cache, Pony returns the object from the cache without sending a query to the database.


    .. py:method:: delete()

        Delete the entity instance. The instance will be marked as deleted and then will be deleted from the database during the :py:func:`flush` function, which is issued automatically on committing the current transaction when exiting from the most outer :py:func:`db_session` or before sending the next query to the database.

        .. code-block:: python

            Order[123].delete()


    .. py:classmethod:: describe()

        Return a string with the entity declaration.

        .. code-block:: python

            >>> print(OrderItem.describe())

            class OrderItem(Entity):
                quantity = Required(int)
                price = Required(Decimal)
                order = Required(Order)
                product = Required(Product)
                PrimaryKey(order, product)


    .. py:classmethod:: drop_table(with_all_data=False)

        Drops the table which is associated with the entity in the database. If the table is not empty and ``with_all_data=False``, the method raises the ``TableIsNotEmpty`` exception and doesn't delete anything. Setting the ``with_all_data=True`` allows you to delete the table even if it is not empty.

        If you need to delete an intermediate table created for many-to-many relationship, you have to call the method :py:meth:`~Set.select` of the relationship attribute.



    .. py:classmethod:: exists(*args, **kwargs)

        Returns ``True`` if an instance with the specified condition or attribute values exists and ``False`` otherwise.

        .. code-block:: python

            Product.exists(price=1000)
            Product.exists(lambda p: p.price > 1000)


    .. py:method:: flush()

        Save the changes made to this object to the database. Usually Pony saves changes automatically and you don't need to call this method yourself. One of the use cases when it might be needed is when you want to get the primary key value of a newly created object which has autoincremented primary key before commit.


    .. py:classmethod:: get(*args, **kwargs)

        Extract one entity instance from the database.

        If the object with the specified parameters exists, then returns the object. Returns ``None`` if there is no such object. If there are more than one objects with the specified parameters, raises the ``MultipleObjectsFoundError: Multiple objects were found. Use select(...) to retrieve them`` exception. Examples:

        .. code-block:: python

            Product.get(price=1000)
            Product.get(lambda p: p.name.startswith('A'))


    .. py:classmethod:: get_by_sql(sql, globals=None, locals=None)

        Select entity instance by raw SQL.

        If you find that you cannot express a query using the standard Pony queries, you always can write your own SQL query and Pony will build an entity instance(s) based on the query results. When Pony gets the result of the SQL query, it analyzes the column names which it receives from the database cursor. If your query uses ``SELECT * ...`` from the entity table, that would be enough for getting the necessary attribute values for constructing entity instances. You can pass parameters into the query, see :ref:`Using the select_by_sql() and get_by_sql() methods <entities_raw_sql_ref>` for more information.


    .. py:classmethod:: get_for_update(*args, **kwargs, nowait=False)

        :param bool nowait: prevent the operation from waiting for other transactions to commit. If a selected row(s) cannot be locked immediately, the operation reports an error, rather than waiting.

        Locks the row in the database using the ``SELECT ... FOR UPDATE`` SQL query. If ``nowait=True``, then the method will throw an exception if this row is already blocked. If ``nowait=False``, then it will wait if the row is already blocked.

        If you need to use ``SELECT ... FOR UPDATE`` for multiple rows then you should use the :py:meth:`~Query.for_update` method.


    .. py:method:: get_pk()

        Get the value of the primary key of the object.

        .. code-block:: python

            >>> c = Customer[1]
            >>> c.get_pk()
            1

        If the primary key is composite, then this method returns a tuple consisting of primary key column values.

        .. code-block:: python

            >>> oi = OrderItem[1,4]
            >>> oi.get_pk()
            (1, 4)

    .. py:method:: load(*args)

        Load all lazy and non-lazy attributes, but not collection attributes, which were not retrieved from the database yet. If an attribute was already loaded, it won't be loaded again. You can specify the list of the attributes which need to be loaded, or it's names. In this case Pony will load only them:

        .. code-block:: python

            obj.load(Person.biography, Person.some_other_field)
            obj.load('biography', 'some_other_field')


    .. py:classmethod:: select(lambda)

        Select objects from the database in accordance with the condition specified in lambda, or all objects if lambda function is not specified.

        The ``select()`` method returns an instance of the :py:class:`Query` class. Entity instances will be retrieved from the database once you start iterating over the ``Query`` object.

        This query example returns all products with the price greater than 100 and which were ordered more than once:

        .. code-block:: python

            Product.select(lambda p: p.price > 100 and count(p.order_items) > 1)[:]


    .. py:classmethod:: select_by_sql(sql, globals=None, locals=None)

        Select entity instances by raw SQL. See :ref:`Using the select_by_sql() and get_by_sql() methods <entities_raw_sql_ref>` for more information.


    .. py:classmethod:: select_random(limit)

        Select ``limit`` random objects. This method uses the algorithm that can be much more effective than using ``ORDER BY RANDOM()`` SQL construct. The method uses the following algorithm:

        1. Determine max id from the table.

        2. Generate random ids in the range (0, max_id]

        3. Retrieve objects by those random ids. If an object with generated id does not exist (e.g. it was deleted), then select another random id and retry.

        Repeat the steps 2-3 as many times as necessary to retrieve the specified amount of objects.

        This algorithm doesn't affect performance even when working with a large number of table rows. However this method also has some limitations:

        * The primary key must be a sequential id of an integer type.

        * The number of "gaps" between existing ids (the count of deleted objects) should be relatively small.

        The ``select_random()`` method can be used if your query does not have any criteria to select specific objects. If such criteria is necessary, then you can use the :py:meth:`Query.random` method.


    .. py:method:: set(**kwargs)

        Assign new values to several object attributes at once:

        .. code-block:: python

            Customer[123].set(email='new@example.com', address='New address')

        This method also can be convenient when you want to assign new values from a dictionary:

        .. code-block:: python

            d = {'email': 'new@example.com', 'address': 'New address'}
            Customer[123].set(**d)


    .. py:method:: to_dict(only=None, exclude=None, with_collections=False, with_lazy=False, related_objects=False)

        Return a dictionary with attribute names and its values. This method can be used when you need to serialize an object to JSON or other format.

        By default this method doesn't include collections (to-many relationships) and lazy attributes. If an attribute's values is an entity instance then only the primary key of this object will be added to the dictionary.

        :param list|str only: use this parameter if you want to get only the specified attributes. This argument can be used as a first positional argument. You can specify a list of attribute names ``obj.to_dict(['id', 'name'])``, a string separated by spaces: ``obj.to_dict('id name')``, or a string separated by spaces with commas: ``obj.to_dict('id, name')``.
        :param list|str exclude: this parameter allows you to exclude specified attributes. Attribute names can be specified the same way as for the ``only`` parameter.
        :param bool related_objects: by default, all related objects represented as a primary key. If ``related_objects=True``, then objects which have relationships with the current object will be added to the resulting dict as objects, not their primary keys. It can be useful if you want to walk the related objects and call the ``to_dict()`` method recursively.
        :param bool with_collections: by default, the resulting dictionary will not contain collections (to-many relationships). If you set this parameter to ``True``, then the relationships to-many will be represented as lists. If ``related_objects=False`` (which is by default), then those lists will consist of primary keys of related instances. If ``related_objects=True`` then to-many collections will be represented as lists of objects.
        :param bool with_lazy: if ``True``, then lazy attributes (such as BLOBs or attributes which are declared with ``lazy=True``) will be included to the resulting dict.
        :param bool related_objects: By default all related objects are represented as a list with their primary keys only. If you want to see the related objects instances, you can specify ``related_objects=True``.

        For illustrating the usage of this method we will use the eStore example which comes with Pony distribution. Let's get a customer object with the id=1 and convert it to a dictionary:

        .. code-block:: python

            >>> from pony.orm.examples.estore import *
            >>> c1 = Customer[1]
            >>> c1.to_dict()

            {'address': u'address 1',
            'country': u'US',
            'email': u'john@example.com',
            'id': 1,
            'name': u'John Smith',
            'password': u'***'}

        If we don't want to serialize the password attribute, we can exclude it this way:

        .. code-block:: python

            >>> c1.to_dict(exclude='password')

            {'address': u'address 1',
            'country': u'US',
            'email': u'john@example.com',
            'id': 1,
            'name': u'John Smith'}

        If you want to exclude more than one attribute, you can specify them as a list: ``exclude=['id', 'password']`` or as a string: ``exclude='id, password'`` which is the same as ``exclude='id password'``.

        Also you can specify only the attributes, which you want to serialize using the parameter ``only``:

        .. code-block:: python

            >>> c1.to_dict(only=['id', 'name'])

            {'id': 1, 'name': u'John Smith'}

            >>> c1.to_dict('name email') # 'only' parameter as a positional argument

            {'email': u'john@example.com', 'name': u'John Smith'}

        By default the collections are not included to the resulting dict. If you want to include them, you can specify ``with_collections=True``. Also you can specify the collection attribute in the ``only`` parameter:

        .. code-block:: python

            >>> c1.to_dict(with_collections=True)

            {'address': u'address 1',
            'cart_items': [1, 2],
            'country': u'USA',
            'email': u'john@example.com',
            'id': 1,
            'name': u'John Smith',
            'orders': [1, 2],
            'password': u'***'}

        By default all related objects (cart_items, orders) are represented as a list with their primary keys. If you want to see the related objects instances, you can specify ``related_objects=True``:

        .. code-block:: python

            >>> c1.to_dict(with_collections=True, related_objects=True)

            {'address': u'address 1',
            'cart_items': [CartItem[1], CartItem[2]],
            'country': u'USA',
            'email': u'john@example.com',
            'id': 1,
            'name': u'John Smith',
            'orders': [Order[1], Order[2]],
            'password': u'***'}


.. _queries_and_functions:

Queries and functions
---------------------

Below is the list of upper level functions defined in Pony:

.. py:function:: avg(gen)

    Return the average value for all selected attributes.

    :param generator gen: Python generator expression
    :rtype: numeric

    .. code-block:: python

        avg(o.total_price for o in Order)

    The equivalent query can be generated using the :py:meth:`~Query.avg` method.

.. py:function:: between(x, a, b)

    This function will be translated into ``x BETWEEN a AND b``. It is equal to the condition ``x >= a AND x <= b``.

    .. code-block:: python

        select(p for p in Person if between(p.age, 18, 65))

.. py:function:: coalesce(*args)

    :param list args: list of arguments

    Returns the first non-null expression in a list.

    .. code-block:: python

        select(coalesce(p.phone, 'UNKNOWN') for p in Person)

.. py:function:: concat(*args)

    :param list args: list of arguments

    Concatenates arguments into one string.

    .. code-block:: python

        select(concat(p.first_name, ' ', p.last_name) for p in Person)

.. py:function:: commit()

    Save all changes which were made within the current :py:func:`db_session` using the :py:func:`flush` function and commits the transaction to the database. This top level :py:func:`commit` function calls the :py:meth:`~Database.commit` method of each database object which was used in current transaction.


.. py:function:: count(gen)

    Return the number of objects that match the query condition.

    :param generator gen: Python generator expression
    :rtype: numeric

    .. code-block:: python

        count(c for c in Customer if len(c.orders) > 2)

    This query will be translated to the following SQL:

    .. code-block:: sql

        SELECT COUNT(*)
        FROM "Customer" "c"
        LEFT JOIN "Order" "order-1"
          ON "c"."id" = "order-1"."customer"
        GROUP BY "c"."id"
        HAVING COUNT(DISTINCT "order-1"."id") > 2

    The equivalent query can be generated using the :py:meth:`~Query.count` method.


.. py:function:: delete(gen)

    Delete objects from the database. Pony loads objects into the memory and will delete them one by one. If you have :py:meth:`before_delete` or :py:meth:`after_delete` defined, Pony will call each of them.

    :param generator gen: Python generator expression

    .. code-block:: python

        delete(o for o in Order if o.status == 'CANCELLED')

    If you need to delete objects without loading them into memory, you should use the :py:meth:`~Query.delete()` method with the parameter ``bulk=True``. In this case no hooks will be called, even if they are defined for the entity.


.. py:function:: desc(attr)

    This function is used inside :py:meth:`~Query.order_by` and :py:meth:`~Query.sort_by` for ordering in descending order.

    :param attribute attr: Entity attribute

    .. code-block:: python

        select(o for o in Order).order_by(desc(Order.date_shipped))

    The same example, using ``lambda``:

    .. code-block:: python

        select(o for o in Order).order_by(lambda o: desc(o.date_shipped))


.. py:function:: distinct(gen)

    When you need to force DISTINCT in a query, it can be done using the ``distinct()`` function. But usually this is not necessary, because Pony adds DISTINCT keyword automatically in an intelligent way. See more information about it in the TODO chapter.

    :param generator gen: Python generator expression

    .. code-block:: python

        distinct(o.date_shipped for o in Order)

    Another usage of the `distinct()` function is with the `sum()` aggregate function - you can write:

    .. code-block:: python

        select(sum(distinct(x.val)) for x in X)

    to generate the following SQL:

    .. code-block:: sql

        SELECT SUM(DISTINCT x.val)
        FROM X x

    but it is rarely used in practice.


.. py:function:: exists(gen, globals=None, locals=None)

    Returns `True` if at least one instance with the specified condition exists and `False` otherwise.

    :param generator gen: Python generator expression.
    :param dict globals:
    :param dict locals: optional parameters which can contain dicts with variables and its values, used within the query.
    :rtype: bool

    .. code-block:: python

        exists(o for o in Order if o.date_delivered is None)


.. py:function:: flush()

    Save all changes from the :py:func:`db_session` cache to the databases, without committing them. It makes the updates made in the :py:func:`db_session` cache visible to all database queries which belong to the current transaction.

    Usually Pony saves data from the database session cache automatically and you don't need to call this function yourself. One of the use cases when it might be needed is when you want to get the primary keys values of newly created objects which has autoincremented primary key before commit.

    This top level ``flush()`` function calls the :py:meth:`~Database.flush` method of each database object which was used in current transaction.

This function is called automatically before executing the following functions: :py:func:`commit`, :py:func:`get`, :py:func:`exists`, :py:func:`select`.


.. py:function:: get(gen, globals=None, locals=None)

    Extracts one entity instance from the database.

    :param generator gen: Python generator expression.
    :param dict globals:
    :param dict locals: optional parameters which can contain dicts with variables and its values, used within the query.
    :return: the object if an object with the specified parameters exists, or ``None`` if there is no such object.

    If there are more than one objects with the specified parameters, the function raises the ``MultipleObjectsFoundError: Multiple objects were found. Use select(...) to retrieve them`` exception.

    .. code-block:: python

        get(o for o in Order if o.id == 123)

    The equivalent query can be generated using the :py:meth:`~Query.get` method.


.. py:function:: getattr(object, name[, default])

    This is `a standard Python built-in function <https://docs.python.org/3/library/functions.html#getattr>`_, that can be used for getting the attribute value inside the query.

    Example:

    .. code-block:: python

        attr_name = 'name'
        param_value = 'John'
        select(c for c in Customer if getattr(c, attr_name) == param_value)
        
.. py:function:: group_concat(gen, sep=',', distinct=False)

    Returns string which is concatenation of given attribute.
    
    .. code-block:: python
        
        group_concat(t.title for t in Tag, sep='-')
        
    The equivalent query can be generated using the :py:meth:`~Query.group_concat()` method.
    
    .. note:: Query should return only single attribute. Also in SQLite you can't use both `distinct` and `sep` arguments at a time.


.. py:function:: JOIN(*args)

    Used for query optimization in cases when Pony doesn't provide this optimization automatically. Serves as a hint saying Pony that we want to use SQL JOIN, instead of generating a subquery inside the SQL query.

    .. code-block:: python

        select(g for g in Group if max(g.students.gpa) < 4)

        select(g for g in Group if JOIN(max(g.students.gpa) < 4))


.. py:function:: left_join(gen, globals=None, locals=None)

    The results of a left join always contain the result from the 'left' table, even if the join condition doesn't find any matching record in the 'right' table.

    :param generator gen: Python generator expression.
    :param dict globals:
    :param dict locals: optional parameters which can contain dicts with variables and its values, used within the query.


    Let's say we need to calculate the amount of orders for each customer. Let's use the example which comes with Pony distribution and write the following query:

    .. code-block:: python

        from pony.orm.examples.estore import *
        populate_database()

        select((c, count(o)) for c in Customer for o in c.orders)[:]

    It will be translated to the following SQL:

    .. code-block:: sql

        SELECT "c"."id", COUNT(DISTINCT "o"."id")
        FROM "Customer" "c", "Order" "o"
        WHERE "c"."id" = "o"."customer"
        GROUP BY "c"."id"

    And return the following result:

    .. code-block:: python

        [(Customer[1], 2), (Customer[2], 1), (Customer[3], 1), (Customer[4], 1)]


    But if there are customers that have no orders, they will not be selected by this query, because the condition ``WHERE "c"."id" = "o"."customer"`` doesn't find any matching record in the Order table. In order to get the list of all customers, we should use the ``left_join()`` function:

    .. code-block:: python

        left_join((c, count(o)) for c in Customer for o in c.orders)[:]

    .. code-block:: sql

        SELECT "c"."id", COUNT(DISTINCT "o"."id")
        FROM "Customer" "c"
        LEFT JOIN "Order" "o"
          ON "c"."id" = "o"."customer"
        GROUP BY "c"."id"

    Now we will get the list of all customers with the number of order equal to zero for customers which have no orders:

    .. code-block:: python

        [(Customer[1], 2), (Customer[2], 1), (Customer[3], 1), (Customer[4], 1), (Customer[5], 0)]

    We should mention that in most cases Pony can understand where LEFT JOIN is needed. For example, the same query can be written this way:

    .. code-block:: python

        select((c, count(c.orders)) for c in Customer)[:]

    .. code-block:: sql

        SELECT "c"."id", COUNT(DISTINCT "order-1"."id")
        FROM "Customer" "c"
        LEFT JOIN "Order" "order-1"
          ON "c"."id" = "order-1"."customer"
        GROUP BY "c"."id"


.. py:function:: len(arg)

    Return the number of objects in the collection. Can be used only within the query, similar to :py:func:`count`.

    :param generator arg: a collection
    :rtype: numeric

    .. code-block:: python

        Customer.select(lambda c: len(c.orders) > 2)


.. py:function:: max(gen)

    Return the maximum value from the database. The query should return a single attribute.

    :param generator gen: Python generator expression.

    .. code-block:: python

        max(o.date_shipped for o in Order)

    The equivalent query can be generated using the :py:meth:`~Query.max` method.


.. py:function:: min(*args, **kwargs)

    Return the minimum value from the database. The query should return a single attribute.

    :param generator gen: Python generator expression.

    .. code-block:: python

        min(p.price for p in Product)

    The equivalent query can be generated using the :py:meth:`~Query.min` method.


.. py:function:: random()

    Returns a random value from 0 to 1. This functions, when encountered inside a query will be translated into RANDOM SQL query.

    Example:

    .. code-block:: python

        select(s.gpa for s in Student if s.gpa > random() * 5)

    .. code-block:: sql

        SELECT DISTINCT "s"."gpa"
        FROM "student" "s"
        WHERE "s"."gpa" > (random() * 5)


.. py:function:: raw_sql(sql, result_type=None)

    This function encapsulates a part of a query expressed in a raw SQL format. If the ``result_type`` is specified, Pony converts the result of raw SQL fragment to the specified format.

    :param str sql: SQL statement text.
    :param type result_type:  the type of the SQL statement result.

    .. code-block:: python

        >>> q = Person.select(lambda x: raw_sql('abs("x"."age")') > 25)
        >>> print(q.get_sql())

    .. code-block:: sql

        SELECT "x"."id", "x"."name", "x"."age", "x"."dob"
        FROM "Person" "x"
        WHERE abs("x"."age") > 25

    .. code-block:: python

        x = 10
        y = 15
        select(p for p in Person if raw_sql('p.age > $(x + y)'))

        names = select(raw_sql('UPPER(p.name)') for p in Person)[:]
        print(names)

        ['JOHN', 'MIKE', 'MARY']

    See more examples :ref:`here <using_raw_sql_ref>`.


.. py:function:: rollback()

    Roll back the current transaction.

    This top level ``rollback()`` function calls the :py:meth:`~Database.rollback` method of each database object which was used in current transaction.


.. py:function:: select(gen)

    Translates the generator expression into SQL query and returns an instance of the :py:class:`Query` class.

    :param generator gen: Python generator expression.
    :param dict globals:
    :param dict locals: optional parameters which can contain dicts with variables and its values, used within the query.
    :rtype:  :py:class:`Query` or list

    You can iterate over the result:

    .. code-block:: python

        for p in select(p for p in Product):
            print p.name, p.price

    If you need to get a list of objects you can get a full slice of the result:

    .. code-block:: python

        prod_list = select(p for p in Product)[:]

    The ``select()`` function can also return a list of single attributes or a list of tuples:

    .. code-block:: python

        select(p.name for p in Product)

        select((p1, p2) for p1 in Product
                        for p2 in Product if p1.name == p2.name and p1 != p2)

        select((p.name, count(p.orders)) for p in Product)

    You can apply any :py:class:`Query` method to the result, e.g. :py:meth:`~Query.order_by` or :py:meth:`~Query.count`.

    If you want to run a query over a relationship attribute, you can use the :py:meth:`~Set.select` method of the relationship attribute.


.. py:function:: show()

    Prints out the entity definition or the value of attributes for an entity instance in the interactive mode.

    :param value: entity class or entity instance

    .. code-block:: python

        >>> show(Person)
        class Person(Entity):
            id = PrimaryKey(int, auto=True)
            name = Required(str)
            age = Required(int)
            cars = Set(Car)


        >>> show(mary)
        instance of Person
        id|name|age
        --+----+---
        2 |Mary|22


.. py:function:: set_sql_debug(value=True, show_values=None)

    Prints SQL statements being sent to the database to the console or to a log file.
    Previous name ``sql_debug`` is deprecated.

    :param bool value: sets debugging on/off
    :param bool show_values: when ``True``, query parameters will be logged in addition to the SQL text *(new in version 0.7.3)*

    Before version 0.7.3 it was a global flag. Now, in multi-threaded application, it should be set for each thread separately.

    By default Pony sends debug information to stdout. If you have the `standard Python logging <https://docs.python.org/3.6/howto/logging.html>`_ configured, Pony will use it instead. Here is how you can store debug information in a file:

    .. code-block:: python

        import logging
        logging.basicConfig(filename='pony.log', level=logging.INFO)

    Note, that we had to specify the ``level=logging.INFO`` because the default standard logging level is ``WARNING`` and Pony uses the ``INFO`` level for its messages by default. Pony uses two loggers: ``pony.orm.sql`` for SQL statements that it sends to the database and ``pony.orm`` for all other messages.


.. py:function:: sql_debugging(value=True, show_values=None)

    Context manager, use it for enabling/disabling logging SQL queries for a specific part of your code. If you need to turn on debugging for the whole db_session, use the similar parameters of :py:func:`db_session` decorator or context manager.

    :param bool value: sets debugging on/off
    :param bool show_values: when ``True``, query parameters will be logged in addition to the SQL text *(new in version 0.7.3)*

    .. code-block:: python

        with sql_debugging:  # turn debug on for a specific query
            result = Person.select()

        with sql_debugging(show_values=True):  # log with query params
            result = Person.select()

        with sql_debugging(False):  # turn debug off for a specific query
            result = Person.select()


.. py:function:: sum(gen)

    Return the sum of all values selected from the database.

    :param generator gen: Python generator expression
    :rtype: numeric
    :return: a number. If the query returns no items, the ``sum()`` method returns 0.

    .. code-block:: python

        sum(o.total_price for o in Order)

    The equivalent query can be generated using the :py:meth:`~Query.sum` method.


.. _query_object:

Query object
------------

The generator expression and lambda queries return an instance of the ``Query`` class. Below is the list of methods that you can apply to it.

.. py:class:: Query

    .. py:method:: [start:end]
                   [index]

        Limit the number of instances to be selected from the database. In the example below we select the first ten instances:

        .. code-block:: python

            # generator expression query
            select(c for c in Customer)[:10]

            # lambda function query
            Customer.select()[:10]

        Generates the following SQL:

        .. code-block:: sql

            SELECT "c"."id", "c"."email", "c"."password", "c"."name", "c"."country", "c"."address"
            FROM "Customer" "c"
            LIMIT 10

        If we need to select instances with offset, we should use ``start`` and ``end`` values:

        .. code-block:: python

            select(c for c in Customer).order_by(Customer.name)[20:30]

        It generates the following SQL:

        .. code-block:: sql

            SELECT "c"."id", "c"."email", "c"."password", "c"."name", "c"."country", "c"."address"
            FROM "Customer" "c"
            ORDER BY "c"."name"
            LIMIT 10 OFFSET 20

        Also you can use the :py:meth:`~Query.limit` or :py:meth:`~Query.page` methods for the same purpose.


    .. py:method:: __len__()

        Return the number of objects selected from the database.

        .. code-block:: python

            len(select(c for c in Customer))

    .. py:method:: avg()

        Return the average value for all selected attributes:

        .. code-block:: python

            select(o.total_price for o in Order).avg()

        The function :py:func:`avg` does the same thing.


    .. py:method:: count()

        Return the number of objects that match the query condition:

        .. code-block:: python

            select(c for c in Customer if len(c.orders) > 2).count()

      The function :py:func:`count` does the same thing.


    .. py:method:: delete(bulk=None)

        Delete instances selected by a query. When ``bulk=False`` Pony loads each instance into memory and call the :py:meth:`Entity.delete` method on each instance (calling :py:meth:`before_delete` and :py:meth:`after_delete` hooks if they are defined). If ``bulk=True`` Pony doesn't load instances, it just generates the SQL DELETE statement which deletes objects in the database.

        .. note:: Be careful with the bulk delete:

            * :py:meth:`before_delete` and :py:meth:`after_delete` hooks will not be called on deleted objects.
            * If an object was loaded into memory, it will not be removed from the :py:func:`db_session` cache on bulk delete.


    .. py:method:: distinct()

        Force DISTINCT in a query:

        .. code-block:: python

            select(c.name for c in Customer).distinct()

        But usually this is not necessary, because Pony adds DISTINCT keyword automatically in an intelligent way. See more information about it in the :ref:`Automatic DISTINCT <automatic_distinct>` section.

        The function :py:func:`distinct` does the same thing.


    .. py:method:: exists()

        Returns ``True`` if at least one instance with the specified condition exists and ``False`` otherwise:

        .. code-block:: python

            select(c for c in Customer if len(c.cart_items) > 10).exists()

        This query generates the following SQL:

        .. code-block:: sql

            SELECT "c"."id"
            FROM "Customer" "c"
              LEFT JOIN "CartItem" "cartitem-1"
                ON "c"."id" = "cartitem-1"."customer"
            GROUP BY "c"."id"
            HAVING COUNT(DISTINCT "cartitem-1"."id") > 20
            LIMIT 1


    .. py:method:: filter(lambda, globals=None, locals=None)
                   filter(str, globals=None, locals=None)
                   filter(**kwargs)

        Filters the result of a query. The conditions which are passed as parameters to the ``filter()`` method will be translated into the WHERE section of the resulting SQL query. The result of the ``filter()`` method is a new query object with the specified additional condition.

        .. note:: This method is similar to the :py:meth:`~Query.where` method. The difference is that the ``filter()`` condition applies to items returned from the previous query, whereas ``where()`` condition applies to the loop variables from the original generator expression. Example:

            .. code-block:: python

                q = select(o.customer for o in Order)

                # c refers to o.customer
                q2 = q.filter(lambda c: c.name.startswith('John'))

                # o refers to Order object
                q3 = q2.where(lambda o: o.total_price > 1000)

            The name of lambda function argument in ``filter()`` may be arbitrary, but in ``where()`` the name of lambda argument should exactly match the name of the loop variable.

        .. note:: The :py:meth:`~Query.where` method was added in version 0.7.3. Before that it was possible to do the same by using ``filter()`` method in which argument of lambda function was not specified:

            .. code-block:: python

                q = select(o.customer for o in Order)

                # using new where() method
                q2a = q.where(lambda o: o.customer.name == 'John Smith')

                # old way to do the same using filter() method
                q2b = q.filter(lambda: o.customer.name == 'John Smith')

            But this old way has a drawback: IDEs and linters don't understand code and warn about "undefined global variable ``o``". With ``where()`` it is no longer the case. Using lambda function without argument in ``filter()`` will be deprecated in the next release.


        **Specifying** ``filter()`` **condition using lambda function**

            Usually the argument of the ``filter()`` method is a lambda function. The argument of the lambda function represents the result of the query. You can use an arbitrary name for this argument:

            .. code-block:: python

                q = select(p.name for p in Product)
                q2 = q.filter(lambda x: x.startswith('Apple iPad'))

            In the example above ``x`` argument corresponds to the result of the query ``p.name``. This way you cannot access the ``p`` variable in the filter method, only ``p.name``. When you need to access the original query loop variable, you can use the :py:meth:`~Query.where` method instead.

            If the query returns a tuple, the number of ``filter()`` lambda function arguments should correspond to the query result:

            .. code-block:: python

                q = select((p.name, p.price) for p in Product)
                q2 = q.filter(lambda n, p: n.startswith('Apple iPad') and p < 500)

        **Specifying** ``filter()`` **condition using keyword arguments**

            Another way to filter the query result is to pass parameters in the form of named arguments:

            .. code-block:: python

                q = select(o.customer for o in Order if o.total_price > 1000)
                q2 = q.filter(name="John Smith", country="UK")

            Keyword arguments can be used only when the result of the query is an object. In the example above it is an object of the ``Customer`` type.

        **Specifying** ``filter()`` **condition as a text string**

            Also the ``filter()`` method can receive a text definition of a lambda function. It can be used when you combine the condition from text pieces:

            .. code-block:: python

                q = select(p for p in Product)
                x = 100
                q2 = q.filter("lambda p: p.price > x")

            In the example above the ``x`` variable in lambda refers to ``x`` defined before. The more secure solution is to specify the dictionary with values as a second argument of the ``filter()`` method:

            .. code-block:: python

                q = select(p for p in Product)
                q2 = q.filter("lambda p: p.price > x", {"x": 100})

    .. py:method:: first()

        Return the first element from the selected results or ``None`` if no objects were found:

        .. code-block:: python

            select(p for p in Product if p.price > 100).first()


    .. py:method:: for_update(nowait=False)

        :param bool nowait: prevent the operation from waiting for other transactions to commit. If a selected row(s) cannot be locked immediately, the operation reports an error, rather than waiting.

        Sometimes there is a need to lock objects in the database in order to prevent other transactions from modifying the same instances simultaneously. Within the database such lock should be done using the SELECT FOR UPDATE query. In order to generate such a lock using Pony you can call the ``for_update`` method:

        .. code-block:: python

            select(p for p in Product if p.picture is None).for_update()

        This query selects all instances of Product without a picture and locks the corresponding rows in the database. The lock will be released upon commit or rollback of current transaction.


    .. py:method:: get()

        Extract one entity instance from the database. The function returns the object if an object with the specified parameters exists, or ``None`` if there is no such object. If there are more than one objects with the specified parameters, raises the ``MultipleObjectsFoundError: Multiple objects were found. Use select(...) to retrieve them`` exception. Example:

        .. code-block:: python

            select(o for o in Order if o.id == 123).get()

        The function :py:func:`get` does the same thing.


    .. py:method:: get_sql()

        Return SQL statement as a string:

        .. code-block:: python

            sql = select(c for c in Category if c.name.startswith('a')).get_sql()
            print(sql)

        .. code-block:: sql

            SELECT "c"."id", "c"."name"
            FROM "category" "c"
            WHERE "c"."name" LIKE 'a%%'
            
    .. py:method:: group_concat(sep=',', distinct=False)
    
        Returns a string which is the concatenation of all non-NULL values of given column. 
        
        The function :py:func:`group_concat` does the same thing.
        
        .. code-block:: python
        
            select(article.tag for article in Article).group_concat(sep=', #')
        
        .. note:: In SQLite you can't use `group_concat()` with both `sep` and `distinct` arguments at a time.


    .. py:method:: limit(limit, offset=None)

        Limit the number of instances to be selected from the database.

        .. code-block:: python

            select(c for c in Customer).order_by(Customer.name)[20:30]

        Also you can use the :py:meth:`~Query.[start:end]` or :py:meth:`~Query.page` methods for the same purpose.


    .. py:method:: max()

        Return the maximum value from the database. The query should return a single attribute:

        .. code-block:: python

            select(o.date_shipped for o in Order).max()

        The function :py:func:`max` does the same thing.


    .. py:method:: min()

        Return the minimum value from the database. The query should return a single attribute:

        .. code-block:: python

            select(p.price for p in Product).min()

        The function :py:func:`min` does the same thing.

    .. py:method:: order_by(attr1 [, attr2, ...])
                   order_by(pos1 [, pos2, ...])
                   order_by(lambda[, globals[, locals]])
                   order_by(str[, globals[, locals]])

        .. note:: The behavior of ``order_by()`` is going to be changed in the next release (0.8). Previous behavior supports by method :py:func:`sort_by` which is introduced in the release 0.7.3. In order to be fully forward-compatible with the release 0.8, you can replace all ``order_by()`` calls to ``sort_by()`` calls.

        Orders the results of a query. Currently ``order_by()`` and ``sort_by()`` methods work in the same way - they are applied to the result of the previous query.

        .. code-block:: python

            q = select(o.customer for o in Order)

            # The following five queries are all equivalent

            # Before the 0.8 release
            q1 = q.order_by(lambda c: c.name)
            q2 = q.order_by(Customer.name)

            # Starting from the 0.7.3 release
            q3 = q.sort_by(lambda c: c.name)
            q4 = q.sort_by(Customer.name)

            # After the 0.8 release
            q5 = q.order_by(lambda o: o.customer.name)

        Most often query returns the same object it iterates. In this case the behavior of ``order_by()`` will remains the same before and after the 0.8 release:

        .. code-block:: python

            # the query returns the loop variable
            q = select(c for c in Customer if c.age > 18)

            # the next line will work the same way
            # before and after the 0.8 release
            q2 = q.order_by(lambda c: c.name)

        There are several ways how it is possible to call ``order_by()`` method:

        **Using entity attributes**

            .. code-block:: python

                select(o for o in Order).order_by(Order.date_created)

            For ordering in descending order, use the function :py:func:`desc()`:

            .. code-block:: python

                select(o for o in Order).order_by(desc(Order.date_created))

        **Using position of query result variables**

            .. code-block:: python

                select((o.customer.name, o.total_price) for o in Order).order_by(-2, 1)

            The position numbers start with 1. Minus means sorting in the descending order. In this example we sort the result by the total price in descending order and by the customer name in ascending order.

        **Using lambda**

            .. code-block:: python

                select(o for o in Order).order_by(lambda o: o.customer.name)

            If the lambda has a parameter (``o`` in our example) then ``o`` represents the result of the ``select`` and will be applied to it. Starting from the release 0.8 it will represent the iterator loop variable from the original query. If you want to continue using the result of a query for ordering, you need to use the ``sort_by`` method instead.

        **Using lambda without parameters**

            If you specify the lambda without parameters, then inside lambda you may access all names defined inside the query:

            .. code-block:: python

                select(o.total_price for o in Order).order_by(lambda: o.customer.id)

            It looks like ``o`` is a global variable, but Pony understand it as a loop variable name ``o`` from the generator expression. This behavior confuses IDEs and linetrs which warn about "access to undefined global variable ``o``". Starting with release 0.8 this way of using ``order_by()`` will be unnecessary: just add ``o`` argument to lambda function instead.

        **Specifying a string expression**

            This approach is similar to the previous one, but you specify the body of a lambda as a string:

            .. code-block:: python

                select(o for o in Order).order_by("o.customer.name")


    .. py:method:: sort_by(attr1 [, attr2, ...])
                   sort_by(pos1 [, pos2, ...])
                   sort_by(lambda[, globals[, locals]])
                   sort_by(str[, globals [, locals]])

        *New in 0.7.3*

        Orders the results of a query. The expression in ``sort_by()`` method call applies to items in query result. Until the 0.8 release it works the same as :py:meth:`~Query.order_by`, then the behavior of ``order_by()`` will change.

        There are several ways how it is possible to call ``order_by()`` method:

        **Using entity attributes**

            .. code-block:: python

                select(o.customer for o in Order).sort_by(Customer.name)

            For ordering in descending order, use the function :py:func:`desc()`:

            .. code-block:: python

                select(o.customer for o in Order).sort_by(desc(Customer.name))

        **Using position of query result variables**

            .. code-block:: python

                select((o.customer.name, o.total_price) for o in Order).sort_by(-2, 1)

            The position numbers start with 1. Minus means sorting in the descending order. In this example we sort the result by the total price in descending order and by the customer name in ascending order.

        **Using lambda**

            .. code-block:: python

                select(o.customer for o in Order).sort_by(lambda c: c.name)

            Lambda inside the ``sort_by`` method receives the result of the previous query.

        **Specifying a string expression**

            This approach is similar to the previous one, but you specify the body of a lambda as a string:

            .. code-block:: python

                select(o for o in Order).sort_by("o.customer.name")


    .. py:method:: page(pagenum, pagesize=10)

        Pagination is used when you need to display results of a query divided into multiple pages. The page numbering starts with page 1. This method returns a slice [start:end] where ``start = (pagenum - 1) * pagesize``, ``end = pagenum * pagesize``.


    .. py:method:: prefetch(*args)

        Allows specifying which related objects or attributes should be loaded from the database along with the query result.

        Usually there is no need to prefetch related objects. When you work with the query result within the ``@db_session``, Pony gets all related objects once you need them. Pony uses the most effective way for loading related objects from the database, avoiding the N+1 Query problem.

        So, if you use Flask, the recommended approach is to use the ``@db_session`` decorator at the top level, at the same place where you put the Flask's ``app.route`` decorator:

        .. code-block:: python

            @app.route('/index')
            @db_session
            def index():
                ...
                objects = select(...)
                ...
                return render_template('template.html', objects=objects)

        Or, even better, wrapping the wsgi application with the :py:func:`db_session` decorator:

        .. code-block:: python

            app.wsgi_app = db_session(app.wsgi_app)

        If for some reason you need to pass the selected instances along with related objects outside of the :py:func:`db_session`, then you can use this method. Otherwise, if you'll try to access the related objects outside of the :py:func:`db_session`, you might get the ``DatabaseSessionIsOver`` exception, e.g. ``DatabaseSessionIsOver: Cannot load attribute Customer[3].name: the database session is over``

        More information regarding working with the :py:func:`db_session` can be found :ref:`here <db_session>`.

        You can specify entities and/or attributes as parameters. When you specify an entity, then all "to-one" and non-lazy attributes of corresponding related objects will be prefetched. The "to-many" attributes of an entity are prefetched only when specified explicitly.

        If you specify an attribute, then only this specific attribute will be prefetched. You can specify attribute chains, e.g. ``order.customer.address``. The prefetching works recursively - it applies the specified parameters to each selected object.

        Examples:

        .. code-block:: python

            from pony.orm.examples.presentation import *

        Loading Student objects only, without prefetching:

        .. code-block:: python

            students = select(s for s in Student)[:]

        Loading students along with groups and departments:

        .. code-block:: python

            students = select(s for s in Student).prefetch(Group, Department)[:]

            for s in students: # no additional query to the DB will be sent
                print s.name, s.group.major, s.group.dept.name

        The same as above, but specifying attributes instead of entities:

        .. code-block:: python

            students = select(s for s in Student).prefetch(Student.group, Group.dept)[:]

            for s in students: # no additional query to the DB will be sent
                print s.name, s.group.major, s.group.dept.name

        Loading students and related courses ("many-to-many" relationship):

        .. code-block:: python

            students = select(s for s in Student).prefetch(Student.courses)

            for s in students:
                print s.name
                for c in s.courses: # no additional query to the DB will be sent
                    print c.name


    .. py:method:: random(limit)

        Select ``limit`` random objects from the database. This method will be translated using the ``ORDER BY RANDOM()`` SQL expression. The entity class method :py:meth:`~Entity.select_random` provides better performance, although doesn't allow to specify query conditions.

        For example, select ten random persons older than 20 years old:

        .. code-block:: python

            select(p for p in Person if p.age > 20).random()[:10]


    .. py:method:: show(width=None)

        Prints the results of a query to the console. The result is formatted in the form of a table. This method doesn't display "to-many" attributes because it would require additional query to the database and could be bulky. But if an instance has a "to-one" relationship, then it will be displayed.

        .. code-block:: python

            >>> select(p for p in Person).order_by(Person.name)[:2].show()

            SELECT "p"."id", "p"."name", "p"."age"
            FROM "Person" "p"
            ORDER BY "p"."name"
            LIMIT 2

            id|name|age
            --+----+---
            3 |Bob |30

            >>> Car.select().show()
            id|make  |model   |owner
            --+------+--------+---------
            1 |Toyota|Prius   |Person[2]
            2 |Ford  |Explorer|Person[3]


    .. py:method:: sum()

        Return the sum of all selected items. Can be applied to the queries which return a single numeric expression only.

        .. code-block:: python

            select(o.total_price for o in Order).sum()

        If the query returns no items, the query result will be 0.


    .. py:method:: where(lambda, globals=None, locals=None)
                   where(str, globals=None, locals=None)
                   where(**kwargs)

        *New in version 0.7.3*

        Filters the result of a query. The conditions which are passed as parameters to the ``where()`` method will be translated into the WHERE section of the resulting SQL query. The result of the ``where()`` method is a new query object with the specified additional condition.

        .. note:: This method is similar to the :py:meth:`~Query.filter` method. The difference is that the ``filter()`` condition applies to items returned from the previous query, whereas ``where()`` condition applies to the loop variables from the original generator expression. Example:

            .. code-block:: python

                q = select(o.customer for o in Order)

                # c refers to o.customer
                q2 = q.filter(lambda c: c.name.startswith('John'))

                # o refers to Order object
                q3 = q2.where(lambda o: o.total_price > 1000)

            The name of lambda function argument in ``filter()`` may be arbitrary, but in ``where()`` the name of lambda argument should exactly match the name of the loop variable.

        .. note:: Before the ``where()`` method was added it was possible to do the same by using :py:meth:`~Query.filter` method in which argument of lambda function was not specified:

            .. code-block:: python

                q = select(o.customer for o in Order)

                # using new where() method
                q2a = q.where(lambda o: o.customer.name == 'John Smith')

                # old way to do the same using filter() method
                q2b = q.filter(lambda: o.customer.name == 'John Smith')

            But this old way has a drawback: IDEs and linters don't understand code and warn about "undefined global variable ``o``". With ``where()`` it is no longer the case. Using lambda function without argument in ``filter()`` will be deprecated in the next release.

        **Specifying** ``where()`` **condition using lambda function**

            Usually the argument of the ``where()`` method is a lambda function. The arguments of the lambda function refer to the query loop variables, and should have the same names.

            .. code-block:: python

                q = select(p.name for p in Product).where(lambda p: p.price > 1000)

            In the example above ``p`` argument corresponds to the ``p`` variable of the query.

            In the ``where()`` method the lambda arguments can refer to all loop variables from the original query:

            .. code-block:: python

                q = select(c.name for c in Customer for o in c.orders)
                q2 = q.where(lambda c, o: c.country == 'US' and o.state == 'DELIVERED')

            When the query is written using the ``select()`` method of an entity, the query does not have any explicitly defined loop variable. In that case the argument of lambda function should be the first letter of the entity name in the lower case:

            .. code-block:: python

                q = Product.select()
                q2 = q.where(lambda p: p.name.startswith('A'))

        **Specifying** ``where()`` **condition using keyword arguments**

            Another way to filter the query result is to pass parameters in the form of named arguments:

            .. code-block:: python

                q = select(o.customer for o in Order if o.total_price > 1000)
                q2 = q.where(state == 'DELIVERED')

            The ``state`` keyword attribute refers to the ``state`` attribute of the ``Order`` object.

        **Specifying** ``where()`` **condition as a text string**

            Also the ``where()`` method can receive an expression text instead of lambda function. It can be used when you combine the condition from text pieces:

            .. code-block:: python

                q = select(p for p in Product)
                x = 100
                q2 = q.where("p.price > x")

            The more secure solution is to specify the dictionary with values as a second argument of the ``where()`` method:

            .. code-block:: python

                q = select(p for p in Product)
                q2 = q.where("p.price > x", {"x": 100})


    .. py:method:: without_distinct()

        By default Pony tries to avoid duplicates in the query result and intellectually adds the ``DISTINCT`` SQL keyword to a query where it thinks it necessary. If you don't want Pony to add ``DISTINCT`` and get possible duplicates, you can use this method. This method returns a new instance of the Query object, so you can chain it with other query methods:

        .. code-block:: python

            select(p.name for p in Person).without_distinct().order_by(Person.name)

        Before Pony Release 0.6 the method ``without_distinct()`` returned query result and not a new query instance.



Statistics - QueryStat
----------------------


The ``Database`` object has a thread-local property :py:attr:`~Database.local_stats` which contains query execution statistics. The property value is a dict, where keys are SQL queries and values are instances of the ``QueryStat`` class. A ``QueryStat`` object has the following attributes:

.. class:: QueryStat

    .. py:attribute:: sql

        The text of SQL query

    .. py:attribute:: db_count

        The number of times this query was sent to the database

    .. py:attribute:: cache_count

        The number of times the query result was taken directly from the :py:func:`db_session` cache (for cases when a query was called repeatedly inside the same :py:func:`db_session`)

    .. py:attribute:: min_time

        The minimum time required for database to execute the query

    .. py:attribute:: max_time

        The maximum time required for database to execute the query

    .. py:attribute:: avg_time

        The average time required for database to execute the query

    .. py:attribute:: sum_time

        Total time spent (is equal to `avg_time` * `db_count`)

Pony keeps all statistics separately for each thread. If you want to see the aggregated statistics for all threads then you need to call the :py:meth:`~Database.merge_local_stats()` method. See also: :py:meth:`~Database.local_stats`, :py:meth:`~Database.global_stats`, .

Example:

.. code-block:: python

    query_stats = sorted(db.local_stats.values(),
            reverse=True, key=attrgetter('sum_time'))
    for qs in query_stats:
        print(qs.sum_time, qs.db_count, qs.sql)

