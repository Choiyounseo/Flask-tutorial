####Part12 : Dates and Times

ch7다시보기!!

* Timezone Hell

  * `datetime.now()` : returns correct time for my location(우리나라에 맞는 시간)
  * `datetime.utcnow()` : returns time ins the UTC time zone(UTC 시간)
  * server must manage times that are consistent and independent of location
    * `post.timestamp` 는 UTC로 그대로 두면서, 각 user들이 게시글을 봤을 때 시각을 다르게 변경해주어야 한다(3시에 올린 게시글(at specific location)이 UTC기준 10시면 3시에 게시글이 올라온것으로 표기 해줘야함)

* Timezone Conversions

  * Solution : convert all timestamps from the stored UTC units to the local time of each user
    * need to know the location of each user

  * web browser knows the user's timezone, exposes it through the standard date and time JavaScript APIs
    * The "old school" approach would be to have the web browser somehow send the timezone information to the server when the user first logs on to the application. This could be done with an [Ajax](http://en.wikipedia.org/wiki/Ajax_(programming)) call, or much more simply with a [meta refresh tag](http://en.wikipedia.org/wiki/Meta_refresh). Once the server knows the timezone it can keep it in the user's session or write it to the user's entry in the database, and from then on adjust all timestamps with it at the time templates are rendered.
    * The "new school" approach would be to not change a thing in the server, and let the conversion from UTC to local timezone happen in the client, using JavaScript. <- 이 방법 이용



* Introducing 'Moment.js' & 'Flask-Moment'

  * `Moment.js` : small open-source JavaScript library that takes date and time rendering to another level, as it provides every imaginable formatting option, and then some
  * 'Flask-Moment' : small Flask extension that makes easy to incorporate `moment.js` into application

  ```
  (venv) $ pip install flask-moment
  ```

  ```python
  # __init__.py
  from flask_moment import Moment
  
  app = Flask(__name__)
  # ...
  moment = Moment(app)
  ```

  * unlike other extensions, **'Flask-Moment' works together with `moment.js`**

    * **All templates of the application must include this library!(`moment.js`)**

    ```html
    // base.html
    {% block scripts %}
        {{ super() }}
        {{ moment.include_moment() }}
    {% endblock %}
    ```

    * `moment.include_moment()` function : generates `<script>` tag (imports the library!!)
    * `flask-bootstrap`에서 {block scripts}의 경우 다른 block들(title. navbar, content..)과 다르게 기존 base content가 갖춰진 형태
      * 이 library에 `moment.js`만 추가로 넣고 싶을 경우, parent 내용을 부르는 `super()` function을 이용,
      * 기존 {block scripts}에 있는 내용 그대로, 원하는 라이브러리 추가로 수정해서 block 사용 가능

* Using Moment.js

  * 'Moment.js' makes `moment.js` available to the browser

  * `ISO 8601` format

    * `{{ year }}-{{ month }}-{{ day }}T{{ hour }}:{{ minute }}:{{ second }}{{ timezone }}`
    * UTC timezone : `Z`

  * `moments` object's some methods

    ```
    moment('2017-09-28T21:45:23Z').format('L')
    "09/28/2017"
    moment('2017-09-28T21:45:23Z').format('LL')
    "September 28, 2017"
    moment('2017-09-28T21:45:23Z').format('LLL')
    "September 28, 2017 2:45 PM"
    moment('2017-09-28T21:45:23Z').format('LLLL')
    "Thursday, September 28, 2017 2:45 PM"
    moment('2017-09-28T21:45:23Z').format('dddd')
    "Thursday"
    moment('2017-09-28T21:45:23Z').fromNow()
    "7 hours ago"
    moment('2017-09-28T21:45:23Z').calendar()
    "Today at 2:45 PM"
    ```

    * 예시 : UTC시간 9:45pm / local(computer)시간 2:45pm
    * **`fromNow()`의 경우, 내 시간 기준, 현재 입력된 시간이 몇시간 전, 후인지를 알려줌** (입력 시간을 UTC기준이 아닌, 내 location time 기준으로 해석!) (`fromNow()`, `calendar()` 해당)

  

  ```html
  // user.html
  {% if user.last_seen %}
  <p>Last seen on: {{ moment(user.last_seen).format('LLL') }}</p>
  {% endif %}
  ```

  * `user.last_seen` = db.Column(db.DateTime, default = date time.utcnow)
    * `user.last_seen = datetime.utcnow()`를 이용해 계속 업데이트
  * 이때 `moment()` : `ISO 8601` string이 아닌, Python `datetime` object를 인자로 받음
    * **`moment()`가 automatically generates the required JS code to insert the rendered timstamp in the proper place of the DOM**

  ```html
  // _post.html
  <a href="{{ url_for('user', username=post.author.username) }}">
      {{ post.author.username }}
  </a>
  said {{ moment(post.timestamp).fromNow() }}:
  <br>
  {{ post.body }}
  ```

  

