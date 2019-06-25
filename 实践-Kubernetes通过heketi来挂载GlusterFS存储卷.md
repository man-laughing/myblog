---
 title: 实践-Kubernetes通过Heketi来挂载GlusterFS存储卷
 date:  2019/06/25
 tags: 
   kubernetes
---

## 环境
|  OS  |  Kernel | IP  |glusterfs|heketi|role|
|:-----|:--------|:-----|:---------|:--------|:-----|:--|
|CentOS 7.6|3.10.0| 10.122.138.241|6.1|9.0.0|GlusterFS/Heketi|
|CentOS 7.6|3.10.0| 10.122.138.242|6.1|9.0.0|GlusterFS|

## 先决条件

* 你需要准备一个可以工作的Kubernetes集群，我这里使用预先准备好的k8s1.12.8版本
* 你还需要确保k8s集群到glusterfs集群之间网络是畅通的且没有任何安全策略阻挡访问
* 因没有实际机器，我这里实验使用vmware虚拟机来模拟，希望你也是

## GlusterFS安装

NOTE: 以下所有操作需要在两台GlusterFS节点执行

* 配置hosts

```bash
10.122.138.241 gfs01
10.122.138.242 gfs02
```

* 添加glusterfs安装yum源

```bash
[root@gfs01 ~]# yum install centos-release-gluster
```

* 安装glusterfs，此刻最新版是6.1

```bash
[root@gfs01 ~]# yum install glusterfs-server
```

* 查看安装

```bash
[root@gfs01 ~]# rpm -qa |grep gluster
glusterfs-client-xlators-6.1-1.el7.x86_64
glusterfs-fuse-6.1-1.el7.x86_64
centos-release-gluster6-1.0-1.el7.centos.noarch
glusterfs-libs-6.1-1.el7.x86_64
glusterfs-6.1-1.el7.x86_64
glusterfs-cli-6.1-1.el7.x86_64
glusterfs-api-6.1-1.el7.x86_64
glusterfs-server-6.1-1.el7.x86_64
[root@gfs01 ~]# 
```

* 启动glusterfs服务

```bash
[root@gfs01 ~]# systemctl start glusterd 
```

* 初始化glusterfs

```bash
[root@gfs01 ~]# gluster peer probe gfs02  //如果在gfs02执行这里就是probe gfs01，视情况操作
[root@gfs01 ~]# gluster peer status 
Number of Peers: 1

Hostname: gfs02
Uuid: e325d4b9-6a74-4fa7-86ec-7db6bb8bc9ed
State: Peer in Cluster (Connected)

[root@gfs02 ~]# gluster peer status     //在gfs02节点执行查看状态
Number of Peers: 1

Hostname: gfs01
Uuid: c57c583d-7638-4450-98bb-e4371731ff72
State: Peer in Cluster (Connected)
```


## Heketi安装

NOTE: 以下所有操作需要在10.122.138.241节点执行即可

* 下载heketi最新版安装包，此刻是v9.0

```bash
[root@gfs01 ~]# cd /tmp
[root@gfs01 tmp]# wget -S "https://github.com/heketi/heketi/releases/download/v9.0.0/heketi-v9.0.0.linux.amd64.tar.gz" 
[root@gfs01 tmp]# tar xf heketi-v9.0.0.linux.amd64.tar.gz  -C /opt/app
```

* 配置heketi及启动服务

```bash
[root@gfs01 tmp]# cd /opt/app/heketi
[root@gfs01 heketi]# mkdir /var/lib/heketi/
[root@gfs01 heketi]# cat heketi.json
{
  "port": "9090",      //heketi的http server监听的端口，默认是8080
  "use_auth": false,   //不使用验证，表示谁都可以使用该heketi接口，生产环境不建议这么配置
  "jwt": {
    "admin": {
      "key": "My Secret"
    },
    "user": {
      "key": "My Secret"
    }
  },
  "glusterfs": {
    "executor": "ssh",
    "sshexec": {
      "keyfile": "/root/.ssh/id_rsa",   //需要配置SSH互信，请自行配置，即heketi->glusterfs节点，这里是私钥文件
      "user": "root",
      "port": "22",
      "fstab": "/etc/fstab"
    },
    "db": "/var/lib/heketi/heketi.db",  //该目录需要提前创建
    "loglevel" : "debug"
  }
}
[root@gfs01 heketi]# ./heketi --config=/opt/app/heketi/heketi.json &> access.log &   //启动heketi服务并放在后台运行
``` 

