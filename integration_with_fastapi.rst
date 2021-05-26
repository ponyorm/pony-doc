Integration with FastAPI
========================

PonyORM can be used with FastAPI, it integrates with Pydantic and can be used in an async environment, as long as you follow a few rules.

Setup
-----

For your first steps, simply follow :doc:`Getting Started with Pony <firststeps>` to create the database object and define your models.
You may want to make use of Pydantic's inbuilt `BaseSettings Model`_ to manage your connection secrets. The specific parameters can be found in :doc:`database`.

.. code-block:: python

    import fastapi
    from src.config import Settings
    from src.models import db

    settings = Settings()
    api = fastapi.FastAPI()
    db.bind(**settings.get_connection)
    db.generate_mapping(create_tables=True)

.. todo:: Should the code-block include the Pydantic settings model or is this too far from the Pony Doc scope?
.. _BaseSettings Model: https://pydantic-docs.helpmanual.io/usage/settings/

Models and Schemas
------------------

To make Pony's models work with Pydantic's schemas...

.. todo:: Point to relevant FastAPI doc: https://fastapi.tiangolo.com/tutorial/extra-models/
.. todo:: Maybe answer this StackOverflow Q: https://stackoverflow.com/questions/64521997/list-with-pony-orm-and-fastapi
.. todo:: Maybe answer this StackOverflow Q: https://stackoverflow.com/questions/65134921/most-efficient-way-to-serialize-ponyorm-query-or-a-list-of-objects
.. todo:: Related Github Issue: https://github.com/samuelcolvin/pydantic/issues/2020
.. todo:: Possible Solution: Work with Pydantic.GetterDict: https://fastapi.tiangolo.com/advanced/sql-databases-peewee/#create-a-peeweegetterdict-for-the-pydantic-models-schemas


Async and db_session
--------------------

Pony was not developed for async usage, and it may be tricky to use it correctly in an async environment.
However, if only part of your application is async, you will be fine if you stick to these two rules:

You cannot use the `@db_session` decorator on async functions, use a context manager for wrapping lines of code that really work with the database.
Secondly, do not call async functions inside a `db_session`, only normal function calls are possible. Use little shortliving sessions and don't interrupt them with async:

.. code-block:: python

    async def func():
        with db_session:
            thing = database.MyEntity()
        await async_func(ABC, thing)

