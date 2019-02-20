#### Part4 : Database

* 'Flask-SQLAlchemy' : extension that provides a Flask-friendly wrapper to the popular 'SQLAlchemy' package, which is an 'Object Relational Mapper' or **ORM**

* 'Flask-Migrate' : extension that is a Flask wrapper for Alembic, a database migration framework for SQLAlchemy

  ```python
  // config.py
  
  import os
  basedir = os.path.abspath(os.path.dirname(__file__))
  
  class Config(object):
      # ...
      SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or \
          'sqlite:///' + os.path.join(basedir, 'app.db')
      SQLALCHEMY_TRACK_MODIFICATIONS = False
  ```

  ....??

* 

* ```python
  // __init__.py
  
  from flask import Flask
  from config import Config
  from flask_sqlalchemy import SQLAlchemy
  from flask_migrate import Migrate
  
  app = Flask(__name__)
  app.config.from_object(Config)
  db = SQLAlchemy(app)
  migrate = Migrate(app, db)
  
  from app import routes, models
  ```
  * `db` object : represents the database
  * **`models`** module : define the structure of the database



*  Database Models

  ```python
  from app import db
  
  class User(db.Model):
      id = db.Column(db.Integer, primary_key=True)
      username = db.Column(db.String(64), index=True, unique=True)
      email = db.Column(db.String(120), index=True, unique=True)
      password_hash = db.Column(db.String(128))
  
      def __repr__(self):
          return '<User {}>'.format(self.username)
  ```

  * Fields are created as 'instances' of the `db.Column` class
  * `__repr__` method : tells how to print objects of this class



* Creating Migration Repository : `flask db init`
  * 'Alembic' maintains a `migration repository`
    * directory in which it stores its migration scripts
    * Each time a change is made to the database schema, a migration script is added to the repository with the details of the change
  * to apply migrations to database, these migration scripts are executed in the sequence they were created
* First Database Migration
  * 2 ways to create a database migration
    * maually
    * automatically : 'Alembic' compares the database schema as defined by the database models, against the actual database schema currently used in database
  * `flask db migrate` : populates(이주) the migration script with the changes necessary to make the database schema match the application models 
    * `-m` : optional, adds short descriptive text to the migration
  * terminal에서, `upgrade()` 와 `downgrade`
    * `upgrade()` function : applies the migration
    * `downgrade()` function : removes it
    * allows 'Alembic' to migrate the database to any point in the history, even to older versions, by using the downgrade path
  * migrate를 통해 변경사항을 script에 저장, upgrade를 통해 실제 database에 변경사항 적용가능. downgrade를 통해 이전버전으로 database복원 가능.

* Database Relationships

  * relational databases are good at storing relations between data items
  * ex) users_table & posts_table -> 어떤 post를 어떤 user가 썻는지, 특정 user가 post들을 모두 보기 가능

  ```python
  from datetime import datetime
  from app import db
  
  class User(db.Model):
      id = db.Column(db.Integer, primary_key=True)
      posts = db.relationship('Post', backref='author', lazy='dynamic')
  
  
      def __repr__(self):
          return '<User {}>'.format(self.id)
  
  class Post(db.Model):
      id = db.Column(db.Integer, primary_key=True)
      body = db.Column(db.String(140))
      timestamp = db.Column(db.DateTime, index=True, default=datetime.utcnow)
      user_id = db.Column(db.Integer, db.ForeignKey('user.id'))
  
      def __repr__(self):
          return '<Post {}>'.format(self.body)
  ```

  * `Post` class의 `user_id` field : *foreign key*   (linked!)
    * relation 'one-to-many' : 'one' user writes 'many' posts
    * references an `id` valud from `users table`
      * 이때 `user` : name of the database table for the model
      * model is referenced by the model class, which typically starts with an uppercase character
      * `ForeginKey`의 경우 model is given by its database table name, for which SQLAlchemy automatically uses lowercase characters and, for multi-word model names, snake case
  * `Post` 의 `timestamp` field
    * `default` = `datetime.utcnow` (function call X, function 자체를 변수로 넘김)
    * SQLAlchemy will set the field to the value of calling that function
  * `User`의 `posts` field
    * `db.relationship` : not an actual database field
      * High-level view of the relationship between users and posts
    * 'one-to-many' relationship, `db.relationship` field is normally defined on the 'one' side
    * Convenient way to get access to 'many'
    * `backref` : defines the name of a field that will be added to the objects of the 'many' class that points back at the 'one' object
      * `Post` class에 `post.author` = 그 post를 쓴 user를 return
    * `lazy` : defines how the database query for relationship will be issued **.....?**



* Practice

  ```python
  >>> u = User(username='john', email='john@example.com')
  >>> db.session.add(u)
  >>> db.session.commit()
  ```

  * `db.session` : changes to database are done in the context of a session
    * multiple changes can be accumulated in a session
  * `db.session.commit()` : writes all the changes(sessions에 모여있는 changes들) atomically
    * changes are only written to the database when `commit` is called
  * `db.session.rollback()` : abort the session & remove any changes stored in it
  * `db.session.delete` : delete
  * 'Sessions' guarantee that db will never be left in an inconsistent state

  

  ```python
  >>> users = User.query.all()
  ```

  * `query` : all models have 'query' attribute that is the entry point to run database queries
  * `all()` : all elements of that class

  

  ```python
  >>> u = User.query.get(1)
  ```

  * user.id = 1 인 user return……? get이 뭐하는 놈이길래..?

  

  ```python
  >>> p = Post(body='my first post!', author=u)
  ```

  * `timestamp`는 default값 존재
  * `user_id`의 경우 `author`를 통해 `db.relationship`이 형성되므로, `user.id` 값 가져오기 가능

  ```python
  # get all users in reverse alphabetical order
  >>> User.query.order_by(User.username.desc()).all()
  ```



* Shell Context

  * `flask shell` command : useful tool in `flask` umbrella of commands
    * Command pre-imports the application instance
  * `shell` command is second 'core' command implemented by Flask, after `run`
  * to start a Python interpreter in the context of the application

  ```python
  // microblog.py
  from app import app, db
  from app.models import User, Post
  
  @app.shell_context_processor
  def make_shell_context():
      return {'db': db, 'User': User, 'Post': Post}
  ```

  * `app.shell_context_processor` decorator : registers the function as a shell context function
  * When `flask shell` command runs, will invoke this function and register the items returned by it in the shell session
  * `FLASK_APP = microblog.py`라는 기본 environment set가 되어있어야 가능 (*.flaskenv* file에 setting)





flask의 SQLITE database더 알아보기 -> 'flask db migrate' , 'flask db upgrade' , 'flask db downgrade'



primary_key...?