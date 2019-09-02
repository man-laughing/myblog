
---
title: 实践-MySQL联合查询
date: 2019/09/02
tags: 
   mysql
---


## 环境
|OS          |Kernel   | Mysql version| 
|------------|--------:| :------------| 
|CentOS 7.6  |3.10.0   |5.6.45   | 
 

## 先决条件

* 保证你的数据库服务运行正常（安装过程略）
 
 
## Join联合查询

* 内连接(取交集)

```bash
mysql> select * from boy;
+-----------+-------+
| bname     | other |
+-----------+-------+
| 屌丝      | A     |
| 李四      | B     |
| 王五      | C     |
| 高富帅    | D     |
| 郑七      | E     |
+-----------+-------+
5 rows in set (0.00 sec)

mysql> select * from girl;
+-----------+-------+
| gname     | other |
+-----------+-------+
| 空姐      | B     |
| 大s       | C     |
| 阿娇      | D     |
| 张柏芝    | D     |
| 林黛玉    | E     |
| 宝钗      | F     |
+-----------+-------+
6 rows in set (0.00 sec)

mysql>
mysql> select * from boy inner join girl on boy.other = girl.other;   //这是标准使用内连接写法，inner可以省略，mysql默认连接就是内连接
+-----------+-------+-----------+-------+
| bname     | other | gname     | other |
+-----------+-------+-----------+-------+
| 李四      | B     | 空姐      | B     |
| 王五      | C     | 大s       | C     |
| 高富帅    | D     | 阿娇      | D     |
| 高富帅    | D     | 张柏芝    | D     |
| 郑七      | E     | 林黛玉    | E     |
+-----------+-------+-----------+-------+
5 rows in set (0.00 sec)

mysql>
mysql> select * from boy,girl where boy.other = girl.other;   //这是使用where的写法，也正确
+-----------+-------+-----------+-------+ 
| bname     | other | gname     | other |
+-----------+-------+-----------+-------+
| 李四      | B     | 空姐      | B     |
| 王五      | C     | 大s       | C     |
| 高富帅    | D     | 阿娇      | D     |
| 高富帅    | D     | 张柏芝    | D     |
| 郑七      | E     | 林黛玉    | E     |
+-----------+-------+-----------+-------+
5 rows in set (0.00 sec)

mysql>
```

* 左连接

```bash
mysql> select * from boy;
+-----------+-------+
| bname     | other |
+-----------+-------+
| 屌丝      | A     |
| 李四      | B     |
| 王五      | C     |
| 高富帅    | D     |
| 郑七      | E     |
+-----------+-------+
5 rows in set (0.00 sec)

mysql> select * from girl;
+-----------+-------+
| gname     | other |
+-----------+-------+
| 空姐      | B     |
| 大s       | C     |
| 阿娇      | D     |
| 张柏芝    | D     |
| 林黛玉    | E     |
| 宝钗      | F     |
+-----------+-------+
6 rows in set (0.00 sec)

mysql>
mysql> select * from boy as b left join girl as g on b.other = g.other ;
+-----------+-------+-----------+-------+
| bname     | other | gname     | other |
+-----------+-------+-----------+-------+
| 李四      | B     | 空姐      | B     |
| 王五      | C     | 大s       | C     |
| 高富帅    | D     | 阿娇      | D     |
| 高富帅    | D     | 张柏芝    | D     |
| 郑七      | E     | 林黛玉    | E     |
| 屌丝      | A     | NULL      | NULL  |
+-----------+-------+-----------+-------+
6 rows in set (0.00 sec)
```

* 右连接

```bash
mysql> select * from boy;
+-----------+-------+
| bname     | other |
+-----------+-------+
| 屌丝      | A     |
| 李四      | B     |
| 王五      | C     |
| 高富帅    | D     |
| 郑七      | E     |
+-----------+-------+
5 rows in set (0.00 sec)

mysql> select * from girl;
+-----------+-------+
| gname     | other |
+-----------+-------+
| 空姐      | B     |
| 大s       | C     |
| 阿娇      | D     |
| 张柏芝    | D     |
| 林黛玉    | E     |
| 宝钗      | F     |
+-----------+-------+
6 rows in set (0.00 sec)

mysql>
mysql> select * from boy right join girl on boy.other = girl.other;
+-----------+-------+-----------+-------+
| bname     | other | gname     | other |
+-----------+-------+-----------+-------+
| 李四      | B     | 空姐      | B     |
| 王五      | C     | 大s       | C     |
| 高富帅    | D     | 阿娇      | D     |
| 高富帅    | D     | 张柏芝    | D     |
| 郑七      | E     | 林黛玉    | E     |
| NULL      | NULL  | 宝钗      | F     |
+-----------+-------+-----------+-------+
6 rows in set (0.00 sec)

mysql>
```

