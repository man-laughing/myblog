
---
title: Docker-CE INSTALL
date: 2019/05/29
tags: 
   docker
   kubernetes
---

###卸载老的Docker引擎
```
yum remove docker \
        docker-client \
        docker-client-latest \
        docker-common \
        docker-latest \
        docker-latest-logrotate \
        docker-logrotate \
        docker-engine
```

###安装依赖的包和YUM管理工具
```
 yum install -y yum-utils device-mapper-persistent-data lvm2
```
####设置Docker的YUM仓库
```
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

###启用Docker仓库
```
 yum-config-manager --enable docker-ce-nightly
```

###查看Docker可选版本
```
yum list docker-ce --showduplicates | sort -r
```

####安装Docker-CE最新版本（默认安装最新版）
```
yum install docker-ce docker-ce-cli containerd.io
```

###启动Docker服务
```
systemctl start docker
```
###设置开机启动Docker
```
systemctl enable docker
```


