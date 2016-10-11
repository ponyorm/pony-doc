Working with entity instances
=============================


Creating an entity instance
---------------------------

Creating an entity instance in Pony is similar to creating a regular object in Python:

.. code-block:: python

   customer1 = Customer(login="John", password="***",
                        name="John", email="john@google.com")

When creating an object in Pony, all the parameters should be specified as keyword arguments. If an attribute has a default value, you can omit it.

All created instances belong to the current :py:func:`db_session`. In some object-relational mappers, you are required to call an object's ``save()`` method in order to save it. This is inconvenient, as a programmer must track which objects were created or updated, and must not forget to call the ``save()`` method on each object.

Pony tracks which objects were created or updated and saves them in the database automatically when current :py:func:`db_session` is over. If you need to save newly created objects before leaving the :py:func:`db_session` scope, you can do so by using the :py:func:`flush` or :py:func:`commit` functions.


Loading objects from the database
---------------------------------

Getting an object by primary key
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The simplest case is when we want to retrieve an object by its primary key. To accomplish this in Pony, the user simply needs to put the primary key in square brackets, after the class name. For example, to extract a customer with the primary key value of 123, we can write:

.. code-block:: python

   customer1 = Customer[123]

The same syntax also works for objects with composite keys; we just need to list the elements of the composite primary key, separated by commas, in the same order that the attributes were defined in the entity class description:

.. code-block:: python

   order_item = OrderItem[order1, product1]

Pony raises the ``ObjectNotFound`` exception if object with such primary key doesn't exist.


Getting one object by unique combination of attributes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you want to retrieve one object not by its primary key, but by another combination of attributes, you can use the :py:meth:`~Entity.get` method of an entity. In most cases, it is used for getting an object by the secondary unique key, but it can also be used to search by any other combination of attributes. As a parameter of the :py:meth:`~Entity.get` method, you need to specify the names of the attributes and their values. For example, if you want to receive a product under the name "Product 1", and you believe that database has only one product under this name, you can write:

.. code-block:: python

   product1 = Product.get(name='Product1')

If no object is found, :py:meth:`~Entity.get` returns ``None``. If multiple objects are found, ``MultipleObjectsFoundError`` exception is raised.

You may want to use the :py:meth:`~Entity.get` method with primary key when we want to get ``None`` instead of ``ObjectNotFound`` exception if the object does not exists in database.

Method :py:meth:`~Entity.get` can also receive a lambda function as a single positioning argument. This method returns an instance of an entity, and not an object of the :py:class:`Query` class.


Getting several objects
~~~~~~~~~~~~~~~~~~~~~~~

In order to retrieve several objects from a database, you should use the :py:meth:`~Entity.select()` method of an entity. Its argument is a lambda function, which has a single parameter, symbolizing an instance of an object in the database. Inside this function, you can write conditions, by which you want to select objects. For example, if you want to find all products with the price higher than 100, you can write:

.. code-block:: python

    products = Product.select(lambda p: p.price > 100)

This lambda function will not be executed in Python. Instead, it will be translated into the following SQL query:

.. code-block:: sql

    SELECT "p"."id", "p"."name", "p"."description",
           "p"."picture", "p"."price", "p"."quantity"
    FROM "Product" "p"
    WHERE "p"."price" > 100

The :py:meth:`~Entity.select()` method returns an instance of the :py:class:`Query` class. If you start iterating over this object, the SQL query will be sent to the database and you will get the sequence of entity instances. For example, this is how you can print out all product names and it's price:

.. code-block:: python

    for p in Product.select(lambda p: p.price > 100):
        print(p.name, p.price)

If you don't want to iterate over a query, but need just to get a list of objects, you can do so this way:

.. code-block:: python

    product_list = Product.select(lambda p: p.price > 100)[:]

Here we get a full slice ``[:]`` from the query. This is an equivalent of converting a query to a list:

.. code-block:: python

    product_list = list(Product.select(lambda p: p.price > 100))



Using parameters in queries
~~~~~~~~~~~~~~~~~~~~~~~~~~~

You can use variables in queries. Pony will pass those variables as parameters to the SQL query. One important advantage of declarative query syntax in Pony is that it offers full protection from SQL-injections, as all external parameters will be properly escaped.

Here is the example:

.. code-block:: python

    x = 100
    products = Product.select(lambda p: p.price > x)

The SQL query which will be generated will look this way:

.. code-block:: sql

    SELECT "p"."id", "p"."name", "p"."description",
           "p"."picture", "p"."price", "p"."quantity"
    FROM "Product" "p"
    WHERE "p"."price" > ?

