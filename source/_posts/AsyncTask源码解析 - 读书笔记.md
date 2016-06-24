---
title: AsyncTask源码解析 - 读书笔记
date: 2016年6月21日21:44:05
categories: 读书笔记
tags: [Android源码设计模式解析与实战，读书笔记，Android]
---

# 来自Android源码设计模式解析与实战的第十五章，抓住问题核心——模板方法模式

---

## 1. AsyncTask源码

我们先看执行异步任务的入口execute方法

```java
	public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }

	public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        ...
        mStatus = Status.RUNNING;
        onPreExecute();
        mWorker.mParams = params;
        exec.execute(mFuture);
        return this;
    }
```

在executeOnExecutor中，先执行了onPreExecute方法，然后将params传给mWorker对象，并且执行exec.execute(mFuture)方法。接下来我们看mWorker和mFuture。

```java
		mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                return postResult(result);
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
            	postResultIfNotInvoked(get());
            }
        };
```

mFuture包装了mWorker对象，在这个mFuture对象的run函数中又会调用mWorker对象的call方法，在call方法中又调用了doInBackground函数。