---
title: MediaPlayerFactory - 读书笔记
date: 2016年5月17日00:04:02
categories: 目标
tags: [Android源码设计模式解析与实战，读书笔记，Android]
---

# 来自Android源码设计模式解析与实战的第六章，Android源码中的抽象工厂方法模式实现

---

## MediaPlayerFactory是典型的抽象工厂方法

MediaPlayerFactory其本质只是用来管理Android内置的4种不同的MediaPlayer，而对于每一种具体的MediaPlayer则由一个具体的Factory类来创建。

* StagefrightPlayerFactory创建StagefrightPlayer
* NuPlayerFactory创建NuPlayerDriver
* SonivoxPlayerFactory创建MidiFile
* TestPlayerFactory创建TestPlayerStub