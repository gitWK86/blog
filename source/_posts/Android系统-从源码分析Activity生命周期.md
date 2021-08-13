---
title: AndroidR-Activity生命周期管理
date: 2021-08-06 09:30:08
tags:
---



从Android-28开始，AMS向客户端进程有关Activity部分的通信封装成一个统一的事务来操作，不再直接使用客户端进程ApplicationThread的本地代理了。

<img src="Android系统-从源码分析Activity生命周期\ClientTransactionItem.png" style="zoom:80%;" />





`frameworks/base/services/core/java/com/android/server/wm/ActivityStackSupervisor.java`

```java
boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {
    			……
				// Create activity launch transaction.
                final ClientTransaction clientTransaction = ClientTransaction.obtain(
                        proc.getThread(), r.appToken);

                final DisplayContent dc = r.getDisplay().mDisplayContent;
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        // TODO: Have this take the merged configuration instead of separate global
                        // and override configs.
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                        r.getSavedState(), r.getPersistentSavedState(), results, newIntents,
                        dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                        r.assistToken, r.createFixedRotationAdjustmentsIfNeeded()));

                // Set desired final state.
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

                // Schedule transaction.
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);
    			……
}
```

创建一个ClientTransaction，并将LaunchActivityItem作为callback添加到mActivityCallbacks，而ResumeActivityItem或者PauseActivityItem作为LifecycleStateRequesst传递给mLifecycleStateRequest。最后通过ClientLifecycleManager执行该事务。

事务的执行是通过Binder调用对应Activity的ApplicationThread中，最后再由`TransactionExecutor`执行事务

`frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java`

```java
	public void execute(ClientTransaction transaction) {
        executeCallbacks(transaction);

        executeLifecycleState(transaction);
    }
```

```java
public void executeCallbacks(ClientTransaction transaction) {
	item.execute(mTransactionHandler, token, mPendingActions);
    item.postExecute(mTransactionHandler, token, mPendingActions);
    
    cycleToPath(r, postExecutionState, shouldExcludeLastTransition, transaction);
}

private void executeLifecycleState(ClientTransaction transaction) {
	// Cycle to the state right before the final requested state.
    cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */, transaction);

	// Execute the final transition with proper parameters.
    lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
    lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
}
```

可以看到先执行了Callback，然后执行LifecycleState，对应就是先执行了`LaunchActivityItem`，然后执行`ResumeActivityItem`。

### LaunchActivityItem (onCreate)

具体执行时调用的Item中的execute和postExecute两个函数。

`frameworks/base/core/java/android/app/servertransaction/LaunchActivityItem.java`

```java
 	@Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
        ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mIsForward,
                mProfilerInfo, client, mAssistToken, mFixedRotationAdjustments);
        client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }

    @Override
    public void postExecute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        client.countLaunchingActivities(-1);
    }
```

调用ActivityThread的handleLaunchActivity，这个函数中通过反射创建Activity，最后调用**onCreate**方法。

### ResumeActivityItem (onResume)

紧接着执行`ResumeActivityItem`的的execute和postExecute

`frameworks/base/core/java/android/app/servertransaction/ResumeActivityItem.java`

```java
 	@Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityResume");
        client.handleResumeActivity(token, true /* finalStateRequest */, mIsForward,
                "RESUME_ACTIVITY");
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }

    @Override
    public void postExecute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        try {
            // TODO(lifecycler): Use interface callback instead of AMS.
            ActivityTaskManager.getService().activityResumed(token);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
```

调用ActivityThread的handleResumeActivity，执行**onResume**方法，并且通知ATMS修改状态。

**那onStart流程去哪里了？**

### onStart

看`TransactionExecutor`中的源码，有一个cycleToPath函数一直没有分析，onStart流程就在这个里面。

`frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java`

