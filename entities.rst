Declaring Entities
==================

Entities are Python classes which store an object’s state in the database. Each instance of an entity corresponds to a row in the database table. Often entities represent objects from the real world (e.g. Customer, Product).

Before creating entity instances you need to map entities to the database tables. Pony can map entities to existing tables or create new tables. After the mapping is generated you can query the database and create new instances of entities.

Pony provides an `entity-relationship diagram editor <https://editor.ponyorm.com>`_ which can be used for creating Python entity declarations.


Declaring an entity
-------------------

Each entity belongs to a database. That is why before defining entities you need to create an object of the ``Database`` class:

.. code-block:: python

    from pony.orm import *

    db = Database()

    class MyEntity(db.Entity):
        attr1 = Required(str)

The Pony’s Database object has the ``Entity`` attribute which is used as a base class for all the entities stored in this database. Each new entity that is defined must inherit from this ``Entity`` class.


Entity attributes
-----------------

Entity attributes are specified as class attributes inside the entity class using the syntax ``attr_name = kind(type, options)``:

.. code-block:: python

    class Customer(db.Entity):
        name = Required(str)
        email = Required(str, unique=True)

In the parentheses, after the attribute type, you can specify attribute options.

Each attribute can be one of the following kinds:

* Required
* Optional
* PrimaryKey
* Set

Required and Optional
~~~~~~~~~~~~~~~~~~~~~

Usually most entity attributes are of ``Required`` or ``Optional`` kind. If an attribute is defined as ``Required`` then it must have a value at all times, while ``Optional`` attributes can be empty.

If you need the value of an attribute to be unique then you can set the attribute option ``unique=True``.



PrimaryKey
~~~~~~~~~~

``PrimaryKey`` defines an attribute which is used as a primary key in the database table. Each entity should always have a primary key. If the primary key is not specified explicitly, Pony will create it implicitly. Let’s consider the following example:

.. code-block:: python

    class Product(db.Entity):
        name = Required(str, unique=True)
        price = Required(Decimal)
        description = Optional(str)

The entity definition above will be equal to the following:

.. code-block:: python

    class Product(db.Entity):
        id = PrimaryKey(int, auto=True)
        name = Required(str, unique=True)
        price = Required(Decimal)
        description = Optional(str)

The primary key attribute which Pony adds automatically always will have the name ``id`` and ``int`` type. The option ``auto=True`` means that the value for this attribute will be assigned automatically using the database’s incremental counter or a sequence.

If you specify the primary key attribute yourself, it can have any name and type. For example, we can define the entity ``Customer`` and have customer’s email as the primary key:

.. code-block:: python

    class Customer(db.Entity):
       email = PrimaryKey(str)
       name = Required(str)


Set
~~~

A ``Set`` attribute represents a collection. Also we call it a relationship, because such attribute relates to an entity. You need to specify an entity as the type for the ``Set`` attribute. This is the way to define one side for the to-many relationships. As of now, Pony doesn’t allow the use of ``Set`` with primitive types. We plan to add this feature later.

We will talk in more detail about this attribute type in the :ref:`Entity relationships <entity_relationships>` chapter.


Composite keys
~~~~~~~~~~~~~~

Pony fully supports composite keys. In order to declare a composite primary key you need to specify all the parts of the key as ``Required`` and then combine them into a composite primary key:

.. code-block:: python

    class Example(db.Entity):
        a = Required(int)
        b = Required(str)
        PrimaryKey(a, b)

Here ``PrimaryKey(a, b)`` doesn’t create an attribute, but combines the attributes specified in the parenthesis into a composite primary key. Each entity can have only one primary key.

In order to declare a secondary composite key you need to declare attributes as usual and then combine them using the ``composite_key`` directive:

.. code-block:: python

    class Example(db.Entity):
        a = Required(str)
        b = Optional(int)
        composite_key(a, b)

In the database ``composite_key(a, b)`` will be represented as the ``UNIQUE ("a", "b")`` constraint.

If have just one attribute, which represents a unique key, you can create such a key by specifying ``unique=True`` by an attribute:

.. code-block:: python

    class Product(db.Entity):
        name = Required(str, unique=True)


Composite indexes
~~~~~~~~~~~~~~~~~

Using the ``composite_index()`` directive you can create a composite index for speeding up data retrieval. It can combine two or more attributes:

.. code-block:: python

    class Example(db.Entity):
        a = Required(str)
        b = Optional(int)
        composite_index(a, b)

You can use the attribute or the attribute name:

.. code-block:: python

    class Example(db.Entity):
        a = Required(str)
        b = Optional(int)
        composite_index(a, 'b')

If you want to create a non-unique index for just one column, you can specify the ``index`` option of an attribute.

The composite index can include a discriminator attribute used for inheritance.


Attribute data types
--------------------

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
* IntArray - array of integers
* StrArray - array of strings
* FloatArray - array of floats

See the :ref:`Attribute types <attribute_types>` part of the API Reference for more information.


Attribute options
-----------------

You can specify additional options during attribute definitions using positional and keyword arguments. See the :ref:`Attribute options <attribute_options>` part of the API Reference for more information.


