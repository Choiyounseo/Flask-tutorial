#### Part10 : Email Support



**-> CH7과 함께 나중에 다시 한번더 체크!**

* Email password reset feature for users that forget their password



* Flask-Mail

  ```
  (venv) $ pip install flask-mail
  ```

  ```
  (venv) $ pip install pyjwt
  ```

  * password reset link의 secure token을 만들기 위한 'JSON Web Tokens'

  * configuration variable들의 경우 CH.7에서 error mail-handling때 app.config에 이미 모두 setting해뒀음

  ```python
  # __init__.py
  from flask_mail import Mail
  
  app = Flask(__name__)
  
  mail = Mail(app)
  ```
  * Test 방법은 사이트 참조..



* Simple Email Framework

  ```python
  # email.py
  from flask_mail import Message
  from app import mail
  
  def send_email(subject, sender, recipients, text_body, html_body):
      msg = Message(subject, sender=sender, recipients=recipients)
      msg.body = text_body
      msg.html = html_body
      mail.send(msg)
  ```

  

* Requesting a Password Reset

  ```html
  // login.html
  <p>
      Forgot Your Password?
      <a href="{{ url_for('reset_password_request') }}">Click to Reset It</a>
  </p>
  ```

  * add link for reset_password

  ```python
  # forms.py
  class ResetPasswordRequestForm(FlaskForm):
      email = StringField('Email', validators=[DataRequired(), Email()])
      submit = SubmitField('Request Password Reset')
  ```

  ```html
  // reset_password_request.html
  {% extends "base.html" %}
  
  {% block content %}
      <h1>Reset Password</h1>
      <form action="" method="post">
          {{ form.hidden_tag() }}
          <p>
              {{ form.email.label }}<br>
              {{ form.email(size=64) }}<br>
              {% for error in form.email.errors %}
              <span style="color: red;">[{{ error }}]</span>
              {% endfor %}
          </p>
          <p>{{ form.submit() }}</p>
      </form>
  {% endblock %}
  ```

  ```python
  # routes.py
  from app.forms import ResetPasswordRequestForm
  from app.email import send_password_reset_email
  
  @app.route('/reset_password_request', methods=['GET', 'POST'])
  def reset_password_request():
      if current_user.is_authenticated:
          return redirect(url_for('index'))
      form = ResetPasswordRequestForm()
      if form.validate_on_submit():
          user = User.query.filter_by(email=form.email.data).first()
          if user:
              send_password_reset_email(user)
          flash('Check your email for the instructions to reset your password')
          return redirect(url_for('login'))
      return render_template('reset_password_request.html',
                             title='Reset Password', form=form)
  ```



* Password Reset Tokens

  * how to generate a password request link
  * only valid reset links can be used to reset an account's password
  * 'JWT(Json Web Token)' 
    * The nice thing about JWTs is that they are self contained. You can send a token to a user in an email, and when the user clicks the link that feeds the token back into the application, it can be verified on its own.

  ```
  >>> import jwt
  >>> token = jwt.encode({'a': 'b'}, 'my-secret', algorithm='HS256')
  >>> token
  b'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJhIjoiYiJ9.dvOo58OBDHiuSHD4uW88nfJikhYAXc_sfUHq1mDi4G0'
  >>> jwt.decode(token, 'my-secret', algorithms=['HS256'])
  {'a': 'b'}
  ```

  * `my-secret` : secret key를 이용해 configuration
  * `HS256` algorithm이 가장 많이 사용됨

  * Payload that going to use for password reset tokens
    * `{'reset_password': user_id, 'exp': token_expiration}`.
    * `exp` field : standard for JWTs / indicates an expiration time for the token(제한시간 개념인듯...)
      * 제한시간 넘으면 이것도 invalid하다고 간주
  * When the user clicks on the emailed link, the token is going to be sent back to the application as part of the URL, and the first thing the view function that handles this URL will do is to verify it. If the signature is valid, then the user can be identified by the ID stored in the payload. Once the user's identity is known, the application can ask for a new password and set it on the user's account.

  

  ```python
  # models.py
  from time import time
  import jwt
  from app import app
  
  class User(UserMixin, db.Model):
      # ...
  
      def get_reset_password_token(self, expires_in=600):
          return jwt.encode(
              {'reset_password': self.id, 'exp': time() + expires_in},
              app.config['SECRET_KEY'], algorithm='HS256').decode('utf-8')
  
      @staticmethod
      def verify_reset_password_token(token):
          try:
              id = jwt.decode(token, app.config['SECRET_KEY'],
                              algorithms=['HS256'])['reset_password']
          except:
              return
          return User.query.get(id)
  ```

  * `get_reset_password_token()` function : JWT token 생성(as a string)
    * `decode('utf-8')` : `jwt.encode()` 가 byte sequence로 return하기 때문에 string으로 변환시켜주는 장치(application에서 좀더 편하게 사용하기 위해)
  * `verify_reset_password_token()` : static method
    * **static method**
      * can be invoked directly from the class
      * class method와는 다르게 first argument로 class(self의미)를 받지 않아도 된다
    * token이 invalid하거나, expired(기간 만료)가 되면 exception raise
  * `reset_password` key : ID of the user **.....?**



