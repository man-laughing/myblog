---
 title: 实践-Kubernetes对外暴露服务
 date:  2019/07/23
 tags: 
   kubernetes
---

## 环境
|  OS  |  Kernel | IP   |role| vesion |
|:-----|:--------|:-----|:---------| :---|
|CentOS 7.6|3.10.0| 10.99.72.4| k8s-master|1.12.8|
|CentOS 7.6|3.10.0| 10.99.72.5| k8s-node|1.12.8|
|CentOS 7.6|3.10.0| 10.99.72.6| k8s-node|1.12.8|

## 暴露服务的方式

* ClusterIP,对内
* NodePort,对外
* LoadBalance,对外(需要开通GCP/AWS等服务)
* Ingress,对外

## 实际操作

##### ClusterIP


* 编辑yaml启动一个nginx集群和暴露service(service类型默认就是ClusterIP方式)

```bash
[root@qs-storage-swift-01 k8s_test]# cat test-nginx-deploy.yaml
---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: mynginx
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: mynginx
    spec:
      containers:
      - name: mynginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: mynginx
spec:
  selector:
      app: mynginx
  ports:
    - name: http
      port: 80
      targetPort: 80
[root@qs-storage-swift-01 k8s_test]#
[root@qs-storage-swift-01 k8s_test]# kubectl create -f test-nginx-deploy.yaml
deployment.apps/mynginx created
service/mynginx created
```

* 查看pod是否运行、service是否正确创建

```bash
[root@qs-storage-swift-01 k8s_test]# kubectl  get pods -o wide|grep mynginx
mynginx-86c55bcbcc-7b7gl                                 1/1     Running   0          80s   172.16.123.220   qs-storage-swift-02.localdomain   <none>
mynginx-86c55bcbcc-bxxzd                                 1/1     Running   0          80s   172.16.32.78     qs-storage-swift-03.localdomain   <none>
[root@qs-storage-swift-01 k8s_test]#
[root@qs-storage-swift-01 k8s_test]#
[root@qs-storage-swift-01 k8s_test]# kubectl  get svc |grep mynginx
mynginx                                  ClusterIP      172.16.250.186   <none>        80/TCP                       106s
[root@qs-storage-swift-01 k8s_test]#
```

* 验证启动的该service是否可以正确被调用（我通过在k8s集群内的一个busyboxx内执行命令验证）

```bash
[root@qs-storage-swift-01 k8s_test]# kubectl exec -it busybox-6f584dc999-q2fwb sh
/ #
/ # wget -SO /dev/null mynginx
Connecting to mynginx (172.16.250.186:80)
  HTTP/1.1 200 OK
  Server: nginx/1.17.1
  Date: Fri, 19 Jul 2019 02:20:54 GMT
  Content-Type: text/html
  Content-Length: 612
  Last-Modified: Tue, 25 Jun 2019 12:19:45 GMT
  Connection: close
  ETag: "5d121161-264"
  Accept-Ranges: bytes

saving to '/dev/null'
null                 100% |******************************************************************************************************************************************************************************************************************************|   612  0:00:00 ETA
'/dev/null' saved
/ #
```


##### NodePort


* 编辑yaml启动一个nginx集群和暴露service(service类型使用NodePort方式)

```bash
[root@qs-storage-swift-01 k8s_test]# cat test-nginx-deploy.yaml
---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: mynginx
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: mynginx
    spec:
      containers:
      - name: mynginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: mynginx
spec:
  type: NodePort        //这里要表明使用NodePort来暴露服务
  selector:
      app: mynginx
  ports:
    - name: http
      port: 80
      targetPort: 80
[root@qs-storage-swift-01 k8s_test]# kubectl  create -f test-nginx-deploy.yaml
deployment.apps/mynginx created
service/mynginx created
```

* 查看pod是否运行、service是否正确创建

```bash
[root@qs-storage-swift-01 k8s_test]# kubectl  create -f test-nginx-deploy.yaml 
[root@qs-storage-swift-01 k8s_test]# 
[root@qs-storage-swift-01 k8s_test]# kubectl  get pods |grep mynginx
mynginx-86c55bcbcc-5lwch                                 1/1     Running   0          5h17m
mynginx-86c55bcbcc-v9cpp                                 1/1     Running   0          5h17m
[root@qs-storage-swift-01 k8s_test]#
[root@qs-storage-swift-01 k8s_test]# kubectl  get svc  |grep mynginx
mynginx                                  NodePort       172.16.250.130   <none>        80:30150/TCP                 5h18m
```

* 验证启动的该service是否可以正确被调用

```bash
[root@qs-storage-swift-01 k8s_test]# curl  http://10.99.72.5:30150  //任意一台k8s-node节点的30150端口都可以被访问
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

##### LoadBalance

*  没有aws/gcp服务，不做测试


##### Ingress

NOTE：请确保你有一个可以工作的Ingress-Controller控制器,我这里使用了Traefik


* 编辑yaml文件，创建一个Deployment、一个Service、一个Ingress

```bash
[root@qs-storage-swift-01 traefik]# cat test.nginx.yaml
---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: nginx-web
  namespace: kube-system
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx-web
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-web
  namespace: kube-system
spec:
  selector:
      app: web
  ports:
    - name: http
      port: 80
      targetPort: 80
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test.com
  namespace: kube-system
spec:
  rules:
  - host: test.com    
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-web  
          servicePort: 80
[root@qs-storage-swift-01 traefik]#
```

* 验证

```bash
[root@qs-storage-swift-01 traefik]# kubectl get services -n kube-system
NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                       AGE
kube-dns                  ClusterIP   172.16.250.250   <none>        53/UDP,53/TCP                 54d
nginx-web                 ClusterIP   172.16.250.195   <none>        80/TCP                        24h
tiller-deploy             ClusterIP   172.16.250.10    <none>        44134/TCP                     54d
traefik-ingress-service   NodePort    172.16.250.247   <none>        80:30377/TCP,9090:30555/TCP   3d21h
[root@qs-storage-swift-01 traefik]# 
[root@qs-storage-swift-01 traefik]# curl   http://test.com/ -x 10.99.72.5:30377
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
[root@qs-storage-swift-01 traefik]#
```