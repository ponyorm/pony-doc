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

To make full use of FastAPI, Pony's models have to work with Pydantic's schemas.
Unfortunately, this requires somewhat duplicate model definitions. We're following `FastAPI's example`_ here.

Here are the Pony models from :doc:`Getting Started with Pony <firststeps>`:

.. code-block:: python

    class Person(db.Entity):
        name = Required(str)
        age = Required(int)
        cars = Set('Car')

    class Car(db.Entity):
        make = Required(str)
        model = Required(str)
        owner = Required(Person)

We want to expose `Person` in FastAPI like this:

.. code-block:: python

    @api.get('/persons')
    async def read_all_persons():
        with db_session:
           persons = Person.select()
           result = [PersonInDB.from_orm(p) for p in persons]
        return result

    @api.get('/person/{pid}')
    async def read_single_person(pid: int):
        with db_session:
            person = Person[pid]
            result = PersonInDB.from_orm(person)
        return result

For this, we can write the following Pydantic schemas:

.. code-block:: python

    class CarOut(BaseModel):
        make: str
        model: str

    class PersonInDB(BaseModel):
        name: str
        age: int
        cars: List[CarOut]

        @validator('cars', pre=True, allow_reuse=True)
        def pony_set_to_list(cls, values):
            return [v.to_dict() for v in values]

        class Config:
            orm_mode = True

Without preparation, Pony's related objects aren't interpreted correctly. Use a validator to make them accessible for Pydantic.

.. _FastAPI's example: https://fastapi.tiangolo.com/tutorial/sql-databases/#create-the-pydantic-models

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

