PonyJS Reference
================

Structure of JSON packet
------------------------

The current user is an object which was assigned at the backend using the ``set_current_user()`` method.

``pony.entity`` - keeps the entities, can be used for creating new entity instances or for getting objects by the primary key
- ``pony.currentUser`` - an object which represents currently logged user

If the frontend receives JSON structure with different ``schenaHash``, 'PonyJS warning: database schema has been changed' will be logged to the browser console. Later we are going to provide an ability to subscribe to this event.


pony.js frontend library reference
----------------------------------

.. js:function:: pony.load()

By default, ``pony.load()`` uses the HTTP GET method.

The interface of this function is absolutely similar to the |jquery_ajax| function, because Pony uses it for making Ajax requests.

.. |jquery_ajax| raw:: html

    <a href="http://api.jquery.com/jquery.ajax/" target="_blank">JQuery.ajax()</a>



.. js:function:: pony.save()

convert it to a string using the ``JSON.stringify`` method and send to the url ``/update`` by HTTP ``POST`` method.

.. js:function:: pony.getCurrentUser()

   Returns the object which was set as the current user at the backend using the :py:func:`set_current_user` function.


.. js:data:: pony.entities

   The object which holds the database schema passed from the backend. The keys are the entity names, the values are the ``Entity`` objects. The ``Entity`` object keeps the information about entity attributes, primary and unique keys and entity inheritance. This object has the following methods: :js:func:`Entity.prototype.toString`, :js:func:`Entity.prototype.getByPk`, :js:func:`Entity.prototype.create`


.. js:data:: pony.objects

   The object represents the Identity Map which keeps all Pony entity instances received from the backend or created at the frontend. The key is an unique client id number assigned at the frontend. It has nothing to do with the primary key assigned to an entity instance at the backend.


.. js:data:: pony.schemaHash

   The md5 hash string of the database schema. Once the database schema is received at the frontend, the md5 hash string will be passed to the backend with every :js:func:`load` request. If the database schema is not changed at the backend, the schema JSON won't be passed to the frontend once again. It allows saving the bandwidth.


.. js:data:: pony.cacheModified

   A KnockoutJS observable which holds a boolean value. Initially the value is `false`. It changes to `true` once any entity instance attribute is changed, or any entity instance is created or deleted.


.. js:function:: pony.unmarshalData(json)

   Unpacks the JSON data prepared by the :py:func:`to_json` function.


.. js:function:: pony.getChanges([data])


.. js:function:: Attribute.prototype.add(obj)

   Adds an object to a relationship attribute. Example:

   .. code-block:: js

      var t = pony.entities.Topic.getByPk(1)
      var p = pony.entities.Post.create({author: pony.getCurrentUser(), text: 'Some text'})
      t.posts.add(p)


.. js:function:: Attribute.prototype.create([vals])

   Creates a new entity instance and establishes a relationship with the current object. If the ``vals`` object is passed, the attributes values will be assigned accordingly.

   .. code-block:: js

      var t = pony.entities.Topic.getByPk(1)
      t.posts.create({author: pony.getCurrentUser(), text: 'Some text'})

   In the example above the `Topic.posts` attribute is related to the `Post` entity. The `create` function creates an new `Post` instance and establishes relationship between newly created instance and the `Topic[1]` object.


.. js:function:: Attribute.prototype.remove(obj)

   Removes an item from the collection and thus breaks the relationship between entity instances.


.. js:function:: Entity.prototype.create([vals])

   Creates an entity instance and assigns attribute values from the ``vals`` object. The attribute values also can be assigned after an object is created.


.. js:function:: Entity.prototype.getByPk(pk)

   Returns an entity instance by its primary key. If object with this primary key was not passed from the backend, throws an exception.


.. js:function:: Instance.prototype.destroy()

   Deletes an entity instance.


.. js:function:: Instance.prototype.toString()

   Returns a string representation of an entity instance. For example:

   .. code-block:: js

      > String(pony.entities.Topic.getByPk(1))
      < "Topic[1]"

   It consists of the entity name and the primary key value in the square brackets. If the primary key is composite, the values will be separated by a comma.



PonyJS backend functions reference
----------------------------------

.. class:: Database
   :noindex:

   .. py:method:: to_json(data, include=(), exclude=(), converter=None, with_schema=True, schema_hash=None)

      Serializes ``data`` to a JSON packet, which then can be sent to the frontend.

      :param data: Data that needs to be serialized. Usually it is a list or a dictionary that contains object.
      :param include: List of entity attributes that needs to be included to the resulting JSON. By default Pony doesn't include lazy attributes and relationships to-many. If an object has a relationship to-one, only a primary key of the related object will be included by default.

The ``include`` parameter receives a list of attributes which should be included into the JSON packet. By default, Pony doesn't include lazy attributes and relationships to-many. For to-one relationships, by default, Pony includes only the primary key of the related object. In our example, we want to display the title of the topic, so we specify it in the include list.

