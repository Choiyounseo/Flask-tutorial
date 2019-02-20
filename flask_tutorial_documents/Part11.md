#### Part11 : Facelift

* Bootstrap

  * One of the most popular 'CSS framework'
  * benefits
    * Similar look in all major web browsers
    * Handling of desktop, tablet and phone screen sizes
    * Customizable layouts
    * Nicely styled navigation bars, forms, buttons, alerts, popups, etc.
  * Simply import the *bootstrap.min.css* file

  ```
  (venv) $ pip install flask-bootstrap
  ```

  ```python
  # __init__.py
  from flask_bootstrap import Bootstrap
  
  app = Flask(__name__)
  # ...
  bootstrap = Bootstrap(app)
  ```

  * With the extension initialized, *bootstrap/base.html* template becomes available, with `extends` clause
  * `base.html`의 맨위에 `{% extends 'bootstrap/base.html %'}`을 이용
    * base.html will export its own `app_content` block for its derived templates to define the page content

  * 예시만 보고 실제로 적용시키지는 않겠음...



* Rendering Bootstrap Forms

  * Instead of having to style the form fields one by one, Flask-Bootstrap comes with a macro that accepts a Flask-WTF form object as an argument and renders the complete form using Bootstrap styles

  ```html
  // register.html
  {% extends "base.html" %}
  {% import 'bootstrap/wtf.html' as wtf %}
  
  {% block app_content %}
      <h1>Register</h1>
      <div class="row">
          <div class="col-md-4">
              {{ wtf.quick_form(form) }}
          </div>
      </div>
  {% endblock %}
  ```

  * **`wtf.quick_form()` **macro가 자동으로 complete form을 만들어줌
    * validation error도 모두 포함하고 있음!

* Rendering Pagination Links

  * Bootstrap: pagination link도 지원함

  ```html
  // index.html
  ...
  <nav aria-label="...">
      <ul class="pager">
          <li class="previous{% if not prev_url %} disabled{% endif %}">
              <a href="{{ prev_url or '#' }}">
                  <span aria-hidden="true">&larr;</span> Newer posts
              </a>
          </li>
          <li class="next{% if not next_url %} disabled{% endif %}">
              <a href="{{ next_url or '#' }}">
                  Older posts <span aria-hidden="true">&rarr;</span>
              </a>
          </li>
      </ul>
  </nav>
  ```

  * 이전 index.html에서 next_url과 prev_url이 없으면 hiding했던 점과 다르게, `disabled` state를 이용하며 make 'link' appear grayed out





css_codecademy 뒤에부분꺼 더 체크하기.. html도 한번더 체크 