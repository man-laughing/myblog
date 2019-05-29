---
  title: awk/sed/grep实践
  date: 2019/05/29
  tags: shell三剑客
---
## awk/sed/grep
#### awk
1、awk统计文本行数

```
[root@prometheus-shanghai-2 github.com]# cat abc
leo   25
zhai  24
wang  29
zhai  24
peng  14
[root@prometheus-shanghai-2 github.com]# awk 'END {print NR}'  abc
5
```
2、awk默认空格分割，打印文本最后一列

```
# awk '{print $NF}' FILE
```
3、awk默认空格分割，打印文本开始一列

```
# awk '{print $1}' FILE
```

4、awk在文本每一行前面输出行号

```
# awk '{print NR,$0}' FILE
```
5、awk美化每一行，添加内容

```
[root@prometheus-shanghai-2 github.com]# cat abc
leo   25
zhai  24
wang  29
[root@prometheus-shanghai-2 github.com]#
[root@prometheus-shanghai-2 github.com]# awk '{printf("Name %s Age %d\n",$1,$2)}'  abc
Name leo Age 25
Name zhai Age 24
Name wang Age 29
[root@prometheus-shanghai-2 github.com]#
```
6、awk进行数值比较

```
[root@prometheus-shanghai-2 github.com]# cat abc
leo   25
zhai  24
wang  29
[root@prometheus-shanghai-2 github.com]# awk '$2>=25 {print}'  abc
leo   25
wang  29
```

7、awk在每行前面插入变量，然后打印

```
[root@prometheus-shanghai-2 github.com]# cat abc
leo   25
zhai  24
wang  29
[root@prometheus-shanghai-2 github.com]#
[root@prometheus-shanghai-2 github.com]# awk '{name="hello"}{print name,$0}'  abc
hello leo   25
hello zhai  24
hello wang  29
```

8、awk进行列求和

```
[root@prometheus-shanghai-2 github.com]# cat abc
leo   25
zhai  24
wang  29
[root@prometheus-shanghai-2 github.com]#
[root@prometheus-shanghai-2 github.com]# awk 'BEGIN{sum=0}{sum+=$2}END{print sum}' abc
78
```
9、awk在首行前插入一行内容

```
[root@prometheus-shanghai-2 github.com]# cat abc
leo   25
zhai  24
wang  29
[root@prometheus-shanghai-2 github.com]#
[root@prometheus-shanghai-2 github.com]#
[root@prometheus-shanghai-2 github.com]# awk 'BEGIN{print "name age"} {print}' abc
name age
leo   25
zhai  24
wang  29
```

10、awk实现grep的效果，找到包含某某的行

```
[root@prometheus-shanghai-2 github.com]# cat abc
leo   25
zhai  24
wang  29
zhai  24
peng  14
[root@prometheus-shanghai-2 github.com]#
[root@prometheus-shanghai-2 github.com]# awk '/zhai/ {print}'  abc
zhai  24
zhai  24
```

11、awk打印每一行的列数

```
[root@prometheus-shanghai-2 github.com]# cat abc
leo   25
zhai  24
wang  29
zhai  24
peng  14
[root@prometheus-shanghai-2 github.com]#
[root@prometheus-shanghai-2 github.com]# awk '{print NF}'  abc
2
2
2
2
2
```
12、awk打印文本的第一行内容

```
[root@prometheus-shanghai-2 github.com]# cat abc
leo   25
zhai  24
wang  29
zhai  24
peng  14
[root@prometheus-shanghai-2 github.com]#
[root@prometheus-shanghai-2 github.com]# awk 'NR==1 {print}' abc
leo   25
```

13、awk打印文本的最后一行内容

```
[root@prometheus-shanghai-2 github.com]# cat abc
leo   25
zhai  24
wang  29
zhai  24
peng  14
[root@prometheus-shanghai-2 github.com]#
[root@prometheus-shanghai-2 github.com]# awk 'END {print}'  abc
peng  14
```



#### sed
1、sed打印文本第一行

```
[root@prometheus-shanghai-2 github.com]# cat abc
leo   25
zhai  24
wang  29
zhai  24
peng  14
[root@prometheus-shanghai-2 github.com]# sed -n '1p' abc
leo   25
```

2、sed打印文本第5行

```
[root@prometheus-shanghai-2 github.com]# sed -n '1p' abc
leo   25
[root@prometheus-shanghai-2 github.com]#
[root@prometheus-shanghai-2 github.com]#
[root@prometheus-shanghai-2 github.com]#
[root@prometheus-shanghai-2 github.com]# sed -n '5p' abc
peng  14
```

3、sed打印文本最后一行

```
[root@prometheus-shanghai-2 github.com]# cat abc
leo   25
zhai  24
wang  29
zhai  24
peng  14
[root@prometheus-shanghai-2 github.com]# sed -n '$p'  abc
peng  14
```

4、sed全文替换hello为HELLO

```
[root@prometheus-shanghai-2 github.com]# cat aa
hello
jafljf
hello
[root@prometheus-shanghai-2 github.com]# sed  's/hello/HELLO/g' aa
HELLO
jafljf
HELLO
```

5、sed在某行后追加一行内容（以某某为索引）

```
[root@prometheus-shanghai-2 github.com]# cat abc
leo   25
zhai  24
wang  29
peng  14
[root@prometheus-shanghai-2 github.com]#
[root@prometheus-shanghai-2 github.com]# sed  '/leo/a HELLO 999' abc
leo   25
HELLO 999
zhai  24
wang  29
peng  14
```

6、sed在某行前追加一行内容（以某某为索引）

```
[root@prometheus-shanghai-2 github.com]# cat abc
leo   25
zhai  24
wang  29
peng  14
[root@prometheus-shanghai-2 github.com]# sed  '/leo/i HELLO 999' abc
HELLO 999
leo   25
zhai  24
wang  29
peng  14
```
6、sed在某行处直接修改内容（以某某为索引）

```
[root@prometheus-shanghai-2 github.com]# cat abc
leo   25
zhai  24
wang  29
peng  14
[root@prometheus-shanghai-2 github.com]#
[root@prometheus-shanghai-2 github.com]# sed  '/leo/c HELLO 999' abc
HELLO 999
zhai  24
wang  29
peng  14
```

7、sed在某行后追加一行内容（以行号为索引）

```
[root@prometheus-shanghai-2 github.com]# cat abc
leo   25
zhai  24
wang  29
peng  14
[root@prometheus-shanghai-2 github.com]# sed  '2a hello' abc
leo   25
zhai  24
hello
wang  29
peng  14
[root@prometheus-shanghai-2 github.com]#
```

8、sed在某行前追加一行内容（以行号为索引)

```
[root@prometheus-shanghai-2 github.com]# cat abc
leo   25
zhai  24
wang  29
peng  14
[root@prometheus-shanghai-2 github.com]# sed  '2i hello' abc
leo   25
hello
zhai  24
wang  29
peng  14
```
9、sed在某行直接修改本行内容（以行号为索引)

```
[root@prometheus-shanghai-2 github.com]# cat abc
leo   25
zhai  24
wang  29
peng  14
[root@prometheus-shanghai-2 github.com]#
[root@prometheus-shanghai-2 github.com]# sed  '2c hello' abc
leo   25
hellos
wang  29
peng  14
```



