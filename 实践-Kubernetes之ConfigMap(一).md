---
 title: 实践-Kubernetes之ConfigMap(一)
 date:  2019/06/27
 tags: 
   kubernetes
---

## 环境
|  OS  |  Kernel | k8s version|
|:-----|:--------| :----------|
|CentOS 7.6|3.10.0| 1.12.8|
   

## 先决条件

* 你需要准备一个可以工作的Kubernetes集群，我这里使用预先准备好的k8s1.12.8版本

## 创建ConfigMap

创建configmap有以下三种方式：

1. 从文件或者目录中创建
2. 从命令行直接指定字符串创建
3. 从yaml文件中创建


#### 从文件中创建configmap，默认key就是文件名称，value就是文件内容

```bash
[root@qs-storage-swift-01 k8s_test]# cat configmap/test1.txt
test1
hello
[root@qs-storage-swift-01 k8s_test]# kubectl create configmap cm01 --from-file=configmap/test1.txt    //创建configmap,from-file可以指定写多次表示指多个文件
configmap/cm01 created
[root@qs-storage-swift-01 k8s_test]# kubectl describe configmap cm01
Name:         cm01
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
test1.txt:    //这个表示key
----
test1         //这里表示key的value
hello         //这里表示key的value

Events:  <none>
[root@qs-storage-swift-01 k8s_test]#
```
#### 从文件中创建configmap，并且指定key的名称（如果你不想使用默认文件名作为key的话）

```bash
[root@qs-storage-swift-01 k8s_test]# cat configmap/test2.txt
test2
hello
[root@qs-storage-swift-01 k8s_test]# kubectl create configmap cm02 --from-file=test2=configmap/test2.txt  //创建configmap指定key的名称
configmap/cm02 created
[root@qs-storage-swift-01 k8s_test]# kubectl describe configmap cm02
Name:         cm02
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
test2:           //这个表示key，与创建的时候指定key相对应
----
test2            //这里表示key的value
hello            //这里表示key的value

Events:  <none>
[root@qs-storage-swift-01 k8s_test]#
```

#### 从文件中创建环境变量configmap

```bash
[root@qs-storage-swift-01 k8s_test]# cat configmap/env.txt
name=laughing
age=18
[root@qs-storage-swift-01 k8s_test]# kubectl create configmap cm03 --from-env-file=configmap/env.txt
configmap/cm03 created
[root@qs-storage-swift-01 k8s_test]# kubectl describe configmap cm03
Name:         cm03
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
age:
----
18
name:
----
laughing
Events:  <none>
[root@qs-storage-swift-01 k8s_test]#
```
#### 从目录中创建configmap

```bash
[root@qs-storage-swift-01 k8s_test]# ll configmap/
total 12
-rw-r--r-- 1 root root 21 Jun 27 10:28 env.txt
-rw-r--r-- 1 root root 12 Jun 27 15:49 test1.txt
-rw-r--r-- 1 root root 12 Jun 27 15:50 test2.txt
[root@qs-storage-swift-01 k8s_test]# cat configmap/*
name=laughing
age=18
test1
hello
test2
hello
[root@qs-storage-swift-01 k8s_test]# kubectl create configmap cm04 --from-file=configmap/
configmap/cm04 created
You have mail in /var/spool/mail/root
[root@qs-storage-swift-01 k8s_test]# kubectl  describe configmap cm04
Name:         cm04
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
env.txt:
----
name=laughing
age=18

test1.txt:
----
test1
hello

test2.txt:
----
test2
hello

Events:  <none>
[root@qs-storage-swift-01 k8s_test]#
```
#### 从命令行指定字符串创建configmap

```bash
[root@qs-storage-swift-01 k8s_test]# kubectl  create configmap cm05 --from-literal=k1=hello --from-literal=k2=world
configmap/cm05 created
You have mail in /var/spool/mail/root
[root@qs-storage-swift-01 k8s_test]# kubectl  describe configmap cm05
Name:         cm05
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
k1:      //这个表示key-k1
----
hello    //这个表示key-k1的值
k2:      //这个表示key-k2
----
world    //这个表示key-k2的值
Events:  <none>
```


#### 从yaml文件中创建configmap,通过json字符串的方式

```bash
[root@qs-storage-swift-01 k8s_test]# cat cm06.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
   name: cm06
   namespace: "default"
   labels:
      env: "test"
      app: "zl-app"
data:
   abc.json: |
     {
       "name":"lauging",
       "age:18
     }
[root@qs-storage-swift-01 k8s_test]# kubectl create -f cm06.yaml
configmap/cm06 created
[root@qs-storage-swift-01 k8s_test]# kubectl  describe configmap cm06
Name:         cm06
Namespace:    default
Labels:       app=zl-app
              env=test
Annotations:  <none>

Data
====
abc.json:
----
{
  "name":"lauging",
  "age:18
}

Events:  <none>
```


