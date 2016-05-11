---
title: AS导入
date: 2016-4-20 23:41:12
categories: blog
tags: [博客，文章]
---

Welcome

## 如何从elicpse导入

* 启动时直接导入主工程，查看依赖关系是否正常，gradle修改对应模块。如果不正常请参考之后操作。

* 将其余工程全部删除，修改gradle

* 一一导入其余工程，改为libs，然后以module的形式import到主工程中

* 修改gradle，sync