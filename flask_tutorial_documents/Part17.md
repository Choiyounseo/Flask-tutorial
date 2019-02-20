#### Part17 : Deployment on Linux

* Traditional Hosting
  * application is installed manually or through a scripted installer on a stock server machine
  *  this application can be deployed on any of the major operating systems, a list which includes a large variety of open-source Linux and BSD distributions, and the commercial OS X and Microsoft Windows
    * Since OS X and Windows are desktop operating systems that are not optimized to work as servers, I'm going to discard those as candidates



* Vagrant VM 사용시

  ```
  $ vagrant ssh
  ```

  * If you are using a Vagrant VM, you can skip this section, since your VM is properly configured to use a non-root account named `ubuntu`, without password automatically by Vagrant