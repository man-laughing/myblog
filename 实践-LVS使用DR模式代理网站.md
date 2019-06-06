---
  title: 实践-LVS使用DR模式代理网站
  date: 2019/06/06
  tags: 
     lvs
---


## 环境

|  OS  |  Kernel |  IP     | VIP       | Role  | lvs version |
|:-----|:--------|:--------|:----------|:------|:------------|
|CentOS 7.6|3.10.0|10.122.138.240|10.122.138.233|virtual server|ipvsadm-1.27
|CentOS 7.6|3.10.0|10.122.138.241|10.122.138.233|real server|-|
|CentOS 7.6|3.10.0|10.122.138.242|10.122.138.233|real server|-|

## 先决条件 
* 确保virtual server节点已加载ipvs内核模块（virtualServer节点执行）

```bash
[root@localhost ~]# modprobe ip_vs
[root@localhost ~]# lsmod |grep ip_vs
ip_vs_wlc              12519  1 
ip_vs                 145497  3 ip_vs_wlc
nf_conntrack          133095  4 ip_vs,nf_nat,nf_nat_ipv4,nf_conntrack_ipv4
libcrc32c              12644  4 xfs,ip_vs,nf_nat,nf_conntrack

```

* 确保virtual server节点已经安装LVS管理工具ipvsadm包（virtualServer节点执行）

```bash
[root@localhost ~]# yum install -y ipvsadm
[root@localhost ~]# rpm -qa |grep ipvsadm
ipvsadm-1.27-7.el7.x86_64
```

* 配置VIP地址（在所有节点执行）

```bash
[root@localhost ~]# ifconfig lo:0 10.122.138.233 netmask 255.255.255.255
[root@localhost ~]# route add -host 10.122.138.233 dev lo
```

* 配置lo本地回环接口的ARP抑制（realServer节点执行）

```bash
[root@localhost ~]# echo "1" > /proc/sys/net/ipv4/conf/lo/arp_ignore 
[root@localhost ~]# echo "2" > /proc/sys/net/ipv4/conf/lo/arp_announce 
[root@localhost ~]# echo "1" > /proc/sys/net/ipv4/conf/all/arp_announce 
[root@localhost ~]# echo "2" > /proc/sys/net/ipv4/conf/all/arp_ignore

```


* 配置real server的WEB服务正确，这里已经提前配置好了nginx服务（realServer节点执行）

```
[root@localhost ~]# curl http://10.122.138.241
<h1>
 web01 
</h1>

[root@localhost ~]# curl http://10.122.138.242
<h1>
 web02
</h1>

```

## 配置ipvs转发规则

* 配置转发virutalServer:80 -> realServer:80 (virtualServer节点执行)

```bash
[root@localhost ~]# ipvsadm -A -t 10.122.138.233:80 
[root@localhost ~]# ipvsadm -a -t 10.122.138.233:80 -r 10.122.138.241:80 -g -w 1 //-g 表示使用DR模式
[root@localhost ~]# ipvsadm -a -t 10.122.138.233:80 -r 10.122.138.242:80 -g -w 1 //-g 表示使用DR模式

```


## 验证转发是否生效

* 验证（任意客户端节点执行）

```bash
laughingdeMacBook-Air:~ laughing$ curl  http://10.122.138.233
<h1>
 web02
</h1>
laughingdeMacBook-Air:~ laughing$
laughingdeMacBook-Air:~ laughing$ curl  http://10.122.138.233
<h1>
 web01
</h1>
//配置正确，以上说明请求被均衡转发到了realserver上
```