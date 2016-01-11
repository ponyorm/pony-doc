PonyJS security and permissions
===============================

When we send information from the backend, there might be some objects or object attributes that we don’t want to show at the frontend. Sometimes we don’t want to allow editing of some objects or attributes, or we want to give this permission only to a specific group of users.
In other words, there needs to be a method to protect data from unauthorized view and edit. For this purpose you should use PonyJS security and permissions.

The PonyJS Security and Permissions workflow is divided into two major parts:

 1. Setting up permission rules
 2. Applying the permission rules onto the objects which are sent or received from the frontend


Setting up permission rules
---------------------------

For setting up a permission rule, you should specify a list of actions followed by conditions on when those actions are permitted.

Usually, it is convenient to create a separate file ``permissions.py`` where you put all your permission rules and then place it along with the file ``models.py`` where you keep your entity declarations. In your program you import the file with the permissions after the file with entity declarations and this way apply your permission rules. By default nothing is permitted, so if you forget to import the file with permission rules, Pony will raise the ``PermissionError`` exception once you'll try to get JSON payload using the ``to_json()`` method.

Here is the permission rules for our Forum:

.. code-block:: python

    # anybody can view User, Section, Tag objects
    # except User.password, User.email attributes
    with db.permissions_for(User, Section, Tag):
        allow('view', group='anybody').exclude(User.password, User.email)

    # anybody can view public (not deleted) Message object
    with db.permissions_for(Message):
        allow('view', group='anybody', label='public')

    # admin can manage sections
    with db.permissions_for(Section):
        allow('edit create delete', group='admin')

    # author can manage their messages
    with db.permissions_for(Message):
        allow('edit create delete', role='author')

    # voter can view their votes
    with db.permissions_for(Vote):
        allow('view', role='voter')

    with db.permissions_for(Message):
        # admin can delete any message
        allow('delete', group='admin')
        # moderator can delete messages in their sections
        allow('delete', role='moderator')


Now let's get into the details.

Permissions
~~~~~~~~~~~

There are four standard permissions: ``view``, ``edit``, ``create``, ``delete``.

- ``view`` - allows viewing an object at the frontend
- ``edit`` - allows editing an object at the frontend. This permission implies the ``view`` permission - if you allow editing, it means that an object can be viewed
- ``create`` - allows creating an object at the frontend
- ``delete`` - allows deleting an object at the frontend


Groups, Roles and Labels
~~~~~~~~~~~~~~~~~~~~~~~~

Pony uses the concept of groups, roles and labels for setting permissions.

| **Group**
| A user can belong to one or more groups. Here are the group examples: ``admin``, ``user``. There is a default group – ``anybody``. Every user is a member of this group. If no group is assigned to a user, this user still belongs to the group ``anybody``.

The set of user groups is assigned in the beginning of the ``db_session`` and cannot be changed till the end of current ``db_session``.

A group is a characteristic of a user.

| **Role**
| The user's role can vary depending on the particular object. In databases it is called Row-level security. This means that you can grant permissions to specific rows in a database table, not the whole table only.

For example, using the concept of roles, we can allow a user updating the ``Post`` objects only if this user is the author of these posts. Examples of roles: ``author``, ``moderator``, ``voter``.

A role is a characteristic of a user in relation to an object.

| **Label**
| A label can be assigned to an object. Labels, as well as roles, can be used as an equivalent of Row-level security. Example of label: ``public``.

A label is a characteristic of an object.


Assigning rules to entities
~~~~~~~~~~~~~~~~~~~~~~~~~~~

By default all permissions on objects are denied and in order to permit, you have to create ``allow`` rules. You can create more than one rule for any entity. All rules for each entity will be summarized. The order of the rules doesn't matter.

The ``allow`` rule can be assigned to entities with the help of the ``permissions_for`` context manager. It is a method of the ``Database`` object:

.. code-block:: python

   with db.permissions_for(User, Section, Tag):
        allow('view', group='anybody').exclude(User.password, User.email)

If you need to apply the allow rule only to some entity attributes, not all of them, you can do so with the help of the ``exclude`` method. In parenthesis you specify the list of attributes which should be excluded from the rule.

In the example above we allow to view ``User`` ,``Section``, ``Tag`` objects at the frontend to anybody. Except the ``User.password`` and ``User.email`` attributes.

Another example:

.. code-block:: python

    with db.permissions_for(Message):
        # admin can delete any message
        allow('delete', group='admin')
        # moderator can delete messages in their sections
        allow('delete', role='moderator')

In this example we allow the owner of the account to edit all attributes of the ``User`` object. Users with the ``employee`` role can view all attributes, and everyone else can view all attributes except of the ``User.email`` attribute.

If you use inheritance for your entities, you need to declare permissions for the base entity only.


Hiding entity attributes by default
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Sometimes we want to be sure that some attributes will never be passed to the frontend, even if the permission rules mistakenly allow it. In this case we can use the ``hidden`` option of an attribute:

.. code-block:: python

    class User(db.Entity):
        username = Required(str)
        password = Required(str, hidden=True)

