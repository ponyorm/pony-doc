.. _entity_relationships:

Working with entity relationships
=================================

In Pony, an entity can relate to other entities through relationship attributes. Each relationship always has two ends, and is defined by two entity attributes:

.. code-block:: python

    class Person(db.Entity):
        cars = Set('Car')

    class Car(db.Entity):
        owner = Optional(Person)

In the example above we've defined one-to-many relationship between the ``Person`` and ``Car`` entities using the ``Person.cars`` and ``Car.owner`` attributes.

Let's add a couple more data attributes to our entities:

.. code-block:: python

    from pony.orm import *

    db = Database()

    class Person(db.Entity):
        name = Required(str)
        cars = Set('Car')

    class Car(db.Entity):
        make = Required(str)
        model = Required(str)
        owner = Optional(Person)

    db.bind('sqlite', ':memory:')
    db.generate_mapping(create_tables=True)

Now let's create instances of ``Person`` and ``Car`` entities:

.. code-block:: python

    >>> p1 = Person(name='John')
    >>> c1 = Car(make='Toyota', model='Camry')
    >>> commit()

Normally, in your program, you don't need to call the function :py:func:`commit` manually, because it should be called automatically by the :py:func:`db_session`. But when you work in the interactive mode, you never leave a :py:func:`db_session`, that is why we need to commit manually if we want to store data in the database.


Establishing a relationship
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Right after we've created the instances ``p1`` and ``c1``, they don't have an established relationship. Let's check the values of the relationship attributes:

.. code-block:: python

    >>> print c1.owner
    None

    >>> print p1.cars
    CarSet([])

The attribute ``cars`` has an empty set.

Now let's establish a relationship between these two instances:

.. code-block:: python

    >>> c1.owner = p1

If we print the values of relationship attributes now, then we'll see the following:

.. code-block:: python

    >>> print c1.owner
    Person[1]

    >>> print p1.cars
    CarSet([Car[1]])

When we assigned an owner to the ``Car`` instance, the ``Person.cars`` relationship attribute reflected the change immediately.

We also could establish a relationship by assigning the relationship attribute during the creation of the ``Car`` instance:

.. code-block:: python

    >>> p1 = Person(name='John')
    >>> c1 = Car(make='Toyota', model='Camry', owner=p1)

In our example the attribute ``owner`` is optional, so we can assign a value to it at any time, either during the creation of the ``Car`` instance, or later.



Operations with collections
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The attribute ``Person.cars`` is represented as a collection and hence we can use regular operations that applicable to collections: ``add()``, ``remove()``, ``in``, ``len()``, ``clear()``.

You can add or remove relationships using the :py:meth:`Set.add` and :py:meth:`Set.remove` methods:

.. code-block:: python

    >>> p1.cars.remove(Car[1])
    >>> print p1.cars
    CarSet([])

    >>> p1.cars.add(Car[1])
    >>> print p1.cars
    CarSet([Car[1]])

You can check if a collection contains an element:

.. code-block:: python

    >>> Car[1] in p1.cars
    True

Or make sure that there is no such element in the collection:

.. code-block:: python

    >>> Car[1] not in p1.cars
    False

Check the collection length:

.. code-block:: python

    >>> len(p1.cars)
    1

If you need to create an instance of a car and assign it with a particular person instance, there are several ways to do it. One of the options is to call the :py:meth:`~Set.create` method of the collection attribute:

.. code-block:: python

    >>> p1.cars.create(model='Toyota', make='Prius')
    >>> commit()

Now we can check that a new ``Car`` instance was added to the ``Person.cars`` collection attribute of our instance:

.. code-block:: python

    >>> print p1.cars
    CarSet([Car[2], Car[1]])
    >>> p1.cars.count()
    2

You can iterate over a collection attribute:

.. code-block:: python

    >>> for car in p1.cars:
    ...     print car.model

    Toyota
    Camry


.. _attribute_lifting:

Attribute lifting
-----------------

In Pony, the collection attributes provides the attribute lifting capability: the collection obtains its items' attributes:

.. code-block:: python

    >>> show(Car)
    class Car(Entity):
        id = PrimaryKey(int, auto=True)
        make = Required(str)
        model = Required(str)
        owner = Optional(Person)
    >>> p1 = Person[1]
    >>> print p1.cars.model
    Multiset({u'Camry': 1, u'Prius': 1})

Here we print out the entity declaration using the :py:func:`show` function and then print the value of the ``model`` attribute of the ``cars`` relationship attribute. The ``cars`` attribute has all the attributes of the ``Car`` entity: ``id``, ``make``, ``model`` and ``owner``. In Pony we call this a Multiset and it is implemented using a dictionary. The dictionary's key represents the value of the attribute - 'Camry' and 'Prius' in our example. And the dictionary's value shows how many times it encounters in this collection.

.. code-block:: python

    >>> print p1.cars.make
    Multiset({u'Toyota': 2})

``Person[1]`` has two Toyotas.

We can iterate over the multiset:

.. code-block:: python

    >>> for m in p1.cars.make:
    ...     print m
    ...
    Toyota
    Toyota


Collection attribute parameters
-------------------------------

Here is the list options that you can apply to collection attributes:

* :option:`cascade_delete`
* :option:`columns`
* :option:`lazy`
* :option:`nplus1_threshold`
* :option:`reverse`
* :option:`reverse_columns`
* :option:`table`

Example:

.. code-block:: python

    class Photo(db.Entity):
        tags = Set('Tag', table='photo_to_tag', column='tag_id')

    class Tag(db.Entity):
        photos = Set(Photo)



Collection attribute queries and other methods
----------------------------------------------

Starting with the release 0.6.1, Pony introduces queries for the relationship attributes.

You can use the following methods of the relationship attribute for retrieving data:

* :py:meth:`Set.select`
* :py:meth:`Set.random`
* :py:meth:`Set.page`
* :py:meth:`Set.order_by`
* :py:meth:`Set.load`
* :py:meth:`Set.filter`

See the :ref:`Collection attribute methods <collection_attribute_methods>` part of th API Reference for more details.

Below you can find several examples of using these methods. We'll use the University schema for showing these queries, here are `python entity definitions <https://github.com/ponyorm/pony/blob/orm/pony/orm/examples/university1.py>`_ and  `Entity-Relationship diagram <https://editor.ponyorm.com/user/pony/University>`_.

The example below selects all students with the ``gpa`` greater than 3 within the group 101:

.. code-block:: python

    g = Group[101]
    g.students.filter(lambda student: student.gpa > 3)[:]

This query can be used for displaying the second page of group 101 student's list ordered by the ``name`` attribute:

.. code-block:: python

    g.students.order_by(Student.name).page(2, pagesize=3)

The same query can be also written in the following form:

.. code-block:: python

    g.students.order_by(lambda s: s.name).limit(3, offset=3)

The following query returns two random students from the group 101:

.. code-block:: python

    g.students.random(2)

And one more example. This query returns the first page of courses which were taken by ``Student[1]`` in the second semester, ordered by the course name:

.. code-block:: python

    s = Student[1]
    s.courses.select(lambda c: c.semester == 2).order_by(Course.name).page(1)