#### 从yaml文件中创建configmap，通过普通字符串的方式

```bash
[root@qs-storage-swift-01 k8s_test]# cat cm07.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
   name: cm07
   namespace: "default"
   labels:
      env: "test"
      app: "zl-app"
data:
   database: mongodb
   database_uri: mongodb://localhost:27017
[root@qs-storage-swift-01 k8s_test]# kubectl create -f cm07.yaml
configmap/cm07 created
[root@qs-storage-swift-01 k8s_test]# kubectl describe configmap cm07
Name:         cm07
Namespace:    default
Labels:       app=zl-app
              env=test
Annotations:  <none>

Data
====
database:
----
mongodb
database_uri:
----
mongodb://localhost:27017
Events:  <none>

```


## Pod应用ConfigMap

Pod使用configmap通常有以下三种方式：

1. 使用configmap设置为pod内的环境变量
2. 使用configmap作为pod启动参数
3. 使用configmap作为挂载对象

#### congfig作为环境变量供pod容器使用

```bash
[root@qs-storage-swift-01 k8s_test]# kubectl  describe configmap t1
Name:         t1
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
age:
----
20
name:
----
laughing
Events:  <none>
[root@qs-storage-swift-01 k8s_test]#
[root@qs-storage-swift-01 k8s_test]# cat busybox_pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
   name: busyboxtest
   namespace: "default"
   labels:
     env: zltest
spec:
   containers:
   - name: busybox
     image: busybox
     imagePullPolicy: IfNotPresent
     command: ["sleep","3600"]
     env:
      - name: name
        valueFrom:
           configMapKeyRef:
              name: t1
              key: name
      - name: age
        valueFrom:
           configMapKeyRef:
              name: t1
              key: age
   restartPolicy: "OnFailure"
[root@qs-storage-swift-01 k8s_test]# kubectl create -f busybox_pod.yaml
pod/busyboxtest created
[root@qs-storage-swift-01 k8s_test]#
[root@qs-storage-swift-01 k8s_test]# kubectl  exec -it busyboxtest env |egrep "(name|age)"   //可以看到生效了
name=laughing
age=20
```


#### congfig作为pod容器启动参数（需要pod事先挂载configmap为容器内的环境变量）
```bash

[root@qs-storage-swift-01 k8s_test]# cat busybox_pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
   name: busyboxtest
   namespace: "default"
   labels:
     env: zltest
spec:
   containers:
   - name: busybox
     image: busybox
     imagePullPolicy: IfNotPresent
     command: [ "/bin/sh", "-c", "echo hello $(name)" ]
     env:
      - name: name
        valueFrom:
           configMapKeyRef:
              name: t1
              key: name
   restartPolicy: "OnFailure"
[root@qs-storage-swift-01 k8s_test]# kubectl create -f busybox_pod.yaml  
[root@qs-storage-swift-01 k8s_test]# kubectl logs  busyboxtest   //可以看到pod的日志输出是这个，表示没问题
hello laughing
```


#### config作为卷被挂载到pod容器内部

```bash
[root@qs-storage-swift-01 k8s_test]# kubectl  describe configmap cm08
Name:         cm08
Namespace:    default
Labels:       app=zl-app
              env=test
Annotations:  <none>

Data
====
database:
----
mongodb
database_uri:
----
mongodb://localhost:27017
ttt:
----
hello
Events:  <none>
[root@qs-storage-swift-01 k8s_test]#
[root@qs-storage-swift-01 k8s_test]#
[root@qs-storage-swift-01 k8s_test]# cat busybox_pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
   name: busyboxtest
   namespace: "default"
   labels:
     env: zltest
spec:
   containers:
   - name: busybox
     image: busybox
     imagePullPolicy: IfNotPresent
     command: ["sleep","3600"]
     volumeMounts:
       - name: test-volume
         mountPath: /mnt
   volumes:
   - name: test-volume
     configMap:
         name: cm08
   restartPolicy: "OnFailure"
[root@qs-storage-swift-01 k8s_test]# kubectl create -f busybox_pod.yaml
pod/busyboxtest created
[root@qs-storage-swift-01 k8s_test]# kubectl  exec -it busyboxtest sh   //以下过程说明configmap已经被成功挂载
~ #
~ # cd /mnt
/mnt #
/mnt # ls -l
total 0
lrwxrwxrwx    1 root     root            15 Jun 27 10:47 database -> ..data/database
lrwxrwxrwx    1 root     root            19 Jun 27 10:47 database_uri -> ..data/database_uri
lrwxrwxrwx    1 root     root            10 Jun 27 10:47 ttt -> ..data/ttt
/mnt # cat database
mongodb/mnt #
/mnt # cat database_uri
mongodb://localhost:27017/mnt #
/mnt # cat ttt
hello/mnt #
/mnt #
```