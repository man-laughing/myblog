
---
title: 实践-CentOS7安装MySQL5.6数据库
date: 2019/08/14
tags: 
   mysql
--


### 环境
|Os          |Kernel   |mysql version|
|------------|--------:|:----|
|CentOS 7.6  |3.10.0   |5.6.45 |


### 先决条件

* 机器可以连通网络


#### 使用YUM仓库来安装mysql-5.6

* 安装官方mysql yum仓库文件

```bash
[root@localhost tmp]# wget -S https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
[root@localhost tmp]# yum localinstall mysql80-community-release-el7-3.noarch.rpm
```

* 查看有哪些可选安装版本

```bash
[root@ localhost tmp]# yum repolist all  |grep mysql
```

* 启用mysql-5.6子仓库，禁用5.7和8.0子仓库

```bash
[root@localhost tmp]# yum-config-manager --disable mysql57-community
[root@localhost tmp]# yum-config-manager --disable mysql80-community
[root@localhost tmp]# yum-config-manager --enable  mysql56-community
[root@localhost tmp]# yum repolist enabled  |grep mysql
```

* 开始安装

```bash
[root@localhost tmp]# yum install mysql-community-server
[root@localhost tmp]# rpm -qa  |grep mysql
```

* 启动服务

```bash
[root@localhost tmp]# systemctl  start mysqld
[root@localhost tmp]# systemctl  status mysqld
```


* 启动后安全设置

```bash
[root@localhost tmp]# mysql_secure_installation //按照提示输入YES即可，主要包含设置root密码、删除匿名用户、禁止root远程登录、删除测试库、刷新权限操作
```


#### 使用二进制包来安装mysql-5.6

* 下载mysql二进制安装包

```bash
[root@localhost tmp]# cd /usr/local/src
[root@localhost src]# wget -S https://dev.mysql.com/get/Downloads/MySQL-5.6/mysql-5.6.45-linux-glibc2.12-x86_64.tar.gz
[root@localhost src]# mkdir /usr/local/mysql-5.6.45/
[root@localhost src]# tar xf mysql-5.6.45-linux-glibc2.12-x86_64.tar.gz -C /usr/local/mysql-5.6.45/
```

* 创建mysql运行用户

```bash
[root@localhost src]# useradd -M -s /sbin/nologin mysql     //没有家目录且禁止该用户登录系统
[root@localhost src]# chown -R mysql:mysql  /usr/local/mysql-5.6.45/
```

* 创建mysqld启动服务

```bash
[root@localhost mysql-5.6.45]# cp /usr/local/mysql-5.6.45/support-files/mysql.server /etc/init.d/mysqld
vim /etc/init.d/mysqld
basedir=/usr/local/mysql-5.6.45/
datadir=/usr/local/mysql-5.6.45/data
[root@localhost mysql-5.6.45]# chmod +x  /etc/init.d/mysqld
```

* 初始化mysql服务

```bash
[root@localhost mysql-5.6.45]#  cd /usr/local/mysql-5.6.45/scripts/
[root@localhost scripts]# ./mysql_install_db  --user=mysql --basedir=/usr/local/mysql-5.6.45/ --datadir=/usr/local/mysql-5.6.45/data/
```


*  启动mysqld服务

```bash
[root@localhost mysql-5.6.45]# /etc/init.d/mysqld  start
[root@localhost mysql-5.6.45]# /etc/init.d/mysqld  status
```

* 启动后安全设置

```bash
[root@localhost mysql-5.6.4]# ./mysql_secure_installation //按照提示输入YES即可，主要包含设置root密码、删除匿名用户、禁止root远程登录、删除测试库、刷新权限操作
```