```java
	private void cycleToPath(ActivityClientRecord r, int finish, boolean excludeLastState,
            ClientTransaction transaction) {
        final int start = r.getLifecycleState();
        final IntArray path = mHelper.getLifecyclePath(start, finish, excludeLastState);
        performLifecycleSequence(r, path, transaction);
    }
```

getLifecyclePath是一个关键点，它会根据start和finish计算出中间的生命周期路径。

首先对于生命周期都是以int值的形式表示。

`frameworks/base/core/java/android/app/servertransaction/ActivityLifecycleItem.java`

```java
public abstract class ActivityLifecycleItem extends ClientTransactionItem {
    public static final int UNDEFINED = -1;
    public static final int PRE_ON_CREATE = 0;
    public static final int ON_CREATE = 1;
    public static final int ON_START = 2;
    public static final int ON_RESUME = 3;
    public static final int ON_PAUSE = 4;
    public static final int ON_STOP = 5;
    public static final int ON_DESTROY = 6;
    public static final int ON_RESTART = 7;
}
```

执行`LaunchActivityItem`和`ResumeActivityItem`，就相当于`start = 1`，`finish = 3`，根据下面的计算逻辑计算中间生命周期，就会往`mLifecycleSequence`中add(2)，也就是增加了ON_START。

`frameworks/base/core/java/android/app/servertransaction/TransactionExecutorHelper.java`

```java
public IntArray getLifecyclePath(int start, int finish, boolean excludeLastState) {
    	mLifecycleSequence.clear();
        if (finish >= start) {
            if (start == ON_START && finish == ON_STOP) {
                // A case when we from start to stop state soon, we don't need to go
                // through the resumed, paused state.
                mLifecycleSequence.add(ON_STOP);
            } else {
                // just go there
                // 1-3, add(2), add ON_START
                for (int i = start + 1; i <= finish; i++) {
                    mLifecycleSequence.add(i);
                }
            }
        } else { // finish < start, can't just cycle down
            if (start == ON_PAUSE && finish == ON_RESUME) {
                // Special case when we can just directly go to resumed state.
                mLifecycleSequence.add(ON_RESUME);
            } else if (start <= ON_STOP && finish >= ON_START) {
                // Restart and go to required state.

                // Go to stopped state first.
                for (int i = start + 1; i <= ON_STOP; i++) {
                    mLifecycleSequence.add(i);
                }
                // Restart
                mLifecycleSequence.add(ON_RESTART);
                // Go to required state
                for (int i = ON_START; i <= finish; i++) {
                    mLifecycleSequence.add(i);
                }
            } else {
                // Relaunch and go to required state

                // Go to destroyed state first.
                for (int i = start + 1; i <= ON_DESTROY; i++) {
                    mLifecycleSequence.add(i);
                }
                // Go to required state
                for (int i = ON_CREATE; i <= finish; i++) {
                    mLifecycleSequence.add(i);
                }
            }
        }
}
```

最终调用`performLifecycleSequence`，根据上面的逻辑，传入的path参数中只有一个2，即ON_START，就会执行`ActivityThread`中的`handleStartActivity`，最终调用Activity的**onStart**

`frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java`

