#### Part13 : I18n and L10n

* Internationalization(I18n) & Localization(L10n)

* Flask-Babel

  * Flask extention that makes working with translations easy

  ```
  (venv) $ pip install flask-babel
  ```

  ```python
  # __init__.py
  from flask_babel import Babel
  
  app = Flask(__name__)
  # ...
  babel = Babel(app)
  ```

  * initialize like most other Flask extensions

  

  * To keep track pf supported languages, need to add a configuration variable

  ```python
  # config.py
  class Config(object):
      # ...
      LANGUAGES = ['en', 'es']
  ```

  ```python
  # __init__.py
  from flask import request
  
  # ...
  
  @babel.localeselector
  def get_locale():
      return request.accept_languages.best_match(app.config['LANGUAGES'])
  ```

  * `localeselector` decorator : decorated function이 request에서 변역된 언어가 필요할 때마다 실행된다

  * `request` object의 `accept_languages` attribute : provides high-level interface to work with '*Accept-Language*' header that clients send with a request

  * header specifies the client language and locale preferences as a weighted list (web browser에 기본적으로 모두 있음)

    * 즉, 선호하는 언어들은 weight를 줘서 기록해 둘 수 있음

    ```
    Accept-Language: ko, en-gb;q=0.8, en;q=0.7
    ```

    * 여기선 ko = default weight = 1.0

  * Best language 를 select하기 위해서는 application이 제공가능한 언어 내에서, client가 가장 선호하는 언어를 골라줘야 함 -> `best_match()` method 이용

    * `best_match()` : takes the list of languages offered by the application as an argument and returns the best choice



* Marking Texts to Translate In Python Source Code

  * source code에서 번역해야 할 text를 모두 mark한 다음, Flask-Babel이 모든 file을 scan하며 번역을 위해 mark되있는 text들을 extract해 따로 seperate translation file에 담아야 한다. -> `gettext` tool 이용

  * translation을 위한 text_marking : **`_()` function**으로 wrapping

    ```python
    from flask_babel import _
    # ...
    flash(_('Your post is now live!'))
    ```

    * `_()` function
      * `localeselector`function을 통해 가장 적합한 언어로 translate 해줌
      * 번역된 text를 return, ex) argument to `flash()`

    

    ```
    flash('User {} not found.'.format(username))
    ```

    * text가 dynamic component를 가진 경우
    * `_()` function : older string substitution syntax만 적용가능 (위 케이스는 적용 불가능)

    ```
    flash(_('User %(username)s not found.', username=username))
    ```

    

    * 더 헷갈리는 경우 : string literals are assigned outside of a request, usually when the application is starting up
      * ex) StringField… client가 뭔가 적기 전까지 알수 없음
      * 이런 text들은 사용될때까지 (language) evaluation을 delay시키는 방법 밖에 없음
      * **`lazy_gettext()` : lazy evaluation version of `_()` function == `_l()` ** ( _l()을 써도 되도록 __init__.py에 저장해뒀음)



* Marking Texts to Translate In Templates

  * `_()` function: templates에도 사용 가능

  ```html
  <h1>File Not Found</h1>
  ```

  ```html
  <h1>{{ _('File Not Found') }}</h1>
  ```

  * `_()`를 이용하려면 `{{ ... }}` 이 필요
    * to force the `_()` to be evaluated instead of being considered a literal in the template

  ```html
  // _post.html
  {% set user_link %}
  <a href="{{ url_for('user', username=post.author.username) }}">
      {{ post.author.username }}
  </a>
  {% endset %}
  {{ _('%(username)s said %(when)s',
  username=user_link, when=moment(post.timestamp).fromNow()) }}
  ```

  * The problem here is that I wanted the `username` to be a link that points to the profile page of the user, not just the name, so I had to create an intermediate variable called `user_link` using the `set` and `endset` template directives, and then pass that as an argument to the translation function.



