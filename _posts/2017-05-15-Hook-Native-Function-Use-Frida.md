---
layout:		post
title:		"Hook Native Function By Frida"
subtitle:	"android 学习(二)"
date:		2017-05-15
author:	"dogewatch"
header-img:	"img/post-bg-android.jpg"
tags:
    - android
---

# 前言

这是前一段时间的学习练手的一个小demo，记下来做个记录。简单介绍下[frida](http://www.frida.re)，frida是一款代码插桩工具，它可以向windows，macOS，Linux，iOS，Android等平台的原生应用中注射自定义的JavaScript代码片段，它的主要作用有：

* Access process memory
* Overwrite functions while the application is running
* Call functions from imported classes
* Find object instances on the heap and use them
* Hook, trace and intercept functions etc.

# 正文

本文使用的案例是2014年的阿里移动安全挑战赛的第二题，下载地址：[https://github.com/Wyc0/AndroidRevStudy/blob/master/note-3/crackme1.apk](https://github.com/Wyc0/AndroidRevStudy/blob/master/note-3/crackme1.apk)。先放一个别人的题解：[http://www.blogfshare.com/ailibaba-crack02.html](http://www.blogfshare.com/ailibaba-crack02.html)，不过因为本文使用的frida，所以与之前的题解方法均不相同，在下面会细说。

## 0x01

打开软件如图，需要输入密码过关：

![img](/img/post/MSC-1.png)

## 0x02

把apk拖到jeb中看看代码逻辑，输入密码那个按钮的代码如下：

```java
public void onClick(View arg6) {
                if(MainActivity.this.securityCheck(MainActivity.this.inputCode.getText().toString())) {
                    MainActivity.this.startActivity(new Intent(MainActivity.this, ResultActivity.class));
                }
                else {
                    Toast.makeText(MainActivity.this.getApplicationContext(), "验证码校验失败", 0).show();
                }
            }
```

可以看到验证密码的核心是这个`securityCheck`的函数，而这个函数声明是native：

```java
public native boolean securityCheck(String arg1) {
    }
```

那么这个函数是在libcrackme.so中。这里如果只是想绕过验证的话，那么非常简单，通常思路是反编译成smali文件然后编辑这里的代码，然后重新打包签名，不过这里我们用frida也很容易实现，用下面的代码在java层hook住这个securityCheck函数，直接重新编写其逻辑：

```javascript
send("Running Script");
Java.perform(function(){
    MainActivity = Java.use("com.yaotong.crackme.MainActivity");
    MainActivity.securityCheck.implementation = function(v){
        send("securityCheck hooked");
        return true;
    }
});
```

## 0x03

本文的核心还是要使用frida来获取正确的密码。我们把这个so文件放进ida里，securityCheck函数如下图：

![img](/img/post/MSC-2.png)

我们的输入最终是和`off_628C`所指向的地址的值做比较，跟过去可以看到该处的值为`wojiushidaan`，然而这个结果并不正确，所以这题没有这么简单，肯定是在加载so的时候程序对这个位置的值做了修改，因此我们需要找到在比较前该处的最新值。

这里有两个思路，一个就是在本文前面贴的那篇文章那样，使用ida或者其他工具动态调试这个so文件，不过这个程序针对这种方法专门提高了难道，它在加载.so时的`JNI_OnLoad`函数里用`pthread_create`新起了一个线程，不断地检测调试状态，只要有调试器附加在上面就会退出，详细情况和解决办法可以看放在开头的那篇文章。另一个办法就是我们今天所用到的，用frida而不是用调试器来获取程序运行时内存里的值。

frida提供了劫持native函数以及操作内存的一系列方法，首先我们需要通过导出函数表获取securityCheck这个函数的地址

```javascript
var securityCheck = undefined;
    exports = Module.enumerateExportsSync("libcrackme.so");
    for(i=0; i<exports.length; i++){
        if(exports[i].name == "Java_com_yaotong_crackme_MainActivity_securityCheck"){
            securityCheck = exports[i].address;
            send("securityCheck is at " + securityCheck);
            break;
        }
    }
```

然后就是读取`off_628C`的值了，在ida的exports里可以看到该函数的偏移为`0x11A8`，因此可以通过函数当前地址减`0x11A8`再加`0x628C`，然后用`Memory.readPointer`获取该处所指向的地址，最后用`Memory.readUtf8String`读取最终的结果，所有的代码如下：

```python
#!/usr/bin/env python
# coding=utf-8
from __future__ import print_function
import frida,sys

native_hook_code = """
Java.perform(function(){
    send("Running Script");

    var securityCheck = undefined;
    exports = Module.enumerateExportsSync("libcrackme.so");
    for(i=0; i<exports.length; i++){
        if(exports[i].name == "Java_com_yaotong_crackme_MainActivity_securityCheck"){
            securityCheck = exports[i].address;
            send("securityCheck is at " + securityCheck);
            break;
        }
    }

    Interceptor.attach(securityCheck,{
        onEnter: function(args){
            send("key is: " + Memory.readUtf8String(Memory.readPointer(securityCheck.sub(0x11a8).add(0x628c))));
        }
    });
});
"""

check_hook_code = """
    send("Running Script");
Java.perform(function(){
    MainActivity = Java.use("com.yaotong.crackme.MainActivity");
    MainActivity.securityCheck.implementation = function(v){
        send("securityCheck hooked");
        return true;
    }
});
"""

def on_message(message, data):
    if message['type'] == 'send':
        print("[*] {0}".format(message['payload']))
    else:
        print(message)

process = frida.get_device_manager().enumerate_devices()[-1].attach("com.yaotong.crackme")
script = process.create_script(native_hook_code)
script.on('message', on_message)
script.load()
sys.stdin.read()

```

结果如图：

![img](/img/post/MSC-3.png)

![img](/img/post/MSC-4.png)

# 后记

看来这些程序在针对调试器做反调试时还需要对这些代码插桩工具最对应的策略了，看雪上面有篇翻译文章讲对frida怎样做检测以及绕过，地址：[http://bbs.pediy.com/thread-217482.htm](http://bbs.pediy.com/thread-217482.htm)