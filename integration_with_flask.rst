Integration with flask
======================

Since Pony 0.7.4 we added support for comfortable using PonyORM with Flask. 
With `pony.flask.Pony` you can wrap your flask's application's `request` with `db_session` automatically right way.

.. code-block:: python

    from flask import Flask
    from pony.flask import Pony
    
    app = Flask(__name__)
    Pony(app) 
    
With this code each of `view` function you will define will be wrapped with `db_session` so you should not care about them.

Flask-Login
-----------

Also you can easily use Flask-Login extension.

Example

.. code-block:: python

    from flask import Flask, render_template
    from flask_login import LoginManager, UserMixin, login_required
    from pony.flask import Pony
    from pony.orm import Database, Required, Optional
    from datetime import datetime

    app = Flask(__name__)
    app.config.update(dict(
        DEBUG = False,
        SECRET_KEY = 'secret_xxx',
        PONY = {
            'provider': 'sqlite',
            'filename': 'db.db3',
            'create_db': True
        }
    ))
    
    db = Database()
    
    class User(db.Entity, UserMixin):
        login = Required(str, unique=True)
        password = Required(str)
        last_login = Optional(datetime)
        
    db.bind(**app.config['PONY'])
    db.generate_mapping(create_tables=True)

    Pony(app)
    login_manager = LoginManager(app)
    login_manager.login_view = 'login'

    @login_manager.user_loader
    def load_user(user_id):
        return db.User.get(id=user_id)
        
You can use `LoginManager.current_user` as `User` instance.

.. code-block:: python

    @app.route('/friends')
    @login_required
    def friends():
        return render_template('friends.html', friends=current_user.friends)
        
You can run another example_ to check it

.. _example: https://github.com/ponyorm/pony/tree/orm/pony/flask/example

.. code-block:: sh

    python -m pony.flask.example
