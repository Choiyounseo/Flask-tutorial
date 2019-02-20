#### Part22 : Background Jobs

* Task Queues

  * `worker process`
  * worker processes run independently of the application and can even be located on a different system
  * communication between the application and the workers is done through `message queue`
  * ![Task Queue Diagram](https://blog.miguelgrinberg.com/static/images/mega-tutorial/ch22-queue-diagram.png)

  * 'Celery' & 'Redis Queue' 등 다양한 `task queue` 존재
    * `RQ`의 경우, only support 'Redis message queue'
    * https://medium.com/sunhyoups-story/celery-b96eb337b9cf



* Using RQ

  * ```
    (venv) $ pip install rq
    (venv) $ pip freeze > requirements.txt
    ```

  * RQ의 경우, 'Redis message queue'만 사용이 가능하기 때문에, redis server가 필요

  ```
  (venv) $ brew install redis
  (venv) $ redis-server
  
  (venv) $ rq worker microblog-tasks 
  // microblog-tasks라는 일을 처리한 worker 생성 ( 여러개 생성하면 여러개 worker가 connected됨 )
  ```

  ```
  [shell]
  >>> from redis import Redis
  >>> import rq
  >>> queue = rq.Queue('microblog-tasks', connection=Redis.from_url('redis://'))
  >>> job = queue.enqueue('app.tasks.example', 23)
  // enqueue()로 add a job to the queue
  // 23 -> function의 parameter
  >>> job.get_id()
  >>> job.is_finished
  ```

  * The data that is stored in the queue regarding a task will stay there for some time (500 seconds by default), but eventually will be removed. This is important, the task queue does not preserve a history of executed jobs.

  

  * The `app.task_queue` is going to be the queue where tasks are submitted. Having the queue attached to the application is convenient because anywhere in the application I can use `current_app.task_queue` to access it
  * The `attach()` method of the `Message` class accepts three arguments that define an attachment: the filename, the media type, and the actual file data. The filename is just the name that the recipient will see associated with the attachment, it does not need to be a real file. The media type defines what type of attachment is this, which helps email readers render it appropriately. For example, if you send `image/png` as the media type, an email reader will know that the attachment is an image, in which case it can show it as such. For the blog post data file I'm going to use the JSON format, which uses a `application/json` media type. The third and last argument is a string or byte sequence with the contents of the attachment



(flask run을 위해...)

- Make sure you have Redis running
- In a first terminal window, start one or more instances of the RQ worker. For this you have to use the command `rq worker microblog-tasks`
- In a second terminal window, start the Flask application with `flask run` (remember to set `FLASK_APP` first)



notification관련 route, js 모르겟음...





docker exec ...bin?sh? check



**import time check**(part 21꺼...)