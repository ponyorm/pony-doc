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

    with db_session:
        Product(name='Apple MacBook', stars=[0, 2, 0, 5, 10], tags=['Apple', 'Notebook'])


.. note::
    Optional arrays are declared as NOT NULL by default and use empty array as default value. To make it nullable you should pass ``nullable=True`` option.

.. note::

    For PostgreSQL if you set ``index=True`` Pony will create gin_ index. For SQLite index will be ignored.

.. _gin: https://www.postgresql.org/docs/9.5/gin.html

Operations with arrays
----------------------

Accessing array items
~~~~~~~~~~~~~~~~~~~~~

In PonyORM array indexes are zero-based, as in Python. It is possible to use negative indexes to access array from the end.
You can also use array slices.

Select specific item of array

.. code-block:: python

    select(p.tags[2] for p in Product)[:]  # third element
    select(p.tags[-1] for p in Product)[:]  # last element

Using slice

.. code-block:: python

    select(p.tags[:5] for p in Product)[:]  # first five elements

.. note::
    Steps are not supported for slices.

Check if item or list of items in or not in array

.. code-block:: python

    select(p for p in Product if 'apple' in p.tags)[:]
    select(p for p in Product if ['LCD', 'DVD', 'SSD'] in p.tags)[:]


Change array's items

.. code-block:: python

    product = Product.select().first()
    product.tags.remove('factory-new')
    product.tags.append('reconstructed')