* 导入gluster集群拓扑到heketi服务

```bash
[root@gfs01 heketi]# export HEKETI_CLI_SERVER=http://10.122.138.241:9090    //配置heketi服务地址以便使用heketi-cli工具来调用
[root@gfs01 heketi]# cat topology.json 
{
    "clusters":[
        {
            "nodes":[
                {
                    "node": {
                        "hostnames":{
                            "manage":[
                              "10.122.138.241"
                            ],
                            "storage":[
                                "10.122.138.241"       //这里是第一个glusterfs节点的地址
                            ]
                        },
                        "zone":1
                    },
                    "devices":[
                        "/dev/sdb"                     //这里填写vmware添加的虚拟磁盘，一定不要分区或格式化
                    ]
                },
                {
                    "node": {
                        "hostnames":{
                            "manage":[
                              "10.122.138.242"
                            ],
                            "storage":[
                                "10.122.138.242"       //这里是第二个glusterfs节点的地址
                            ]
                        },
                        "zone":1
                    },
                    "devices":[
                        "/dev/sdb"                     //这里填写vmware添加的虚拟磁盘，一定不要分区或格式化
                    ]
                }
            ]
        }
    ]
}
[root@gfs01 heketi]# heketi-cli topology load --json=/opt/app/heketi/topology.json   //导入glusterfs的拓扑结构
```

* 验证heketi是否可以驱动glusterfs

```bash
[root@gfs01 heketi]# heketi-cli  volume list
[root@gfs01 heketi]# heketi-cli  volume create --size=1 --replica=2    //创建复制卷且指定大小为2GB
Name: vol_d3c7844e6670f9f1d8d9e85e5b75b42c
Size: 1
Volume Id: d3c7844e6670f9f1d8d9e85e5b75b42c
Cluster Id: eb60e834247de9a2948113dba5f372d3
Mount: 10.122.138.241:vol_d3c7844e6670f9f1d8d9e85e5b75b42c
Mount Options: backup-volfile-servers=10.122.138.242
Block: false
Free Size: 0
Reserved Size: 0
Block Hosting Restriction: (none)
Block Volumes: []
Durability Type: replicate
Distribute Count: 1
Replica Count: 2
[root@gfs01 heketi]# 
[root@gfs01 heketi]# gluster volume list  //在glusterfs中查看卷，说明heketi-cli操作生效了
vol_d3c7844e6670f9f1d8d9e85e5b75b42c
[root@gfs01 heketi]# 
```

* 挂载验证由heketi创建出来的glusterfs存储卷

```bash
[root@gfs01 ~]# mkdir /test
[root@gfs01 ~]# mount -t glusterfs gfs01:vol_d3c7844e6670f9f1d8d9e85e5b75b42c /test 
[root@gfs01 ~]# mount |grep test 
gfs01:vol_d3c7844e6670f9f1d8d9e85e5b75b42c on /test type fuse.glusterfs (rw,relatime,user_id=0,group_id=0,default_permissions,allow_other,max_read=131072)
[root@gfs01 ~]# cd /test
[root@gfs01 test]# echo "hello,world" > index.html
[root@gfs01 test]# ll
total 1
-rw-r--r-- 1 root root 12 Jun 25 17:35 index.html
[root@gfs01 test]# cat index.html 
hello,world
```

## Kubernetes配置

NOTE: 以下所有操作需要在预先准备的k8s节点执行即可

* 创建存储类