This way the value of ``x`` will be passed as the SQL query parameter, which completely eliminates the risk of SQL-injection.


Sorting query results
~~~~~~~~~~~~~~~~~~~~~

If you need to sort objects in a certain order, you can use the :py:meth:`Query.order_by` method.

.. code-block:: python

    Product.select(lambda p: p.price > 100).order_by(desc(Product.price))

In this example, we display names and prices of all products with price higher than 100 in a descending order.

The methods of the :py:class:`Query` object modify the SQL query which will be sent to the database. Here is the SQL generated for the previous example:

.. code-block:: sql

    SELECT "p"."id", "p"."name", "p"."description",
           "p"."picture", "p"."price", "p"."quantity"
    FROM "Product" "p"
    WHERE "p"."price" > 100
    ORDER BY "p"."price" DESC

The :py:meth:`Query.order_by` method can also receive a lambda function as a parameter:

.. code-block:: python

   Product.select(lambda p: p.price > 100).order_by(lambda p: desc(p.price))

Using the lambda function inside the .. code-block:: python method allows using advanced sorting expressions. For example, this is how you can sort our customers by the total price of their orders in the descending order:

.. code-block:: python

    Customer.select().order_by(lambda c: desc(sum(c.orders.total_price)))

In order to sort the result by several attributes, you need to separate them by a comma. For example, if you want to sort products by price in descending order, while displaying products with similar prices in alphabetical order, you can do it this way:

.. code-block:: python

    Product.select(lambda p: p.price > 100).order_by(desc(Product.price), Product.name)

The same query, but using lambda function will look this way:

.. code-block:: python

    Product.select(lambda p: p.price > 100).order_by(lambda p: (desc(p.price), p.name))

Note that according to Python syntax, if you return more than one element from lambda, you need to put them into parenthesis.


Limiting the number of selected objects
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

It is possible to limit the number of objects returned by a query by using the :py:meth:`Query.limit` method, or by more compact Python slice notation. For example, this is how you can get the ten most expensive products:

.. code-block:: python

    Product.select().order_by(lambda p: desc(p.price))[:10]

The result of a slice is not a query object, but a final list of entity instances.

You can also use the :py:meth:`Query.page` method as a convenient way of pagination the query results:

.. code-block:: python

    Product.select().order_by(lambda p: desc(p.price)).page(1)


Traversing relationships
~~~~~~~~~~~~~~~~~~~~~~~~

In Pony you can traverse object relationships:

.. code-block:: python

    order = Order[123]
    customer = order.customer
    print customer.name

Pony tries to minimize the number of queries sent to the database. In the example above, if the requested ``Customer`` object was already loaded to the cache, Pony will return the object from the cache without sending a query to the database. But, if an object was not loaded yet, Pony still will not send a query immediately. Instead, it will create a "seed" object first. The seed is an object which has only the primary key initialized. Pony does not know how this object will be used, and there is always the possibility that only the primary key is needed.

In the example above, Pony get the object from database in the third line in, when accessing the ``name`` attribute. By using the "seed" concept, Pony achieves high efficiency and solves the "N+1" problem, which is a weakness of many other mappers.

Traversing is possible in the "to-many" direction as well. For example, if you have a ``Customer`` object and you loop through its ``orders`` attribute, you can do it this way:

.. code-block:: python

    c = Customer[123]
    for order in c.orders:
        print order.state, order.price


Updating an object
------------------

When you assign new values to object attributes, you don't need to save each updated object manually. Changes will be saved in the database automatically on leaving the :py:func:`db_session` scope.

For example, in order to increase the number of products by 10 with a primary key of 123, you can use the following code:

.. code-block:: python

    Product[123].quantity += 10

For changing several attributes of the same object, you can do so separately:

.. code-block:: python

    order = Order[123]
    order.state = "Shipped"
    order.date_shipped = datetime.now()

or in a single line, using the :py:meth:`~Entity.set` method of an entity instance:

.. code-block:: python

    order = Order[123]
    order.set(state="Shipped", date_shipped=datetime.now())

The :py:meth:`~Entity.set` method can be convenient when you need to update several object attributes at once from a dictionary:

.. code-block:: python

    order.set(**dict_with_new_values)

If you need to save the updates to the database before the current database session is finished, you can use the :py:func:`flush` or :py:func:`commit` functions.

