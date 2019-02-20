
####Part16 : Full-Text Search

* Intall Elasticsearch

  `brew install elasticsearch`

  * 제대로 설치되었다면 `http://localhost:9200`에서 json format의 기본 글들 확인가능
  * 파이썬에서도 사용하기 위해 install 해주기

  ```
  (venv) $ pip install elasticsearch
  (venv) $ pip freeze > requirements.txt
  ```

  

* Elasticsearch Tutorial

  ```
  >>> from elasticsearch import Elasticsearch
  >>> es = Elasticsearch('http://localhost:9200')
  
  >>> es.index(index='my_index', doc_type='my_index', id=1, body={'text': 'this is a test'}) // 추가
  
  >>> es.search(index='my_index', doc_type='my_index',
  ... body={'query': {'match': {'text': 'this test'}}}) // 검색
  
  >>> es.indices.delete('my_index') // 삭제
  ```

  * An index can store documents of different types if desired, and in that case the `doc_type`argument can be set to different values according to those different format



* Elastic Configuration

  * app.config에 환경 추가
  * 'ELASTICSEARCH_URL' = http://localhost:9200 변수값은 따로 직접 터미널에 치거나 / *.env* file에 저장하기

  ```python
  # __init__.py
  # ...
  from elasticsearch import Elasticsearch
  
  # ...
  
  def create_app(config_class=Config):
      app = Flask(__name__)
      app.config.from_object(config_class)
  
      # ...
      app.elasticsearch = Elasticsearch([app.config['ELASTICSEARCH_URL']]) \
          if app.config['ELASTICSEARCH_URL'] else None
  
      # ...
  ```

  

* Full-Text Search Abstraction

  * Post에 `__searchable__` class attribute 추가

  ```python
  # models.py
  class Post(db.Model):
      __searchable__ = ['body']
  ```

  * `app/search.py`파일에 elasticsearch 관련 코드, 함수 정의

  ```python
  # search.py
  from flask import current_app
  
  def add_to_index(index, model):
      if not current_app.elasticsearch:
          return
      payload = {}
      for field in model.__searchable__:
          payload[field] = getattr(model, field)
      current_app.elasticsearch.index(index=index, doc_type=index, id=model.id,
                                      body=payload)
  
  def remove_from_index(index, model):
      if not current_app.elasticsearch:
          return
      current_app.elasticsearch.delete(index=index, doc_type=index, id=model.id)
  
  def query_index(index, query, page, per_page):
      if not current_app.elasticsearch:
          return [], 0
      search = current_app.elasticsearch.search(
          index=index, doc_type=index,
          body={'query': {'multi_match': {'query': query, 'fields': ['*']}},
                'from': (page - 1) * per_page, 'size': per_page})
      ids = [int(hit['_id']) for hit in search['hits']['hits']]
      return ids, search['hits']['total']
  ```

  * `model.__searchable__`에 저장된 attirbute 가져와서 text로 저장
    * `getattr`을 이용해 `model.{{ model.__searchable__ }}` 값 가져오기 가능
  * `query_index()` 
    * `multi-match` : can search across multiple fields
    * field name `*` : look in all the fields, searching the entire index
    * pagination가능
      * `body`, `size` argument : control what subset of the entire result set needs to be retured
      * 직접 page, page_size로 계산해서 적어줘야함.....
    * `return` : query와 동일한 문자를 가진 글들과 & 총 매칭된 글의 개수
      * es.search()의 결과물 직접 보면 바로 이해됨!



* Integrating Searches with SQLAlchemy

  * Problem1 : results come as a list of numeric IDs(현재 매칭된 글들의 id들만 return됨)
    * template에 rendering을 위한 SQLAlchemy models가 필요함
    * need to replace list of numbers with corresponding models from database
  * Problem2 : this solution requires the application to explicitly issue indexing calls as posts are added or removed, which is not terrible, but less than ideal, since a bug that causes a missed indexing call when making a change on the SQLAlchemy side is not going to be easily detected, the two databases will get out of sync more and more each time the bug occurs and you will probably not notice for a while. **A better solution would be for these calls to be triggered automatically as changes are made on the SQLAlchemy database** **???**

