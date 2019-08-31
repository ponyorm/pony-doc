Migrations
==========

In Pony version 0.9 we release migrations support. 
Now migrations are implemented only for PostgreSQL and SQLite.


Command Line Inteface
---------------------

You can work with migrations via command line interface (cli). To do so you define your entities as always but instead of calling ``db.generate_mapping`` you call ``db.migrate``.

Example of file ``migrate.py``

.. code-block:: python

    from pony.orm import *
    
    db = Database('sqlite', 'db.sqlite')
    
    
    class Student(db.Entity):
        fullname = Required(str)
        age = Required(int)
        group = Required('Group')
        
        
    class Group(db.Entity):
        number = Required(int)
        students = Set(Student)
        
        
    db.migrate()
    
After you have python file with ``db.migrate`` call you can use it with cli (e.g. ``python migrate.py make``).

Pony stores migrations in files..

Below is the list of all available operations with cli.

make
~~~~

This command creates new migration based on difference it can calculate automatically. Pony uses old migrations to restore the previous state of entities and compare them. Difference is stores as list of operations. !!MAKE LINK!!.

.. note::
    Pony does not automatically detect renaming entities or attributes. If you rename attribute and run ``make`` command - Pony will detect attribute deletion and creating new attribute. For renaming use rename command !!MAKE LINK!!
    
After calculating the difference Pony saves it in migration file. Default migration file contains of dependecies list and operations list. 

    
