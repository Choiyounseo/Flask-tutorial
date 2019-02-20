```
echo "# Flask-tutorial" >> README.md
git init
git add README.md
git commit -m "first commit"
git remote add origin https://github.com/Choiyounseo/Flask-tutorial.git
git push -u origin master
```



```
source venv/bin/activate
```

```
deactivate
```



###Flask Tutorial

---

#### Part 1

```python
from flask import Flask

app = Flask(__name__)

from app import routes
```

* `__name__` : **Python predefined variable**, set to the name of the `module` which it is used
  * `__name__` variable passed to the `Flask` 'class'
  * Flask uses the location of the `module` passed here as a starting point when it needs to load associated resources such as template files

* `routes` module

  * You are going to see that the `routes` module needs to import the `app` variable defined in this script, *so putting one of the reciprocal imports at the bottom avoids the error that results from the mutual references between these two files.* ??
  * `routes` : different URLs that the application implements
  * Flask : handlers for the application routes are written as 'Python functions' == **`view` functions**

  

  ```python
  from app import app
  
  @app.route('/')
  @app.route('/index')
  def index():
      return "Hello, World!"
  ```

  * `@app.route` : *decorators*
    * Creates an association between the URL given as an argument and the function
    * when web browser requests either two URLs('/', '/index'), Flask is going to invoke `index()` function and pass the return value of it back to the browser as a 'response'



```
(venv) $ export FLASK_APP=microblog.py
```

* Flask needs to be told how to import, by setting the `FLASK_APP` environment variable



* `Python decorator`

  * `@decorator` : 대상 함수를 wrapping, 이 wrapping된 함수의 앞뒤에 추가적으로 꾸며질 구문 들을 정의해서 손쉽게 재사용 가능하게 해주는 것

  * Modifies the function that follows it

  * ```python
    import datetime
    
    def datetime_decorator(func):
    
            def decorated():
    
                    print datetime.datetime.now()
    
                    func()
    
                    print datetime.datetime.now()
    
            return decorated
    
    
    @datetime_decorator
    
    def main_function_1():
    
            print "MAIN FUNCTION 1 START"
    
    
    
    @datetime_decorator
    
    def main_function_2():
    
            print "MAIN FUNCTION 2 START"
    
    
    @datetime_decorator
    
    def main_function_3():
    
            print "MAIN FUNCTION 3 START"
    ```