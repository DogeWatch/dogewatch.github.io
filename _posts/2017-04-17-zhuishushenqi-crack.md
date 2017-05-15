---
layout:		post
title:		"追书神器破解手记"
subtitle:	"android 学习(一)"
date:		2017-04-17
author:	"dogewatch"
header-img:	"img/post-bg-android.jpg"
tags:
    - android
---

# 前言

依然是学习练手，android包括java以前都没有接触过，看了两周《android软件安全与逆向分析》感觉光看书也不知道到底学到了什么就打算找个目标练练手。[追书神器](http://www.zhuishushenqi.com/)是我用了很多年的一款看小说软件，资源很全，android和ios上都有，但是在某次更新以后就不能免费看书了。网上有很多关于它的破解资源，但是却没有讲述如何破解其android版本的相关资料，可能是因为这个软件破解比较简单的原因，不过挺适合我这种新手练手的。

# 正文

## 0x01

先说说背景吧，这款软件看小说的时候在正版源处只能够免费看一部分内容，后面的内容就需要一个叫追书券的东西购买，不过在某次更新之前该软件在正版源之外还有个换源的功能，可以通过更换为盗版源达到免费看书的目的。因此一开始我是打算破解正版源处的功能，如果能够免费看正版源的书那当然是坠吼的了。

## 0x02 失败的尝试

既然是想破解正版源的书那么先抓个包看看免费和收费的区别吧。抓包工具是burpsuit，免费的章节直接就能看到正文的内容

![img](/img/post/zhuishu-1.jpg)

而收费章节在购买前访问会得到正文的密文

![img](/img/post/zhuishu-2.jpg)

这个软件每天签到会给点追书券，攒够看一章的券购买一下试试

![img](/img/post/zhuishu-3.jpg)

可以看到发送请求的时候带上了token和bookId，返回一个key值，这个key就是解密前面密文的密钥了。那么从这个流程上来看，在客户端这边下手是不行的了，最核心的key是在网站后台查询token相关信息后给的。用web的手段大致浏览了下这个网站发现了一些登陆的入口也就没有继续了，毕竟搞这个软件还是想练练android破解的技术。

## 0x03 换个思路

既然不能免费看正版源那么让我免费看看盗版源总行吧。记得某次更新前所有的书都能看盗版源，之后就只有一些没什么人看的书可以了，不过官方依然保留了所有书的接口。web选手做这种事第一反应还是抓包看看，找两本书，一本叫《蛊真人》，挺火的小说，在看书的页面里没有换源的选项，抓到的包如下：

![img](/img/post/zhuishu-4.jpg)

另外找了个没什么人看的小说《极品家丁后传》：

![img](/img/post/zhuishu-5.jpg)

经过对比两个包发现了一个名叫`_le`的参数返回值有所区别，于是找了些其它的书实验发现凡是能够看盗版的该参数值都返回`true`，而只能看正版的则相反。确定了这一点后就要去程序里修改使用到这个参数的相关代码了，不过在这之前有一件事要做，就是去掉签名校验。

### 去签名校验

这个软件在libtest.so里对整个包有个签名的校验，其校验核心函数是`isZhuishuSignKey`，代码如下

```c
signed int __fastcall isZhuishuSignKey(JNIEnv *a1, int a2)
{
  JNIEnv *v2; // r4@1
  int v3; // r5@1
  int v4; // r0@1
  int v5; // r7@1
  int v6; // r0@1
  int v7; // r0@1
  int v8; // r6@1
  int v9; // r0@1
  int v10; // ST0C_4@1
  int v11; // r0@1
  int v12; // r5@1
  int v13; // r0@1
  int v14; // r5@1
  int v15; // r0@1
  int v16; // r0@1
  int v17; // r0@1
  int v18; // r0@1
  int v19; // r5@1
  int v20; // r0@1
  int v21; // r0@1
  int v22; // r0@1
  signed int result; // r0@2

  v2 = a1;
  v3 = a2;
  v4 = ((int (*)(void))(*a1)->GetObjectClass)();
  v5 = v4;
  v6 = ((int (__fastcall *)(JNIEnv *, int, const char *, const char *))(*v2)->GetMethodID)(
         v2,
         v4,
         "getPackageManager",
         "()Landroid/content/pm/PackageManager;");
  v7 = ((int (__fastcall *)(JNIEnv *, int, int))(*v2)->CallObjectMethod)(v2, v3, v6);
  v8 = v7;
  v9 = ((int (__fastcall *)(JNIEnv *, int))(*v2)->GetObjectClass)(v2, v7);
  v10 = ((int (__fastcall *)(JNIEnv *, int, const char *, const char *))(*v2)->GetMethodID)(
          v2,
          v9,
          "getPackageInfo",
          "(Ljava/lang/String;I)Landroid/content/pm/PackageInfo;");
  v11 = ((int (__fastcall *)(_DWORD, _DWORD, const char *, const char *))(*v2)->GetMethodID)(
          v2,
          v5,
          "getPackageName",
          "()Ljava/lang/String;");
  v12 = ((int (__fastcall *)(JNIEnv *, int, int))(*v2)->CallObjectMethod)(v2, v3, v11);
  ((void (__fastcall *)(JNIEnv *, int, _DWORD))(*v2)->GetStringUTFChars)(v2, v12, 0);
  v13 = ((int (__fastcall *)(JNIEnv *, int, int, int))(*v2)->CallObjectMethod)(v2, v8, v10, v12);
  v14 = v13;
  v15 = ((int (__fastcall *)(JNIEnv *, int))(*v2)->GetObjectClass)(v2, v13);
  v16 = ((int (__fastcall *)(JNIEnv *, int, const char *, const char *))(*v2)->GetFieldID)(
          v2,
          v15,
          "signatures",
          "[Landroid/content/pm/Signature;");
  v17 = ((int (__fastcall *)(JNIEnv *, int, int))(*v2)->GetObjectField)(v2, v14, v16);
  v18 = ((int (__fastcall *)(JNIEnv *, int, _DWORD))(*v2)->GetObjectArrayElement)(v2, v17, 0);
  v19 = v18;
  v20 = ((int (__fastcall *)(_DWORD, _DWORD))(*v2)->GetObjectClass)(v2, v18);
  v21 = ((int (__fastcall *)(_DWORD, _DWORD, const char *, const char *))(*v2)->GetMethodID)(v2, v20, "hashCode", "()I");
  v22 = ((int (__fastcall *)(JNIEnv *, int, int))(*v2)->CallIntMethod)(v2, v19, v21);
  if ( v22 == 1902783089 )
    result = 1;
  else
    result = (unsigned int)(v22 - 722848853) <= 0;
  return result;
}
```

这个校验过程并不复杂，我们甚至不需要关心它是怎么校验的， 只需要把最后的`return result;`替换成`return true;`就行了。用ida自带的patch功能或者找个16进制编辑器修改对应指令的机器码就行。

### 修改apk

改好了libtest.so后，我们就能自由地修改apk了。用jeb打开该apk，全局搜索字符串'_le'，发现有两个与其相关的方法名字，一个是is_le,另一个是set_le，看了下数量也不是很多，就懒得一个一个看了，编辑相关的smali文件，把所有的名叫`is_le`的方法返回值都改为true，然后用androidcracktool重新打包签名生成新的apk文件，这次破解也就结束了，安装上看看效果：

![img](/img/post/zhuishu-6.png)

打开《蛊真人》已经可以选取别的源了。

# 后记

这个软件破起来确实没什么难度，不过还可以有更进一步的发挥，比如说解锁随机看书，解锁男生区女生区限制，去掉广告和一些无用的功能等。不过这些东西个人没什么需求，也就没有动力去弄了，先就这样吧。