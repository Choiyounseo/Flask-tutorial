#### Part5 : User Logins

* Password Hashing

  * `Werkzeug` : one of the packages that implement password hashing

  

  ```python
  >>> from werkzeug.security import generate_password_hash
  >>> hash = generate_password_hash('foobar')
  ```

  * password `foobar` is transformed into a long encoded string through a series of cryptographic operations that have no known reverse operation
  * same password도 다른 hash값 배정받음 -> hash_password를 통해 original_password 유추 불가능

  

  ```python
  >>> from werkzeug.security import check_password_hash
  >>> check_password_hash(hash, 'foobar')
  True
  ```

  * `check_password_hash` 로 체크가능

  

  ```python
  from werkzeug.security import generate_password_hash, check_password_hash
  
  # ...
  
  class User(db.Model):
      # ...
  
      def set_password(self, password):
          self.password_hash = generate_password_hash(password)
  
      def check_password(self, password):
          return check_password_hash(self.password_hash, password)
  ```

  ```python
  >>> u = User(username='susan', email='susan@example.com')
  >>> u.set_password('mypassword')
  >>> u.check_password('anotherpassword')
  False
  >>> u.check_password('mypassword')
  True
  ```



* Flask-Login

  * `Flask-Login` : Flask extension

  ```
  (venv) $ pip install flask-login
  ```

  ```python
  # __init__.py
  from flask_login import LoginManager
  
  app = Flask(__name__)
  # ...
  login = LoginManager(app)
  
  # ...
  ```

  * need to be created & initialized

  * 4 required items

    * `is_authenticated`: a property that is `True` if the user has valid credentials or `False`otherwise.
    * `is_active`: a property that is `True` if the user's account is active or `False` otherwise.
    * `is_anonymous`: a property that is `False` for regular users, and `True` for a special, anonymous user.
    * `get_id()`: a method that returns a unique identifier for the user as a string (unicode, if using Python 2).

  * `UserMixin` : Flask-Login *mixin* class

    ```python
    # models.py
    from flask_login import UserMixin
    
    class User(UserMixin, db.Model):
    ```



* User Loader Function

  * `Flask-Login` keeps track of logged in user by storing its unique identifier in Flask's user session

    * Storage space assigned to each user who connects to the application

    * needs application's help in loading a user (F-L knows nothing about db)

      ```python
      # models.py
      from app import login
      # ...
      
      @login.user_loader
      def load_user(id):
          return User.query.get(int(id))
      ```

      * user loader is registered with Flask-Login with `@login.user_loader` decorator
      * `id` argument is string -> need to convert string to integer

* Logging Users In

  ```python
  # routes.py
  from flask_login import current_user, login_user
  from app.models import User
  
  # ...
  
  @app.route('/login', methods=['GET', 'POST'])
  def login():
      if current_user.is_authenticated:
          return redirect(url_for('index'))
      form = LoginForm()
      if form.validate_on_submit():
          user = User.query.filter_by(username=form.username.data).first()
          if user is None or not user.check_password(form.password.data):
              flash('Invalid username or password')
              return redirect(url_for('login'))
          login_user(user, remember=form.remember_me.data)
          return redirect(url_for('index'))
      return render_template('login.html', title='Sign In', form=form)
  ```

  * If `current_user.is_authenticated` : 이미 로그인한 유저가 다시 로그인창으로 들어가려 할 경우 방지하는 케이스
    * **`current_user`** variable
      * comes from 'Flask-Login'
      * can be used at any time during the handling to obtain user object
    * `is_authenticated` : Flask-Login property, check if the user is logged in or not
  * `filter_by` : result = query that only includes objects that have a matching username
    * `first()` : return the user object if exists, or `None` if it doesn't
  * `login_user()` function : (Flask-Login) basic function, register the user as logged in, so that any future pages the user navigates to will have `current_user` variable set to that user!!



* Logging Users Out

  * `logout_user()` function : (Flask-Login) basic function

  ```python
  # routes.py
  # ...
  from flask_login import logout_user
  
  # ...
  
  @app.route('/logout')
  def logout():
      logout_user()
      return redirect(url_for('index'))
  ```

  ```html
  // base.html
   <div>
          Microblog:
          <a href="{{ url_for('index') }}">Home</a>
          {% if current_user.is_anonymous %}
          <a href="{{ url_for('login') }}">Login</a>
          {% else %}
          <a href="{{ url_for('logout') }}">Logout</a>
          {% endif %}
      </div>
  ```

  * `is_anonymous` property : one of the attiributes that 'Flask-Login' adds to user objects through `UserMixin` class
  * `current_user.is_anonymous` : true only when the user is not logged in



