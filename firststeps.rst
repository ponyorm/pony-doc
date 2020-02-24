Getting Started with Pony
=========================

Installing
----------

To install Pony, type the following command into the command prompt:

.. code-block:: text

    pip install pony

Pony can be installed on Python 2.7 or Python 3. If you are going to work with SQLite database, you don't need to install anything else. If you wish to use another database, you need to have the access to the database and have the corresponding database driver installed:

* PostgreSQL: `psycopg2 <http://initd.org/psycopg/docs/install.html#installation>`_ or `psycopg2cffi <https://pypi.python.org/pypi/psycopg2cffi>`_
* MySQL: `MySQL-python <https://pypi.python.org/pypi/MySQL-python/>`_ or `PyMySQL <https://pypi.python.org/pypi/PyMySQL>`_
* Oracle: `cx_Oracle <https://pypi.python.org/pypi/cx_Oracle>`_
* CockroachDB: `psycopg2 <http://initd.org/psycopg/docs/install.html#installation>`_ or `psycopg2cffi <https://pypi.python.org/pypi/psycopg2cffi>`_

To make sure Pony has been successfully installed, launch a Python interpreter in interactive mode and type:

.. code-block:: python

    >>> from pony.orm import *

This imports the entire (and not very large) set of classes and functions necessary for working with Pony. Eventually you can choose what to import, but we recommend using ``import *`` at first.

If you don't want to import everything into global namespace, you can import the ``orm`` package only:

.. code-block:: python

    >>> from pony import orm

In this case you don't load all Pony's functions into the global namespace, but it will require you to use ``orm`` as a prefix to any Pony's function and decorator.

The best way to become familiar with Pony is to play around with it in interactive mode. Let's create a sample database containing the entity class ``Person``, add three objects to it, and write a query. 


Creating the database object
----------------------------

Entities in Pony are connected to a database. This is why we need to create the database object first. In the Python interpreter, type:

.. code-block:: python

    >>> db = Database()


Defining entities
-----------------

Now, let's create two entities -- Person and Car. The entity Person has two attributes -- name and age, and Car has attributes make and model. In the Python interpreter, type the following code:

.. code-block:: python

    >>> class Person(db.Entity):
    ...     name = Required(str)
    ...     age = Required(int)
    ...     cars = Set('Car')
    ... 
    >>> class Car(db.Entity):
    ...     make = Required(str)
    ...     model = Required(str)
    ...     owner = Required(Person)
    ... 
    >>> 

The classes that we have created are derived from the :py:attr:`Database.Entity` attribute of the :py:class:`Database` object. It means that they are not ordinary classes, but entities. The entity instances are stored in the database, which is bound to the ``db`` variable. With Pony you can work with several databases at the same time, but each entity belongs to one specific database.

Inside the entity ``Person`` we have created three attributes -- ``name``, ``age`` and ``cars``. The ``name`` and ``age`` are mandatory attributes. In other words, they these attributes cannot have the ``None`` value. The ``name`` is a string attribute, while ``age`` is numeric.

The ``cars`` attribute is declared as :py:class:`Set` and has the ``Car`` type. This means that this is a relationship. It can keep a collection of instances of the ``Car`` entity. ``"Car"`` is specified as a string here because we didn't declare the entity ``Car`` by that moment yet.

The ``Car`` entity has three mandatory attributes: ``make`` and ``model`` are strings, and the ``owner`` attribute is the other side of the one-to-many relationship. Relationships in Pony are always defined by two attributes which represent both sides of a relationship.

If we need to create a many-to-many relationship between two entities, we should declare two :py:class:`Set` attributes at both ends. Pony creates the intermediate database table automatically.

The ``str`` type is used for representing an unicode string in Python 3. Python 2 has two types for strings - ``str`` and ``unicode``. Starting with the Pony Release 0.6, you can use either ``str`` or ``unicode`` for string attributes, both of them mean an unicode string. We recommend using the ``str`` type for string attributes, because it looks more natural in Python 3.

If you need to check an entity definition in the interactive mode, you can use the :py:func:`show` function. Pass the entity class or the entity instance to this function for printing out the definition:

.. code-block:: python

    >>> show(Person)
    class Person(Entity):
        id = PrimaryKey(int, auto=True)
        name = Required(str)
        age = Required(int)
        cars = Set(Car)

