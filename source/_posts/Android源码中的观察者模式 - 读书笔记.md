---
title: Android源码中的观察者模式 - 读书笔记
date: 2016年6月7日22:25:58
categories: 读书笔记
tags: [Android源码设计模式解析与实战，读书笔记，Android]
---

# 来自Android源码设计模式解析与实战的第十二章，解决、解耦的钥匙——观察者模式

---

## 1. ListView中的观察者模式

通常我们往ListView添加数据后，都会调用Adapter的notifyDataSetChanged方法，这就是一个观察者模式。

```java
public void notifyDataSetChanged() {
        mDataSetObservable.notifyChanged();
    }

public void notifyChanged() {
        synchronized(mObservers) {
            for (int i = mObservers.size() - 1; i >= 0; i--) {
                mObservers.get(i).onChanged();
            }
        }
    }
```

这个就是观察者模式主要特征，遍历所有项目一一通知。那么观察者从哪里来？其实是通过ListView的setAdapter方法产生的。

```java
@Override
    public void setAdapter(ListAdapter adapter) {
        if (mAdapter != null && mDataSetObserver != null) {
            mAdapter.unregisterDataSetObserver(mDataSetObserver);
        }
     	...
        // AbsListView#setAdapter will update choice mode states.
        super.setAdapter(adapter);
        if (mAdapter != null) {
            ...
            mDataSetObserver = new AdapterDataSetObserver();
            mAdapter.registerDataSetObserver(mDataSetObserver);
        }
      	...
        requestLayout();
    }
```

在设置Adapter时会创建一个AdapterDataSetObserver，之后会注册到Adapter中。继续深入

```java
class AdapterDataSetObserver extends AdapterView<ListAdapter>.AdapterDataSetObserver {
        @Override
        public void onChanged() {
            super.onChanged();
            if (mFastScroll != null) {
                mFastScroll.onSectionsChanged();
            }
        }

        @Override
        public void onInvalidated() {
            super.onInvalidated();
            if (mFastScroll != null) {
                mFastScroll.onSectionsChanged();
            }
        }
    }

		@Override
        public void onChanged() {
            mDataChanged = true;
            mOldItemCount = mItemCount;
            mItemCount = getAdapter().getCount();

            // Detect the case where a cursor that was previously invalidated has
            // been repopulated with new data.
            if (AdapterView.this.getAdapter().hasStableIds() && mInstanceState != null
                    && mOldItemCount == 0 && mItemCount > 0) {
                AdapterView.this.onRestoreInstanceState(mInstanceState);
                mInstanceState = null;
            } else {
                rememberSyncState();
            }
            checkFocus();
            requestLayout();
        }
```

这里就知道了，在数据发送变化后，会调用notifyDataSetChanged函数，然后会调用其观察对象的onChanged方法，最后通知ListView重新布局，也就是这里的requestLayout。

## 2. BroadcastReceiver中的观察者模式

### 2.1  广播的注册

直接去ContextImpl中查找registerReceiver，其最后调用registerReceiverInternal

```java
private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context) {
        IIntentReceiver rd = null;
        if (receiver != null) {
            if (mPackageInfo != null && context != null) {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = new LoadedApk.ReceiverDispatcher(
                        receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
            return ActivityManagerNative.getDefault().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName,
                    rd, filter, broadcastPermission, userId);
        } catch (RemoteException e) {
            return null;
        }
    }
```

这里看到，当外部传入的context不为空时，会调用getReceiverDispatcher方法，跟进去发现

```java
public IIntentReceiver getReceiverDispatcher(BroadcastReceiver r,
            Context context, Handler handler,
            Instrumentation instrumentation, boolean registered) {
        synchronized (mReceivers) {
            LoadedApk.ReceiverDispatcher rd = null;
            ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> map = null;
            if (registered) {
                map = mReceivers.get(context);
                if (map != null) {
                    rd = map.get(r);
                }
            }
            if (rd == null) {
                rd = new ReceiverDispatcher(r, context, handler,
                        instrumentation, registered);
                if (registered) {
                    if (map == null) {
                        map = new ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>();
                        mReceivers.put(context, map);
                    }
                    map.put(r, rd);
                }
            } else {
                rd.validate(context, handler);
            }
            rd.mForgotten = false;
            return rd.getIIntentReceiver();
        }
    }
```

如果receiver在map中，直接取出，否则new一个ReceiverDispatcher，添加到map中，完成IIntentReceiver的获取。

回到最早的registerReceiverInternal中，最后一步是将其注册到ActivityManagerService中

```java
public Intent registerReceiver(IApplicationThread caller, String callerPackage,
            IIntentReceiver receiver, IntentFilter filter, String permission, int userId) {
        ...
        ArrayList<Intent> allSticky = null;
        if (stickyIntents != null) {
            final ContentResolver resolver = mContext.getContentResolver();
            // Look for any matching sticky broadcasts...
            for (int i = 0, N = stickyIntents.size(); i < N; i++) {
                Intent intent = stickyIntents.get(i);
                if (filter.match(resolver, intent, true, TAG) >= 0) {
                    if (allSticky == null) {
                        allSticky = new ArrayList<Intent>();
                    }
                    allSticky.add(intent);
                }
            }
        }

        // The first sticky in the list is returned directly back to the client.
        Intent sticky = allSticky != null ? allSticky.get(0) : null;
        if (receiver == null) {
            return sticky;
        }

        synchronized (this) {
            if (callerApp != null && (callerApp.thread == null
                    || callerApp.thread.asBinder() != caller.asBinder())) {
                // Original caller already died
                return null;
            }
            ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
            if (rl == null) {
                rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                        userId, receiver);
                if (rl.app != null) {
                    rl.app.receivers.add(rl);
                } else {
                    try {
                        receiver.asBinder().linkToDeath(rl, 0);
                    } catch (RemoteException e) {
                        return sticky;
                    }
                    rl.linkedToDeath = true;
                }
                mRegisteredReceivers.put(receiver.asBinder(), rl);
            } 
          	...
            BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
                    permission, callingUid, userId);
            rl.add(bf);
            mReceiverResolver.addFilter(bf);
			...
            return sticky;
        }
    }
```

这里主要做了几件事

* 根据Action查找匹配的sticky接收器，也就是sendStickyBroadcast发送的广播类型
* 获取对应的ReceiverList
* 创建BroadcastFilter对象，并且添加到ReceiverList中

完成了注册。

### 2.2 分发

ActivityManagerService在消息循环中处理这个广播，并通过Binder机制把这个广播分发给ReceiverDispatcher中，完成异步分发。

### 2.3 处理

ReceiverDispatcher内部类Args在MainActivity所在的消息循环中处理这个广播，最终是将这个广播发给注册的BroadcastReceiver实例的onReceiver函数进行处理：

```java
			public void run() {
                final BroadcastReceiver receiver = mReceiver;
                final boolean ordered = mOrdered;
                
                final IActivityManager mgr = ActivityManagerNative.getDefault();
                final Intent intent = mCurIntent;
                mCurIntent = null;
				...
                try {
                    ClassLoader cl =  mReceiver.getClass().getClassLoader();
                    intent.setExtrasClassLoader(cl);
                    setExtrasClassLoader(cl);
                    receiver.setPendingResult(this);
                    receiver.onReceive(mContext, intent);
                } catch (Exception e) {
                }
            }
```

### 2.4 总结

广播，通过一些map存储BroadcastReceiver，key就是封装了这些广播信息。当发布广播时通过AMS到map查询注册了广播的BroadcastReceiver，通过ReceiverDispatcher将广播分发给各个订阅对象，从而完成这个发布——订阅过程。