* Sending a Password Reset Email

  ```python
  # email.py
  from flask import render_template
  from app import app
  
  def send_password_reset_email(user):
      token = user.get_reset_password_token()
      send_email('[Microblog] Reset Your Password',
                 sender=app.config['ADMINS'][0],
                 recipients=[user.email],
                 text_body=render_template('email/reset_password.txt',
                                           user=user, token=token),
                 html_body=render_template('email/reset_password.html',
                                           user=user, token=token))
  ```

  * `render_template`에서 user, token을 함께 받음 -> personalized email message 생성 가능

  ```txt
  // email로 보내는 txt파일 글!
  Dear {{ user.username }},
  
  To reset your password click on the following link:
  
  {{ url_for('reset_password', token=token, _external=True) }}
  
  If you have not requested a password reset simply ignore this message.
  
  Sincerely,
  
  The Microblog Team
  ```

  * email로 링크를 보내는경우 http:// 부터 시작하는 full URL이 필요
  * 단순히 `url_for('user', username='susan')`식으로 할경우, */user/susan* 만 return해주기 때문에 메일 링크로 사용 불가능
  * 때문에 `_external=True`를 이용해 complete URL return



* Resetting a User Password

  ```python
  # forms.py
  class ResetPasswordForm(FlaskForm):
      password = PasswordField('Password', validators=[DataRequired()])
      password2 = PasswordField(
          'Repeat Password', validators=[DataRequired(), EqualTo('password')])
      submit = SubmitField('Request Password Reset')
  ```

  * `ResetPasswordForm` class 새로 만들기

  

  ```python
  # routes.py
  from app.forms import ResetPasswordForm
  
  @app.route('/reset_password/<token>', methods=['GET', 'POST'])
  def reset_password(token):
      if current_user.is_authenticated:
          return redirect(url_for('index'))
      user = User.verify_reset_password_token(token)
      if not user:
          return redirect(url_for('index'))
      form = ResetPasswordForm()
      if form.validate_on_submit():
          user.set_password(form.password.data)
          db.session.commit()
          flash('Your password has been reset.')
          return redirect(url_for('login'))
      return render_template('reset_password.html', form=form)
  ```

  * `/reset_password/<token>` view function 만들기

  ```html
  // reset_password.html
  
  {% extends "base.html" %}
  
  {% block content %}
      <h1>Reset Your Password</h1>
      <form action="" method="post">
          {{ form.hidden_tag() }}
          <p>
              {{ form.password.label }}<br>
              {{ form.password(size=32) }}<br>
              {% for error in form.password.errors %}
              <span style="color: red;">[{{ error }}]</span>
              {% endfor %}
          </p>
          <p>
              {{ form.password2.label }}<br>
              {{ form.password2(size=32) }}<br>
              {% for error in form.password2.errors %}
              <span style="color: red;">[{{ error }}]</span>
              {% endfor %}
          </p>
          <p>{{ form.submit() }}</p>
      </form>
  {% endblock %}
  ```

  * `reset_password` route를 위한 html

  

* **login창에서 `reset_password_request`를 통해 비밀번호 변경을 위한 이메일 전송, 이메일 전송때 `user.get_reset_password_token`을 이용해 토큰 생성, 이후 이메일 링크(`{{ url_for('reset_password', token=token, _external=True) }}`)를 통해 `reset_password/<token>`으로 이동, `user.verify_reset_password_token`을 통해 decode로 token 맞는지 체크후, 비밀번호 변경가능!**



* Asynchronous Emails

  * sending an email slows the application down considerably
  * All the interactions that need to happen when sending an email make the task slow
  * make `send_email()` function to be *asynchronous*
    * `threading` & `multiprocessing` modules can both do this

  

  ```python
  from threading import Thread
  # ...
  
  def send_async_email(app, msg):
      with app.app_context():
          mail.send(msg)
  
  
  def send_email(subject, sender, recipients, text_body, html_body):
      msg = Message(subject, sender=sender, recipients=recipients)
      msg.body = text_body
      msg.html = html_body
      Thread(target=send_async_email, args=(app, msg)).start()
  ```

  * `send_async_email` function : runs in a background thread, by `Thread()`
  * the sending of the email will run in the thread, process가 완료되면 thread도 함께 clean up됨
  * `app_context()`
    * **The reason many extensions need to know the application instance is because they have their configuration stored in the `app.config` object.**
    * **The application context that is created with the `with app.app_context()` call makes the application instance accessible via the `current_app` variable from Flask.**
  * You probably expected that only the `msg` argument would be sent to the thread, but as you can see in the code, I'm also sending the application instance. When working with threads there is an important design aspect of Flask that needs to be kept in mind. **Flask uses *contexts* to avoid having to pass arguments across functions??????**. I'm not going to go into a lot of detail on this, but know that there are two types of contexts, the ***application context* and the *request context***. In most cases, these contexts are automatically managed by the framework, but when the application starts custom threads, contexts for those threads may need to be manually created.

  