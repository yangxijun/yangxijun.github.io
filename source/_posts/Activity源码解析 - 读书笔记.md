---
title: Activity源码解析 - 读书笔记
date: 2016年7月3日22:09:13
categories: 读书笔记
tags: [Android源码设计模式解析与实战，读书笔记，Android]
---

# 来自Android源码设计模式解析与实战的第十五章，抓住问题核心——模板方法模式

---

## 1. Activity启动

Activity是一个比较好的模板方法模式。在Android系统启动时，第一个启动的进程是zygote进程，然后由zygote启动SystemServer，再后就是启动AWS、WMS等系统核心服务，这些服务承载着整个Android系统与客户端程序交互。zygote除了启动系统服务与进程之外，普通的用户进程也是由zygote进程fork而来，当一个进程启动起来后，就会加载用户在AndroidManifest.xml中配置的默认加载的Activity，此时加载的入口是ActivityThread.main方法，是整个程序的入口。

main方法主要功能就是创建Application和Activity，并且调用Activity的生命周期函数。

```java
	public static void main(String[] args) {
        SamplingProfilerIntegration.start();
        CloseGuard.setEnabled(false);
        Environment.initForCurrentUser();
        EventLogger.setReporter(new EventLoggingReporter());
        Security.addProvider(new AndroidKeyStoreProvider());

        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
        Looper.loop();
    }

 	private void attach(boolean system) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
			...
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                // Ignore
            }
        }
    }
```

上述程序可以看到，在ActivityThread.main中主要的功能就是创建了UI线程消息循环，并启动循环消息循环。最重要的是创建了ActivityThread并调用了attach方法。在方法中又调用了AWS中的attachApplication方法。我们先看看ApplicationThread是什么

```java
private class ApplicationThread extends ApplicationThreadNative {
	public final void scheduleResumeActivity(IBinder token, int processState,
			boolean isForward, Bundle resumeArgs) {
		sendMessage(H.RESUME_ACTIVITY, token, isForward ? 1 : 0);
    }  	
  	
	public final void scheduleLaunchActivity(...) {
            updateProcessState(procState, false);
            ActivityClientRecord r = new ActivityClientRecord();

            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.activityInfo = info
            r.state = state;
            updatePendingConfiguration(curConfig);
			...
            sendMessage(H.LAUNCH_ACTIVITY, r);
	}

}
```

ApplicationThread通过handler管理Activity的生命周期。继续关注AWS中的attachApplication方法

```java
	public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
	}

    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {
        ProcessRecord app;
        try {
            ...
            thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                    profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                    app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(mConfiguration), app.compat,
                    getCommonServicesLocked(app.isolated),
                    mCoreSettingsObserver.getCoreSettingsLocked());
            updateLruProcessLocked(app, false, null);
            app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
        } catch (Exception e) {
            return false;
        }
        // See if the top visible activity is waiting to run in this process...
        if (normalMode) {
            try {
                if (mStackSupervisor.attachApplicationLocked(app)) {
                    didSomething = true;
                }
            } catch (Exception e) {
            }
        }
        return true;
    }
```

忽略些代码，这里关注mStackSupervisor.attachApplicationLocked(app)。

```java
boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
        final String processName = app.processName;
        boolean didSomething = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                if (!isFrontStack(stack)) {
                    continue;
                }
                ActivityRecord hr = stack.topRunningActivityLocked(null);
                if (hr != null) {
                    if (hr.app == null && app.uid == hr.info.applicationInfo.uid
                            && processName.equals(hr.processName)) {
                        try {
                            if (realStartActivityLocked(hr, app, true, true)) {
                                didSomething = true;
                            }
                        } catch (RemoteException e) {
                        }
                    }
                }
            }
        }
        return didSomething;
    }

```

这里会遍历整个Activity栈并且找出栈顶的ActivityStack，然后调用realStartActivityLocked来真正启动 Activity

```java
	final boolean realStartActivityLocked(ActivityRecord r,
            ProcessRecord app, boolean andResume, boolean checkConfig)
            throws RemoteException {
        final ActivityStack stack = r.task.stack;
        try {
            mService.ensurePackageDexOpt(r.intent.getComponent().getPackageName());
            app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_TOP);
            app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    r.compat, r.launchedFromPackage, r.task.voiceInteractor, app.repProcState,
                    r.icicle, r.persistentState, results, newIntents, !andResume,
                    mService.isNextTransitionForward(), profilerInfo);
        } catch (RemoteException e) {}
        return true;
    }

```

ActivityThread的内部类ApplicationThread的scheduleLaunchActivity方法。之后会调ActivityThread的handleLaunchActivity函数，再调用performLaunchActivity来创建Activity，并将Activity和Application关联上。然后调用onCreate方法。

## 2. setContentView

```java
	public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }

	@Override
    public void setContentView(int layoutResID) {
        if (mContentParent == null) {
            installDecor();
        }
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }

	private void installDecor() {
        if (mDecor == null) {
            mDecor = return new DecorView(getContext(), -1);
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);
        }
    }
```

setContentView基本流程

* 构建mDecor对象
* 设置窗口属性
* inflate布局

