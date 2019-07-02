---
 title: 实践-制作第一个Docker镜像
 date:  2019/07/01
 tags: 
   docker
---

## 环境
|  OS  |  Kernel | Docker|
|:-----|:--------| :----------|
|CentOS 7.6|3.10.0| 18.06.1-ce|
   
   
   
## 先决条件

*  你需要有一个Docker环境，请自行安装
   

## 制作镜像

* 我们这里制作一个基于flask项目的容器镜像，查看Dockerfile文件

```bash
[root@qs-storage-swift-01 flask_test]# ls -l
total 8
-rw-r--r-- 1 root root 259 Jul  2 16:48 app.py
-rw-r--r-- 1 root root 222 Jul  2 17:31 Dockerfile
[root@qs-storage-swift-01 flask_test]# cat app.py
#!/usr/bin/python
#coding:utf8

from flask import Flask
app = Flask(__name__)

@app.route("/")
def index():
    return "<h1>hello,wold</h1>"

@app.route("/check")
def check():
    return "ok"

if __name__ == "__main__":
    app.run(host="0.0.0.0",debug=True)
[root@qs-storage-swift-01 flask_test]# 
[root@qs-storage-swift-01 flask_test]# cat Dockerfile
FROM frolvlad/alpine-python2
MAINTAINER zhanglei_zl@ehousechina.com

WORKDIR /opt/app/flask_test
RUN     ["pip","install","flask","flask_redis"]
ADD     .  /opt/app/flask_test/
EXPOSE  5000
ENTRYPOINT ["python", "app.py"]
[root@qs-storage-swift-01 flask_test]#
```

* 制作

```bash
[root@qs-storage-swift-01 flask_test]# docker build -t flasktest:v1.0 .
Sending build context to Docker daemon  3.072kB
Step 1/7 : FROM frolvlad/alpine-python2
 ---> a34ce29854ce
Step 2/7 : MAINTAINER 305835227@ehousechina.com
 ---> Running in c45903c64063
Removing intermediate container c45903c64063
 ---> 575c94c217f5
Step 3/7 : WORKDIR /opt/app/flask_test
 ---> Running in 5b20432228ff
Removing intermediate container 5b20432228ff
 ---> 2d78cbc0b666
Step 4/7 : RUN     ["pip","install","flask","flask_redis"]
 ---> Running in d183718a5994
DEPRECATION: Python 2.7 will reach the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 won't be maintained after that date. A future version of pip will drop support for Python 2.7.
Collecting flask
  Downloading https://files.pythonhosted.org/packages/9a/74/670ae9737d14114753b8c8fdf2e8bd212a05d3b361ab15b44937dfd40985/Flask-1.0.3-py2.py3-none-any.whl (92kB)
Collecting flask_redis
  Downloading https://files.pythonhosted.org/packages/9d/9c/cead8fff1c8da2bd31a83ec476c3364812ee74f3c7c3445d070555f681d1/flask_redis-0.4.0-py2.py3-none-any.whl
Collecting itsdangerous>=0.24 (from flask)
  Downloading https://files.pythonhosted.org/packages/76/ae/44b03b253d6fade317f32c24d100b3b35c2239807046a4c953c7b89fa49e/itsdangerous-1.1.0-py2.py3-none-any.whl
Collecting Jinja2>=2.10 (from flask)
  Downloading https://files.pythonhosted.org/packages/1d/e7/fd8b501e7a6dfe492a433deb7b9d833d39ca74916fa8bc63dd1a4947a671/Jinja2-2.10.1-py2.py3-none-any.whl (124kB)
Collecting Werkzeug>=0.14 (from flask)
  Downloading https://files.pythonhosted.org/packages/9f/57/92a497e38161ce40606c27a86759c6b92dd34fcdb33f64171ec559257c02/Werkzeug-0.15.4-py2.py3-none-any.whl (327kB)
Collecting click>=5.1 (from flask)
  Downloading https://files.pythonhosted.org/packages/fa/37/45185cb5abbc30d7257104c434fe0b07e5a195a6847506c074527aa599ec/Click-7.0-py2.py3-none-any.whl (81kB)
Collecting redis>=2.7.6 (from flask_redis)
  Downloading https://files.pythonhosted.org/packages/ac/a7/cff10cc5f1180834a3ed564d148fb4329c989cbb1f2e196fc9a10fa07072/redis-3.2.1-py2.py3-none-any.whl (65kB)
Collecting MarkupSafe>=0.23 (from Jinja2>=2.10->flask)
  Downloading https://files.pythonhosted.org/packages/b9/2e/64db92e53b86efccfaea71321f597fa2e1b2bd3853d8ce658568f7a13094/MarkupSafe-1.1.1.tar.gz
Installing collected packages: itsdangerous, MarkupSafe, Jinja2, Werkzeug, click, flask, redis, flask-redis
  Running setup.py install for MarkupSafe: started
    Running setup.py install for MarkupSafe: finished with status 'done'
Successfully installed Jinja2-2.10.1 MarkupSafe-1.1.1 Werkzeug-0.15.4 click-7.0 flask-1.0.3 flask-redis-0.4.0 itsdangerous-1.1.0 redis-3.2.1
WARNING: You are using pip version 19.1, however version 19.1.1 is available.
You should consider upgrading via the 'pip install --upgrade pip' command.
Removing intermediate container d183718a5994
 ---> bc73f0f575d0
Step 5/7 : ADD     .  /opt/app/flask_test/
 ---> 834f59752007
Step 6/7 : EXPOSE  5000
 ---> Running in 8d00f4d9dad4
Removing intermediate container 8d00f4d9dad4
 ---> d883a6092c45
Step 7/7 : ENTRYPOINT ["python", "app.py"]
 ---> Running in 4ba18efddb71
Removing intermediate container 4ba18efddb71
 ---> 42eb5593c837
Successfully built 42eb5593c837
Successfully tagged flasktest:v1.0
```

* 查看

```bash
[root@qs-storage-swift-01 flask_test]# docker images |grep test
flasktest                                                        v1.0                       42eb5593c837        About a minute ago   58.9MB
```


## Docker测试

* 基于这个镜像运行一个容器试试看

```bash
[root@qs-storage-swift-01 flask_test]#
[root@qs-storage-swift-01 flask_test]# docker run -it -d --name flaskTest01 flasktest:v1.0
a5bebc04194562542d3ece01e778c0c8ec43a83387eae5a86a9d8187b7487c3b
[root@qs-storage-swift-01 flask_test]# docker ps |grep flask
a5bebc041945        flasktest:v1.0                           "python app.py"          6 seconds ago       Up 5 seconds        5000/tcp            flaskTest01
[root@qs-storage-swift-01 flask_test]#
```


