Aggregation
===========

You can use the following five aggregate functions for declarative queries:  :py:func:`sum`, :py:func:`count`, :py:func:`min`, :py:func:`max`, and :py:func:`avg`. Let's see some examples of simple queries using these functions.

Total GPA of students from group 101:

.. code-block:: python

	sum(s.gpa for s in Student if s.group.number == 101)

Number of students with a GPA above three:

.. code-block:: python

	count(s for s in Student if s.gpa > 3)

First name of a student, who studies philosophy, sorted alphabetically:

.. code-block:: python

	min(s.name for s in Student if "Philosophy" in s.courses.name)

Birth date of the youngest student in group 101:

.. code-block:: python

	max(s.dob for s in Student if s.group.number == 101)

Average GPA in department 44:

.. code-block:: python

	avg(s.gpa for s in Student if s.group.dept.number == 44)


.. note:: Although Python already has the standard functions ``sum()``, ``count()``, ``min()``, and ``max()``, Pony adds its own functions under the same names. Also, Pony adds its own :py:func:`avg` function. These functions are implemented in the ``pony.orm`` module and they can be imported from there either "by the star", or by its name.

    The functions implemented in Pony expand the behavior of standard functions in Python; thus, if in a program these functions are used in their standard way, the import will not affect their behavior. But it also allows specifying a declarative query inside the function.

    If one forgets to import these functions from the ``pony.orm`` package, then an error will appear upon use of the Python standard functions ``sum()``, ``count()``, ``min()``, and ``max()`` with a declarative query as a parameter:
  
    .. code-block:: text

        TypeError: Use a declarative query in order to iterate over entity

Aggregate functions can also be used inside a query. For example, if you need to find not only the birth date of the youngest student in the group, but also the student himself, you can write the following query:

.. code-block:: python

    select(s for s in Student if s.group.number == 101
               and s.dob == max(s.dob for s in Student
                                            if s.group.number == 101))

Or, for example, to get all groups with an average GPA above 4.5:

.. code-block:: python

    select(g for g in Group if avg(s.gpa for s in g.students) > 4.5)

This query can be shorter if we use Pony :ref:`attribute lifting <attribute_lifting>` feature:

.. code-block:: python

    select(g for g in Group if avg(g.students.gpa) > 4.5)


Query object aggregate functions
--------------------------------

You can call the aggregate methods of the :py:class:`Query` object:

.. code-block:: python

    select(sum(s.gpa) for s in Student)

Is equal to the following query:

.. code-block:: python

    select(s.gpa for s in Student).sum()

Here is the list of the aggregate functions:

* :py:meth:`Query.avg`
* :py:meth:`Query.count`
* :py:meth:`Query.min`
* :py:meth:`Query.max`
* :py:meth:`Query.sum`


Several aggregate functions in one query
----------------------------------------

SQL allows you including several aggregate functions in the same query. For example, we might want to receive both the lowest and the highest GPA for each group. In SQL, such a query would look like this:

.. code-block:: sql

    SELECT s.group_number, MIN(s.gpa), MAX(s.gpa)
    FROM Student s
    GROUP BY s.group_number

This query will return the lowest and the highest GPA for each group. With Pony you can use the same approach:

.. code-block:: python

    select((s.group, min(s.gpa), max(s.gpa)) for s in Student)


Function ``count``
------------------

Aggregate queries often need to calculate the quantity of something. Here is how we get the number of students in Group 101:

.. code-block:: python

    count(s for s in Student if s.group.number == 101)

The number of students in each group related to the department 44:

.. code-block:: python

    select((g, count(g.students)) for g in Group if g.dept.number == 44)

or this way:

.. code-block:: python

    select((s.group, count(s)) for s in Student if s.group.dept.number == 44)

In the first example the aggregate function :py:func:`count` receives a collection, and Pony will translate it into a subquery. (Actually, this subquery will be optimized by Pony and will be replaced with ``LEFT JOIN``).

In the second example, the function :py:func:`count` receives a single object instead of a collection. In this case Pony will add a ``GROUP BY`` section to the SQL query and the grouping will be done on the ``s.group`` attribute.

If you use the :py:func:`count` function without arguments, this will be translated to SQL ``COUNT(*)``. If you specify an argument, it will be translated to ``COUNT(DISTINCT column)``.


Conditional ``count``
---------------------

There is another way of using the :py:func:`count` function. Let's assume that we want to get three numbers for each group:

* The number of students that have a GPA less than 3
* The number of students with GPA between 3 to 4
* The number of students with GPA higher than 4

The query can be constructed this way:

.. code-block:: python

    select((g, count(s for s in g.students if s.gpa <= 3),
               count(s for s in g.students if s.gpa > 3 and s.gpa <= 4),
               count(s for s in g.students if s.gpa > 4)) for g in Group)

Although this query will work, it is pretty long and not very effecive - each ``count`` will be translated into a separate subquery. For such situations, Pony provides a "conditional COUNT" syntax:

.. code-block:: python

    select((s.group, count(s.gpa <= 3),
                     count(s.gpa > 3 and s.gpa <= 4),
                     count(s.gpa > 4)) for s in Student)

This way, we put our condition into the :py:func:`count` function. This query will not have subqueries, which makes it more effective.

