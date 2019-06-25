---
 title: 实践-Kubernetes使用GlusterFS存储卷
 date:  2019/06/25
 tags: 
   kubernetes
---

## 环境
|  OS  |  Kernel |kubernetes|glusterfs|  
|:-----|:--------|:---------|:--------|
|CentOS 7.6|3.10.0| 1.12.8|6.1|


## 先决条件
* 准备好一个正常工作的GlusterFS集群和复制卷（这里已经提前准备好)

```bash
[root@gfs01 ~]# gluster volume list
vol_ea651725875294f309bf5b675b1647af
[root@gfs01 ~]# gluster volume info vol_ea651725875294f309bf5b675b1647af  //一会就挂载此复制卷
 
Volume Name: vol_ea651725875294f309bf5b675b1647af
Type: Replicate
Volume ID: d3dee37e-476b-4f30-b648-f4eef72f0eb7
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 2 = 2
Transport-type: tcp
Bricks:
Brick1: 10.122.138.242:/var/lib/heketi/mounts/vg_af0c602307a73142002d4d49f15f67de/brick_d7f597422c0821612331f9d1f5a80cf6/brick
Brick2: 10.122.138.241:/var/lib/heketi/mounts/vg_7eda4fe745458b81ddad8518fb350e76/brick_549811a1c9353d4d90ec88a194f16be4/brick
Options Reconfigured:
user.heketi.id: ea651725875294f309bf5b675b1647af
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
```

## 配置kubernetes


* 创建endpoints

NOTE: 这里请注意GlusterFS集群正常工作以及kubernetes到对应端口的连通性

```bash
[root@qs-storage-swift-01 glusterfs]# cat gluster_endpoints.json
{
  "kind": "Endpoints",
  "apiVersion": "v1",
  "metadata": {
    "name": "glusterfs-cluster"
  },
  "subsets": [
    {
      "addresses": [
        {
          "ip": "10.122.138.241"
        }
      ],
      "ports": [
        {
          "port": 24007
        }
      ]
    },
    {
      "addresses": [
        {
          "ip": "10.122.138.242"
        }
      ],
      "ports": [
        {
          "port": 24007
        }
      ]
    }
  ]
}

[root@qs-storage-swift-01 glusterfs]# kubectl create -f gluster_endpoints.json
```


* 创建pv

```bash
[root@qs-storage-swift-01 glusterfs]# cat gluster_pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  glusterfs:
    endpoints: "glusterfs-cluster"
    path: "vol_ea651725875294f309bf5b675b1647af"  //这里即是预先配置好的GlusterVolume
    readOnly: false
[root@qs-storage-swift-01 glusterfs]# kubectl  create -f gluster_pv.yaml
```

* 创建pvc

```bash
[root@qs-storage-swift-01 glusterfs]# cat gluster_pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
[root@qs-storage-swift-01 glusterfs]# kubectl create -f gluster_pvc.yaml
```


* 创建测试的ngix服务

```bash
[root@qs-storage-swift-01 glusterfs]# cat test_nginx_pod.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: test1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: test1
    spec:
      containers:
      - name: test1
        image: nginx
        volumeMounts:
        - name: gs
          mountPath: /usr/share/nginx/html
      volumes:
      - name: gs
        persistentVolumeClaim:
          claimName: pvc   //指定我们创建的pvc名称

---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: test1
  name: test1
  namespace: default
spec:
  selector:
    app: test1
  ports:
    - port: 80
  type: NodePort
[root@qs-storage-swift-01 glusterfs]# kubectl create -f test_nginx_pod.yaml
```

## 验证卷

*  往卷中写入文件并测试nginx服务

```bash
[root@qs-storage-swift-01 glusterfs]# kubectl  exec -it test1-584f875466-dgs9d bash  //进入容器
root@test1-584f875466-dgs9d:/# cd /usr/share/nginx/html/
root@test1-584f875466-dgs9d:/# rm -rf *
root@test1-584f875466-dgs9d:/usr/share/nginx/html# ls -l
total 0
root@test1-584f875466-dgs9d:/usr/share/nginx/html#
root@test1-584f875466-dgs9d:/usr/share/nginx/html# echo "hello,world" > test.html
root@test1-584f875466-dgs9d:/usr/share/nginx/html# ls -l
total 1
-rw-r--r-- 1 root root 12 Jun 25 02:20 test.html
root@test1-584f875466-dgs9d:/usr/share/nginx/html#
root@test1-584f875466-dgs9d:/usr/share/nginx/html# exit       //退出容器
[root@qs-storage-swift-01 glusterfs]# curl http://172.16.123.196/test.html
hello,world  //可以看到我们刚才写入的test.html文件生效了
```

* 关闭这个nginx服务看看卷中的数据是否还在

```bash
[root@qs-storage-swift-01 glusterfs]# kubectl  delete -f test_nginx_pod.yaml  //关闭nginx服务

[root@gfs01 ~]# gluster volume list    //在glusterfs中查看卷
vol_ea651725875294f309bf5b675b1647af
[root@gfs01 ~]# 
[root@gfs01 ~]# mkdir /test            //创建挂载点
[root@gfs01 ~]# mount -t glusterfs 10.122.138.241:/vol_ea651725875294f309bf5b675b1647af /test   //挂载卷
[root@gfs01 ~]# ls -l /test
total 1
-rw-r--r-- 1 root root 12 Jun 25 10:20 test.html
[root@gfs01 ~]# cat /test/test.html 
hello,world     //文件还在，持久性没有问题
```

