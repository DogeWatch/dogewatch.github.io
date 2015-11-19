---
layout:     post
title:      "RCTF"
subtitle:   "web writeup"
date:       2015-11-19
author:     "dogewatch"
header-img: "img/home-bg-o.jpg"
tags:
    - ctf
    - web
---

> "here we go."

## 前言

这次RCTF只搞了web题，其他题完全没时间去碰啊。。
虽然这次天枢没能进决赛，只差一名，但是大家都努力了，希望下次能有更好成绩。

---

## web 1

注册登陆进去后发现有上传，随后试验各种后缀名发现只有.jpg可以，顺便一提每次上传要过验证码真是蛋疼。
随便上传个文件后发现会返回文件名和uid(这个uid一开始没发现作用，后来看了tomato大牛的题解才明白是啥用，后面再说)
，随后会在上传的页面回显文件名。于是就在这里猜测是文件名有注入。在文件名后面加单引号成功上传，再次回到上传页面发现无任何文件名回显，因此猜测是二次注入。
随后开始注入，先来一发aaaaaa'+version()+'.jpg，发现返回文件名为5.1。
![img](/img/post/rctf-web1-1.png)
继续，测试select和from发现被过滤了，分别改成selselectect和frfromom可绕过，后来又发现所有的字母都不会显示，于是只能一位一位猜表名、列名和字段。

```
payload：'+(selselectect if((selselectect count(*) frfromom information_schema.tables)>0,2,3))+'
'+(selselectect if (1<2,ascii(mid((selselectect table_name frfromom information_schema.tables limit 29,1),1,1)),2))+'
'+(selselectect if (1<2,ascii(mid((selselectect I_am_flag frfromom hello_flag_is_here limit 0,1),1,1)),2))+'

```

最后得到flag，!!_@m_Th.e_F!lag。

后来看tomato的题解发现，这题在insert的时候格式是('文件名','uid','uid')，拖了一晚上验证码的我看到这真是哭了，
因为做的时候也想过用这种方式注入，但是死活没成功，还是太菜了啊。


---

## web 2

注册的时候发现过滤了空格，注册一个账号aaaaa'"，然后进去发现一堆赵日天，叶良辰神马的然并卵的东西，最后在改密码的时候发现了报错。
![img](/img/post/rctf-web2-1.png)
明显的二次注入，用updatexml报错注入，先猜flag表，
"&&updatexml(0x7e,concat(0x7e,(select(flag)from(flag))),0)#，
告诉我flag is not here，真是哔了狗了。没办法，只有从表名开始猜了，因为报错注入只会返回一行，但是过滤了空格所以不能使用limit，
在这我又get了一个新的姿势，那就是用mysql的regexp正则匹配函数。直接regexp('flag')找到表名，读取内容是只返回前几个字符，没办法，再次用regexp('RCTF')成功读到flag。
RCTF{sql_1njecti0n_is_f4n_6666} 


---

## web 3

注册一个账号，改密码的时候抓包发现可以修改要改密码的用户名，改成admin后登陆进去提示not allow ip，把XFF改成127.0.0.1后成功登陆。
进去发现叫我们猜action，果断猜upload出现一个上传页面，发现没有验证码真是感动哭了。随后fuzz出可用的后缀名php4,php5。
但是仍然提示不是真正的php，根据php的这几种格式，
```
<?php?>
<??>
<script language="php"></script>
asp的<%%>
```
挨个尝试，发现script标签可以于是成功拿到flag。


---

## web 4

一开始没什么头绪，后来放出提示是nosql就猜测是mongodb，使用[$ne]和[$regex]跑出账号密码，ROIS_ADMIN pas5woRd_i5_45e2884c4e5b9df49c747e1d。
登陆进去后，看源码发现上传需要先过useragent和cookie的验证。

```
$Agent = $_SERVER['HTTP_USER_AGENT'];
$backDoor = $_COOKIE['backdoor'];
$msg = json_encode("no privilege");
$iterations = 1000;
$salt = "roisctf";
$alg = "sha1";
$keylen = "20";
if ($Agent == $backDoor || strlen($Agent) != 65) {
    exit($msg);
}
if (substr($Agent,0,23) != "rois_special_user_agent") {
    exit($msg);
}
if (pbkdf2($alg, $Agent, $salt, $iterations, $keylen) != pbkdf2($alg, $backDoor, $salt, $iterations, $keylen)) {
    exit($msg);
}

```

搜了一发php下的pbkdf2函数发现了一个hash_pbkdf2函数，也没多想就当作是一回事了，因为php的==是弱比较，所以想着只要找俩字符串加密后与0e1相等就行了。
于是就开始跑，跑出来后发现上传依然过不了，在这卡了好久。最后无奈去找客服搅基，客服说这个函数不是php的函数，他只是用php表示出来而已，囧。
最后搜到这篇<a hef="https://mathiasbynens.be/notes/pbkdf2-hmac">文章</a>，用里面提供的脚本跑出一对字符串，rois_special_user_agentaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaamipvkd
3-Rfm^Bq%3BZZAcl]mS&eE。
测试能成功上传了。然后下载提供的备份文件，打开发现是一个关于zip解压的php文件。找官网下了原生版本后diff了一下发现了一处不同点。

```

if ($p_header['filename_len'] != 0)  {
      $p_header['filename'] = fread($this->zip_fd, $p_header['filename_len']);
      $preNum = substr_count($p_header['filename'], '../');
      $prefix = str_repeat('../', $preNum);
      $element = explode('.', str_replace($prefix, '', $p_header['filename']));
      $fname = $prefix . md5($element[0]. 'RoisFighting'). '.' .end($element);
      $p_header['filename'] = $fname;
}

```
知道这是解压后对文件进行了重命名，于是尝试计算出文件名后拼接到给出的上传路径去访问发现是404。
多上传几次发现给出的路径都不一样，于是尝试构造一个解压后存放至上一级目录的zip文件，用winhex编辑zip压缩包，在里面文件名前三位改成../，成功上传得到flag。

---

## 后记

这次RCTF只撸了web里的四道题，后面的xss和500分的题完全没时间看了。看题解发现xss不是很难，不知道时间足够能做出来不。个人成绩依然是60多名，从今年的绿盟举办的CTF开始，陆续打了几次都是这个名次，相信积累足够技术和经验后能有进步。