Pony always saves the changes accumulated in the :py:func:`db_session` cache automatically before executing the following methods: ``select()``, ``get()``, ``exists()``, ``execute()`` and ``commit()``.

In future, Pony is going to support bulk update. It will allow updating multiple objects on the disk without loading them to the cache:

.. code-block:: python

    update(p.set(price=price * 1.1) for p in Product
                                    if p.category.name == "T-Shirt")



Deleting an object
------------------

When you call the :py:meth:`~Entity.delete` method of an entity instance, Pony marks the object as deleted. The object will be removed from the database during the following commit.

For example, this is how we can delete an order with the primary key equal to 123:

.. code-block:: python

    Order[123].delete()


Bulk delete
~~~~~~~~~~~

Pony supports bulk delete for objects using the :py:func:`delete` function. This way you can delete multiple objects without loading them to the cache:

.. code-block:: python

    delete(p for p in Product if p.category.name == 'SD Card')
    #or
    Product.select(lambda p: p.category.name == 'SD Card').delete(bulk=True)

.. note:: Be careful with the bulk delete:

    * :py:meth:`before_delete` and :py:meth:`after_delete` hooks will not be called on deleted objects.
    * If an object was loaded into memory, it will not be removed from the :py:func:`db_session` cache on bulk delete.


.. _cascade_delete:

Cascade delete
~~~~~~~~~~~~~~

When Pony deletes an instance of an entity it also needs to delete its relationships with other objects. The relationships between two objects are defined by two relationship attributes. If another side of the relationship is declared as a ``Set``, then we just need to remove the object from that collection. If another side is declared as ``Optional``, then we need to set it to ``None``. If another side is declared as ``Required``,  we cannot just assign ``None`` to that relationship attribute. In this case, Pony will try to do a cascade delete of the related object.

This default behavior can be changed using the :option:`cascade_delete` option of an attribute. By default this option is set to ``True`` when another side of the relationship is declared as ``Required`` and ``False`` for all other relationship types.

If the relationship is defined as ``Required`` at the other end and ``cascade_delete=False`` then Pony raises the ``ConstraintError`` exception on deletion attempt.

Let's consider a couple of examples.

The example below raises the ``ConstraintError`` exception on an attempt to delete a group which has related students:

.. code-block:: python

    class Group(db.Entity):
        major = Required(str)
        items = Set("Student", cascade_delete=False)

    class Student(db.Entity):
        name = Required(str)
        group = Required(Group)


In the following example, if a ``Person`` object has a related ``Parssport`` object, then if you'll try to delete the ``Person`` object, the ``Passport`` object will be deleted as well due to cascade delete:

.. code-block:: python

    class Person(db.Entity):
        name = Required(str)
        passport = Optional("Passport", cascade_delete=True)

    class Passport(db.Entity):
        number = Required(str)
        person = Required("Person")



Saving objects in the database
------------------------------

Normally you don't need to bother of saving your entity instances in the database manually - Pony automatically commits all changes to the database on leaving the :py:func:`db_session` context. It is very convenient. In the same time, in some cases you might want to :py:func:`flush` or :py:func:`commit` data in the database before leaving the current database session.

If you need to get the primary key value of a newly created object, you can do :py:func:`commit` manually within the :py:func:`db_session` in order to get this value:


.. code-block:: python

    class Customer(db.Entity):
        id = PrimaryKey(int, auto=True)
        email = Required(str)

    @db_session
    def handler(email):
        c = Customer(email=email)
        # c.id is equal to None
        # because it is not assigned by the database yet
        commit()
        # the new object is persisted in the database
        # c.id has the value now
        print(c.id)


Order of saving objects
~~~~~~~~~~~~~~~~~~~~~~~

Usually Pony saves objects in the database in the same order as they are created or modified. In some cases Pony can reorder SQL INSERT statements if this is required for saving objects. Let's consider the following example:

.. code-block:: python

    from pony.orm import *

    db = Database()

    class TeamMember(db.Entity):
        name = Required(str)
        team = Optional('Team')

    class Team(db.Entity):
        name = Required(str)
        team_members = Set(TeamMember)

    db.bind('sqlite', ':memory:')
    db.generate_mapping(create_tables=True)
    sql_debug(True)

    with db_session:
        john = TeamMember(name='John')
        mary = TeamMember(name='Mary')
        team = Team(name='Tenacity', team_members=[john, mary])

In the example above we create two team members and then a team object, assigning the team members to the team. The relationship between TeamMember and Team objects is represented by a column in the TeamMember table:

