#### Part14 : AJAX

* **Server-side vs. Client-side**
  * Server-side : server does all the work, while the client just displays the web pages and accepts user input
  * Client-side : client takes a more active role
  * **`Single Page Applications` or SPAs**
    *  In a strict client-side application the entire application is downloaded to the client with the initial page request, and then the application runs entirely on the client, only contacting the server to retrieve or store data and making dynamic changes to the appearance of that first and only web page.
  *  To do real time translations of user posts, **the client browser will send *asynchronous requests* to the server, to which the server will respond without causing a page refresh**. The client will then insert the translations into the current page dynamically
  * **`AJAX` : Asynchronous JavaScript and XML** ( XML대신 JSON을 요즘 사용하기도 한다)

* Live Translation Workflow

  * 외국인이 쓴 post의 번역본을 실시간으로 확인하고 싶을때 (이전 Part13처럼 미리 번역한 파일로 compile하려면 매번 번역될때마다 창이 redirect ( page replaced ) -> 비효율적)
  * user가 번역 버튼을 클릭했을때 Ajax request 요청, 번역해서 보여주기

* Language Identification

  * 먼저, post가 어떤 언어로 작성되었는지 알아야함

    * Automated detection works fairly well...
    * Python `guess_language`

    ```
    (venv) $ pip install guess_language-spirit
    ```

    * post 게시글이 작성될때 바로 어떤 언어의 글인지 미리 저장해두기

    ```python
    # models.py
    class Post(db.Model):
        # ...
        language = db.Column(db.String(5))
    ```

    

    ```python
    # routes.py
    from guess_language import guess_language
    
    @app.route('/', methods=['GET', 'POST'])
    @app.route('/index', methods=['GET', 'POST'])
    @login_required
    def index():
        form = PostForm()
        if form.validate_on_submit():
            language = guess_language(form.post.data)
            if language == 'UNKNOWN' or len(language) > 5:
                language = ''
            post = Post(body=form.post.data, author=current_user,
                        language=language)
            ...
    ```

    * `guess_language` function 을 이용해 post.data가 어떤 언어인지 미리 정보 저장해두기

* Displaying a "Translate" Link

  ```html
  // _post.html
  {% if post.language and post.language != g.locale %}
  <br><br>
  <a href="#">{{ _('Translate') }}</a>
  {% endif %}
  ```

  * localeselector에 포함되어있지않은 임시번역이 필요한 경우 사용...

* Using a Third-Party Translation Service

* ```
  export MS_TRANSLATOR_KEY=<paste-your-key-here>
  ```

  ```
  (venv) $ pip install requests
  ```

  * Microsoft Translator API : web service that accepts HTTP requests
  * `requests` package

  

  ```python
  # translate.py
  import json
  import requests
  from flask_babel import _
  from app import app
  
  def translate(text, source_language, dest_language):
      if 'MS_TRANSLATOR_KEY' not in app.config or \
              not app.config['MS_TRANSLATOR_KEY']:
          return _('Error: the translation service is not configured.')
      auth = {'Ocp-Apim-Subscription-Key': app.config['MS_TRANSLATOR_KEY']}
      r = requests.get('https://api.microsofttranslator.com/v2/Ajax.svc'
                       '/Translate?text={}&from={}&to={}'.format(
                           text, source_language, dest_language),
                       headers=auth)
      if r.status_code != 200:
          return _('Error: the translation service failed.')
      return json.loads(r.content.decode('utf-8-sig'))
  ```

  * `json.loads()` function : Python standard library to decode 'JSON' into a Python string that i can use
  * The `content` attribute of the response object contains the raw body of the response as a bytes object, which is converted to a UTF-8 string and sent to `json.loads()`

  ```
  // examples...
  >>> from app.translate import translate
  >>> translate('Hi, how are you today?', 'en', 'es')  # English to Spanish
  ```

  

