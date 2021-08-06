---
title: Android系统 Activity启动IDLE流程分析
date: 2021-07-21 09:22:20
tags:
---

9221系统启动时，tvlauncher出现两次`"ActivityRecord idle"`流程，在该函数中会执行内存回收操作，所以需要对该流程进行分析，确认是客制化修改后造成的还是Android原生逻辑，是否会对开机速度有影响。

## IdleHandler

在流程分析之前，首先理解下基础概念，什么是`IdleHandler`？

首先看下源码

```java
    /**
     * Callback interface for discovering when a thread is going to block
     * waiting for more messages.
     */
    public static interface IdleHandler {
        /**
         * Called when the message queue has run out of messages and will now
         * wait for more.  Return true to keep your idle handler active, false
         * to have it removed.  This may be called if there are still messages
         * pending in the queue, but they are all scheduled to be dispatched
         * after the current time.
         */
        boolean queueIdle();
    }
```

在注释中可以看到当消息队列空闲时会执行`IdelHandler`的`queueIdle()`方法，该方法返回一个`boolean`值，
如果为`false`则执行完毕之后移除这条消息，
如果为`true`则保留，等到下次空闲时会再次执行。

可以在`MessageQueue`的`next()`方法中看下具体的处理逻辑。

```java
Message next() {
        ......
        for (;;) {
            ......
            synchronized (this) {
                // 此处为正常消息队列的处理
                ......
                if (mQuitting) {
                    dispose();
                    return null;
                }
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }
                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }
            pendingIdleHandlerCount = 0;
            nextPollTimeoutMillis = 0;
        }
    }
```

所以可以看出`IdleHandler`的本质就是在当前线程 消息队列空闲时 做些事情。

要使用`IdleHandler`只需要调用`MessageQueue#addIdleHandler(IdleHandler handler)`方法即可。

## Activity启动流程

![](Android系统-Activity启动IDLE流程分析\startActivity.png)

## IDLE分析

### 第一次IDLE分析

通过对日志分析

```java
ActivityTaskManager: handleMessage: IDLE_TIMEOUT_MSG: r=ActivityRecord{195e76f u0 com.google.android.tvlauncher/.MainActivity t6}

ActivityTaskManager: Activity idle: ActivityRecord{195e76f u0 com.google.android.tvlauncher/.MainActivity t6}

ActivityTaskManager: ActivityRecord idle....com.google.android.tvlauncher====name = 101=====msg.obj = com.google.android.tvlauncher
```

可以看到第一次IDLE是`Handler`中接收到了`IDLE_TIMEOUT_MSG`消息，最终又调用`activityIdleInternal()`

`frameworks/base/services/core/java/com/android/server/wm/ActivityStackSupervisor.java`

```java
    case IDLE_TIMEOUT_MSG: {
        Slog.d(TAG_IDLE,"handleMessage: IDLE_TIMEOUT_MSG: r=" + msg.obj);
        // We don't at this point know if the activity is fullscreen, so we need to be
        // conservative and assume it isn't.
        activityIdleFromMessage((ActivityRecord) msg.obj, true /* fromTimeout */);
    } break;
```

```java
    private void activityIdleFromMessage(ActivityRecord idleActivity, boolean fromTimeout) {
        activityIdleInternal(idleActivity, fromTimeout,
                             fromTimeout /* processPausingActivities */, null /* config */);
    }

    void activityIdleInternal(ActivityRecord r, boolean fromTimeout,
              boolean processPausingActivities, Configuration config) {
        ......
        Slog.w(TAG,"ActivityRecord idle...."+r.packageName +"====name = " + msg.what);
        ......
        mHandler.removeMessages(IDLE_TIMEOUT_MSG, r)；
    }
```

这里知道了`IDLE_TIMEOUT_MSG`引起第一次IDLE，那么这个消息是什么时候发送的呢？经过对源码分析，得到如下时序图：



![](Android系统-Activity启动IDLE流程分析\startActivity-idle1.png)

在ActivityStack的`resumeTopActivityInnerLocked`方法中，当判断APP进程存在时，将会调度Activity resume事务。

```java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
	......
	
	if (next.attachedToProcess()) {
		next.completeResumeLocked();
	}
    
    ......
}
```

调用ActivityRecord中的`completeResumeLocked`函数：

```java
void completeResumeLocked() {
	mStackSupervisor.scheduleIdleTimeout(this);
}
```

最后调用ActivityStackSupervisor的`scheduleIdleTimeoutLocked`方法：

```java
void scheduleIdleTimeout(ActivityRecord next) {
    if (DEBUG_IDLE) Slog.d(TAG_IDLE, "scheduleIdleTimeout: Callers=" + Debug.getCallers(4));
    Message msg = mHandler.obtainMessage(IDLE_TIMEOUT_MSG, next);
    mHandler.sendMessageDelayed(msg, IDLE_TIMEOUT);
}
```

通过mHandler发送一个**10s**延时的**IDLE_TIMEOUT_MSG**消息。

当10s后没有取消这条消息的发送（即Activity没有启动完成，后面会有分析），那么就会执行一次IDLE。



### 第二次IDLE分析

这一次IDLE是当Activity启动完成，onResume执行完成，应用主线程处于空闲时，会通过IdleHandler调用ATMS接口，由ATMS再去调用`ActivityStackSupervisor`中的`activityIdleInternal`函数执行IDLE去回收内存。并在函数中取消`IDLE_TIMEOUT_MSG`的消息。

![](Android系统-Activity启动IDLE流程分析\startActivity-idle2.png)

`frameworks/base/core/java/android/app/ActivityThread.java`

```java
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
      ......
      Looper.myQueue().addIdleHandler(new Idler());
}
```

```java
private class Idler implements MessageQueue.IdleHandler {
        @Override
        public final boolean queueIdle() {
            
            try {
            	am.activityIdle(a.token, a.createdConfig, stopProfiling);
            	a.createdConfig = null;
            } catch (RemoteException ex) {
            	throw ex.rethrowFromSystemServer();
            }
               
            return false;
        }
}
```



## 分析总结

如果应用Activity启动时间低于10s，那么当IdleHandler执行IDLE流程后就会移除`IDLE_TIMEOUT_MSG`消息，也就不会触发两次IDLE。

根据日志分析，目前Activity的启动时间已经超过10s，所以才会出现两次执行IDLE的情况。

而超过10s的原因则是9221设备开机时内存占用过高，导致卡顿造成的。

```java
07-20 05:01:08.709  1102  1139 D TGuardMemoryManagerMemoryStrategy: getStrategy, type is LOW_MEMORY
07-20 05:01:08.709  1102  1139 D TGuardMemoryManagerMemoryStrategy: run LowMemoryStrategy
```

后期使用最新的9221版本重新测试3次，时间均为8秒左右，IDLE两次的情况也不会出现。