.. note:: The queries above are not entirely equivalent: if a group doesn't have any students, then the first query will select that group having zeros as the result of :py:func:`count`, while the second query simply will not select the group at all. This happens because the second query selects the rows from the table Student, and if the group doesn't have any students, then the table Student will not have any rows for this group.

    If you want to get rows with zeros, then an effective SQL query should use the :py:func:`left_join` function:

    .. code-block:: python

        left_join((g, count(s.gpa <= 3),
               count(s.gpa > 3 and s.gpa <= 4), 
               count(s.gpa > 4)) for g in Group for s in g.students)



More sophisticated aggregate queries
------------------------------------

Using Pony you can do even more complex grouping. For example, you can group by an attribute part:

.. code-block:: python

    select((s.dob.year, avg(s.gpa)) for s in Student)

The birth year in this case is not a distinct attribute – it is a part of the ``dob`` attribute.

You can have expressions inside the aggregate functions:

.. code-block:: python

    select((item.order, sum(item.price * item.quantity))
            for item in OrderItem if item.order.id == 123)

Here is another way of making the same query:

.. code-block:: python

    select((order, sum(order.items.price * order.items.quantity))
            for order in Order if order.id == 123)

In the second case, we use the :ref:`attribute lifting <attribute_lifting>` concept. The expression ``order.items.price`` creates an array of prices, while ``order.items.quantity`` generates an array of quantities. As the result, in this example, we'll have the sum of quantity multiplied by the price for each order item.


Queries with HAVING
-------------------

The ``SELECT`` statement has two different sections which are used for conditions: ``WHERE`` and ``HAVING``. The ``WHERE`` section is used more often and contains conditions which will be applied to each row. If a query contains aggregate functions, such as ``MAX`` or ``SUM``, the ``SELECT`` statement may also contain ``GROUP BY`` and ``HAVING`` sections. The conditions of the ``HAVING`` section are applied after grouping the SQL query results. Typically the conditions of the ``HAVING`` section always contain aggregate functions, while conditions in the ``WHERE`` section may only contain aggregate functions inside a subquery.

When you write a query which contains aggregate functions, Pony needs to determine if the resulting SQL will contain the ``GROUP BY`` and ``HAVING`` sections and where it should put each condition from the Python query. If a condition contains an aggregate function, Pony places the condition into the ``HAVING`` section. Otherwise it places the condition into the ``WHERE`` section.

Consider the following query, which returns the tuples (``Group``, count_of_students):

.. code-block:: python

    select((s.group, count(s)) for s in Student
           if s.group.dept.number == 44 and avg(s.gpa) > 4)

In this query we have two conditions. The first condition is ``s.group.dept.number == 44``. Since it doesn`t include an aggregate function, Pony will place this condition into the ``WHERE`` section. The second condition ``avg(s.gpa) > 4`` contains the aggregate function ``avg`` and will be placed into the ``HAVING`` section.

Another question is what columns Pony should add to the ``GROUP BY`` section. According to the SQL standard, any non-aggregated column which placed into the ``SELECT`` statement should be added to the ``GROUP BY`` section too. Let's consider the following query:

.. code-block:: sql

    SELECT A, B, C, SUM(D), MAX(E), COUNT(F)
    FROM T1
    WHERE ...
    GROUP BY ...
    HAVING ...

According to the SQL standard, we need to include the columns ``A``, ``B`` and ``C`` into the ``GROUP BY`` section, because these columns appear in the ``SELECT`` list and don't wrapped with any aggregate function. Pony does exactly this. If your aggregated Pony query returns a tuple with several expressions, any non-aggregated expression will be placed into the ``GROUP BY`` section. Let's consider the same Pony query again:

.. code-block:: python

    select((s.group, count(s)) for s in Student
           if s.group.dept.number == 44 and avg(s.gpa) > 4)

This query returns the tuples (``Group``, count_of_students). The first element of the tuple, the ``Group`` instance, is not aggregated, so it will be placed into the ``GROUP BY`` section:

.. code-block:: sql

    SELECT "s"."group", COUNT(DISTINCT "s"."id")
    FROM "Student" "s", "Group" "group-1"
    WHERE "group-1"."dept" = 44
      AND "s"."group" = "group-1"."number"
    GROUP BY "s"."group"
    HAVING AVG("s"."gpa") > 4

The ``s.group`` expression was placed into the ``GROUP BY`` section, and the condition ``avg(s.gpa) > 4`` was placed into the ``HAVING`` section of the query.

Sometimes the condition which should be placed into the ``HAVING`` section contains some non-aggregated columns. Such columns will be added to the ``GROUP BY`` section, because according to the SQL standard it is forbidden to use a non-aggregated column inside the ``HAVING`` section, if it was not added to the ``GROUP BY`` list.

Another example:

.. code-block:: python

    select((item.order, item.order.total_price,
         sum(item.price * item.quantity))
         for item in OrderItem
         if item.order.total_price < sum(item.price * item.quantity))

This query has the following condition: ``item.order.total_price < sum(item.price * item.quantity)``, which contains an aggregate function and should be added to the ``HAVING`` section. But the part ``item.order.total_price`` is not aggregated. Hence, it will be added to the ``GROUP BY`` section in order to satisfy the SQL requirements.



Aggregate functions in order by section
---------------------------------------

The aggregate functions can be used inside the :py:meth:`Query.order_by` function. Here is an example:


.. code-block:: python

    select((s.group, avg(s.gpa)) for s in Student) \
            .order_by(lambda s: desc(avg(s.gpa)))


Another way of ordering by an aggregated value is specifying the position number inside the :py:meth:`Query.order_by` method:

.. code-block:: python

    select((s.group, avg(s.gpa)) for s in Student).order_by(-2)