```java
	/** Transition the client through previously initialized state sequence. */
    private void performLifecycleSequence(ActivityClientRecord r, IntArray path,
            ClientTransaction transaction) {
        final int size = path.size();
        for (int i = 0, state; i < size; i++) {
            state = path.get(i);
            switch (state) {
                case ON_CREATE:
                    mTransactionHandler.handleLaunchActivity(r, mPendingActions,
                            null /* customIntent */);
                    break;
                case ON_START:
                    mTransactionHandler.handleStartActivity(r.token, mPendingActions);
                    break;
                case ON_RESUME:
                    mTransactionHandler.handleResumeActivity(r.token, false /* finalStateRequest */,
                            r.isForward, "LIFECYCLER_RESUME_ACTIVITY");
                    break;
                case ON_PAUSE:
                    mTransactionHandler.handlePauseActivity(r.token, false /* finished */,
                            false /* userLeaving */, 0 /* configChanges */, mPendingActions,
                            "LIFECYCLER_PAUSE_ACTIVITY");
                    break;
                case ON_STOP:
                    mTransactionHandler.handleStopActivity(r.token, 0 /* configChanges */,
                            mPendingActions, false /* finalStateRequest */,
                            "LIFECYCLER_STOP_ACTIVITY");
                    break;
                case ON_DESTROY:
                    mTransactionHandler.handleDestroyActivity(r.token, false /* finishing */,
                            0 /* configChanges */, false /* getNonConfigInstance */,
                            "performLifecycleSequence. cycling to:" + path.get(size - 1));
                    break;
                case ON_RESTART:
                    mTransactionHandler.performRestartActivity(r.token, false /* start */);
                    break;
                default:
                    throw new IllegalArgumentException("Unexpected lifecycle state: " + state);
            }
        }
    }
```

这样onCreate -》onStart -》onResume的完整流程就执行完成了。

### PauseActivityItem (onPause)

之前在看Activity生命周期的时候都会了解到A启动B时，对应的生命周期流程为

**(A)onPause→(B)onCreate→(B)onStart→(B)onResume→(A)onStop**

基于上面的Activity启动流程时序图，在执行到`ActivityStack.java`中的`resumeTopActivityInnerLocked`函数时

```java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
		……
		if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
            pausing |= startPausingLocked(userLeaving, false /* uiSleeping */, next);
        }
        ……
        // 执行 B Activity启动流程
}
```

判断是否已经有已经处于`resumed`状态的Activity，如果存在，就执行`startPausingLocked`函数让该Activity执行`onPause`流程，然后再继续去执行B Activity的启动流程。

```java
final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
            ActivityRecord resuming) {
    
     mAtmService.getLifecycleManager().scheduleTransaction(prev.app.getThread(),
             prev.appToken, PauseActivityItem.obtain(prev.finishing, userLeaving,
                      prev.configChangeFlags, pauseImmediately));
    
}
```

这里也是通过ClientTransaction传递PauseActivityItem事务，TransactionExecutor去执行，最终调用ActivityThread的`handlePauseActivity`方法，执行**onPause**，并通知ATMS Activity的状态修改。

`frameworks/base/core/java/android/app/servertransaction/PauseActivityItem.java`

```java
  	@Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        client.handlePauseActivity(token, mFinished, mUserLeaving, mConfigChanges, pendingActions,
                "PAUSE_ACTIVITY_ITEM");
    }

    @Override
    public void postExecute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        try {
            // TODO(lifecycler): Use interface callback instead of AMS.
            ActivityTaskManager.getService().activityPaused(token);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
```

#### 为什么要先将A Activity onPause？

一个Activity或多或少会占有系统资源，而在官方的建议中，onPause方法将会释放掉很多系统资源，为切换Activity提供流畅性的保障，而不需要再等多两个阶段，这样做切换更快。

### StopActivityItem (onStop)

![](Android系统-从源码分析Activity生命周期\ActivityStop.png)

在上一步onPause执行时，最终调用了ATMS的`activityPaused`函数。

当执行完pause方法之后会执行completePauseLocked方法，其中会判断当前ActivityRecord已经绑定了App端的数据，说明已经启动了，并且当前的ActivityRecord的visible为false，或者点击了锁屏使其睡眠，都会调用ActivityStatck.addToStopping。然后会调用`ActivityStackSupervisor`中的IDLE流程`activityIdleInternal`，最后调用`processStoppingAndFinshingActivities`函数，其中就会调用`ActivityRecord`的`stopIfPossible`函数，最终调用我们熟悉的`StopActivityItem`通知Activity执行**onStop**。



### DestroyActivityItem (onDestory)

onDestory的流程在上面已经看到了，流程基本一致，就不再分析。



onCreate -》onStart -》onResume -》onPause -》onStop -》onDestory的流程分析结束。