* For the problem of triggering the indexing changes automatically, I decided to drive updates to the Elasticsearch index from SQLAlchemy *events*

  *  SQLAlchemy provides a large list of [events](http://docs.sqlalchemy.org/en/latest/core/event.html)that applications can be notified about. For example, each time a session is committed, I can have a function in the application invoked by SQLAlchemy, and in that function I can apply the same updates that were made on the SQLAlchemy session to the Elasticsearch index

* `SearchableMixin`이라는 새 class를 정의

  * ```python
    # models.py
    from app.search import add_to_index, remove_from_index, query_index
    
    class SearchableMixin(object):
        @classmethod
        def search(cls, expression, page, per_page):
            ids, total = query_index(cls.__tablename__, expression, page, per_page)
            if total == 0:
                return cls.query.filter_by(id=0), 0
            when = []
            for i in range(len(ids)):
                when.append((ids[i], i))
            return cls.query.filter(cls.id.in_(ids)).order_by(
                db.case(when, value=cls.id)), total
    
        @classmethod
        def before_commit(cls, session):
            session._changes = {
                'add': list(session.new),
                'update': list(session.dirty),
                'delete': list(session.deleted)
            }
    
        @classmethod
        def after_commit(cls, session):
            for obj in session._changes['add']:
                if isinstance(obj, SearchableMixin):
                    add_to_index(obj.__tablename__, obj)
            for obj in session._changes['update']:
                if isinstance(obj, SearchableMixin):
                    add_to_index(obj.__tablename__, obj)
            for obj in session._changes['delete']:
                if isinstance(obj, SearchableMixin):
                    remove_from_index(obj.__tablename__, obj)
            session._changes = None
    
        @classmethod
        def reindex(cls):
            for obj in cls.query:
                add_to_index(cls.__tablename__, obj)
    
    db.event.listen(db.session, 'before_commit', SearchableMixin.before_commit)
    db.event.listen(db.session, 'after_commit', SearchableMixin.after_commit)
    ```

  * `class method` : special method that is associated with the class and not a particular instance

  * `self` -> `cls` 표현 사용 ( instance가 아닌 class를 받는다는 점을 헷갈리지 않기 위함)

    * 예를 들어 `Post` model에 attached되면, `Post.search()` 만으로 `search()` method 사용 가능(without have actual instance of class `Post`)

  * **`search()` class** ???

    * `query_index`에서 가져온 list of object IDs를 actual object로 변경
    * `cls.__tablename__` : as index name **???**
      * **all indexes will be named with the name Flask-SQLAlchemy assigned to the relational table**
    * The SQLAlchemy query that retrieves the list of objects by their IDs is based on a `CASE` statement from the SQL language, which needs to be used to ensure that the results from the database come in the same order as the IDs are given. This is important because the Elasticsearch query returns results sorted from more to less relevant
    * `search()` function returns the query that replaces the list of IDs, and also passes through the total number of search results as a second return value
    * `cls.id.in_(ids)` ? total==0일때 뭘 reutrn하는거지? db.case(when)?

  * **`before_commit()` class, `after_commit()` class**

    * **SQLAlchemy event: db commit의 'before', 'after'로 triggered됨** ( `db.event.listen` 이용!)
    *  before handler is useful because the session hasn't been committed yet, so I can look at it and figure out what objects are going to be added, modified and deleted, available as `session.new`, `session.dirty` and `session.deleted` respectively
      * 위 세 objects들의 경우, commit후 사용불가, 때문에 따로 `session._changes` dictionary에 저장해서 함께 commit
    * `after commit()`이 된 후엔, elasticsearch side를 변경해줄 차례(아직 commit으로 인한 db만 변경된 상태)
      * `before_commit`때 저장해둔 `_changes` variable을 이용

  * **`reindex()` class**

    * simple helper method that you can use to refresh an index with all the data from the relational side

  

  * 이제 `Post` class에 incorporate하기

  ```python
  class Post(SearchableMixin, db.Model):
  ```



* Search Form

  * URL에서 검색한 argument(`q`)를 가져오려면 `GET` method 사용해야함

  ```python
  # main/forms.py
  from flask import request
  
  class SearchForm(FlaskForm):
      q = StringField(_l('Search'), validators=[DataRequired()])
  
      def __init__(self, *args, **kwargs):
          if 'formdata' not in kwargs:
              kwargs['formdata'] = request.args
          if 'csrf_enabled' not in kwargs:
              kwargs['csrf_enabled'] = False
          super(SearchForm, self).__init__(*args, **kwargs)
  ```

  * `__init__` constructor 

    * `formdata` argument : determines from where Flask-WTF gets form submissions
      * default : `request.form` -> `POST` request를 통해 전송
      * `GET` request : get have the field values in the query string, so I need to point Flask-WTF at `request.args`, which is where Flask writes the query string arguments **?????**
    * For clickable search links to work, CSRF needs to be disabled, so I'm setting `csrf_enabled` to `False` so that Flask-WTF knows that it needs to bypass CSRF validation for this form.

  * user가 어떤 페이지('/index', '/user'…) 던 상관없이 SearchForm은 보여져야한다. user가 로그인했을때 항상 보여줘야 함!

    * 즉, 어떤 request가 오던 일단 무조건 `SearchForm()`을 만들어 버리자!

    ```python
    # main/routes.py
    from flask import g
    from app.main.forms import SearchForm
    
    @bp.before_app_request
    def before_request():
        if current_user.is_authenticated:
            current_user.last_seen = datetime.utcnow()
            db.session.commit()
            g.search_form = SearchForm()
        g.locale = str(get_locale())
    ```

    ```html
    // base.html
    ...
    <div class="collapse navbar-collapse" id="bs-example-navbar-collapse-1">
    <ul class="nav navbar-nav">
    ... home and explore links ...
    </ul>
    {% if g.search_form %}
    <form class="navbar-form navbar-left" method="get"
    action="{{ url_for('main.search') }}">
    <div class="form-group">
    {{ g.search_form.q(size=20, class='form-control',
    placeholder=g.search_form.q.label.text) }}
    </div>
    </form>
    {% endif %}
    ...
    ```

    * `g.search_form` = SearchForm()
    * `g.search_form.q`= SearchForm.q = StringField!



* Search View Function

  ```python
  # main/routes.py
  @bp.route('/search')
  @login_required
  def search():
      if not g.search_form.validate():
          return redirect(url_for('main.explore'))
      page = request.args.get('page', 1, type=int)
      posts, total = Post.search(g.search_form.q.data, page,
                                 current_app.config['POSTS_PER_PAGE'])
      next_url = url_for('main.search', q=g.search_form.q.data, page=page + 1) \
          if total > page * current_app.config['POSTS_PER_PAGE'] else None
      prev_url = url_for('main.search', q=g.search_form.q.data, page=page - 1) \
          if page > 1 else None
      return render_template('search.html', title=_('Search'), posts=posts,
                             next_url=next_url, prev_url=prev_url)
  ```

  * Post.search한 결과 pagination으로 보여주는게 끝

  ```html
  // search.html
  {% extends "base.html" %}
  
  {% block app_content %}
      <h1>{{ _('Search Results') }}</h1>
      {% for post in posts %}
          {% include '_post.html' %}
      {% endfor %}
      <nav aria-label="...">
          <ul class="pager">
              <li class="previous{% if not prev_url %} disabled{% endif %}">
                  <a href="{{ prev_url or '#' }}">
                      <span aria-hidden="true">&larr;</span>
                      {{ _('Previous results') }}
                  </a>
              </li>
              <li class="next{% if not next_url %} disabled{% endif %}">
                  <a href="{{ next_url or '#' }}">
                      {{ _('Next results') }}
                      <span aria-hidden="true">&rarr;</span>
                  </a>
              </li>
          </ul>
      </nav>
  {% endblock %}
  ```





* **Elasticsearch engine을 이용, 기본 함수들 `search.py`에 저장, `POST` 모델에서 사용하기위한 새 `SearchableMixin`에서 이 `search.py` 함수들 사용(db변경될때 마다 search engine내 db도 변경해줘야 하기 때문 -> Flask event를 이용해 commit전,후로 함수 자동 call하도록 만듬). `SearchForm`을 따로 만들어서 로그인한 유저가 어떤 페이지에 있던 검색이 가능하도록 만들어주기 위해 `@before_app_request`에서 미리 form을 불러버림. 이후 검색클릭시 해당 `/search` route로 이동, `Post.search()`를 이용해 검색후 `render_template`**



* 궁금한점
  * SearchableMixin.search() 함수
  * base.html에 `<form method="get">`으로 `g.search_form`을 사용하는데, 이 `GET` METHOD는 따로 어느 route에서도 `methods=['GET']`을 적어줄 필요가 없는건가 아님 내가 못찾고 있는건가……….ㅜ 필요없나….?뭘까…
    * 따로 어떠한 route에서 GET으로 필요한게 아니고, 그냥 g.search.q.data를 가져오면 되는거라 필요없는듯...