---
 title: 实践-CentOS 7信任自签名证书
 date:  2019/06/11 
 tags: 
   openssl
---

## 环境
|  OS  |  Kernel |  
|:-----|:--------|
|CentOS 7.6|3.10.0|

## 先决条件

```bash
yum install -y openssl ca-certificates 
```

## 配置

```bash
cp ca.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust
```

## 验证
```bash
[root@prometheus-shqs-1 ~]# curl https://registry.leocloud.club
<!doctype html>
<html>

<head>
    <meta charset="utf-8">
    <title>Harbor</title>
    <base href="/">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link rel="icon" type="image/x-icon" href="favicon.ico?v=2">
<link rel="stylesheet" href="styles.c1fdd265f24063370a49.css"></head>

<body>
    <harbor-app>
        <div class="spinner spinner-lg app-loading">
            Loading...
        </div>
    </harbor-app>
<script type="text/javascript" src="runtime.26209474bfa8dc87a77c.js"></script><script type="text/javascript" src="scripts.c04548c4e6d1db502313.js"></script><script type="text/javascript" src="main.186a000e569d8c62efd2.js"></script></body>

</html>[root@prometheus-shqs-1 ~]#
```