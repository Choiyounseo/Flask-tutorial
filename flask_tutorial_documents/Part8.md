#### Part8 : Followers

* Database Relationships

  * relational database doesn't have a 'list type'
  * only 'tables' with records and relationships between these records
  * `One-to-Many`
    * `One` : *db.relationship* 
    * `Many` side : db with the use of  *foreignkey*
    * ![One-to-many Relationship](https://blog.miguelgrinberg.com/static/images/mega-tutorial/ch04-users-posts.png)
  * `Many-to-Many`
    * *association* table with two foreign keys
    * ![many-to-many](https://blog.miguelgrinberg.com/static/images/mega-tutorial/ch08-students-teachers.png)
  * `Many-to-One` , `One-to-One`
    * `One-to-One` : `One-to-Many`와 유사, Many side에서 not to have more than one link

* Representing Followers

  * User follows many users / user has many followers
  * 즉, 위의 `Many-to-Many` 처럼, 같은 user class 두개를 연결지어두는 *association table* 이 필요!
    * **self-referential relationship**
    * ![many-to-many](https://blog.miguelgrinberg.com/static/images/mega-tutorial/ch08-followers-schema.png)
    * `followers` : 'association table'

* Database Model Representation

  ```python
  # models.py
  followers = db.Table('followers',
      db.Column('follower_id', db.Integer, db.ForeignKey('user.id')),
      db.Column('followed_id', db.Integer, db.ForeignKey('user.id'))
  )
  ```

  * Foreign keys만 존재 ->  굳이 class로 선언하지 않았음..

  ```python
  # models.py
  class User(UserMixin, db.Model):
      # ...
      followed = db.relationship(
          'User', secondary=followers,
          primaryjoin=(followers.c.follower_id == id),
          secondaryjoin=(followers.c.followed_id == id),
          backref=db.backref('followers', lazy='dynamic'), lazy='dynamic')
  ```

  * `db.relationship` : links `User` instances to other `User` instances
  * `followed` : 내(left side)가 following하는 사람들(right side)을 모아둔 곳 -> 나로 인해 followed된 사람들을 모아둔 곳
  * `'User'` : right side enttity of the relationship
  * `Secondary` : association table that is used for this relationship
  * `primaryjoin` : condition that links the left side entity with association table
  * `secondaryjoin` : condition that links the right side entity with association table
    * **만약 이게 `User` class가 아닌 다른 class를 사용한다면?`== id` 대신 어떤걸 넣어야 할까? ex) `post.id` ?????**
  * `backref` : defines how this relationship will be accessed from the right side entity
    * left side : `followed` / right side : `followers`
    * `lazy` : execution mode for this query
      * `dynamic` : sets up the query not to run until specifically requested



* Adding & Removing "follows"

  ```python
  user1.followed.append(user2)
  # user1 follow the user2
  
  user1.followed.remove(user2)
  ```

  ```python
  # models.py
  class User(UserMixin, db.Model):
      #...
  
      def follow(self, user):
          if not self.is_following(user):
              self.followed.append(user)
  
      def unfollow(self, user):
          if self.is_following(user):
              self.followed.remove(user)
  
      def is_following(self, user):
          return self.followed.filter(
              followers.c.followed_id == user.id).count() > 0
  ```

  

* Obtaining the Posts from Followed Users

  * `user.followed.all()` 을 통해 모든 followed user들을 모은 뒤 이 user들의 `post`를 모아 합쳐서 정렬
    * (Problem1) : followed user가 많은 경우 time, memory적으로 부담
    * (Problem2) : 가장 최근 followed user들의 post만 보고 싶어도 정렬을 위해 모든 post가 필요하기 때문에 매우 시간 낭비

  ```python
  # models.py
  class User(db.Model):
      #...
      def followed_posts(self):
          return Post.query.join(
              followers, (followers.c.followed_id == Post.user_id)).filter(
                  followers.c.follower_id == self.id).order_by(
                      Post.timestamp.desc())
  ```

  * application 상에서 하지말고, db에서 훨씬 효율적으로 sorting을 할수 있으니 let the database figure out how to extract that information in the most efficient way (모두 합쳐서 sorting하는것은 불가피, db가 더 효울적인 방법으로 sorting하도록 하는것이 좋음(직접 코드를 짜는것보다))

  

  ```python
  Post.query.join(...).filter(...).order_by(...)
  ```

  * Joins

    * **A JOIN clause is used to combine rows from two or more tables, based on a related column between them**

    ```python
    Post.query.join(followers, (followers.c.followed_id == Post.user_id))
    ```

    * `followers` : association table

    * Want the database to create a temporary table that combines data from posts and followers tables

      * followed_id사람들의 post를 association table과 함께 합쳐서 보고 싶은것

    * | id   | text            | user_id | follower_id | followed_id |
      | ---- | --------------- | :-----: | ----------- | ----------- |
      | 1    | post from susan |    2    | 1           | 2           |
      | 2    | post from mary  |    3    | 2           | 3           |
      | 3    | post from david |    4    | 1           | 4           |
      | 3    | post from david |    4    | 3           | 4           |

  * Filters

    ```python
    filter(followers.c.follower_id == self.id)
    ```

    * 그중 follower_id = 현재 user id인 글들만 골라오기

  * Sorting

    ```python
    order_by(Post.timestamp.desc())
    ```

    * timestamp field of the post in descending order (가장 최근 글이 가장 먼저 보이도록)

* Combining Own and Followed Posts

  * user들은 followed user들의 post와 자신의 post도 함께 보고싶어함
  * (Method1) : user가 자기자신은 follow하도록 만든다
    * 단점은 followed수가 1개 + 되버림(자기자신으로 인해)
  * (Method2) : create second query that returns the user's own posts, and then use `union` operator to combine the two queries into a single one

  ```python
  # models.py
  def followed_posts(self):
      followed_posts = Post.query.join(
          followers, (followers.c.followed_id == Post.user_id)).filter(
          followers.c.follower_id == self.id)
      own_posts = Post.query.filter_by(user_id = self.id)
      return followed_posts.union(own_posts).order_by(Post.timestamp.desc())    
  ```

  

* Unit Testing the User Model

  ```python
  # tests.py
  from datetime import datetime, timedelta
  import unittest
  from app import app, db
  from app.models import User, Post
  
  class UserModelCase(unittest.TestCase):
      def setUp(self):
          app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite://'
          db.create_all()
  
      def tearDown(self):
          db.session.remove()
          db.drop_all()
      def test_....
      ....
  ```

  

  * Python `unittest` package : easy to write, execute unit tests
  * `setUp()`, `tearDown()` methods : special methods that the unit testing framework executes before and after each test respectively
  *  I have implemented a little hack in `setUp()`, to prevent the unit tests from using the regular database that I use for development. By changing the application configuration to `sqlite://` I get SQLAlchemy to use an in-memory SQLite database during the tests.
  * `db.create_all()` call creates all the database tables. This is a quick way to create a database from scratch that is useful for testing



* Integrating Followers with the Application

  ```python
  # routes.py
  @app.route('/follow/<username>')
  @login_required
  def follow(username):
      user = User.query.filter_by(username=username).first()
      if user is None:
          flash('User {} not found.'.format(username))
          return redirect(url_for('index'))
      if user == current_user:
          flash('You cannot follow yourself!')
          return redirect(url_for('user', username=username))
      current_user.follow(user)
      db.session.commit()
      flash('You are following {}!'.format(username))
      return redirect(url_for('user', username=username))
  
  @app.route('/unfollow/<username>')
  @login_required
  def unfollow(username):
      user = User.query.filter_by(username=username).first()
      if user is None:
          flash('User {} not found.'.format(username))
          return redirect(url_for('index'))
      if user == current_user:
          flash('You cannot unfollow yourself!')
          return redirect(url_for('user', username=username))
      current_user.unfollow(user)
      db.session.commit()
      flash('You are not following {}.'.format(username))
      return redirect(url_for('user', username=username))
  ```

  * `db.session.commit()` 필수!

  ```html
  // user.html
  
  ```

  



Query