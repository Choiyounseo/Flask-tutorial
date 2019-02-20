#### Part15 : A Better Application Structure

* 현재까지의 subsystems들 정리
  * The user authentication subsystem, which includes some view functions in *app/routes.py*, some forms in *app/forms.py*, some templates in *app/templates* and the email support in *app/email.py*.
  * The error subsystem, which defines error handlers in *app/errors.py* and templates in *app/templates*.
  * The core application functionality, which includes displaying and writing blog posts, user profiles and following, and live translations of blog posts, which is spread through most of the application modules and templates.

* A better solution would be to not use a global variable for the application, and instead use an ***application factory* function** to create the function at runtime. This would be a function that **accepts a configuration object as an argument, and returns a Flask application instance, configured with those settings**. If I could modify the application to work with an application factory function, then writing tests that require special configuration would become easy, because **each test can create its own application**.



* Blueprints

  *  `blueprint` : Flask logical structure that represents a subset of the application
    * can include `routes`, `view functions)(handlers for the application routes)`, `forms`, `templates`, `static files`
    * if write the bluprint in a separate Python package, then you have a component that encapsulates the elements related to specific feature of the application
    * Contents of `blueprint` : initially `dormant` state (application과 register되야 가능)
    * a temporary storage for application functionality that helps in organizing the code

* Error Handling Blueprint

  ```
  app/
      errors/                             <-- blueprint package
          __init__.py                     <-- blueprint creation
          handlers.py                     <-- error handlers
      templates/
          errors/                         <-- error templates
              404.html
              500.html
      __init__.py                         <-- blueprint registration
  ```

  * 두개의 `__init__.py` 파일!

  * Flask blueprints can be configured to have separate directory for templates or static files

  

  ```python
  # app/errors/__init__.py
  from flask import Blueprint
  
  bp = Blueprint('errors', __name__)
  
  from app.errors import handlers
  ```

  * create blueprint ( name of blueprint, name of base module(`__name__`))
  * In the *handlers.py* module, instead of attaching the error handlers to the application with the `@app.errorhandler` decorator, I use the blueprint's `@bp.app_errorhandler` decorator. While both decorators achieve the same end result, the idea is to try to make the blueprint independent of the application so that it is more portable. I also need to modify the path to the two error templates to account for the new *errors* sub-directory where they were moved.

  

  ```python
  # app/__init__.py
  app = Flask(__name__)
  
  from app.errors import bp as errors_bp
  app.register_blueprint(errors_bp)
  
  from app import routes, models  # <-- remove errors from this import!
  ```

  * register the blueprint with the application
    * `register_bluprint()` method
  * When a blueprint is registered, any view functions, templates, static files, error handlers, etc. are connected to the application

  

* Authentication Blueprint

  * `@app.route` 대신 `@bp.route`로 decorate 변경

  * **`url_for()`의 경우, 이제 argument가 must include blueprint name and the view function name, separated by a period**

    * ex) `url_for('login')` -> `url_for('auth.login')` 으로 변경해주기

    ```python
    # app/__init__.py
    # .. (creat_app 함수 내부)
    from app.auth import bp as auth_bp
    app.register_blueprint(auth_bp, url_prefix='/auth')
    # ...
    ```



* Application Factory Pattern

  * 이전까지 단계에서는 application had to be 'global variable'

    * because, all `view functions` and `error handlers` needed to be decorated with decorators that come from `app`
    * ex) `@app.route`
    * blueprints -> app이 굳이 global일 필요가 없음...

  * new function **`create_app()`**

    * **constructs a Flask application instance**, and eliminate the global variable

    ```python
    # app/__init__.py
    # ...
    db = SQLAlchemy()
    migrate = Migrate()
    login = LoginManager()
    login.login_view = 'auth.login'
    login.login_message = _l('Please log in to access this page.')
    mail = Mail()
    bootstrap = Bootstrap()
    moment = Moment()
    babel = Babel()
    
    def create_app(config_class=Config):
        app = Flask(__name__)
        app.config.from_object(config_class)
    
        db.init_app(app)
        migrate.init_app(app, db)
        login.init_app(app)
        mail.init_app(app)
        bootstrap.init_app(app)
        moment.init_app(app)
        babel.init_app(app)
    
        # ... no changes to blueprint registration
    
        if not app.debug and not app.testing:
            # ... no changes to logging setup
    
        return app
    ```

    * 대부분의 Flask extensions : initialized by creating an instance of the extension & passing the application as an argument
    * application이 더이상 global variable이 아닌 경우, extension을 두단계로 나누어 initialize시킬수가 있음
      * (1) extension instance를 argument passing없이 create
        * Instance of the extension that is not attached to the application
      * (2) `init_app()` method를 이용해 extension instance를 application과 bind하기

  * `app` -> `current_app` 으로 replace : import app같은걸 안해도 되도록 만들어줌

    * `current_user`, `g` 등과 유사..

  * `current_app._get_current_obejct()` 

    * extracts the actual application instance from inside the proxy object
    * `current_app`의 경우 proxy object (dynamically mapped to the application instance)

    

  * 전체적으로 모두 `import app`을 바로 하는게 아닌, create된 app instance를 인자로 받아 사용하도록 변경해주기

  

  * `test.py`에서, create_app을 사용할때, `db.create_all()` (create the database tables)

    * `db` 가 방금 create_app된 instance만 사용할수 있는 방법?

    * **`current_app` variable 이용**

      *  `current_app` variable, which somehow acts as a proxy for the application when there is no global application to import? **This variable looks for an active application context in the current thread, and if it finds one, it gets the application from it**. If there is no context, then there is no way to know what application is active, so `current_app` raises an exception.

      ```
      >>> from flask import current_app
      >>> current_app.config['SQLALCHEMY_DATABASE_URI']
      Traceback (most recent call last):
          ...
      RuntimeError: Working outside of application context.
      
      >>> from app import create_app
      >>> app = create_app()
      >>> app.app_context().push()
      >>> current_app.config['SQLALCHEMY_DATABASE_URI']
      'sqlite:////home/miguel/microblog/app.db'
      ```

      (예시 terminal)

* Environment Variables

  * `.env` file in root app directory로 관리

  ```
  (venv) $ pip install python-dotenv
  ```

  * `config.py` module이 모든 environment variables를 저장하고 있으므로,  `Config` class 생성전, `.env`file을 load해준다

* Requirements File

  * if need to regenerate the environment on another machine, intall한 모든 packages들을 다시 설치해줘야함
  * 미리 `requirements.txt` file (in root folder) 에 all dependencies with version 를 적어두면 편하다

  ```
  (venv) $ pip freeze > requirements.txt
  ```

  * `pip freeze` : dump all the packages that are installed on your virtual environment in the correct format

  

  이제 다른 컴퓨터에서 같은 environment를 사용하고 싶을때, 

  ```
  (venv) $ pip install -r requirements.txt
  ```

  



git rm --cached app.db -> git의 repository delte로 status 변경(commit, push하면 파일 사라짐)