By default, the ``to_json()`` method includes all attributes except lazy attributes and collections to-many. In case the object has to-one relationship, only the primary key of the related object will be included by default. If you need to get lazy attributes, to-many collections or all attributes of the related to-one object, you need to specify this attribute in the ``include`` list, as we do in the following example:



      :param exclude: List of entity attributes to exclude from the resulting JSON.
      :param converter: A function that
      :param with_schema: ``True`` means the database schema will be included into the JSON packet
      :param schema_hash: If schema
      :return: JSON packet of ``str`` type.
      :rtype: str
      :raises TypeError: if ``include`` or ``exclude`` list contains a non-attribute value.
      :raises PermissionError: if ``data`` parameter contains entity instances that are not allowed by the permission rules.
      :raises TransactionError: if ``data`` parameter contains entity instances that were extracted from the database in another transaction, or if an instance belongs to another database.


   .. py:method:: from_json(changes, observer=None)

      Extracts data from the JSON packet received from the frontend and apply changes to the ``Database`` object.

      :param changes: the JSON packet received from the frontend.
      :param observer: a function that will be called on each changed object. The function receives 2, 3 or 4 arguments:

         * action - a string description of an action. The following actions are possible:

            * "create" - an object was created, the function receives ``action``, ``obj`` and ``vals``.
            * "delete" - an object was deleted, the function receives ``action`` and ``obj``.
            * "update" - an object was updated, the function receives ``action``, ``obj``, ``vals`` and ``old_vals``.
            * "add" - a relationship between objects was established, the function receives ``action``, ``obj`` and ``vals``.
            * "remove" - a relationship between objects was dropped, the function receives ``action``, ``obj`` and ``vals``.

         * obj - an object which is changing
         * vals - optional argument, a dictionary with attribute names as keys, and values of those attributes as values. In case of "add" and "remove" actions, the value is a list of objects to be added or removed. Will not be passed on "delete" action.
         * old_vals - optional argument, a dictionary with attribute names as keys, and values of those attributes as values. Will be passed only on "update" action.

         Below is the example of such a function:

         .. code-block:: python

            def log_changes(action, obj, vals=None, old_vals=None):
                print(action, obj, vals, old_vals)

      :return: Data which were sent from the frontend. Besides applying the changes made to entity instances, you can send arbitrary data from the frontend and receive them as the value, returned by this function.
      :raises PermissionError: if actions that need to be performed for applying ``changes`` are not allowed by the permission rules.
      :raises TransactionError: if the same ``changes`` set is trying to be applied more than once.
      :raises OptimisticCheckError: if the ``changes`` JSON has an updated value for attribute that was not read from the database.


   .. py:method:: permissions_for(entities)

      :param entities: An entity or a list of entities

      A context manager which is used for setting permissions for entities. The entities are specified as parameters, the :py:func:`allow` function is used for setting rules. Example:

      .. code-block:: python

         with db.permissions_for(User, Topic, Post):
             allow('edit')

      In this example we allow "view" and "edit" any instance of ``User``, ``Topic`` and ``Post`` entities to anyone at the frontend.

      .. py:function:: allow(operations, group=None, role=None, label=None, site=None)

         Sets a rule within the ``permissions_for`` context manager.

         :param operations: a space separated string with allowed operations. The standard operations are:

               * view
               * edit
               * create
               * delete

            The "edit" operation implies "view", because a user cannot edit something without viewing it. So if you specify just "edit", it is equal to "view edit".

         :param group: a user group
         :param role: a user role to an object
         :param label: an object label
         :param site: the current site. Handy when you want to have different rules for admin and public interfaces

         The rule will be applied when all specified parameters (``group``, ``role``, ``label``, ``site`` will match.

         By default, the operations are applied to all entity attributes. If you don't want to allow viewing or editing a particular attribute, you can exclude it using the ``exclude`` function. Example:

         .. code-block:: python

            with db.permissions_for(Topic, Post):
                allow('edit', role='author').exclude(Topic.author, Post.author)

         In this example a use with the 'author' role can edit a ``Topic`` and ``Post`` objects, but cannot change the ``author`` attribute of these objects.

         If you want to exclude an attribute completely and don't show it at any circumstances, you can use the


.. py:decorator:: groups_getter(user_class)
.. py:decorator:: roles_getter(user_class, obj_class)
.. py:decorator:: labels_getter(obj_class)

   Functions decorated with these decorators are called by Pony on permission check during the execution of :py:meth:`Database.to_json` and :py:meth:`Database.from_json` methods. First, Pony calls the functions decorated with the ``groups_getter`` decorator, passes the current user object (an object which was set by the :py:func:`set_current_user` function) and receives a list of groups assigned to this user. Then it calls the functions decorated with the ``roles_getter`` and ``labels_getter`` decorators for each entity instance.

   The decorated function should return a string or a list of strings.

   Once Pony has a list of groups, roles and labels for current user and an object, it can check it against the permission rules set with the :py:func:`permissions_for` context manager.

.. py:function:: get_user_groups(user)

   Returns a list of groups associated with the ``user`` object.


.. py:function:: get_user_roles(user, obj)

   Returns a list of ``user`` roles to ``obj`` object.


.. py:function:: get_object_labels(obj)

   Returns a list of labels associated with the ``obj`` object.


.. py:function:: get_current_user()

   Returns an object which was set as the current user for current :ref:`db_session <db_session_ref>` using the :py:func:`set_current_user` function.


.. py:function:: set_current_user(user)

   Assigns ``user`` object as the current user for current :ref:`db_session <db_session_ref>`.


.. py:function:: get_current_site()

   Returns a string set as the current site.


.. py:function:: set_current_site(site)

   Sets current site.

.. py:decorator:: default_filter
.. py:decorator:: default_filters_disabled