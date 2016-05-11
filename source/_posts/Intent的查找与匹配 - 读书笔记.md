---
title: Intent的查找与匹配 - 读书笔记
date: 2016年5月2日11:36:04
categories: 读书笔记
tags: [Android源码设计模式解析与实战，读书笔记，Android]
---

# 来自Android源码设计模式解析与实战的第四章使程序运行更高效-原型模式

---

## PackageManagerService的介绍

PMS加载Framework资源与核心库，之后才开始对扫描的指定目录下的apk文件进行解析。其中scanDirLI方法是扫描指定目录下的所有apk文件，然后通过调用scanPackageLI函数进行解析。

在scanPackageLI中首先先构造了一个PackageParser，也就是一个apk包解析器，之后调用PackageParser的parsePackage函数进行解析，最终会调用到parseApkLite函数，该函数做的事是：

* 构建AssetManager
* 构建资源
* 获取AndroidManifest.xml 的解析器
* 解析获取到的AndroidManifest.xml

继续，会对Application和use-permission解析，而Application中包含了Activity、Service、Provider、Receiver等标签，之后会把这些分别添加到对应的mActivities、mServices等列表中。到这一步，整个已安装apk的信息树就建立了，每个apk的应用名、包名、图标、Activity、Service等信息都存储在系统中。

## 启动Activity

对于启动一个Activity，最终都是调用startActivityForResult，最后是调用Instrumentation的execStartActivity函数。execStartActivity里面其实就是调用了ActivityManagerService的startActivity方法，整个方法里其实就是调用了ActivityStackSupervisor的startActivityMayWait方法，最后会调用PMS的resolveIntent方法，而其中继续调用了自身的queryIntentActivities方法。而queryIntentActivities会返回一个ActivityInfo对象列表，也就是符合Intent的Activity列表。