* Extracting Text to Translate

  * `pybabel` command : marked text들을 *.pot* file로 extract

    * *portable object template*
    * marked text들은 모두 한꺼번에 모아둔 text file

    

  ```cfg
  // babel.cfg
  [python: app/**.py]
  [jinja2: app/templates/**.html]
  extensions=jinja2.ext.autoescape,jinja2.ext.with_
  ```

  * `pybabel`에게 어떤 파일들은 translate을 위해 scan해야 하는지 알려주는 configuration file
  * 1,2 line : define the filename patterns for Python and Jinja2 template files respectively
  * 3 line : defines two extensions provided by Jinja2 template engine that help Flask-Babel properly parse template files

  

  ```
  (venv) $ pybabel extract -F babel.cfg -k _l -o messages.pot .
  ```

  * marked 해둔 모든 text들을 *.pot* file로 extract하기 위한 명령어
  * The `pybabel extract` command reads the configuration file given in the `-F` option, then scans all the code and template files in the directories that match the configured sources, starting from the directory given in the command (the current directory or `.` in this case). By default, `pybabel`will look for `_()` as a text marker, but I have also used the lazy version, which I imported as `_l()`, so I need to tell the tool to look for those too with the `-k _l`. The `-o` option provides the name of the output file.



* Generating a Language Catalog

  * Create a translation for each language

  ```
  (venv) $ pybabel init -i messages.pot -d app/translations -l es
  creating catalog app/translations/es/LC_MESSAGES/messages.po based on messages.pot
  ```

  * `pybabel init` command : `messages.pot` file을 input으로 받아서 `-d` option에 쓰인 directory로 새 언어 catalog를 적음
  * `-l` option : specified language option
  * 명령어 이후, message.po파일에 모든 marked text들이 모여있는걸 볼수 있음
  * `**.po` 파일 내부 `msgstr`의 경우 모두 직접 변역글들을 입력해줘야함.....**

  

  * to compile all the translations for the applications, use `pybabel compile` command

  ```
  (venv) $ pybabel compile -d app/translations
  compiling catalog app/translations/es/LC_MESSAGES/messages.po to
  app/translations/es/LC_MESSAGES/messages.mo
  ```

  * es -> ko 다 넣으면 한국어로 translate가능!
  * `.mo` file 생성 -> file that Flask-Babel will use to load translations for application
  * added to project languages, these are ready to be used in the application

* Updating the Translations

  ```
  (venv) $ pybabel extract -F babel.cfg -k _l -o messages.pot .
  (venv) $ pybabel update -i messages.pot -d app/translations
  ```

  * 추가로 translate하고 싶은 text가 생겼을 때 update하는 방법
  *  `update` call takes the new `messages.pot` file and merges it into all the *messages.po* files associated with the project

* Translating Dates and Times

  * `moment.js` library : localization&internationalization을 지원하지 않는다. -> 우리가 직접 _(), _l() 작업을 해주지도 않았기 때문에 번역되지 않은 상태로 보이게 됌
  * Flask-Babel returns the selected language and locale for given request via **`get_locale()` function**
  * `g` object에 locale한 후, base template에서 접근해 사용할 수 있도록 함
  * `get_locale()` : returns a *locale object* -> `__init__.py`에서 정의해서 만들었음!

  ```python
  # routes.py
  from flask import g
  from flask_babel import get_locale
  
  # ...
  
  @app.before_request
  def before_request():
      # ...
      g.locale = str(get_locale())
  ```

  ```html
  // base.html
  {% block scripts %}
      {{ super() }}
      {{ moment.include_moment() }}
      {{ moment.lang(g.locale) }}
  {% endblock %}
  ```



* Command-Line Enhancements
  * `pybabel` 명령어들이 길어서 짧게 단축시켜보자! (`babel extract` step은 없음...)
    * `flask translate init LANG` to add a new language
    * `flask translate update` to update all language repositories
    * `flask translate compile` to compile all language repositories
  * 단축 명령어를 위한 `cli.py` 파일 추가...