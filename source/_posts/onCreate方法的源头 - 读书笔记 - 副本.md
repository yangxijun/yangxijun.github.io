---
title: onCreate方法的源头 - 读书笔记
date: 2016年5月11日23:52:11
categories: 读书笔记
tags: [Android源码设计模式解析与实战，读书笔记，Android]
---

# 来自Android源码设计模式解析与实战的第五章，应用最广泛的模式——工厂方法模式

---

## onCreate方法的源头

 该系列的[第一篇文章](http://yangxijun.github.io/2016/04/30/LayoutInflater%E7%9A%84%E6%9D%A5%E9%BE%99%E5%8E%BB%E8%84%89%20-%20%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0/)提到过，一个应用的真在入口是在ActivityThread类中的main方法。

### 关于ActivityThread

ActivityThread是一个final类，一个应用对应一个ActivityThread，当Zygote进程孵化出一个新的应用进程后，会执行ActivityThread的main方法，main方法中做了一些比较常规的逻辑，比如准备Looper和消息队列，然后调用ActivityThread的attach方法将其绑定到ActivityManagerService中，开始不断地读取消息队列中的消息并分发消息。