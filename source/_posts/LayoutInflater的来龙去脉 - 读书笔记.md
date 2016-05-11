---
title: LayoutInflater的来龙去脉 - 读书笔记
date: 2016年4月30日02:24:06
categories: 读书笔记
tags: [Android源码设计模式解析与实战，读书笔记，Android]
---

# 该系列第一篇文章，来自Android源码设计模式解析与实战，主要是讲源码的

---

## LayoutInflater中的单例模式简单介绍

```java
/**
* Obtains the LayoutInflater from the given context.
*/
public static LayoutInflater from(Context context) {
   LayoutInflater LayoutInflater =
           (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
   if (LayoutInflater == null) {
       throw new AssertionError("LayoutInflater not found.");
   }
   return LayoutInflater;
}
```
## 深入理解LayoutInflater
---

### 在这之前，简单介绍Activity的启动过程

Activity的入口是ActivityThread的main函数，在main函数中创建一个新的ActivityThread对象，并且启动消息循环Looper.prepareMainLooper()，创建新的Activity, 新的Context对象, 然后将Context传递给Activity。之后调用attach函数，其中，会通过Binder机制与ActivityManagerService通信，并且最终调用handleLaunchActivity函数。该函数会继续调用performLaunchActivity，之后会调用callActivityOnCreate方法。

---

context的实现类是ContextImpl，所以要到ContextImpl中寻找getSystemService方法

```java
	@Override
    public Object getSystemService(String name) {
        return SystemServiceRegistry.getSystemService(this, name);
    }
	/**
     * Gets a system service from a given context.
     */
    public static Object getSystemService(ContextImpl ctx, String name) {
        ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
        return fetcher != null ? fetcher.getService(ctx) : null;    
	}
```

在SystemServiceRegistry中可以找到LAYOUT_INFLATER_SERVICE的生成的地方。

```java
registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class,
                new CachedServiceFetcher<LayoutInflater>() {
            @Override
            public LayoutInflater createService(ContextImpl ctx) {
                return new PhoneLayoutInflater(ctx.getOuterContext());
            }});
```

所以，LayoutInflater实现类是PhoneLayoutInflater，而其中用XmlPullParser和反射对布局文件xml进行解析。