You may notice that the entity got one extra attribute named ``id``. Why did that happen?

Each entity must contain a primary key, which allows distinguishing one entity from the other. Since we have not set the primary key attribute manually, it was created automatically. If the primary key is created automatically, it is named as ``id`` and has a numeric format. If the primary key attribute is created manually, you can specify the name and type of your choice. Pony also supports composite primary keys.

When the primary key is created automatically, it always has the option ``auto`` set to ``True``. It means that the value for this attribute will be assigned automatically using the database’s incremental counter or a database sequence.


Database binding
----------------

The database object has the :py:func:`Database.bind()` method. It is used for attaching declared entities to a specific database. If you want to play with Pony in the interactive mode, you can use the SQLite database created in memory:

.. code-block:: python

    >>> db.bind(provider='sqlite', filename=':memory:')

Currently Pony supports 5 database types: ``'sqlite'``, ``'mysql'``, ``'postgresql'``, ``'cockroach'`` and ``'oracle'``. The subsequent parameters are specific to each database. They are the same ones that you would use if you were connecting to the database through the DB-API module.

For SQLite, either the database filename or the string ':memory:' must be specified as the parameter, depending on where the database is being created. If the database is created in-memory, it will be deleted once the interactive session in Python is over. In order to work with the database stored in a file, you can replace the previous line with the following:

.. code-block:: python

    >>> db.bind(provider='sqlite', filename='database.sqlite', create_db=True)

In this case, if the database file does not exist, it will be created. In our example, we can use a database created in-memory.

If you're using another database, you need to have the specific database adapter installed. For PostgreSQL Pony uses psycopg2. For MySQL either MySQLdb or pymysql adapter. For Oracle Pony uses the cx_Oracle adapter.

Here is how you can get connected to the databases:

.. code-block:: python

    # SQLite
    db.bind(provider='sqlite', filename=':memory:')
    # or
    db.bind(provider='sqlite', filename='database.sqlite', create_db=True)

    # PostgreSQL
    db.bind(provider='postgres', user='', password='', host='', database='')

    # MySQL
    db.bind(provider='mysql', host='', user='', passwd='', db='')

    # Oracle
    db.bind(provider='oracle', user='', password='', dsn='')

    # CockroachDB
    db.bind(provider='cockroach', user='', password='', host='', database='', )

Mapping entities to database tables
-----------------------------------

Now we need to create database tables where we will persist our data. For this purpose, we need to call the :py:meth:`~Database.generate_mapping` method on the :py:class:`Database` object:

.. code-block:: python

    >>> db.generate_mapping(create_tables=True)

The parameter ``create_tables=True`` indicates that, if the tables do not already exist, then they will be created using the ``CREATE TABLE`` command.

All entities connected to the database must be defined before calling :py:meth:`~Database.generate_mapping` method.


Using the debug mode
--------------------

Using the :py:func:`set_sql_debug` function, you can see the SQL commands that Pony sends to the database. In order to turn the debug mode on, type the following:

.. code-block:: python

    >>> set_sql_debug(True)

If this command is executed before calling the :py:meth:`~Database.generate_mapping` method, then during the creation of the tables, you will see the SQL code used to generate them.



Creating entity instances
-------------------------

Now, let's create five objects that describe three persons and two cars, and save this information in the database:

.. code-block:: python

    >>> p1 = Person(name='John', age=20)
    >>> p2 = Person(name='Mary', age=22)
    >>> p3 = Person(name='Bob', age=30)
    >>> c1 = Car(make='Toyota', model='Prius', owner=p2)
    >>> c2 = Car(make='Ford', model='Explorer', owner=p3)
    >>> commit()

Pony does not save objects in the database immediately. These objects will be saved only after the :py:func:`commit` function is called. If the debug mode is turned on, then during the :py:func:`commit`, you will see five ``INSERT`` commands sent to the database.


db_session
----------

The code which interacts with the database has to be placed within a database session. When you work with Python’s interactive shell you don't need to worry about the database session, because it is maintained by Pony automatically. But when you use Pony in your application, all database interactions should be done within a database session. In order to do that you need to wrap the functions that work with the database with the :py:func:`db_session` decorator:

.. code-block:: python

    @db_session
    def print_person_name(person_id):
        p = Person[person_id]
        print p.name
        # database session cache will be cleared automatically
        # database connection will be returned to the pool

    @db_session
    def add_car(person_id, make, model):
        Car(make=make, model=model, owner=Person[person_id])
        # commit() will be done automatically
        # database session cache will be cleared automatically
        # database connection will be returned to the pool

The :py:func:`db_session` decorator performs the following actions on exiting function:

* Performs rollback of transaction if the function raises an exception
* Commits transaction if data was changed and no exceptions occurred
* Returns the database connection to the connection pool
* Clears the database session cache

Even if a function just reads data and does not make any changes, it should use the :py:func:`db_session` in order to return the connection to the connection pool.

The entity instances are valid only within the :py:func:`db_session`. If you need to render an HTML template using those objects, you should do this within the :py:func:`db_session`.

Another option for working with the database is using the :py:func:`db_session` as the context manager instead of the decorator:

.. code-block:: python

    with db_session:
        p = Person(name='Kate', age=33)
        Car(make='Audi', model='R8', owner=p)
        # commit() will be done automatically
        # database session cache will be cleared automatically
        # database connection will be returned to the pool


Writing queries
---------------

Now that we have the database with five objects saved in it, we can try some queries. For example, this is the query which returns a list of persons who are older than twenty years old:

.. code-block:: python

    >>> select(p for p in Person if p.age > 20)
    <pony.orm.core.Query at 0x105e74d10>

The :py:func:`select` function translates the Python generator into a SQL query and returns an instance of the :py:class:`Query` class. This SQL query will be sent to the database once we start iterating over the query. One of the ways to get the list of objects is to apply the slice operator ``[:]`` to it:

.. code-block:: python

    >>> select(p for p in Person if p.age > 20)[:]

    SELECT "p"."id", "p"."name", "p"."age"
    FROM "Person" "p"
    WHERE "p"."age" > 20

    [Person[2], Person[3]]

As the result you can see the text of the SQL query which was sent to the database and the list of extracted objects. When we print out the query result, the entity instance is represented by the entity name and its primary key written in square brackets, e.g. ``Person[2]``.

For ordering the resulting list you can use the :py:meth:`Query.order_by` method. If you need only a portion of the result set, you can use the slice operator, the exact same way as you would do that on a Python list. For example, if you want to sort all people by their name and extract the first two objects, you do it this way:

.. code-block:: python

    >>> select(p for p in Person).order_by(Person.name)[:2]

    SELECT "p"."id", "p"."name", "p"."age"
    FROM "Person" "p"
    ORDER BY "p"."name"
    LIMIT 2

    [Person[3], Person[1]]

Sometimes, when working in the interactive mode, you might want to see the values of all object attributes. For this purpose, you can use the :py:meth:`Query.show` method:

.. code-block:: python

    >>> select(p for p in Person).order_by(Person.name)[:2].show()

    SELECT "p"."id", "p"."name", "p"."age"
    FROM "Person" "p"
    ORDER BY "p"."name"
    LIMIT 2

    id|name|age
    --+----+---
    3 |Bob |30 
    1 |John|20

The :py:meth:`Query.show` method doesn't display "to-many" attributes because it would require additional query to the database and could be bulky. That is why you can see no information about the related cars above. But if an instance has a "to-one" relationship, then it will be displayed:

.. code-block:: python

    >>> Car.select().show()
    id|make  |model   |owner    
    --+------+--------+---------
    1 |Toyota|Prius   |Person[2]
    2 |Ford  |Explorer|Person[3]

If you don't want to get a list of objects, but need to iterate over the resulting sequence, you can use the ``for`` loop without using the slice operator:

.. code-block:: python

    >>> persons = select(p for p in Person if 'o' in p.name)
    >>> for p in persons:
    ...     print p.name, p.age
    ...
    SELECT "p"."id", "p"."name", "p"."age"
    FROM "Person" "p"
    WHERE "p"."name" LIKE '%o%'

    John 20
    Bob 30

In the example above we get all Person objects with the name attribute containing the letter 'o' and display the person's name and age.

A query does not necessarily have to return entity objects. For example, you can get a list, consisting of the object attribute:

.. code-block:: python

    >>> select(p.name for p in Person if p.age != 30)[:]

    SELECT DISTINCT "p"."name"
    FROM "Person" "p"
    WHERE "p"."age" <> 30

    [u'John', u'Mary']

Or a list of tuples:

.. code-block:: python

    >>> select((p, count(p.cars)) for p in Person)[:]

    SELECT "p"."id", COUNT(DISTINCT "car-1"."id")
    FROM "Person" "p"
      LEFT JOIN "Car" "car-1"
        ON "p"."id" = "car-1"."owner"
    GROUP BY "p"."id"

    [(Person[1], 0), (Person[2], 1), (Person[3], 1)]

In the example above we get a list of tuples consisting of a ``Person`` object and the number of cars they own.

With Pony you can also run aggregate queries. Here is an example of a query which returns the maximum age of a person:

.. code-block:: python

    >>> print max(p.age for p in Person)
    SELECT MAX("p"."age")
    FROM "Person" "p"

    30

In the following parts of this manual you will see how you can write more complex queries.


Getting objects
---------------

To get an object by its primary key you need to specify the primary key value in the square brackets:

.. code-block:: python

    >>> p1 = Person[1]
    >>> print p1.name
    John

You may notice that no query was sent to the database. That happened because this object is already present in the database session cache. Caching reduces the number of requests that need to be sent to the database.

For retrieving the objects by other attributes, you can use the :py:meth:`Entity.get` method:

.. code-block:: python

    >>> mary = Person.get(name='Mary')

    SELECT "id", "name", "age"
    FROM "Person"
    WHERE "name" = ?
    [u'Mary']

    >>> print mary.age
    22

In this case, even though the object had already been loaded to the cache, the query still had to be sent to the database because the ``name`` attribute is not a unique key. The database session cache will only be used if we lookup an object by its primary or unique key.

You can pass an entity instance to the :py:func:`show` function in order to display the entity class and attribute values:

.. code-block:: python

    >>> show(mary)
    instance of Person
    id|name|age
    --+----+---
    2 |Mary|22



Updating an object 
------------------

.. code-block:: python

    >>> mary.age += 1
    >>> commit()

Pony keeps track of all changed attributes. When the :py:func:`commit` function is executed, all objects that were updated during the current transaction will be saved in the database. Pony saves only those attributes, that were changed during the database session.


Writing raw SQL queries
-----------------------

If you need to select entities by a raw SQL query, you can do it this way:

.. code-block:: python

    >>> x = 25
    >>> Person.select_by_sql('SELECT * FROM Person p WHERE p.age < $x')

    SELECT * FROM Person p WHERE p.age < ?
    [25]

    [Person[1], Person[2]]

If you want to work with the database directly, avoiding entities, you can use the :py:meth:`Database.select` method:

.. code-block:: python

    >>> x = 20
    >>> db.select('name FROM Person WHERE age > $x')
    SELECT name FROM Person WHERE age > ?
    [20]

    [u'Mary', u'Bob']


Pony examples
-------------

Instead of creating models manually, you can check the examples from the Pony distribution package:

.. code-block:: python

    >>> from pony.orm.examples.estore import *

Here you can see the database diagram for this example: `https://editor.ponyorm.com/user/pony/eStore <https://editor.ponyorm.com/user/pony/eStore>`_.

During the first import, there will be created the SQLite database with all the necessary tables. In order to fill it in with the data, you need to call the following function:

.. code-block:: python

    >>> populate_database()

This function will create objects and place them in the database.

After the objects have been created, you can try some queries. For example, here is how you can display the country where we have most of the customers:

.. code-block:: python

    >>> select((customer.country, count(customer))
    ...        for customer in Customer).order_by(-2).first()

    SELECT "customer"."country", COUNT(DISTINCT "customer"."id")
    FROM "Customer" "customer"
    GROUP BY "customer"."country"
    ORDER BY 2 DESC
    LIMIT 1

In this example, we are grouping objects by the country, sorting them by the second column (the number of customers) in the reverse order, and then extracting the first row.

You can find more query examples in the ``test_queries()`` function in the `pony.orm.examples.estore <https://github.com/ponyorm/pony/blob/orm/pony/orm/examples/estore.py>`_ module.