This option has a preference over the permission rules and, when set to ``True`` guarantees that this attribute will never be passed to the frontend.



Applying the permission rules onto the objects which are being sent or received from the frontend
-------------------------------------------------------------------------------------------------

Pony checks the permissions during preparing objects for sending to the frontend (``to_json()`` method) and during saving object updates received from the frontend (``from_json()`` method). Pony goes through the list of the objects and checks permissions for each object.

For making a decision if an object can be passed through, Pony needs to get the following information:

- what groups the current user belongs to
- what roles the current user has against each object
- what labels assigned to each object

In order to provide Pony with this information, you need to write functions and decorate them with special decorators::

    @groups_getter(user_class)

    @roles_getter(user_class, obj_class)

    @labels_getter(obj_class)

Pony will call these functions whenever it needs to get groups, roles and labels associated with the current user and objects that are going to be sent or received from the backend. Pony applies the information received from these functions to the permission rules and makes a decision if it can allow the requested action(view, edit, create or delete) on a particular object.

But first of all, before Pony calls these functions, we need to set current user for the HTTP request.


Setting current user
~~~~~~~~~~~~~~~~~~~~

The current user is usually set in the beginning of processing a request from a user. The username is usually stored in cookies. Using this username we can extract the User object from the database and set it as the current user::

    set_current_user(user)

At the end of the request processing you need to reset the current user. This can be done by passing ``None`` as the user object::

    set_current_user(None)


Getting current user at the frontend
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``to_json()`` method adds the JSON representation of the current user to the resulting JSON payload. At the frontend this object is stored under the ``pony.currentUser`` variable.


Using decorators for getting groups, roles and labels
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When Pony sends objects to the frontend using the method ``to_json()`` it needs to check if the current user has a permission to ``view`` each of those object. For this purpose Pony calls the function decorated with the ``@groups_getter`` decorator. This function should return a single group, or a list of groups which this user belongs to. Pony caches the result and keeps it till the end of the db_session.

Then Pony iterates over the objects go be sent to the frontend and calls functions decorated with the ``@roles_getter`` and ``@labels_getter`` decorators. Then Pony applies the result, received from these functions to the permission rules declared earlier. If Pony encounters an object that doesn't have a corresponding rule which allows to ``view`` this object, it raises the ``PermissionError`` exception.

When Pony receives objects from the frontend using the method ``from_json()``, it performs the same steps. The only difference is that instead of checking the ``view`` permission, it checks the permission for ``edit``, ``create`` or ``delete``, depending on what was done with an object at the frontend.

In the parenthesis following the decorator name, you need to write a class, which will work as a filter for this function. When Pony checks permissions for an object, it calls only those functions which have a class of this object (or its ancestor) specified in the decorator. In the same time, the function will receive the object itself, not the class.

``@groups_getter(user_class)`` - decorates functions used for getting user groups

``@roles_getter(user_class, obj_class)`` - decorates functions used for getting user roles

``@labels_getter(obj_class)`` - decorates functions used for getting object labels

You can declare more then one function decorated with the same decorator and having the same filter. Pony will call all functions which correspond to a particular object. All returned results will be combined together. The function declared with one of these decorators can return:

- A string, meaning that this group, role or label will be added to the resulting list of groups, roles and labels
- A list of strings, meaning that each string from this list will be added to the resulting list
- ``None``, meaning that this function adds nothing

You can call these getters functions recursively.



default_filter
~~~~~~~~~~~~~~

Sometimes it is convenient to have a filter which would be applied to all queries automatically. In our Forum data model the ``Message`` entity has the ``deleted_at`` attribute. The idea here is that when the message needs to be deleted, we don't delete it, but mark it deleted by assigning the current datetime to the ``deleted_at`` attribute. When we write a query which extracts ``Message`` objects from the database, we should remember about this and always add the ``and m.deleted_at is None`` condition, in order not to show deleted messages. Another option would be to define a ``default_filter``:

.. code-block:: python

    @default_filter(Message)
    def not_deleted(obj):
        return obj.deleted_at is None

Here we declare a function and decorate it with the ``@default_filter`` decorator. In the parenthesis we specify the entity where we apply the default filter. Once it is declared, all queries that select instances of the specified entity will have the default filter applied.

.. note:: If your query returns something other than the entity instances, say an attribute, or a tuple, this filter will not be applied.

The function that is decorated with the ``@default_filter`` receives an instance of the entity. The body of this function consists of the ``return`` keyword and the condition.

If you need to apply the default filter only to specific user groups, you can use the ``except_groups`` option:

.. code-block:: python

    @default_filter(Message, except_groups=['admin', 'moderator'])
    def not_deleted(obj):
        return obj.deleted_at is None

In the example above, the default filter wouldn't be applied if the user belongs to the ``admin`` or ``moderator`` groups.

Also, you can disable the default filter at any moment by using the ``default_filters_disabled`` context manager or decorator:

.. code-block:: python

    with default_filter_disabled():
        posts = Post.select(lambda p: p.author == User[1])

The query example above will return all posts of the ``User[1]``.


