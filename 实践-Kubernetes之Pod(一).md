---
 title: Kubernetes之Pod基本操作(一)
 date:  2019/07/01
 tags: 
   kubernetes
---

## 环境
|  OS  |  Kernel | k8s version|
|:-----|:--------| :----------|
|CentOS 7.6|3.10.0| 1.12.8|
   

## 通过yaml描述文件操作Pod

* 查看当前k8s集群有哪些pod

```bash
[root@qs-storage-swift-01 pods]# kubectl  get pods --all-namespaces  //--all-namespaces表示查看所有命名空间
NAMESPACE     NAME                                                     READY   STATUS    RESTARTS   AGE
default       busybox01                                                1/1     Running   0          8m15s
default       centos-7474788896-bgg85                                  1/1     Running   1          6d21h
default       messageboard-flask-api-744c4f5479-dv8s8                  1/1     Running   1          32d
default       messageboard-flask-api-744c4f5479-xjl9b                  1/1     Running   0          32d
default       messageboard-redis-master-5d65bf6db4-gfvcr               1/1     Running   1          32d
default       my-nginx-nginx-ingress-controller-64bcf46c6c-ssr2n       1/1     Running   1          32d
default       my-nginx-nginx-ingress-default-backend-f98975fc4-fgqgw   1/1     Running   0          32d
kube-system   coredns-59fc846856-g7jst                                 1/1     Running   1          32d
kube-system   coredns-59fc846856-xgwdc                                 1/1     Running   0          32d
kube-system   tiller-deploy-8448976d-nk8gs                             1/1     Running   0          32d
```

* 创建一个busybox的pod

```bash
[root@qs-storage-swift-01 pods]# cat busybox_pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
   name: busybox01
   namespace: "default"
   labels:
     owner: laughing
     env: test
spec:
   containers:
   - name: busybox
     image: busybox
     imagePullPolicy: IfNotPresent
     command: ["sleep","3600"]
   restartPolicy: "OnFailure"
[root@qs-storage-swift-01 pods]# kubectl create -f busybox_pod.yaml
pod/busybox01 created
[root@qs-storage-swift-01 pods]#
```

* 更新yaml描述文件使之立即生效

```bash
[root@qs-storage-swift-01 pods]# cat busybox_pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
   name: busybox01
   namespace: "default"
   labels:
     owner: laughing
     env: test
     app: sts     //这里增加了一个标签
spec:
   containers:
   - name: busybox
     image: busybox
     imagePullPolicy: IfNotPresent
     command: ["sleep","3600"]
   restartPolicy: "OnFailure"
[root@qs-storage-swift-01 pods]# kubectl apply -f busybox_pod.yaml
Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
pod/busybox01 configured
[root@qs-storage-swift-01 pods]# kubectl get pods -l app=sts -o wide  //通过app这个标签能成功找到pod，说明更新生效了
NAME        READY   STATUS    RESTARTS   AGE     IP             NODE                              NOMINATED NODE
busybox01   1/1     Running   0          5m21s   172.16.32.66   qs-storage-swift-03.localdomain   <none>
[root@qs-storage-swift-01 pods]#
```

* 删除busybox这个pod

```bash
[root@qs-storage-swift-01 pods]# kubectl delete -f busybox_pod.yaml
pod "busybox01" deleted
[root@qs-storage-swift-01 pods]#
//或者以下命令
[root@qs-storage-swift-01 pods]# kubectl  get pods -o wide -l owner=laughing
NAME        READY   STATUS    RESTARTS   AGE   IP               NODE                              NOMINATED NODE
busybox01   1/1     Running   0          21s   172.16.123.206   qs-storage-swift-02.localdomain   <none>
[root@qs-storage-swift-01 pods]#
[root@qs-storage-swift-01 pods]#
[root@qs-storage-swift-01 pods]# kubectl delete pod busybox01  //找到该pod的名字通过kubectl来删除
pod "busybox01" deleted
[root@qs-storage-swift-01 pods]#
```

## 通过kubectl命令操作Pod

* 创建busybox的pod且伴随有两个副本、两个标签

```bash
[root@qs-storage-swift-01 pods]# kubectl run busybox02 --image=busybox --replicas=2 -l owner=xiaobai,env=test -- sleep 86400
kubectl run --generator=deployment/apps.v1beta1 is DEPRECATED and will be removed in a future version. Use kubectl create instead.
deployment.apps/busybox02 created   
[root@qs-storage-swift-01 pods]#
[root@qs-storage-swift-01 pods]# kubectl get deployment -o wide -l env=test,owner=xiaobai     //deployment被创建
NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES    SELECTOR
busybox02   2         2         2            2           3m34s   busybox02    busybox   env=test,owner=xiaobai
[root@qs-storage-swift-01 pods]#
[root@qs-storage-swift-01 pods]# kubectl get replicaset -o wide -l env=test,owner=xiaobai     //replicaset被创建
NAME                  DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES    SELECTOR
busybox02-796677f8b   2         2         2       3m45s   busybox02    busybox   env=test,owner=xiaobai,pod-template-hash=796677f8b
[root@qs-storage-swift-01 pods]#
[root@qs-storage-swift-01 pods]# kubectl  get pods -o wide  -l env=test,owner=xiaobai         //实际的两个pod被创建
NAME                        READY   STATUS    RESTARTS   AGE   IP               NODE                              NOMINATED NODE
busybox02-796677f8b-2hss2   1/1     Running   0          88s   172.16.32.69     qs-storage-swift-03.localdomain   <none>
busybox02-796677f8b-9p8h9   1/1     Running   0          88s   172.16.123.207   qs-storage-swift-02.localdomain   <none>

```

* 删除这个busybox的pod

```bash
[root@qs-storage-swift-01 pods]# kubectl  delete pod busybox02
deployment.apps/busybox02 deleted
```

* 更新pod的配置

```bash
无
```

NOTE: 这里不建议直接更新pod的配置，因为通过kubectl run运行的pod都是会被自动托管到对应的ReplicaSet及Deployment下面，管理器会一直维持初始状态，如果更改了某个pod的配置它就会成为孤立pod，此pod不会归属为任何一个ReplicaSet和Deployment。

