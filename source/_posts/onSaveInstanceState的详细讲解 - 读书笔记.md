---
title: onSaveInstanceState的详细讲解 - 读书笔记
date: 2016年6月13日22:17:44
categories: 读书笔记
tags: [Android源码设计模式解析与实战，读书笔记，Android]
---

# 来自Android源码设计模式解析与实战的第十三章，编程中的“后悔药”——备忘录模式

---

## 1. onSaveInstanceState简介

当Activity不是正常方式退出，且Activity在随后的时间被系统杀死之前就会调用这两个方法。

```java
protected void onSaveInstanceState(Bundle outState) {
        outState.putBundle(WINDOW_HIERARCHY_TAG, mWindow.saveHierarchyState());
        Parcelable p = mFragments.saveAllState();
        if (p != null) {
            outState.putParcelable(FRAGMENTS_TAG, p);
        }
        getApplication().dispatchActivitySaveInstanceState(this, outState);
    }
```

主要分三步：

* 存储窗口的视图树状态
* 存储Fragment的状态
* 调用Activity的onSaveInstanceState来存储状态

###  1.1 存储各种状态

```java
 public Bundle saveHierarchyState() {
        Bundle outState = new Bundle();
        if (mContentParent == null) {
            return outState;
        }

        SparseArray<Parcelable> states = new SparseArray<Parcelable>();
        mContentParent.saveHierarchyState(states);
        outState.putSparseParcelableArray(VIEWS_TAG, states);

        // save the focused view id
        View focusedView = mContentParent.findFocus();
        if (focusedView != null) {
            if (focusedView.getId() != View.NO_ID) {
                outState.putInt(FOCUSED_ID_TAG, focusedView.getId());
            }
        }

        // save the panels
        SparseArray<Parcelable> panelStates = new SparseArray<Parcelable>();
        savePanelState(panelStates);
        if (panelStates.size() > 0) {
            outState.putSparseParcelableArray(PANELS_TAG, panelStates);
        }
		// save the actionBar
        if (mDecorContentParent != null) {
            SparseArray<Parcelable> actionBarStates = new SparseArray<Parcelable>();
            mDecorContentParent.saveToolbarHierarchyState(actionBarStates);
            outState.putSparseParcelableArray(ACTION_BAR_TAG, actionBarStates);
        }

        return outState;
    }

```

具体步骤：

* 存储整棵视图树的结构
* 存储获取了焦点的view
* 存储整个面板状态
* 存储ActionBar的状态

我们只看如何存储整棵视图树的结构，这里mContentParent就是我们通过Activity的setContentView函数设置的内容视图，它是整个内容视图的根节点。直接看源代码

```java
	public void saveHierarchyState(SparseArray<Parcelable> container) {
        dispatchSaveInstanceState(container);
    }

	protected void dispatchSaveInstanceState(SparseArray<Parcelable> container) {
        if (mID != NO_ID && (mViewFlags & SAVE_DISABLED_MASK) == 0) {
            mPrivateFlags &= ~PFLAG_SAVE_STATE_CALLED;
            Parcelable state = onSaveInstanceState();
            if ((mPrivateFlags & PFLAG_SAVE_STATE_CALLED) == 0) {
                throw new IllegalStateException(
                        "Derived class did not call super.onSaveInstanceState()");
            }
            if (state != null) {
                container.put(mID, state);
            }
        }
    }
```

注意mID，如果View没有设置id时，这个View的状态不会被存储到Bundle中，因为之后还原时需要这个作为唯一key值。

View类中的saveHierarchyState函数调用了dispatchSaveInstanceState函数来存储自身的状态，而ViewGroup则覆写了dispatchSaveInstanceState来存储自身以及子view视图，代码如下

```java
	protected void dispatchSaveInstanceState(SparseArray<Parcelable> container) {
        super.dispatchSaveInstanceState(container);
        final int count = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < count; i++) {
            View c = children[i];
            if ((c.mViewFlags & PARENT_SAVE_DISABLED_MASK) != PARENT_SAVE_DISABLED) {
                c.dispatchSaveInstanceState(container);
            }
        }
    }
```

### 1.2 Bundle数据存储在哪？

调用onSaveInstanceState是在Activity被销毁前，更确切的说法是在调用Activity的onStop之前。Activity的onStop在ActivityThread的performStopActivity中。

```java
	final void performStopActivity(IBinder token, boolean saveState) {
        ActivityClientRecord r = mActivities.get(token);
        performStopActivityInner(r, null, false, saveState);
    }

	private void performStopActivityInner(ActivityClientRecord r,
            StopInfo info, boolean keepShown, boolean saveState) {
        if (r != null) {
            if (info != null) {
                try {
                    info.description = r.activity.onCreateDescription();
                } catch (Exception e) {
                    ...
                }
            }
            // Next have the activity save its current state and managed dialogs...
            if (!r.activity.mFinished && saveState) {
                if (r.state == null) {
                    callCallActivityOnSaveInstanceState(r);
                }
            }
            if (!keepShown) {
                try {
                    // Now we are idle.
                    r.activity.performStop();
                } catch (Exception e) {
                  	...
                }
                r.stopped = true;
            }
            r.paused = true;
        }
    }
```

这里很明显，在调用performStop前会调用callCallActivityOnSaveInstanceState。而状态信息存储到ActivityClientRecord的state字段中。

## 2. onSaveInstanceState的调用时机

* 按下Home键
* 长按Home键，选择运行其他程序
* 按下电源键，屏幕关闭
* 启动一个新的Activity
* 屏幕方向切换
* 电话打入等情况