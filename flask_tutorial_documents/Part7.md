#### Part7 : Error Handling

* (Flask) Debug Mode

  * a mode in which Flask outputs a really nice debugger directly on your browser

  ```
  (venv) $ export FLASK_DEBUG=1(개발자모드) or 0(일반모드)
  ```

  * debugger allows you expand each stack frame and see the corresponding source code
  * can also open a Python prompt on any of the frames and execute any valid Python expressions
  * 개발 모드에서만 사용하고, 실제 배포때는 적용시키면 안됌
  * *reloader* : `flask run`한 상태에서 application 변경시, file save할 때마다 변경된 사항이 바로 적용됌!



* Custom Error Pages

  * Flask provides a mechanism for an application to install its own error pages

  ```python
  # errors.py
  from flask import render_template
  from app import app, db
  
  @app.errorhandler(404)
  def not_found_error(error):
      return render_template('404.html'), 404
  
  @app.errorhandler(500)
  def internal_error(error):
      db.session.rollback()
      return render_template('500.html'), 500
  ```

  * `@errorhandler` decorator
    * return second value(404, 500 - error code number) after the template
    * **`view function`의 defuault second_value = '200'**
      * 200 : status code for successful response
  * `500 error` : could be invoked after a 'database error'
    * username 이 duplicate하게 등록되는것을 방지, *session을 rollback*에서 clean state로 바꿔준다

* Sending Errors by Email

  ```python
  # config.py
  class Config(object):
      # ...
      MAIL_SERVER = os.environ.get('MAIL_SERVER')
      MAIL_PORT = int(os.environ.get('MAIL_PORT') or 25)
      MAIL_USE_TLS = os.environ.get('MAIL_USE_TLS') is not None
      MAIL_USERNAME = os.environ.get('MAIL_USERNAME')
      MAIL_PASSWORD = os.environ.get('MAIL_PASSWORD')
      ADMINS = ['your-email@example.com']
  ```

  * add the email server details to the configuration file
  * If the email server is not set in the environment, then I will use that as a sign that emailing errors needs to be disabled **……?????**
  * The email server port can also be given in an environment variable, but if not set, the standard port 25 is used
  * `ADMINS` configuration variable is a list of the email addresses that will receive error reports, so your own email address should be in that list

  

  ```python
  # __init__.py
  
  import logging
  from logging.handlers import SMTPHandler
  
  # ...
  
  if not app.debug:
      if app.config['MAIL_SERVER']:
          auth = None
          if app.config['MAIL_USERNAME'] or app.config['MAIL_PASSWORD']:
              auth = (app.config['MAIL_USERNAME'], app.config['MAIL_PASSWORD'])
          secure = None
          if app.config['MAIL_USE_TLS']:
              secure = ()
          mail_handler = SMTPHandler(
              mailhost=(app.config['MAIL_SERVER'], app.config['MAIL_PORT']),
              fromaddr='no-reply@' + app.config['MAIL_SERVER'],
              toaddrs=app.config['ADMINS'], subject='Microblog Failure',
              credentials=auth, secure=secure)
          mail_handler.setLevel(logging.ERROR)
          app.logger.addHandler(mail_handler)
  ```

  * Python's `logging` package to write its logs
    * package already has ability to send logs by email
  * add a 'SMTPHandler' instance to the Flask logger object, `app.logger`
    * `SMTPHandler` : class, located in the `logging.handlers` module, support sending logging messages to an email address via SMTP
    * *class* `logging.handlers.SMTPHandler`(*mailhost*, *fromaddr*, *toaddrs*, *subject*, *credentials=None*, *secure=None*, *timeout=1.0*)
  * `app.debug` : debug 모드가 아닐때에만 'email logger' 실행
  * `SMTPHandler` instance : sets its level so that it only reports errors and not warnings, informational or debugging messages
  * finally attach it to `app.logger` object from Flask

**---> 나중에 한번더 체크!!**



* Logging to a File

  * maintain a log file for the application

  ```python
  # __init__.py
  # ...
  from logging.handlers import RotatingFileHandler
  import os
  
  # ...
  
  if not app.debug:
      # ...
  
      if not os.path.exists('logs'):
          os.mkdir('logs')
      file_handler = RotatingFileHandler('logs/microblog.log', maxBytes=10240,
                                         backupCount=10)
      file_handler.setFormatter(logging.Formatter(
          '%(asctime)s %(levelname)s: %(message)s [in %(pathname)s:%(lineno)d]'))
      file_handler.setLevel(logging.INFO)
      app.logger.addHandler(file_handler)
  
      app.logger.setLevel(logging.INFO)
      app.logger.info('Microblog startup')
  ```

  * `RotatingFileHandler` : class, located in the `logging.handlers` module, supports rotation of disk log files
    * *class* `logging.handlers.RotatingFileHandler`(*filename*, *mode='a'*, *maxBytes=0*, *backupCount=0*, *encoding=None*, *delay=False*)
    * Locates the logs, 'maxBytes' limit을 걸어주면서 application을 너무 오래 run 했을때 log file이 더이상 커지지 않도록 방지
    * keeping the last ten log files as backup!
    * Writing the log file with name `microblog.log` in a *logs* directory
      * 해당 directory가 없을 경우, 알아서 새로 생성됌
  * `logging.Formatter` class : provides custom formatting for the log messages
  * `logger.setLevel(loggin.INFO)` : `DEBUG`, `INFO`, `WARNING`, `ERROR` and `CRITICAL` in increasing order of severity



* Fixing the Duplicate Username Bug

  ```python
  class EditProfileForm(FlaskForm):
      username = StringField('Username', validators=[DataRequired()])
      about_me = TextAreaField('About me', validators=[Length(min=0, max=140)])
      submit = SubmitField('Submit')
  
      def __init__(self, original_username, *args, **kwargs):
          super(EditProfileForm, self).__init__(*args, **kwargs)
          self.original_username = original_username
  
      def validate_username(self, username):
          if username.data != self.original_username:
              user = User.query.filter_by(username=self.username.data).first()
              if user is not None:
                  raise ValidationError('Please use a different username.')
  ```

  * edit창에 들어가서 user가 username을 아예 변경하지않아도 문제가 없어야 한다.
  * `current_user` : flask_login에서 제공해주는 친구! 어떻게 작동하는지 알아보기
  * 내 기존 username은 이미 db에 존재하기 때문에 변경하지않은 상태로 edit submit을 눌렀을때 original_username 체크를 안하는경우 `ValidationError` 발생하게 됌!!!
  * `super`? 안하면 생기는 문제는..?
    * `super()` : 자식 클래스에서 부모클래스의 내용을 사용하고 싶은 경우
      * `super().부모클래스 내용`
    * 부모인 FlaskForm의 `__init__` method 호출을 왜 하는 거지..?

  ```python
  # routes.py
  @app.route('/edit_profile', methods=['GET', 'POST'])
  @login_required
  def edit_profile():
      form = EditProfileForm(current_user.username)
      # ...
  ```

  

  ```python
  # routes.py
  @app.route('/edit_profile', methods=['GET', 'POST'])
  @login_required
  def edit_profile():
      form = EditProfileForm(current_user.username)
      # ...
  ```