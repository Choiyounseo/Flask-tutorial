#### Part6 : Profile Page and Avatars

* User Profile Page

  ```python
  # routes.py
  @app.route('/user/<username>')
  @login_required
  def user(username):
      user = User.query.filter_by(username=username).first_or_404()
      posts = [
          {'author': user, 'body': 'Test post #1'},
          {'author': user, 'body': 'Test post #2'}
      ]
      return render_template('user.html', user=user, posts=posts)
  ```

  * `<username>` URL component
    * when route has a dynamic component, *Flask will accept any text in the portion of the URL*, and will invoke the view function with the actual text as an argument
    * ex) `/user/susan` -> view function이 username을 'susan'으로 setting

  ```html
  // base.html
  <a href="{{ url_for('user', username=current_user.username) }}">Profile</a>
  ```

  

* Avatars

  * 'Gravatar'
    * URL with the format *https://www.gravatar.com/avatar/<hash>*
    * `<hash>` : MD5 hash of the user's email address
    * `s` argument : size 조정가능
    * `d` argument : determines what image Gravatar provides for users that do not have an avatar registered with the service

  ```python
  # models.py
  from hashlib import md5
  # ...
  
  class User(UserMixin, db.Model):
      # ...
      def avatar(self, size):
          digest = md5(self.email.lower().encode('utf-8')).hexdigest()
          return 'https://www.gravatar.com/avatar/{}?d=identicon&s={}'.format(
              digest, size)
  ```

  * `avatar()`
    * returns URL of the user's avatar image, scaled to the requested size in pixels
    * for users that don't have an avatar registered, 'identicon' image will be generated!
  * Since `MD5` support in Python works on bytes (not on strings) -> need to encode string to bytes



* Using Jinja2 Sub-Templates

  * *user.html*과 *index.html* 에 동시 사용할 sub-template 만들기

  ```python
  {% include '_post.html' %}
  ```

  * **`include` statement 을 통해 sub-template 소환 가능**



* More Interesting Profiles

  ```python
  # models.py
  class User(UserMixin, db.Model):
      # ...
      about_me = db.Column(db.String(140))
      last_seen = db.Column(db.DateTime, default=datetime.utcnow)
  ```

  * **database가 변경될 때 마다 db-migration 필수적으로 해주기!!**

* Recording the Last Visit Time For a User

  * `last_seen` field :  write current time on this field for a given user whenever that user sends a request to the server

  ```python
  # routes.py
  from datetime import datetime
  
  @app.before_request
  def before_request():
      if current_user.is_authenticated:
          current_user.last_seen = datetime.utcnow()
          db.session.commit()
  ```

  * `@before_request` decorator : register the decorated function to be executed **right before the view function**
  * can execute this before *any* view function in the application
  * `db.session.commit()` 필수!!



* Profile Editor

  ```python
  # forms.py
  from wtforms import StringField, TextAreaField, SubmitField
  from wtforms.validators import DataRequired, Length
  
  # ...
  
  class EditProfileForm(FlaskForm):
      username = StringField('Username', validators=[DataRequired()])
      about_me = TextAreaField('About me', validators=[Length(min=0, max=140)])
      submit = SubmitField('Submit')
  ```

  * `TextAreaField` : multi-line box in which the user can enter text

  

  ```python
  # routes.py
  from app.forms import EditProfileForm
  
  @app.route('/edit_profile', methods=['GET', 'POST'])
  @login_required
  def edit_profile():
      form = EditProfileForm()
      if form.validate_on_submit():
          current_user.username = form.username.data
          current_user.about_me = form.about_me.data
          db.session.commit()
          flash('Your changes have been saved.')
          return redirect(url_for('edit_profile'))
      elif request.method == 'GET':
          form.username.data = current_user.username
          form.about_me.data = current_user.about_me
      return render_template('edit_profile.html', title='Edit Profile',
                             form=form)
  ```

  * `db.session.commit()` 필수!
  * **`form.validate_on_submit()` : `GET` request에선 return false!!**
  * `request.method`
    * initial request는 `GET`-> username과 about_me에 대한 이전의 데이터 미리 불러옴
    * 이후 validate_on_submit되지않은 상태에서 `POST` request가 실행됨 -> html이 화면에 띄워짐!!
    * 따로 서버에서 POST, GET을 언제보내라 하지않아도 알아서 둘다 신호를 항상 보내는건가……..? 뭐지





React 다시 보기(component)