.. _entity_inheritance:

Entity inheritance
------------------

Entity inheritance in Pony is similar to inheritance for regular Python classes. Let’s consider an example of a data diagram where entities ``Student`` and ``Professor`` inherit from the entity ``Person``:

.. code-block:: python

    class Person(db.Entity):
        name = Required(str)

    class Student(Person):
        gpa = Optional(Decimal)
        mentor = Optional("Professor")

    class Professor(Person):
        degree = Required(str)
        students = Set("Student")

All attributes and relationships of the base entity ``Person`` are inherited by all descendants.

In some mappers (e.g. Django) a query on a base entity doesn’t return the right class: for derived entities the query returns just a base part of each instance. With Pony you always get the correct entity instances:

.. code-block:: python

    for p in Person.select():
        if isinstance(p, Professor):
            print p.name, p.degree
        elif isinstance(p, Student):
            print p.name, p.gpa
        else:  # somebody else
            print p.name

.. note:: Since version 0.7.7 you can use `isinstance()` inside query

.. code-block:: python

    staff = select(p for p in Person if not isinstance(p, Student))

In order to create the correct entity instance Pony uses a discriminator column. By default this is a string column and Pony uses it to store the entity class name:

.. code-block:: python

    classtype = Discriminator(str)

By default Pony implicitly creates the ``classtype`` attribute for each entity class which takes part in inheritance. You can use your own discriminator column name and type. If you change the type of the discriminator column, then you have to specify the ``_discrimintator_`` value for each entity.

Let’s consider the example above and use ``cls_id`` as the name for our discriminator column of ``int`` type:

.. code-block:: python

    class Person(db.Entity):
        cls_id = Discriminator(int)
        _discriminator_ = 1
        ...

    class Student(Person):
        _discriminator_ = 2
        ...

    class Professor(Person):
        _discriminator_ = 3
        ...


Multiple inheritance
~~~~~~~~~~~~~~~~~~~~

Pony also supports multiple inheritance. If you use multiple inheritance then all the parent classes of the newly defined class should inherit from the same base class (a "diamond-like" hierarchy).

Let’s consider an example where a student can have a role of a teaching assistant. For this purpose we’ll introduce the entity ``Teacher`` and derive ``Professor`` and ``TeachingAssistant`` from it. The entity ``TeachingAssistant`` inherits from both the ``Student`` class and the ``Teacher`` class:

.. code-block:: python

    class Person(db.Entity):
        name = Required(str)

    class Student(Person):
        ...

    class Teacher(Person):
        ...

    class Professor(Teacher):
        ...

    class TeachingAssistant(Student, Teacher):
        ...

The ``TeachingAssistant`` objects are instances of both ``Teacher`` and ``Student`` entities and inherit all their attributes. Multiple inheritance is possible here because both ``Teacher`` and ``Student`` have the same base class ``Person``.

Inheritance is a very powerful tool, but it should be used wisely. Often the data diagram is much simpler if it has limited usage of inheritance.


Representing inheritance in the database
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

There are three ways to implement inheritance in the database:

1. Single Table Inheritance: all entities in the hierarchy are mapped to a single database table.
2. Class Table Inheritance: each entity in the hierarchy is mapped to a separate table, but each table stores only the attributes which the entity doesn’t inherit from its parents.
3. Concrete Table Inheritance: each entity in the hierarchy is mapped to a separate table and each table stores the attributes of the entity and all its ancestors.

The main problem of the third approach is that there is no single table where we can store the primary key and that is why this implementation is rarely used.

The second implementation is used often, this is how the inheritance is implemented in Django. The disadvantage of this approach is that the mapper has to join several tables together in order to retrieve data which can lead to the performance degradation.

Pony uses the first approach where all entities in the hierarchy are mapped to a single database table. This is the most efficient implementation because there is no need to join tables. This approach has its disadvantages too:

* Each table row has columns which are not used because they belong to other entities in the hierarchy. It is not a big problem because the blank columns keep ``NULL`` values and it doesn’t use much space.
* The table can have large number of columns if there are a lot of entities in the hierarchy. Different databases have different limits for maximum columns per table, but usually that limit is pretty high.


Adding custom methods to entities
---------------------------------

Besides data attributes, entities can have methods. The most straightforward way of adding methods to entities is defining those methods in the entity class. Let's say we would like to have a method of the Product entity which returns concatenated name and price. It can be done the following way:

.. code-block:: python

    class Product(db.Entity):
        name = Required(str, unique=True)
        price = Required(Decimal)

        def get_name_and_price(self):
            return "%s (%s)" % (self.name, self.price)

Another approach is using mixin classes. Instead of putting custom methods directly to the entity definition, you can define them in a separate mixin class and inherit entity class from that mixin:

.. code-block:: python

    class ProductMixin(object):
        def get_name_and_price(self):
            return "%s (%s)" % (self.name, self.price)

    class Product(db.Entity, ProductMixin):
        name = Required(str, unique=True)
        price = Required(Decimal)

