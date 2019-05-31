---
  title: 实践-Nginx配置PHP7.2
  date: 2019/05/30
  tags: 
     php
---

## 环境

|OS          |Kernel   |
|:-----------|:--------|
|CentOS 7.6  |3.10.0   |

## 添加第三方PHP安装YUM仓库源

```bash
rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
```

## 安装PHP7.2

```bash
yum install php72w-common php72w-fpm php72w-opcache php72w-gd php72w-mysqlnd php72w-mbstring php72w-pecl-redis php72w-pecl-memcached php72w-devel php72w-cli

```

## 验证

```bash
[root@instance-1 yum.repos.d]# php -v
PHP 7.2.17 (cli) (built: May 13 2019 18:03:04) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.2.0, Copyright (c) 1998-2018 Zend Technologies
    with Zend OPcache v7.2.17, Copyright (c) 1999-2018, by Zend Technologies
```


## 配置PHP-FPM(/etc/php-fpm.d/www.conf)

```bash
[www]
user = nobody
group = nobody
listen =  /var/lib/php-fpm/php-fpm.sock
listen.owner = nobody
listen.group = nobody
listen.mode = 0660
listen.allowed_clients = any
pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35
access.log = /var/log/php-fpm/$pool.access.log
slowlog = /var/log/php-fpm/www-slow.log
catch_workers_output = yes
php_admin_value[error_log] = /var/log/php-fpm/www-error.log
php_admin_flag[log_errors] = on
php_value[session.save_handler] = files
php_value[session.save_path]    = /var/lib/php/session
php_value[soap.wsdl_cache_dir]  = /var/lib/php/wsdlcache
```

## 启动PHP-FPM服务

```bash
systemctl start php-fpm
```

## Nginx安装

```bash
tar xf nginx.tar.gz
./configure --prefix=/opt/app/nginx --with-stream --with-http_ssl_module --with-http_v2_module --with-debug
make 
make install
```

## Nginx PHP虚拟机主机配置

```bash
    server {
        listen  80;
        server_name  a.a.com;
        access_log   logs/$host.access.log  main;
        error_log    logs/$host.error.log;
        root         /opt/app/nginx/html/phptest; //必须保证nginx运行用户能访问

        location ~ \.php$ {
            fastcgi_intercept_errors on;
            include         fastcgi_params;
            fastcgi_index   index.php;
            fastcgi_pass    unix:/var/lib/php-fpm/php-fpm.sock;
            fastcgi_param   SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }
    }
```
## PHP测试页(/opt/app/nginx/html/phptest/info.php)

```bash
<?php
  phpinfo();
?>
```

## 启动Nginx服务

```bash
/opt/app/nginx/sbin/nginx -t
/opt/app/nginx/sbin/nginx 
```