* 外链接(取并集,又叫全连接)

```bash
mysql> select * from boy;
+-----------+-------+
| bname     | other |
+-----------+-------+
| 屌丝      | A     |
| 李四      | B     |
| 王五      | C     |
| 高富帅    | D     |
| 郑七      | E     |
+-----------+-------+
5 rows in set (0.00 sec)

mysql> select * from girl;
+-----------+-------+
| gname     | other |
+-----------+-------+
| 空姐      | B     |
| 大s       | C     |
| 阿娇      | D     |
| 张柏芝    | D     |
| 林黛玉    | E     |
| 宝钗      | F     |
+-----------+-------+
6 rows in set (0.00 sec)

mysql>
mysql>
mysql> select * from boy full join girl;
+-----------+-------+-----------+-------+
| bname     | other | gname     | other |
+-----------+-------+-----------+-------+
| 屌丝      | A     | 空姐      | B     |
| 李四      | B     | 空姐      | B     |
| 王五      | C     | 空姐      | B     |
| 高富帅    | D     | 空姐      | B     |
| 郑七      | E     | 空姐      | B     |
| 屌丝      | A     | 大s       | C     |
| 李四      | B     | 大s       | C     |
| 王五      | C     | 大s       | C     |
| 高富帅    | D     | 大s       | C     |
| 郑七      | E     | 大s       | C     |
| 屌丝      | A     | 阿娇      | D     |
| 李四      | B     | 阿娇      | D     |
| 王五      | C     | 阿娇      | D     |
| 高富帅    | D     | 阿娇      | D     |
| 郑七      | E     | 阿娇      | D     |
| 屌丝      | A     | 张柏芝    | D     |
| 李四      | B     | 张柏芝    | D     |
| 王五      | C     | 张柏芝    | D     |
| 高富帅    | D     | 张柏芝    | D     |
| 郑七      | E     | 张柏芝    | D     |
| 屌丝      | A     | 林黛玉    | E     |
| 李四      | B     | 林黛玉    | E     |
| 王五      | C     | 林黛玉    | E     |
| 高富帅    | D     | 林黛玉    | E     |
| 郑七      | E     | 林黛玉    | E     |
| 屌丝      | A     | 宝钗      | F     |
| 李四      | B     | 宝钗      | F     |
| 王五      | C     | 宝钗      | F     |
| 高富帅    | D     | 宝钗      | F     |
| 郑七      | E     | 宝钗      | F     |
+-----------+-------+-----------+-------+
30 rows in set (0.00 sec)

mysql>
```

* 自然连接

```bash
mysql> select * from boy;
+-----------+-------+
| bname     | other |
+-----------+-------+
| 屌丝      | A     |
| 李四      | B     |
| 王五      | C     |
| 高富帅    | D     |
| 郑七      | E     |
+-----------+-------+
5 rows in set (0.01 sec)

mysql>
mysql> select * from girl;
+-----------+-------+
| gname     | other |
+-----------+-------+
| 空姐      | B     |
| 大s       | C     |
| 阿娇      | D     |
| 张柏芝    | D     |
| 林黛玉    | E     |
| 宝钗      | F     |
+-----------+-------+
6 rows in set (0.01 sec)

mysql>
mysql> select * from boy natural join girl;
+-------+-----------+-----------+
| other | bname     | gname     |
+-------+-----------+-----------+
| B     | 李四      | 空姐      |
| C     | 王五      | 大s       |
| D     | 高富帅    | 阿娇      |
| D     | 高富帅    | 张柏芝    |
| E     | 郑七      | 林黛玉    |
+-------+-----------+-----------+
5 rows in set (0.00 sec)
```