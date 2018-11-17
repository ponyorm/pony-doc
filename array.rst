Array Data Type Support
=======================


Overview
--------

Since Pony version 0.7.7 we add support of Array type for PostgreSQL and SQLite. It implements PostgreSQL's arrays.
JSON type is more flexible, but in some cases Array type might be more efficient.

Declaring an Array attribute
----------------------------

Each array should have a specified type of items that this array can store. Supported types are: ``int``, 
``float`` and ``str``.

For declaring an Array attribute with Pony you should use one of the ``IntArray``, ``FloatArray`` or ``StrArray`` types. 
These types can be imported from ``pony.orm`` package:

.. code-block:: python

   from pony.orm import *

   db = Database()


   class Product(db.Entity):
       id = PrimaryKey(int, auto=True)
       name = Required(str)
       stars = Optional(IntArray)
       tags = Optional(StrArray)


   db.bind('sqlite', ':memory:')
   db.generate_mapping(create_tables=True)


.. note::

    For PostgreSQL if you set ``index=True`` Pony will create gin_ index. For SQLite index will be ignored.

.. _gin: https://www.postgresql.org/docs/9.5/gin.html

Operations with arrays
----------------------

Change array's items

.. code-block:: python

    product = Product.select().first()
    product.tags.append('reconstructed')

Select specific item of array

.. code-block:: python

    tags1 = select(p.tags[2] for p in Product)[:]
    tags2 = select(p.tags[-5] for p in Product)[:]

Using slice

.. code-block:: python

    early_tags = select(p.tags[:10] for p in Product)[:]

.. note::
    Steps are not supported for slices.

Check if item or list of items in or not in array

.. code-block:: python

    products1 = select(p for p in Product if 'apple' in p.tags)[:]
    products2 = select(p for p in Product if ['LCD', 'DVD', 'SSD'] in p.tags)[:]



