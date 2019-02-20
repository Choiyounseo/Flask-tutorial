#### Part3 : Web Forms

- Web forms : one of the most basic building blocks in any web application

- `Flask-WTF` extension : thin wrapper around `WTForms` package

- Configuration **——> 다시 체크!!!!!!!**

  - `app.config` : basic solution for application to **specify configuration options**
  - use 'dictionary style' to work with variables

  ```python
  app = Flask(__name__)
  app.config['SECRET_KEY'] = 'you-will-never-guess'
  ```

  - `app.config.from-object()` method

  ```python
  // config.py
  import os
  
  class Config(object):
      SECRET_KEY = os.environ.get('SECRET_KEY') or 'you-will-never-guess'
  ```

  ```python
  // __init__.py
  from flask import Flask
  from config import Config
  
  app = Flask(__name__)
  app.config.from_object(Config)
  
  from app import routes
  ```

  

- User Login Form

- ```python
  # forms.py
  from flask_wtf import FlaskForm
  from wtforms import StringField, PasswordField, BooleanField, SubmitField
  from wtforms.validators import DataRequired
  
  class LoginForm(FlaskForm):
      username = StringField('Username', validators=[DataRequired()])
      password = PasswordField('Password', validators=[DataRequired()])
      remember_me = BooleanField('Remember Me')
      submit = SubmitField('Sign In')
  ```

  - `validators` : to attach validation behaviors to fields
  - **`DataRequired` validator simply checks that the field is not submitted empty**

- Form Templates

  ```html
  // login.html
  {% extends "base.html" %}
  
  {% block content %}
      <h1>Sign In</h1>
      <form action="" method="post" novalidate>
          {{ form.hidden_tag() }}
          <p>
              {{ form.username.label }}<br>
              {{ form.username(size=32) }}
          </p>
          <p>
              {{ form.password.label }}<br>
              {{ form.password(size=32) }}
          </p>
          <p>{{ form.remember_me() }} {{ form.remember_me.label }}</p>
          <p>{{ form.submit() }}</p>
      </form>
  {% endblock %}
  ```

  - `<form>` element in HTML
    - `action` : submit된 후 browser의 URL 결정
    - `method` : default value = 'GET'
    - `novalidate` : used to tell the web_browser to not apply validation to the fields in thie form
  - `form.hidden_tag()` : template argument generates a hidden field that includes a token that is used to protect the form against CSRF attacks
    - Hidden field와 `SECRET_KEY` variable defined in Flask configuration을 통해 form protect 가능
  - `{{ form.<field_name>.label }}` , where I wanted the field label
  - `{{ form.<field_name>() }}` , where I wanted the field
  - Fields from the form object know how to render themselves as HTML!

- Receiving Form Data

  ```python
  // routes.py
  
  from flask import render_template, flash, redirect
  
  @app.route('/login', methods=['GET', 'POST'])
  def login():
      form = LoginForm()
      if form.validate_on_submit():
          flash('Login requested for user {}, remember_me={}'.format(
              form.username.data, form.remember_me.data))
          return redirect('/index')
      return render_template('login.html', title='Sign In', form=form)
  ```

  - `methods` argument

    - this `view function` accepts `GET` and `POST` requests
    - **overriding the default, which is to accept only `GET` requests**

  - **`form.validate_on_submit()`** method : does all form processing work

    - If `GET` request, method returns **false**
    - If `POST` request, method gathers all the data, *run all the validators attached to fields*, and if everythingis all right, return **true**
      - -> POST 되려는 data(field)들이 valid한 form들이면 can be processed by application
      - field가 적어도 하나 invalid시, return **false**

  - `flash()` function : useful to show message

    ```html
    // base.html
    
    {% with messages = get_flashed_messages() %}
    {% if messages %}
    <ul>
    {% for message in messages %}
    <li>{{ message }}</li>
    {% endfor %}
    </ul>
    {% endif %}
    {% endwith %}
    ```

    - **`get_flashed_messages()` **function : comes from Flask, and returns a list of all the messages that have been registered with `flash()` previously

- Improving Field Validation

  - field의 form이 invalid 할 때, 체크하는 방법

    ```html
    {{ form.username.label }}<br>
    {{ form.username(size=32) }}<br>
    {% for error in form.username.errors %}
    <span style="color: red;">[{{ error }}]</span>
    {% endfor %}
    ```



- Generating Links
  - For better control of 'links' variables, Flask provides function called `url_for()`
    - generates URLs using its internal mapping of URLs to view functions
    - ex) `url_for('login')` returns `/login` url
      - argument to `url_for()` is *endpoint name*, which is the name of the view function