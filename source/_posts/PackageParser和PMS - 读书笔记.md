---
title: PackageParser和PMS - 读书笔记
date: 2016-6-5 12:01:45
categories: 读书笔记
tags: [Android源码设计模式解析与实战，读书笔记，Android]
---

# 来自Android源码设计模式解析与实战的第十章，化繁为简的翻译机——解释器模式

---

## PackageParser

该类用于对AndroidManifest.xml中每一个组件标签创建了相应的类用以存储相应的信息

类似之前的方法，我们可以从SystemServiceRegistry中找到对应Context.WIFI_SERVICE的具体实现方法

```java
registerService(Context.WIFI_SERVICE, WifiManager.class,
                new CachedServiceFetcher<WifiManager>() {
		@Override
        public WifiManager createService(ContextImpl ctx) {
            IBinder b = ServiceManager.getService(Context.WIFI_SERVICE);
            IWifiManager service = IWifiManager.Stub.asInterface(b);
            return new WifiManager(ctx.getOuterContext(), service);
	}});
```

这样我们可以看到WiFi的具体控制是在WifiManager之中，我们可以找到改变wifi状态的方法setWifiEnable

```java
public boolean setWifiEnabled(boolean enabled) {
        try {
            return mService.setWifiEnabled(enabled);
        } catch (RemoteException e) {
            return false;
        }
    }
```

其中mService是IWifiManager，一个AIDL的方法，通过查看源码很容易发现其具体实现是在WifiServiceImpl中

```java
    public synchronized boolean setWifiEnabled(boolean enable) {
        ...
        mWifiController.sendMessage(CMD_WIFI_TOGGLED);
        return true;
    }
```

这还不是真正实现功能的地方！实际上是通过mWifiController发送了一个CMD_WIFI_TOGGLED消息，其中mWifiController的类型为WifiController，sendMessage这个方法在WifiController的父类StateMachine中。

```java
public final void sendMessage(int what) {
        // mSmHandler can be null if the state machine has quit.
        SmHandler smh = mSmHandler;
        if (smh == null) return;

        smh.sendMessage(obtainMessage(what));
    }
```

那么我们继续查看SmHandler源码中的handleMessage方法

```java
@Override
        public final void handleMessage(Message msg) {
            if (!mHasQuit) {
                if (mDbg) mSm.log("handleMessage: E msg.what=" + msg.what);

                /** Save the current message */
                mMsg = msg;

                /** State that processed the message */
                State msgProcessedState = null;
                if (mIsConstructionCompleted) {
                    /** Normal path */
                    msgProcessedState = processMsg(msg);
                } else if (!mIsConstructionCompleted && (mMsg.what == SM_INIT_CMD)
                        && (mMsg.obj == mSmHandlerObj)) {
                    /** Initial one time path. */
                    mIsConstructionCompleted = true;
                    invokeEnterMethods(0);
                } else {
                    throw new RuntimeException("StateMachine.handleMessage: "
                            + "The start method not called, received msg: " + msg);
                }
                performTransitions(msgProcessedState, msg);

                // We need to check if mSm == null here as we could be quitting.
                if (mDbg && mSm != null) mSm.log("handleMessage: X");
            }
        }
```

处理消息的方法就是processMsg，处理完成之后会调用performTransitions

```java
 private void performTransitions(State msgProcessedState, Message msg) {
            State orgState = mStateStack[mStateStackTopIndex].state;
            boolean recordLogMsg = mSm.recordLogRec(mMsg) && (msg.obj != mSmHandlerObj)

            State destState = mDestState;
            if (destState != null) {
                /**
                 * Process the transitions including transitions in the enter/exit methods
                 */
                while (true) {
                    StateInfo commonStateInfo = setupTempStateStackWithStatesToEnter(destState);
                    invokeEnterMethods(commonStateInfo);
                    int stateStackEnteringIndex = moveTempStateStackToStateStack();
                    invokeEnterMethods(stateStackEnteringIndex);

                    moveDeferredMessageAtFrontOfQueue();

                    if (destState != mDestState) {
                        // A new mDestState so continue looping
                        destState = mDestState;
                    } else {
                        // No change in mDestState so we're done
                        break;
                    }
                }
                mDestState = null;
            }
			....
        }
```

这其中的invokeEnterMethods和invokeEnterMethods可以发现运用了状态模式，其接口是IState

```java
public interface IState {
	void enter();
  	
  	void exit();
  
  	boolean processMessage(Message msg);
}
```

状态之间是不可以随意切换的，它们有一种层级关系，这些关系在StateMachine中被定义

```java
addState(mDefaultState);
addState(mInitialState, mDefaultState);
addState(mSupplicantStartingState, mDefaultState);
addState(mSupplicantStartedState, mDefaultState);
addState(mDriverStartingState, mSupplicantStartedState);
addState(mDriverStartedState, mSupplicantStartedState);
addState(mScanModeState, mDriverStartedState);
addState(mConnectModeState, mDriverStartedState);
addState(mL2ConnectedState, mConnectModeState);
addState(mObtainingIpState, mL2ConnectedState);
addState(mVerifyingLinkState, mL2ConnectedState);
addState(mConnectedState, mL2ConnectedState);
addState(mRoamingState, mL2ConnectedState);
addState(mDisconnectingState, mConnectModeState);
addState(mDisconnectedState, mConnectModeState);
addState(mWpsRunningState, mConnectModeState);
addState(mWaitForP2pDisableState, mSupplicantStartedState);
addState(mDriverStoppingState, mSupplicantStartedState);
addState(mDriverStoppedState, mSupplicantStartedState);
addState(mSupplicantStoppingState, mDefaultState);
addState(mSoftApStartingState, mDefaultState);
addState(mSoftApStartedState, mDefaultState);
addState(mTetheringState, mSoftApStartedState);
addState(mTetheredState, mSoftApStartedState);
addState(mUntetheringState, mSoftApStartedState);

setInitialState(mInitialState);
```

其中addState最终都会调用SmHandler中的addState方法，从而绑定关系