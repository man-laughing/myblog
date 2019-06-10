---
  title: 实践-LVS+KeepAlived高可用代理往网站
  date: 2019/06/10
  tags: 
     lvs
     keepalived
---


## 环境

|  OS  |  Kernel |  IP     | VIP       | Role  | Version |
|:-----|:--------|:--------|:----------|:------|:------------|
|CentOS 7.6|3.10.0|10.122.138.240|10.122.138.233|LVS+KeepAlived|lvs1.27/keepalived1.3.5|
|CentOS 7.6|3.10.0|10.122.138.250|10.122.138.233|LVS+KeepAlived|-|
|CentOS 7.6|3.10.0|10.122.138.241|10.122.138.233|RealServer |-|
|CentOS 7.6|3.10.0|10.122.138.242|10.122.138.233|RealServer |-|

## 先决条件 
* 确保负载均衡节点已加载ipvs内核模块（lvs&keepalived所有节点执行）

```bash
[root@localhost ~]# modprobe ip_vs
[root@localhost ~]# lsmod |grep ip_vs
ip_vs_wlc              12519  1 
ip_vs                 145497  3 ip_vs_wlc
nf_conntrack          133095  4 ip_vs,nf_nat,nf_nat_ipv4,nf_conntrack_ipv4
libcrc32c              12644  4 xfs,ip_vs,nf_nat,nf_conntrack

```

* 确保负载均衡节点已经安装ipvsadm包和keepalived包（lvs&keepalived所有节点执行）

```bash
[root@localhost ~]# yum install -y ipvsadm keepalived
```

## 配置RealSerer

* 配置VIP地址且添加路由（realServer所有节点执行）

```bash
[root@localhost ~]# ifconfig lo:0 10.122.138.233 netmask 255.255.255.255
[root@localhost ~]# route add -host 10.122.138.233 dev lo
```

* 配置lo本地回环接口的ARP抑制（realServer所有节点执行）

```bash
[root@localhost ~]# echo "1" > /proc/sys/net/ipv4/conf/lo/arp_ignore 
[root@localhost ~]# echo "2" > /proc/sys/net/ipv4/conf/lo/arp_announce 
[root@localhost ~]# echo "1" > /proc/sys/net/ipv4/conf/all/arp_announce 
[root@localhost ~]# echo "2" > /proc/sys/net/ipv4/conf/all/arp_ignore

```

* 配置real server的web服务正确，这里已经提前配置好了nginx服务（realServer所有节点执行）

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

## 配置KeepAlived
* 配置主节点keepalived.conf（lvs&keepalived主节点执行）

```bash
 ! Configuration File for keepalived
 
 global_defs {
  router_id LVS_DEVEL
 }
 
 vrrp_instance VI_1 {
    state MASTER           //表示自己状态为MASTER   
    interface ens33        //表示VIP绑定在此接口上 
    virtual_router_id 51     
    priority 100           //这里声明优先级
    advert_int 1             
    authentication {         
        auth_type PASS       
        auth_pass 1111      
   }
    virtual_ipaddress {      
       10.122.138.233      //虚拟的VIP
    }
 }

   virtual_server 10.122.138.233 80 {
     delay_loop 6       
     lb_algo rr        
     lb_kind DR        
     nat_mask 255.255.255.255
     persistence_timeout 0
     protocol TCP            

   real_server 10.122.138.241 80 {      
        weight 1                       
        TCP_CHECK {   
            connect_timeout 3       
            nb_get_retry 3
            delay_before_retry 3
            connect_port 80
        }
   }
    real_server 10.122.138.242 80 {
       weight 1
       TCP_CHECK {
           connect_timeout 3
           nb_get_retry 3
           delay_before_retry 3
           connect_port 80
      }
   }
}
```

* 配置从节点keepalived.conf（lvs&keepalived从节点执行）

```bash
 ! Configuration File for keepalived
 
 global_defs {
  router_id LVS_DEVEL
 }
 
 vrrp_instance VI_1 {
    state BACKUP         //表示自己状态为BACKUP   
    interface ens33      //表示VIP绑定在此接口上 
    virtual_router_id 51     
    priority 76          //这里声明优先级，可以看到比MASTER的要低
    advert_int 1             
    authentication {         
        auth_type PASS       
        auth_pass 1111      
   }
    virtual_ipaddress {      
       10.122.138.233    //虚拟的VIP
    }
 }

   virtual_server 10.122.138.233 80 {
     delay_loop 6       
     lb_algo rr        
     lb_kind DR        
     nat_mask 255.255.255.255
     persistence_timeout 0
     protocol TCP            

   real_server 10.122.138.241 80 {      
        weight 1                       
        TCP_CHECK {   
            connect_timeout 3       
            nb_get_retry 3
            delay_before_retry 3
            connect_port 80
        }
   }
    real_server 10.122.138.242 80 {
       weight 1
       TCP_CHECK {
           connect_timeout 3
           nb_get_retry 3
           delay_before_retry 3
           connect_port 80
      }
   }
}
```

* 启动keepalived服务（lvs&keepalived所有节点执行）

```bash
systemctl start keepalived
```





## 验证

* 查看LVS规则(任意lvs&keepalived节点执行)

```bash
[root@localhost ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.122.138.233:80 rr
  -> 10.122.138.241:80            Route   1      0          0         
  -> 10.122.138.242:80            Route   1      0          0         
```

* 验证负载均衡（任意客户端节点执行）

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


* 验证自动故障转移（lvs&keepalived主节点执行，人工关闭网卡模拟故障）

```bash
ifdown ens33
```

* 验证VIP是否漂移到了从节点（lvs&keepalived从节点执行）

```bash
[root@localhost ~]# ip  a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:50:56:3c:76:9f brd ff:ff:ff:ff:ff:ff
    inet 10.122.138.250/24 brd 10.122.138.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet 10.122.138.233/32 scope global ens33    //可以看到VIP顺利漂移过来了
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:fe3c:769f/64 scope link 
       valid_lft forever preferred_lft forever

```