.. code-block:: sql

    CREATE TABLE "Team" (
      "id" INTEGER PRIMARY KEY AUTOINCREMENT,
      "name" TEXT NOT NULL
    )

    CREATE TABLE "TeamMember" (
      "id" INTEGER PRIMARY KEY AUTOINCREMENT,
      "name" TEXT NOT NULL,
      "team" INTEGER REFERENCES "Team" ("id")
    )

When Pony creates ``john``, ``mary`` and ``team`` objects, it understands that it should reorder SQL INSERT statements and create an instance of the ``Team`` object in the database first, because it will allow using the team id for saving TeamMember rows:

.. code-block:: sql

    INSERT INTO "Team" ("name") VALUES (?)
    [u'Tenacity']

    INSERT INTO "TeamMember" ("name", "team") VALUES (?, ?)
    [u'John', 1]

    INSERT INTO "TeamMember" ("name", "team") VALUES (?, ?)
    [u'Mary', 1]


Cyclic chains during saving objects
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now let's say we want to have an ability to assign a captain to a team. For this purpose we need to add a couple of attributes to our entities: ``Team.captain`` and reverse attribute ``TeamMember.captain_of``

.. code-block:: python

    class TeamMember(db.Entity):
        name = Required(str)
        team = Optional('Team')
        captain_of = Optional('Team')

    class Team(db.Entity):
        name = Required(str)
        team_members = Set(TeamMember)
        captain = Optional(TeamMember, reverse='captain_of')

And here is the code for creating entity instances with a captain assigned to the team:

.. code-block:: python

    with db_session:
        john = TeamMember(name='John')
        mary = TeamMember(name='Mary')
        team = Team(name='Tenacity', team_members=[john, mary], captain=mary)

When Pony tries to execute the code above it raises the following exception:

.. code-block:: text

    pony.orm.core.CommitException: Cannot save cyclic chain: TeamMember -> Team -> TeamMember

Why did it happen? Let's see. Pony sees that for saving the ``john`` and ``mary`` objects in the database it needs to know the id of the team, and tries to reorder the insert statements. But for saving the ``team`` object with the ``captain`` attribute assigned, it needs to know the id of ``mary`` object. In this case Pony cannot resolve this cyclic chain and raises an exception.

In order to save such a cyclic chain, you have to help Pony by adding the :py:func:`flush` command:

.. code-block:: python

    with db_session:
        john = TeamMember(name='John')
        mary = TeamMember(name='Mary')
        flush() # saves objects created by this moment in the database
        team = Team(name='Tenacity', team_members=[john, mary], captain=mary)

In this case, Pony will save the ``john`` and ``mary`` objects in the database first and then will issue SQL UPDATE statement for building the relationship with the ``team`` object:

.. code-block:: sql

    INSERT INTO "TeamMember" ("name") VALUES (?)
    [u'John']

    INSERT INTO "TeamMember" ("name") VALUES (?)
    [u'Mary']

    INSERT INTO "Team" ("name", "captain") VALUES (?, ?)
    [u'Tenacity', 2]

    UPDATE "TeamMember"
    SET "team" = ?
    WHERE "id" = ?
    [1, 2]

    UPDATE "TeamMember"
    SET "team" = ?
    WHERE "id" = ?
    [1, 1]



Entity methods
--------------

See the :ref:`Entity methods <entity_methods>` part of the API Reference for details.



Entity hooks
------------

See the :ref:`Entity hooks <entity_methods>` part of the API Reference for details.



Serializing entity instances using pickle
-----------------------------------------

Pony allows pickling entity instances, query results and collections. You might want to use it if you want to cache entity instances in an external cache (e.g. memcache). When Pony pickles entity instances, it saves all attributes except collections in order to avoid pickling a large set of data. If you need to pickle a collection attribute, you must pickle it separately. Example:

.. code-block:: python

    >>> from pony.orm.examples.estore import *
    >>> products = select(p for p in Product if p.price > 100)[:]
    >>> products
    [Product[1], Product[2], Product[6]]
    >>> import cPickle
    >>> pickled_data = cPickle.dumps(products)

Now we can put the pickled data to a cache. Later, when we need our instances again, we can unpickle it:

.. code-block:: python

    >>> products = cPickle.loads(pickled_data)
    >>> products
    [Product[1], Product[2], Product[6]]

You can use pickling for storing objects in an external cache for improving application performance. When you unpickle objects, Pony adds them to the current :py:func:`db_session` as if they were just loaded from the database. Pony doesn't check if objects hold the same state in the database.