* Requiring Users To Login

  * 로그인 해야만 특정 페이지들 볼수 있게 구현

  ```python
  # __init__.py
  login = LoginManager(app)
  login.login_view = 'login'
  ```

  * 로그인 되어 있지않을 시, '/login' url로 가게 해주는 장치 `'login'`

  

  ```python
  # routes.py
  from flask_login import login_required
  
  @app.route('/')
  @app.route('/index')
  @login_required
  def index():
  ```

  * `@login_required` decorator 이용
  * `@app.route` decorator 아래에 이 decorator를 추가시, 해당  view function이 protected, will not allow access to users that are not authenticated

  * 로그인 하지않은 사용자가 '/index' url로 접근시, '/login' 으로 redirect됌
    * 이때, 로그인을 하면 다시 '/index' 페이지에 접근할수 있도록 redirect해주는 장치 존재
    * URL이 `/login?next=/index`
    * `next` query string이 original URL에 setting되어 있음 -> application이 이 URL을 redirect back after login구현 장치로 사용가능

  ```python
  # routes.py
  
  from flask import request
  from werkzeug.urls import url_parse
  
  @app.route('/login', methods=['GET', 'POST'])
  def login():
      # ...
      if form.validate_on_submit():
          user = User.query.filter_by(username=form.username.data).first()
          if user is None or not user.check_password(form.password.data):
              flash('Invalid username or password')
              return redirect(url_for('login'))
          login_user(user, remember=form.remember_me.data)
          next_page = request.args.get('next')
          if not next_page or url_parse(next_page).netloc != '':
              next_page = url_for('index')
          return redirect(next_page)
      # ...
  ```

  * **`request`** variable : contains all the information that the client sent with the reques
  * `request.args` attribute : exposes the contents of the 'query string' in friendly distionary format
  * `url_parse()` : function that check if URL is relative or absolute. **.....??**
  * 3 cases to consider
    * If the login URL does not have a `next` argument, then the user is redirected to the index page.
    * If the login URL includes a `next` argument that is set to a relative path (or in other words, a URL without the domain portion), then the user is redirected to that URL.
    * If the login URL includes a `next` argument that is set to a full URL that includes a domain name, then the user is redirected to the index page. -> for security



* Showing the Logged In User in Templates

  ```html
  // index.html
  {% block content %}
      <h1>Hi, {{ current_user.username }}!</h1>
  {% endblock %}
  ```

  * use `current_user`



* User Registration

  ```python
  # forms.py
  from flask_wtf import FlaskForm
  from wtforms import StringField, PasswordField, BooleanField, SubmitField
  from wtforms.validators import ValidationError, DataRequired, Email, EqualTo
  from app.models import User
  
  # ...
  
  class RegistrationForm(FlaskForm):
      username = StringField('Username', validators=[DataRequired()])
      email = StringField('Email', validators=[DataRequired(), Email()])
      password = PasswordField('Password', validators=[DataRequired()])
      password2 = PasswordField(
          'Repeat Password', validators=[DataRequired(), EqualTo('password')])
      submit = SubmitField('Register')
  
      def validate_username(self, username):
          user = User.query.filter_by(username=username.data).first()
          if user is not None:
              raise ValidationError('Please use a different username.')
  
      def validate_email(self, email):
          user = User.query.filter_by(email=email.data).first()
          if user is not None:
              raise ValidationError('Please use a different email address.')
  ```

  * `Email` : stock validator that comes with 'WTForms' that ensure that what the user types in this field matches the structure of an email address
  * `EqualTo` : validator that make sure that its value is identical to the one for the first password field
  * `ValidationError` 
  * **`validate_<field_name>` 으로 method를 만들경우, 'WTForms'가 자동으로 custom validators도 가져가서 validator로 사용함!!!!**

  

  ```python
  # routes.py
  from app import db
  from app.forms import RegistrationForm
  
  # ...
  
  @app.route('/register', methods=['GET', 'POST'])
  def register():
      if current_user.is_authenticated:
          return redirect(url_for('index'))
      form = RegistrationForm()
      if form.validate_on_submit():
          user = User(username=form.username.data, email=form.email.data)
          user.set_password(form.password.data)
          db.session.add(user)
          db.session.commit()
          flash('Congratulations, you are now a registered user!')
          return redirect(url_for('login'))
      return render_template('register.html', title='Register', form=form)
  ```

  

  **login은 Flask-Login의  login_user, login_required, logout_user등등 이 있으나, register은 알맞은 형태의 form을 알아서 만든후, router에서 User형태의 db로 잘 저장만 해주면 되기 때문에 특별한 helper function들이 사용되지는 않음**