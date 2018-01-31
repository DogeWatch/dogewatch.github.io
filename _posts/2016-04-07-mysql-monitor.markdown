---
layout:     post
title:      "mysql过程监控"
subtitle:   "php"
date:       2016-04-07
author:     "dogewatch"
header-img: "img/contact-bg.jpg"
tags:
    - web
---

> "here喂狗"

## 前言

最近审代码在windows下用的seay的源码审计工具，发现里面的mysql监控小插件挺好用的。
于是就想写一个类似的小东西在mac上也能下断点监听mysql执行了哪些语句。

## 正文

实现mysql监控的主要原理是开启general_log来记录历史执行语句，可以记录到数据库也可以记录到文件。
这里我们通过以下两条命令使其记录到mysql的general_log表中。

```
set global general_log=on;
set global log_output='table';
```

直接上代码：

```php+HTML
<!DOCTYPE html>
<html>
<head>
	<meta http-equiv="content-type" content="text/html; charset=utf-8">
	<title></title>
</head>
<body>
<form id="sql_connect" method="post" action=""/>
<input type="text" id="name" name="sqlname" value="root"/>
<input type="text" id="name" name="sqlpass" value="123456"/>
<input type="submit" value="断点" name="sub1"/>
<input type="submit" value="更新" name="sub2"/>
</form>
<?php
session_start();
$sqlpass = '';
$sqlname = '';
$sqlold = array();
$sqlnew = array();
$result = array();
if(isset($_POST['sqlname'])&&isset($_POST['sqlpass'])&&(!empty($_POST['sub1']))){
	$sqlold = array();
	$sqlname = $_POST['sqlname'];
	$sqlpass = $_POST['sqlpass'];
	$sql = mysql_connect("localhost", $sqlname, $sqlpass);
	if(!$sql){
		die ('mysql connection failed!!!');
	}
	else{
		mysql_select_db("mysql", $sql);
		$query1 = "select count(*) from general_log";
		$rs1 = mysql_query($query1);
		$_SESSION['count'] = intval(mysql_fetch_assoc($rs1)['count(*)']);
	}
}

if (isset($_POST['sqlname'])&&isset($_POST['sqlpass'])&&(!empty($_POST['sub2']))){
	$num = $_SESSION['count'];
	$sqlnew = array();
	$sqlname = $_POST['sqlname'];
	$sqlpass = $_POST['sqlpass'];
	$sql = mysql_connect("localhost", $sqlname, $sqlpass);
	if(!$sql){
		die ('mysql connection failed!!!');
	}
	else{
		mysql_select_db("mysql", $sql);
		$query2 = "select * from general_log";
		$rs2 = mysql_query($query2);
		while($row2 = mysql_fetch_assoc($rs2)){
			array_push($sqlnew, $row2);
		}
		$result = array_slice($sqlnew, $num-count($sqlnew));
		foreach ($result as $key => $value) {
			echo "<tr><td>ID:	".$key."</td><td>时间:		</td><td>".$value['event_time']."</td><td>操作:		</td><td>".$value['argument']."</td></tr><br>";
		}
		$_SESSION['count'] = count($sqlnew);
	}
}
?>
</body>
</html>
```

使用结果如图:
![img](/img/post/mysql-monitor.png)

## 后记

挖不出洞啊挖不出洞，好不容易绕过了xssfilter乌云还不给审啊不给审，什么鬼！
