JSON Data Type Support
======================


Overview
--------

Recently native JSON data type support has been added in all major database systems. JSON support introduces dynamic data structures commonly found in NoSQL databases. Usually they are used when working with highly varying data or when the exact data structure is hard to predict.

Pony allows working with JSON data stored in your database using Python syntax.



Declaring a JSON attribute
--------------------------

For declaring a JSON attribute with Pony you should use the ``Json`` type. This type can imported from ``pony.orm`` package:

.. code-block:: python

   from pony.orm import *

   db = Database()


   class Product(db.Entity):
       id = PrimaryKey(int, auto=True)
       name = Required(str)
       info = Required(Json)
       tags = Optional(Json)


   db.bind('sqlite', ':memory:', create_db=True)
   db.generate_mapping(create_tables=True)


The ``info`` attribute in the ``Product`` entity is declared as ``Json``. This allows us to have different JSON structures for different product types, and make queries to this data.


Assigning value to the JSON attribute
-------------------------------------

Usually, a JSON structure contains a combination of dictionaries and lists containing simple types, such as numbers, strings and boolean. Let's create a couple of objects with a simple JSON structure:

.. code-block:: python

    p1 = Product(name='Samsung Galaxy S7 edge',
                 info={
                     'display': {
                        'size': 5.5,
                     },
                     'battery': 3600,
                     '3.5mm jack': True,
                     'SD card slot': True,
                     'colors': ['Black', 'Grey', 'Gold'],
                 },
                 tags=['Smartphone', 'Samsung'])

    p2 = Product(name='iPhone 6s',
                 info={
                     'display': {
                        'size': 4.7,
                        'resolution': [750, 1334],
                        'multi-touch': True,
                     },
                     'battery': 1810,
                     '3.5mm jack': True,
                     'colors': ['Silver', 'Gold', 'Space Gray', 'Rose Gold'],
                 },
                 tags=['Smartphone', 'Apple', 'Retina'])

In Python code a JSON structure is represented with the help of the standard Python dict and list. In our example, the ``info`` attribute is assigned with a dict. The ``tags`` attribute keeps a list of tags. These attributes will be serialized to JSON and stored in the database on commit.


Reading JSON attribute
----------------------

You can read a JSON attribute as any other entity attribute:

.. code-block:: python

   >>> Product[1].info
   {'battery': 3600, '3.5mm jack': True, 'colors': ['Black', 'Grey', 'Gold'],
   'display': 5.5}

Once JSON attribute is extracted from the database, it is deserialized and represented as a combination of dicts and lists. You can use the standard Python dict and list API for working with the value:

.. code-block:: python

   >>> Product[1].info['colors']
   ['Black', 'Grey', 'Gold']

   >>> Product[1].info['colors'][0]
   'Black'

   >>> 'Black' in Product[1].info['colors']
   True


Modifying JSON attribute
------------------------

For modifying the JSON attribute value, you use the standard Python list and dict API as well:

.. code-block:: python

   >>> Product[1].info['colors'].append('Silver')
   >>> Product[1].info['colors']
   ['Black', 'Grey', 'Gold', 'Silver']

Now, on commit, the changes will be stored in the database. In order to track the changes made in the JSON structure, Pony uses its own dict and list implementations which inherit from the standard Python dict and list.

Below is a couple more examples of how you can modify the the JSON value.

.. code-block:: python

   p = Product[1]

   # assigning a new value
   p.info['display']['size'] = 4.7

   # popping a dict value
   display_size = p.info['display'].pop('size')

   # removing a dict key using del
   del p.info['display']

   # adding a dict key
   p.info['display']['resolution'] = [1440, 2560]

   # removing a list item
   del p.info['colors'][0]

   # replacing a list item
   p.info['colors'][1] = ['White']

   # replacing a number of list items
   p.info['colors'][1:] = ['White']

All of the actions above are regular Python operations with attributes, lists and dicts.



Querying JSON structures
------------------------

Native JSON support in databases allows not only read and modify structured data, but also making queries. It is a very powerful feature - the queries use the same syntax and run in the same ACID transactional environment, in the same time offering NoSQL capabilities of a document store inside the relational database.

Pony allows selecting objects by filtering them by JSON sub-elements. To access JSON sub-element Pony constructs JSON path expression which then will be used inside a SQL query:

.. code-block:: python

   # products with display size greater than 5
   Product.select(lambda p: p.info['display']['size'] > 5)

In order to specify values you can use parameters:

.. code-block:: python

   x = 2048
   # products with width resolution greater or equal to x
   Product.select(lambda p: p.info['display']['resolution'][0] >= x)

In MySQL, PostgreSQL and SQLite it is also possible to use parameters inside JSON path expression:

.. code-block:: python

   index = 0
   Product.select(lambda p: p.info['display']['resolution'][index] < 2000)

   key = 'display'
   Product.select(lambda p: p.info[key]['resolution'][index] > 1000)

.. note:: Oracle does not support parameters inside JSON paths. With Oracle you can use constant keys only.

For JSON array you can calculate length:

.. code-block:: python

   # products with more than 2 tags
   Product.select(lambda p: len(p.info['tags']) > 2)

Another query example is checking if a string key is a part of a JSON dict or array:

.. code-block:: python

   # products which have the resolution specified
   Product.select(lambda p: 'resolution' in p.info['display'])

   # products of black color
   Product.select(lambda p: 'Black' in p.info['colors'])

When you compare JSON sub-element with ``None``, it will be evaluated to ``True`` in the following cases:

 * When the sub-element contains JSON ``null`` value
 * When the sub-element does not exist

.. code-block:: python

   Product.select(lambda p: p.info['SD card slot'] is None)

You can test JSON sub-element for truth value:

.. code-block:: python

   # products with multi-touch displays
   select(p for p in Product if p.info['display']['multi-touch'])

In Python, the following values are treated as false for conditionals: ``None``, 0, ``False``, empty string, empty dict and empty list. Pony keeps this behavior for conditions applied for JSON structures. Also, if the JSON path is not found, it will be evaluated to false.

In previous examples we used JSON structures in query conditions. But it is also possible to retrieve JSON structures or extract its parts as the query result:

.. code-block:: python

   select(p.info['display'] for p in Product)

When retrieving JSON structures this way, they will not be linked to entity instances. This means that modification of such JSON structures will not be saved to the database. Pony tracks JSON changes only when you select an object and modify its attributes.

MySQL and Oracle allows using wildcards in JSON path. Pony support wildcards by using special syntax:

 * [...] means 'any dictionary element'
 * [:] means 'any list item'

Here is a query example:

.. code-block:: python

   select(p.info['display'][...] for p in Product)

The result of such query will be an array of JSON sub-elements. With the current situation of JSON support in databases, the wildcards can be used only in the expression part of the generator expression.


JSON Support in Databases
-------------------------

For storing JSON in the database Pony uses the following types:

 * `SQLite <https://sqlite.org/json1.html>`_ - TEXT

 * `PostgreSQL <https://www.postgresql.org/docs/current/static/functions-json.html>`_ - JSONB (binary JSON)

 * `MySQL <https://dev.mysql.com/doc/refman/5.7/en/json.html>`_ - JSON (binary JSON, although it doesn't have 'B' in the name)

 * `Oracle <https://docs.oracle.com/database/121/ADXDB/json.htm>`_ - CLOB

Starting with the version 3.9 SQLite provides the `JSON1 extension module <https://www.sqlite.org/json1.html>`_. This extension improves performance when working with JSON queries, although Pony can work with JSON in SQLite even without this module.
