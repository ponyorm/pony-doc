What is Pony ORM?
=================

Pony is an advanced object-relational mapper. An ORM allows developers to work with the content of a database in the form of objects. A relational database contains rows that are stored in tables. However, when writing a program in a high level object-oriented language, it is much more convenient when the data retrieved from the database can be accessed in the form of objects. Pony ORM is a library for Python language that allows you to conveniently work with objects that are stored as rows in a relational database.

There are other popular mappers implemented in Python such as Django and SQLAlchemy, but we believe that Pony has some distinct advantages:

 * An exceptionally convenient syntax for writing queries
 * Automatic query optimization
 * An elegant solution for the N+1 problem
 * The `online database schema editor <https://editor.ponyorm.com>`_

In comparison to Django, Pony provides:

 * The IdentityMap pattern
 * Automatic transaction management 
 * Automatic caching of queries and objects
 * Full support of composite keys
 * The ability to easily write queries using LEFT JOIN, HAVING and other features of SQL

One interesting feature of Pony is that it allows interacting with the database in pure Python using the generator expressions or lambda functions, which are then translated into SQL. Such queries may easily be written by a developer familiar with Python, even without being a database expert. Here is an example of a query using the generator expression syntax:

.. code-block:: python

   select(c for c in Customer if sum(c.orders.total_price) > 1000)

Here is the same query written using the lambda function:

.. code-block:: python

   Customer.select(lambda c: sum(c.orders.total_price) > 1000)

In this query, we retrieve all customers with total amount of purchases exceeding 1000. The query to the database is described in the form of Python generator expression and passed to the :py:func:`select` function as an argument. Pony doesn't execute this generator, but translates it into SQL and then sends to the database. Using such approach any developer can write database queries without being an expert in SQL.

The ``Customer`` is an entity class that is initially described when the application is created, and linked to a table in the database.

Not every object-relational mapper offers such a convenient query syntax. In addition to ease of use, Pony ensures efficient work with data. Queries are translated into SQL that is executed quickly and efficiently. Depending on the DBMS, the syntax of the generating SQL may vary in order to use features of the chosen database. The query code written in Python will look the same regardless of the DBMS, which ensures the application’s portability.

With Pony any developer can write complex and effective queries, even without being an expert in SQL. At the same time, Pony does not "fight" with SQL – if a developer needs to write a query in raw SQL, for example to call up a stored procedure, he or she can easily do this with Pony. The main goal of Pony ORM is to simplify the process of development of web applications.

Starting with the version 0.7, Pony ORM is released under the Apache License, Version 2.0.


PonyORM community
-----------------

If you have any questions, please post them on `Stack Overflow <http://stackoverflow.com/questions/tagged/ponyorm>`_.
Meet the PonyORM team, chat with the community members, and get your questions answered on our community `Telegram group <https://telegram.me/ponyorm>`_. Join our newsletter at `ponyorm.com <https://ponyorm.com>`_. Reach us on `Twitter <https://twitter.com/ponyorm>`_ or contact the Pony ORM team by e-mail: team (at) ponyorm.com.