```bash
[root@qs-storage-swift-01 glusterfs]# cat gluster_storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gluster-heketi                               //存储类的名字
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://10.122.138.241:9090"              //这里即是我们刚配置好的heketi的地址，按实际情况填写  
  clusterid: "2ad9814ec008476947d88214b8c4704a"      //这里是heketi的集群ID，因为heketi可以管理多个glusterfs集群，可通过heketi-cli cluster list查看
  restauthenabled: "false"                           //表示不使用验证
  restuser: "admin"
  secretNamespace: "default"
  secretName: "heketi-secret"
  gidMin: "40000"
  gidMax: "50000"
  volumetype: "replicate:2"                          //表示使用复制卷，2个副本
[root@qs-storage-swift-01 glusterfs]#
[root@qs-storage-swift-01 glusterfs]# kubectl create -f gluster_storageclass.yaml
storageclass.storage.k8s.io/gluster-heketi created
[root@qs-storage-swift-01 glusterfs]# kubectl  get storageclass
NAME             PROVISIONER               AGE
gluster-heketi   kubernetes.io/glusterfs   3s
[root@qs-storage-swift-01 glusterfs]#
```

* 创建pvc持久卷声明

```bash
[root@qs-storage-swift-01 glusterfs]# cat gluster_pvc_auto.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc                             //此pvc的名字
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: gluster-heketi      //填写存储类的名字
  resources:
    requests:
      storage: 3Gi                      //实际应该按照需求来，这里测试写了3GB（Gi是规范写法）
[root@qs-storage-swift-01 glusterfs]# kubectl  create -f gluster_pvc_auto.yaml
persistentvolumeclaim/pvc created
[root@qs-storage-swift-01 glusterfs]# kubectl  get pvc
NAME                                         STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
pvc                                          Bound     pvc-56619e9e-972e-11e9-834e-fa163e0d03a0   3Gi        RWX            gluster-heketi   51s   
[root@qs-storage-swift-01 glusterfs]# 
[root@qs-storage-swift-01 glusterfs]# kubectl  get pv   //由于pvc的创建，这里pv也进行了自动创建
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM         STORAGECLASS     REASON   AGE
pvc-56619e9e-972e-11e9-834e-fa163e0d03a0   3Gi        RWX            Delete           Bound    default/pvc   gluster-heketi            63s
[root@gfs01 ~]# 
[root@gfs01 ~]# heketi-cli  volume list    //在heketi节点查看，此刻多了一个卷，此时说明k8s已经成功通过heketi驱动了glusterfs卷
Id:ab5984c9559beff50e4ccde2c9c417ce    Cluster:eb60e834247de9a2948113dba5f372d3    Name:vol_ab5984c9559beff50e4ccde2c9c417ce
Id:d3c7844e6670f9f1d8d9e85e5b75b42c    Cluster:eb60e834247de9a2948113dba5f372d3    Name:vol_d3c7844e6670f9f1d8d9e85e5b75b42c
```

* 实际启动nginx测试容器并挂载该存储卷

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
          claimName: pvc         //这里填写刚才创建的pvc名字

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
[root@qs-storage-swift-01 glusterfs]# kubectl  create -f test_nginx_pod.yaml       //启动nginx容器
```

* 写入文件验证&测试nginx服务

```bash
[root@qs-storage-swift-01 glusterfs]# kubectl  get pods -o wide |grep test1
test1-584f875466-jhlx7                                   1/1     Running   0          2m38s   172.16.123.198   qs-storage-swift-02.localdomain   <none>
[root@qs-storage-swift-01 glusterfs]# kubectl  exec -it test1-584f875466-jhlx7 bash   //进入nginx容器
root@test1-584f875466-jhlx7:/# cd /usr/share/nginx/html/
root@test1-584f875466-jhlx7:/usr/share/nginx/html# rm -rf *
root@test1-584f875466-jhlx7:/usr/share/nginx/html# echo "hello,nginx" > index.html    //写入存储卷一个测试文件
root@test1-584f875466-jhlx7:/usr/share/nginx/html# cat index.html
hello,nginx
root@test1-584f875466-jhlx7:/usr/share/nginx/html# exit
exit
[root@qs-storage-swift-01 glusterfs]# curl -s http://172.16.123.198    //验证正确
hello,nginx
[root@qs-storage-swift-01 glusterfs]#
```