* Ajax From The Server

  * An asynchronous (or Ajax) request is similar to the routes and view functions that I have created in the application, with the only difference that instead of returning HTML or a redirect, it just returns data, formatted as [XML](http://en.wikipedia.org/wiki/XML) or more commonly [JSON](http://en.wikipedia.org/wiki/JSON)

  ```python
  # routes.py
  from flask import jsonify
  from app.translate import translate
  
  @app.route('/translate', methods=['POST'])
  @login_required
  def translate_text():
      return jsonify({'text': translate(request.form['text'],
                                        request.form['source_language'],
                                        request.form['dest_language'])})
  ```

  * `request.form` attribute : dictionary that Flask exposes with all data that has included in the submission
  * 이전의 경우, 항상 flask-wtf의 render_template에서 알아서 모두 해줬기 때문에 `request.form` 형식을 사용할 필요가 없었음..
  * **근데 `POST`로 보낼 data 'text', 'source_language', 'dest_language'는 어디서 가져오는거지.....?????**



* Ajax From The Client

  * When working with JavaScript in the browser, the page currently being displayed is internally represented in as the Document Object Model or just the DOM

  * The JavaScript code running in this context can make changes to the DOM to trigger changes in the page

  * 위의 코드 세개 변수 받아오는 방법!

    * need to locate the node within the DOM that contains the blog post body and read its contents

    ```html
    // _post.html
    <span id="post{{ post.id }}">{{ post.body }}</span>
    ```

    ```html
    // _post.html
    <span id="translation{{ post.id }}">
        <a href="#">{{ _('Translate') }}</a>
    </span>
    ```

    * assign unique identifier to each blog post

      * ex) `post1`, `post2`...

    * can use jQuery to locate `<span>` element for that post and extract the text in it (주어진 id를 통해 post.body값 접근 가능)

      ```
      $('#post1').text()
      ```

      * `$` : name of a function provided by jQuery library
      * **`#` : 'selector' syntax used by jQuery**

    

    ```html
    // base.html
    {% block scripts %}
        ...
        <script>
            function translate(sourceElem, destElem, sourceLang, destLang) {
                $(destElem).html('<img src="{{ url_for('static', filename='loading.gif') }}">');
                $.post('/translate', {
                    text: $(sourceElem).text(),
                    source_language: sourceLang,
                    dest_language: destLang
                }).done(function(response) {
                    $(destElem).text(response['text'])
                }).fail(function() {
                    $(destElem).text("{{ _('Error: Could not contact server.') }}");
                });
            }
        </script>
    {% endblock %}
    ```

    * 4 arguments

      * `sourceElem` : unique ID for the post
      * `destElem` : translate link nodes
      * `sourceLang` : source language code
      * `destLang` : destination language code

    * Translation 진행 중일 때, *spinner* 가 보이도록 해줌(jQuery 이용)

      * `$(destElem).html()` function
      * **언제 spinner가 보이고, 사라지는지 어떻게 아는거지.....?**

    * **`$.post()` : jQuery, send `POST` request**

      * **Flask가 routes.py에 있는 `request.form` dictionary에서 변수값 받기 가능**

    * JavaScript는 기본이 *asynchronous* -> callback function을 이용해서 response를 받았을때만 추가 작업을 하도록 promise 생성

      ```javascript
      $.post(<url>, <data>).done(function(response) {
          // success callback
      }).fail(function() {
          // error callback
      })
      ```

    ```html
    // _post.html
    <span id="translation{{ post.id }}">
        <a href="javascript:translate(
                 '#post{{ post.id }}',
                 '#translation{{ post.id }}',
                 '{{ post.language }}',
                 '{{ g.locale }}');">{{ _('Translate') }}</a>
    </span>
    ```

    * The `href` element of a link can accept any JavaScript code if it is prefixed with  `javascript:`, so that is a convenient way to make the call to the translation function
    * The `#` that you see as a prefix to the `post<ID>` and `translation<ID>` elements indicates that what follows is an element ID
    * `translation{{post.id}}` : 해당 html을 번역된 글로 띄워주기 위해 element가 필요!
    * **`_post.html`에서 사용하는 translate() 함수 : `base.html`의 `<script>`에 define된 함수**
      * `base.html`의 translate() 함수에서 `routes.py`의 `/translate`으로 `POST` request를 보냄
    * **`routes.py`에서 사용하는 translate() 함수 : `translate.py`에 define된 함수**



js랑 같이 공부 한번 더하기.....

css.html.js.react codecademy얼른 다 다시 보기!(주말에 하기)