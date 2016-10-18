---
layout:		post
title:		HttpOnly Cookie Sniffer
subtitle:	chrome extension
data:		2016-09-14
author:		"dogewatch"
header-img:	"img/post-bg-js-version.jpg"
tags:
    - web
---

> alert(+1s)

# 前言

看到@fridayy美女黑阔写的一篇关于百度的利用selfxss登录账户的文章后，觉得里面描述的HttpOnly隐私嗅探器挺有意思的，决定来实现一发。

# 正文

一般情况下带httponly标志的cookie我们是不能通过js获取的，但是如果泄漏到了页面中的话我们就能通过各种手段来获取了，因此有必要写一个监测工具来监测页面中是否有httponly的cookie。
因为chrome的插件有权限来获取所有存储在本地的cookie，所以我们这个工具就以浏览器插件的形式来扫描页面。原理是非常简单的，只要获取带httponly标识的值，然后在浏览页面时扫描下html字符就可以了。
首先获取浏览器内存储的所有cookie，提取出我们想要的。

```
function getcookie(){
	chrome.cookies.getAll({secure: false},function(cookies){
		for(var i=0; i<cookies.length; i++){
			var value = cookies[i].value;
			if(cookies[i].httpOnly == true){
				if(value.length > 10){
					list.push(value);
					cookie.push(cookies[i]);
				}
			}
		}
	});
}
```
因为httonly的cookie中有些值很简单，例如1、true、ok、时间之类的，所以这里只取了长度大于10的值。
manifest.json中需要添加对应的权限

```
"permissions":["cookies","http://*/*","https://*/*"],
```
获取到了这些cookie作为关键字了以后，就需要扫面页面了。
新建一个content.js注入到页面中，content.js与background.js用sendRequest来通信。

```
function sendr(){
	chrome.extension.sendRequest('GET_LIST',function(list){
		console.log(list.lentgh);
		var html = document.documentElement.outerHTML;
		list.forEach(function(item, index){
			if(html.indexOf(item) >= 0){
				alert('NUM: ' + index + ' Value: ' + item);
			}
		});
	});
}

window.addEventListener('DOMContentLoaded', sendr());
```
在background.js中添加代码用来发送cookie列表

```
function sendr(){
	chrome.extension.onRequest.addListener(function(message, sender, sendResponse){
		if(message == 'GET_LIST'){
			console.log('accept');
			sendResponse(list);
		}
	});
}
```
当cookie值有所变化时同步刷新cookie列表

```
chrome.cookies.onChanged.addListener(function(){getcookie();});
```

到此，一个很简单的cookie嗅探器就完成了，我们来试验一下有没有效果。
正常访问网页，大半天时间都没有弹窗，正当我以为这个插件写得有问题时，163邮箱居然弹了！！！
![img](/img/post/cookie-1.png)
看一下具体信息
![img](/img/post/cookie-2.png)

# 后记

后续想要添加的功能就是劫持ajax函数看ajax数据中是否有cookie值以及监测url中有没有泄漏这些信息.