This approach can be beneficial if you are using our `online ER diagram editor <https://editor.ponyorm.com>`_. The editor automatically generates entity definitions in accordance with the diagram. In this case, if you add some custom methods to the entity definition, these methods will be overwritten once you change your diagram and save newly generated entity definitions. Using mixins would allow you to separate entity definitions and mixin classes with methods into two different files. This way you can overwrite your entity definitions without losing your custom methods.

For our example above the separation can be done in the following way.

File mixins.py:

.. code-block:: python

    class ProductMixin(object):
        def get_name_and_price(self):
            return "%s (%s)" % (self.name, self.price)

File models.py:

.. code-block:: python

    from decimal import Decimal
    from pony.orm import *
    from mixins import *

    class Product(db.Entity, ProductMixin):
        name = Required(str, unique=True)
        price = Required(Decimal)



.. _mapping_customization:

Mapping customization
---------------------

When Pony creates tables from entity definitions, it uses the name of entity as the table name and attribute names as the column names, but you can override this behavior.

The name of the table is not always equal to the name of an entity: in MySQL and PostgreSQL the default table name generated from the entity name will be converted to the lower case, in Oracle - to the upper case. You can always find the name of the entity table by reading the ``_table_`` attribute of an entity class.

If you need to set your own table name use the ``_table_`` class attribute:

.. code-block:: python

    class Person(db.Entity):
        _table_ = "person_table"
        name = Required(str)

Also you can set schema name:

.. code-block:: python

    class Person(db.Entity):
        _table_ = ("my_schema", "person_table")
        name = Required(str)

If you need to set your own column name, use the option ``column``:

.. code-block:: python

    class Person(db.Entity):
        _table_ = "person_table"
        name = Required(str, column="person_name")

Also you can specify the ``_table_options_`` for the table. It can be used when you need to set options like ``ENGINE`` or ``TABLESPACE``. See :ref:`Entity options <entity_options>` part of the API reference for more detail.

For composite attributes use the option ``columns`` with the list of the column names specified:

.. code-block:: python

    class Course(db.Entity):
        name = Required(str)
        semester = Required(int)
        lectures = Set("Lecture")
        PrimaryKey(name, semester)

    class Lecture(db.Entity):
        date = Required(datetime)
        course = Required(Course, columns=["name_of_course", "semester"])

In this example we override the column names for the composite attribute ``Lecture.course``. By default Pony will generate the following column names: ``"course_name"`` and ``"course_semester"``. Pony combines the entity name and the attribute name in order to make the column names easy to understand to the developer.

If you need to set the column names for the intermediate table for many-to-many relationship, you should specify the option ``column`` or ``columns`` for the ``Set`` attributes. Let’s consider the following example:

.. code-block:: python

    class Student(db.Entity):
        name = Required(str)
        courses = Set("Course")

    class Course(db.Entity):
        name = Required(str)
        semester = Required(int)
        students = Set(Student)
        PrimaryKey(name, semester)

By default, for storing many-to-many relationships between ``Student`` and ``Course``, Pony will create an intermediate table ``"Course_Student"`` (it constructs the name of the intermediate table from the entity names in the alphabetical order). This table will have three columns: ``"course_name"``, ``"course_semester"`` and ``"student"`` - two columns for the ``Course``’s composite primary key and one column for the ``Student``. Now let’s say we want to name the intermediate table as ``"Study_Plans"`` which have the following columns: ``"course"``, ``"semester"`` and ``"student_id"``. This is how we can achieve this:

.. code-block:: python

    class Student(db.Entity):
        name = Required(str)
        courses = Set("Course", table="Study_Plans", columns=["course", "semester"]))

    class Course(db.Entity):
        name = Required(str)
        semester = Required(int)
        students = Set(Student, column="student_id")
        PrimaryKey(name, semester)
        
You can find more examples of mapping customization in `an example which comes with Pony ORM package <https://github.com/ponyorm/pony/blob/orm/pony/orm/examples/university1.py>`_
        
.. _hybrid_methods_and_properties:

Hybrid methods and properties
-----------------------------

*(new in version 0.7.4)*

You can declare methods and properties inside your entity that you can use in queries. Important that hybrids and properties should contain single line return statement.

.. code-block:: python

    class Person(db.Entity):
        first_name = Required(str)
        last_name = Required(str)
        cars = Set(lambda: Car)

        @property
        def full_name(self):
            return self.first_name + ' ' + self.last_name

        @property
        def has_car(self):
            return not self.cars.is_empty()

        def cars_by_color(self, color):
            return select(car for car in self.cars if car.color == color)
            # or return self.cars.select(lambda car: car.color == color)

        @property
        def cars_price(self):
            return sum(c.price for c in self.cars)
            
        
    class Car(db.Entity):
        brand = Required(str)
        model = Required(str)
        owner = Optional(Person)
        year = Required(int)
        price = Required(int)
        color = Required(str)
            
    with db_session:
        # persons' full name
        select(p.full_name for p in Person)
            
        # persons who have a car
        select(p for p in Person if p.has_car)
            
        # persons who have yellow cars
        select(p for p in Person if count(p.cars_by_color('yellow')) > 1)
            
        # sum of all cars that have owners
        sum(p.cars_price for p in Person)





