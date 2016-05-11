---
title: WindowManager的简单介绍 - 读书笔记
date: 2016年5月1日21:21:22
categories: 读书笔记
tags: [Android源码设计模式解析与实战，读书笔记，Android]
---

# 来自Android源码设计模式解析与实战的第三章Android源码中的Builder模式实现

---

## WindowManager的介绍

所有需要显示到屏幕上的内容（包括Activity）都是通过WindowManager来操作的。这里我们只关心WindowManager和WindowManagerService、Surface、SurfaceFlinger等建立关联以及交互的一个基本过程。

与之前一样我们可以通过context的getSystemService方法追踪到WindowManager的源码位置。

```java
registerService(Context.WINDOW_SERVICE, WindowManager.class, new CachedServiceFetcher<WindowManager>() {
            @Override
            public WindowManager createService(ContextImpl ctx) {
                return new WindowManagerImpl(ctx.getDisplay());
		}});
```

其实这里就可以发现，WindowManager是通过WindowManagerImpl进行工作的。

进入WindowManagerImpl我们看到，实际上WindowManagerImpl也不是干活的，是通过代理模式交给WindowManagerGlobal来干活的。

```java
private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();

@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
	applyDefaultToken(params);
	mGlobal.addView(view, params, mDisplay, mParentWindow);
}
```

而addView的关键代码是

```java
root = new ViewRootImpl(view.getContext(), display);
view.setLayoutParams(wparams);
mViews.add(view);
mRoots.add(root);
mParams.add(wparams);

...
root.setView(view, wparams, panelParentView);
...
```

这个addView做了以下几件事：

* 创建ViewRootImpl
* 给view设置布局参数
* 将view，root，布局参数存到各自的列表当中
* 调用ViewRootImpl的setView方法，加载view

ViewRootImpl是一个Handler，是作为native层与java层View系统通信的桥梁，这里简单看下ViewRootImpl的构造方法

```java
 public ViewRootImpl(Context context, Display display) {
        mContext = context;
        mWindowSession = WindowManagerGlobal.getWindowSession();
        mDisplay = display;
        mBasePackageName = context.getBasePackageName();

		...
        mThread = Thread.currentThread();
		...
}
```

---

NOTICE：这个mThread是当前线程，因为ViewRootImpl是在UI线程中创建的，所以不能在子线程中更新UI。

---

很明显的，这里继续观察WindowManagerGlobal中的getWindowSession方法

```java
public static IWindowSession getWindowSession() {
        synchronized (WindowManagerGlobal.class) {
            if (sWindowSession == null) {
                try {
                    InputMethodManager imm = InputMethodManager.getInstance();
                    IWindowManager windowManager = getWindowManagerService();
                    sWindowSession = windowManager.openSession(
                            new IWindowSessionCallback.Stub() {
                                @Override
                                public void onAnimatorScaleChanged(float scale) {
                                    ValueAnimator.setDurationScale(scale);
                                }
                            },
                            imm.getClient(), imm.getInputContext());
                } catch (RemoteException e) {
                    Log.e(TAG, "Failed to open window session", e);
                }
            }
            return sWindowSession;
        }
}

public static IWindowManager getWindowManagerService() {
        synchronized (WindowManagerGlobal.class) {
            if (sWindowManagerService == null) {
                sWindowManagerService = IWindowManager.Stub.asInterface(
                        ServiceManager.getService("window"));
                try {
                    sWindowManagerService = getWindowManagerService();
                    ValueAnimator.setDurationScale(sWindowManagerService.getCurrentAnimatorScale());
                } catch (RemoteException e) {
                    Log.e(TAG, "Failed to get WindowManagerService, cannot set animator scale", e);
                }
            }
            return sWindowManagerService;
        }
}
```

Framework层首先通过getWindowManagerService函数获取IWindowManager对象，该函数中通过ServiceManager的getService函数获取WMS。

```java
 /**
     * Returns a reference to a service with the given name.
     * 
     * @param name the name of the service to get
     * @return a reference to the service, or <code>null</code> if the service doesn't exist
     */
    public static IBinder getService(String name) {
        try {
            IBinder service = sCache.get(name);
            if (service != null) {
                return service;
            } else {
                return getIServiceManager().getService(name);
            }
        } catch (RemoteException e) {
            Log.e(TAG, "error in getService", e);
        }
        return null;
    }
```

显而易见了，由于返回的是个IBinder对象，也就说这里已经和WMS建立了初步的联系。

我们先理一下view的绘制：

* 发起requestLayout
* scheduleTraversals
* performTraversals（包括performMeasure，performLayout，performDraw）
* draw方法

而draw方法就是绘制视图的方法，其主要步骤是：

* 判断是使用CPU还是GPU绘制
* 获取绘制表面Surface对象
* 通过Surface对象获取并且锁住Canvas绘制对象
* 从DecorView开始发起整棵视图树的绘制流程
* Surface对象解锁Canvas，并且通知SurfaceFlinger更新视图













