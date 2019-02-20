#### Part 19 : Deployment on Docker Containers

[https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xix-deployment-on-docker-containers]

* Docker container
  * Container : built on a lightweight virtualization technology that allows an application, along with its dependencies and configuration to run in complete isolation
  * A system configured as a container host can execute many containers, all of them sharing the host's kernel and direct access to the host's hardware. This is in contrast to virtual machines, which have to emulate a complete system, including CPU, disk, other hardware, kernel, etc
  * In spite of having to share the kernel, the level of isolation in a container is pretty high. A container has its own file system, and can be based on an operating system that is different than the one used by the container host
  * https://bcho.tistory.com/805 (참고 사이트...)
  * https://subicura.com/2017/01/19/docker-guide-for-beginners-1.html



* Installing Docker CE
  * 2 editions of Docker
    * free community edition(CE)
    * Subscription based enterprise edition(EE)



* Building a Container Image
  * `container image` : a template that is used to create a container
  * The most basic way to create a container image for your application is to start a container for the base operating system you want to use (Ubuntu, Fedora, etc.), connect to a bash shell process running in it, and then manually install your application ??
  * 더 쉬운 방법 : `docker build` : reads and executes build instructions from a file called Dockerfile
    * `Dockerfile` : installer script of sorts that executes the installation steps to get the application deployed, plus some container specific settings



* Gunicorn
  * https://paphopu.tistory.com/entry/WSGI%EC%97%90-%EB%8C%80%ED%95%9C-%EC%84%A4%EB%AA%85-WSGI%EB%9E%80-%EB%AC%B4%EC%97%87%EC%9D%B8%EA%B0%80
  * `Gunicorn` : Pythion WSGI HTTP server
  * `WSGI` **?????**
  * Note the `exec` that precedes the gunicorn command. In a shell script, `exec` triggers the process running the script to be replaced with the command given, instead of starting it as a new process. This is important, because **Docker associates the life of the container to the first process that runs on it.** In cases like this one, **where the start up process is not the main process of the container(왜지...?)**, you need to make sure that the main process takes the place of that first process to ensure that the container is not terminated early by Docker.
* `ENV` 변수 설정시, docker container를 만들때 static-variable(build-time environment variables)로 만드는 방법 이외에, (run-time environment variables)로 설정하는 방법 : `docker run` command 때 `-e` option 사용



* Using Third-Party 'Containerized' Services
  * `DATABASE_URL` environment variable을 설정하지 않은 상태: app이 SQLite db를 default로 사용, which is supported by a file on disk
  * container를 stop하고 지울 경우, 해당 SQLite file도 없어져 버림...
  * file system in a container is *ephemeral*(수명이 짧다), meaning that it goes away when the container goes away
  * database를 위한 container를 하나더 만들자! 
    * 이제 application container: application code and no data, now can throw it away and replace it with a new one without any problems
    * **`--link` option** : tells Docker to make another container accessible to this one

```
$ docker version

$ docker build -t microblog:latest .

$ docker images

$ docker run --name microblog -d -p 8000:5000 --rm microblog:latest
// -e: (environ variable) option & -v: host_path와 container_path volumn해주는 option

$ docker ps

$ docker stop 021da2e1e0d3
```



* 'Amazon Container Service' (ECS) : gives you the ability to create a cluster of container hosts on which to run your containers
* 'Kubernetes' : container orchestration platform

네트워크공부