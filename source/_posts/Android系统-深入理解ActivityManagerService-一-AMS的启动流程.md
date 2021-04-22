---
title: Android系统-深入理解ActivityManagerService(一)AMS的启动流程
date: 2021-03-27 19:18:08
categories: 
- Android系统
tags:
- AMS
- 系统
---

AMS应该是作为Android开发经常听到的一个系统服务了，它是系统的引导服务，Android中**最核心的服务**，主要负责系统中四大组件的**启动、切换、调度及应用进程的管理和调度**等工作，在Android系统中非常重要。所以，接下来就深入了解一下AMS的世界，首先我们从它的启动流程开始。

## AMS的启动

在之前的文章[Android包管理机制(一)PMS服务启动]()中，我们有提到在SystemServer中启动引导服务，而AMS也是系统的引导服务，所以首先去看下SystemServer。

```java
private void startBootstrapServices() {
        //启动installer服务
        Installer installer = mSystemServiceManager.startService(Installer.class);
        // AMS
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);
  			
  			mActivityManagerService.initPowerManagement();
  			mActivityManagerService.setSystemProcess();
 }
```

启动AMS首先调用SystemServiceManager的startService方法，参数为ActivityManagerService.Lifecycle.class

`frameworks/base/services/core/java/com/android/server/SystemServiceManager.java`

```java
 @SuppressWarnings("unchecked")
    public <T extends SystemService> T startService(Class<T> serviceClass) {
        try {
            final T service;
            try {
                Constructor<T> constructor = serviceClass.getConstructor(Context.class);
                service = constructor.newInstance(mContext);
            } catch (InstantiationException ex) {
            ......
            }

            // Register it.
            mServices.add(service);

            // Start it.
            try {
                service.onStart();
            } catch (RuntimeException ex) {
                throw new RuntimeException("Failed to start service " + name
                        + ": onStart threw an exception", ex);
            }
            return service;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        }
    }
```

在startService中，首先根据传进来的Lifecycle得到它的构造器constructor，再调用constructor的newInstance方法来创建Lifecycle类型的service对象。接着将刚创建的service添加到ArrayList类型的mServices对象中来完成注册。最后调用service的onStart方法来启动service，并返回该service。

`frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java`

```java
 public static final class Lifecycle extends SystemService {
        private final ActivityManagerService mService;

        public Lifecycle(Context context) {
            super(context);
            mService = new ActivityManagerService(context);
        }

        @Override
        public void onStart() {
            mService.start();
        }

        public ActivityManagerService getService() {
            return mService;
        }
    }
```

结合Lifecycle的代码，可以看到，startService中newInstance实际调用了Lifecycle(Context context){}，创建了ActivityManagerService实例mService，当调用Lifecycle类型的service的onStart方法时，实际上是调用了AMS的start方法。

所以获取AMS实例代码，最后getService()会返回Lifecycle中创建的mService。

```java
mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
```

## AMS构造函数

```java
    public ActivityManagerService(Context systemContext) {
        mContext = systemContext;
        mFactoryTest = FactoryTest.getMode();
        mSystemThread = ActivityThread.currentActivityThread();

        mPermissionReviewRequired = mContext.getResources().getBoolean(
                com.android.internal.R.bool.config_permissionReviewRequired);

        //创建名为"ActivityManager"的前台线程，并获取mHandler
        mHandlerThread = new ServiceThread(TAG,
                android.os.Process.THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
        mHandlerThread.start();
        mHandler = new MainHandler(mHandlerThread.getLooper());
        
        //通过UiThread类，创建名为"android.ui"的线程
        mUiHandler = new UiHandler();

        /* static; one-time init here */
        if (sKillHandler == null) {
            //创建名为"ActivityManager:kill"的后台线程，sKillHandler
            sKillThread = new ServiceThread(TAG + ":kill",
                    android.os.Process.THREAD_PRIORITY_BACKGROUND, true /* allowIo */);
            sKillThread.start();
            sKillHandler = new KillHandler(sKillThread.getLooper());
        }

        //前台广播接收器，在运行超过10s将放弃执行
        mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "foreground", BROADCAST_FG_TIMEOUT, false);
        //后台广播接收器，在运行超过60s将放弃执行
        mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "background", BROADCAST_BG_TIMEOUT, true);
        mBroadcastQueues[0] = mFgBroadcastQueue;
        mBroadcastQueues[1] = mBgBroadcastQueue;

        //创建ActiveServices
        mServices = new ActiveServices(this);
        //创建ProviderMap
        mProviderMap = new ProviderMap(this);
        mAppErrors = new AppErrors(mContext, this);

        // TODO: Move creation of battery stats service outside of activity manager service.
        //创建目录/data/system
        File dataDir = Environment.getDataDirectory();
        File systemDir = new File(dataDir, "system");
        systemDir.mkdirs();
        // 电池状态服务
        mBatteryStatsService = new BatteryStatsService(systemDir, mHandler);
        mBatteryStatsService.getActiveStatistics().readLocked();
        mBatteryStatsService.scheduleWriteToDisk();
        mOnBattery = DEBUG_POWER ? true
                : mBatteryStatsService.getActiveStatistics().getIsOnBattery();
        mBatteryStatsService.getActiveStatistics().setCallback(this);

        //创建进程统计服务，信息保存在目录/data/system/procstats
        mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));

        //应用程序的操作（权限）管理
        mAppOpsService = new AppOpsService(new File(systemDir, "appops.xml"), mHandler);
        mAppOpsService.startWatchingMode(AppOpsManager.OP_RUN_IN_BACKGROUND, null,
                new IAppOpsCallback.Stub() {
                    @Override public void opChanged(int op, int uid, String packageName) {
                        if (op == AppOpsManager.OP_RUN_IN_BACKGROUND && packageName != null) {
                            if (mAppOpsService.checkOperation(op, uid, packageName)
                                    != AppOpsManager.MODE_ALLOWED) {
                                runInBackgroundDisabled(uid);
                            }
                        }
                    }
                });

        mGrantFile = new AtomicFile(new File(systemDir, "urigrants.xml"));

        mUserController = new UserController(this);

        GL_ES_VERSION = SystemProperties.getInt("ro.opengles.version",
            ConfigurationInfo.GL_ES_VERSION_UNDEFINED);

        if (SystemProperties.getInt("sys.use_fifo_ui", 0) != 0) {
            mUseFifoUiScheduling = true;
        }

        mTrackingAssociations = "1".equals(SystemProperties.get("debug.track-associations"));

        mConfiguration.setToDefaults();
        mConfiguration.setLocales(LocaleList.getDefault());

        mConfigurationSeq = mConfiguration.seq = 1;
      
        //CPU使用情况的追踪器执行初始化
        mProcessCpuTracker.init();

        mCompatModePackages = new CompatModePackages(this, systemDir, mHandler);
        mIntentFirewall = new IntentFirewall(new IntentFirewallInterface(), mHandler);
        mStackSupervisor = new ActivityStackSupervisor(this);
        mActivityStarter = new ActivityStarter(this, mStackSupervisor);
        mRecentTasks = new RecentTasks(this, mStackSupervisor);

        //创建名为"CpuTracker"的线程
        mProcessCpuThread = new Thread("CpuTracker") {
            @Override
            public void run() {
                while (true) {
                    try {
                        try {
                            synchronized(this) {
                                final long now = SystemClock.uptimeMillis();
                                long nextCpuDelay = (mLastCpuTime.get()+MONITOR_CPU_MAX_TIME)-now;
                                long nextWriteDelay = (mLastWriteTime+BATTERY_STATS_TIME)-now;
                                //Slog.i(TAG, "Cpu delay=" + nextCpuDelay
                                //        + ", write delay=" + nextWriteDelay);
                                if (nextWriteDelay < nextCpuDelay) {
                                    nextCpuDelay = nextWriteDelay;
                                }
                                if (nextCpuDelay > 0) {
                                    mProcessCpuMutexFree.set(true);
                                    this.wait(nextCpuDelay);
                                }
                            }
                        } catch (InterruptedException e) {
                        }
                        updateCpuStatsNow();
                    } catch (Exception e) {
                        Slog.e(TAG, "Unexpected exception collecting process stats", e);
                    }
                }
            }
        };

        //添加Watchdog监控
        Watchdog.getInstance().addMonitor(this);
        Watchdog.getInstance().addThread(mHandler);
    }
```

在构造函数中进行各个变量的初始化，创建ActivityManager，ActivityManager:kill，android.ui，CpuTracker线程

## AMS.start

```java
private void start() {
        Process.removeAllProcessGroups();//移除所有的进程组
        mProcessCpuThread.start();//启动CpuTracker线程

        mBatteryStatsService.publish(mContext);//启动电池统计服务
        mAppOpsService.publish(mContext);
        //创建LocalService，并添加到LocalServices
        LocalServices.addService(ActivityManagerInternal.class, new LocalService());
    }
```

## AMS.setSystemProcess

```java
   public void setSystemProcess() {
        try {
            ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true);
            ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
            ServiceManager.addService("meminfo", new MemBinder(this)); //内存
            ServiceManager.addService("gfxinfo", new GraphicsBinder(this)); //图像信息
            ServiceManager.addService("dbinfo", new DbBinder(this)); //数据库
            if (MONITOR_CPU_USAGE) {
                ServiceManager.addService("cpuinfo", new CpuBinder(this)); //CPU
            }
            ServiceManager.addService("permission", new PermissionController(this)); //权限
            ServiceManager.addService("processinfo", new ProcessInfoService(this)); //进程服务

            ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
                    "android", STOCK_PM_FLAGS | MATCH_SYSTEM_ONLY);
            mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());

            synchronized (this) {
                ProcessRecord app = newProcessRecordLocked(info, info.processName, false, 0);
                app.persistent = true;
                app.pid = MY_PID;
                app.maxAdj = ProcessList.SYSTEM_ADJ;
                app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
                synchronized (mPidsSelfLocked) {
                    mPidsSelfLocked.put(app.pid, app);
                }
                updateLruProcessLocked(app, false, null);
                updateOomAdjLocked();
            }
        } catch (PackageManager.NameNotFoundException e) {
            throw new RuntimeException(
                    "Unable to find android system package", e);
        }
    }
```

setSystemProcess就是注册各种服务到ServiceManager，加载名为“android”的package，然后为“android”创建ProcessRecord，用于描述进程的数据结构。

## startOtherServices

上面的流程是在startBootstrapServices时，执行的AMS的创建等操作，但AMS的流程还没结束，在startOtherServices中仍然有AMS的后续操作。

```java
private void startOtherServices() {
    //安装系统Provider
    mActivityManagerService.installSystemProviders();
    mActivityManagerService.setWindowManager(wm);
    mActivityManagerService.systemReady(new Runnable() {
    });
}
```

```java
public final void installSystemProviders() {
        List<ProviderInfo> providers;
        synchronized (this) {
            ProcessRecord app = mProcessNames.get("system", Process.SYSTEM_UID);
            providers = generateApplicationProvidersLocked(app);
            if (providers != null) {
                for (int i=providers.size()-1; i>=0; i--) {
                    ProviderInfo pi = (ProviderInfo)providers.get(i);
                    if ((pi.applicationInfo.flags&ApplicationInfo.FLAG_SYSTEM) == 0) {
                        Slog.w(TAG, "Not installing system proc provider " + pi.name
                                + ": not system .apk");
                        //移除非系统的provider
                        providers.remove(i);
                    }
                }
            }
        }
        if (providers != null) {
            //安装所有的系统provider
            mSystemThread.installSystemProviders(providers);
        }

        // 创建核心Settings Observer，用于监控Settings的改变。
        mCoreSettingsObserver = new CoreSettingsObserver(this);
        //监控字体的变化
        mFontScaleSettingObserver = new FontScaleSettingObserver();
    }
```

##systemReady

在startOtherServices中调用mActivityManagerService.systemReady(），参数为Runable类型的goingCallback

在AMS中systemReady函数可以分为三个部分

```
public void systemReady(final Runnable goingCallback) {
    before goingCallback; 
    goingCallback.run(); 
    after goingCallback; 
}
```

###before goingCallback

```java
   synchronized(this) {
            if (mSystemReady) {//第一次进入为false，不执行下面代码
                // If we're done calling all the receivers, run the next "boot phase" passed in
                // by the SystemServer
                if (goingCallback != null) {
                    goingCallback.run();
                }
                return;
            }

            mLocalDeviceIdleController
                    = LocalServices.getService(DeviceIdleController.LocalService.class);

            // Make sure we have the current profile info, since it is needed for security checks.
            mUserController.onSystemReady();
            mRecentTasks.onSystemReadyLocked();
            mAppOpsService.systemReady();
            mSystemReady = true;
        }

        ArrayList<ProcessRecord> procsToKill = null;
        synchronized(mPidsSelfLocked) {
            for (int i=mPidsSelfLocked.size()-1; i>=0; i--) {
                ProcessRecord proc = mPidsSelfLocked.valueAt(i);
                //非persistent进程,加入procsToKill
                if (!isAllowedWhileBooting(proc.info)){
                    if (procsToKill == null) {
                        procsToKill = new ArrayList<ProcessRecord>();
                    }
                    procsToKill.add(proc);
                }
            }
        }

        synchronized(this) {
            if (procsToKill != null) {
                for (int i=procsToKill.size()-1; i>=0; i--) {
                    //杀掉procsToKill中的进程, 杀掉进程且不允许重启
                    ProcessRecord proc = procsToKill.get(i);
                    Slog.i(TAG, "Removing system update proc: " + proc);
                    removeProcessLocked(proc, true, false, "system update done");
                }
            }

            // Now that we have cleaned up any update processes, we
            // are ready to start launching real processes and know that
            // we won't trample on them any more.
            //process处于ready状态
            mProcessesReady = true;
        }

        Slog.i(TAG, "System now ready");
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_AMS_READY,
            SystemClock.uptimeMillis());
```

非persistent进程加入procsToKill，杀掉procsToKill中的进程, 杀掉进程且不允许重启

### goingCallback.run()

```java
if (goingCallback != null) goingCallback.run();
```

这个goingCallback是在startOtherServices时传入的Callback

```java
private void startOtherServices() {
	mActivityManagerService.systemReady(new Runnable() {
            @Override
            public void run() {
                Slog.i(TAG, "Making services ready");
                mSystemServiceManager.startBootPhase(
                        SystemService.PHASE_ACTIVITY_MANAGER_READY);
                Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "PhaseActivityManagerReady");

                Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "StartObservingNativeCrashes");
                try {
                    // 开始监听Native Crash
                    mActivityManagerService.startObservingNativeCrashes();
                } catch (Throwable e) {
                    reportWtf("observing native crashes", e);
                }
                Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);

                if (!mOnlyCore) {
                    //启动WebView
                    Slog.i(TAG, "WebViewFactory preparation");
                    Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "WebViewFactoryPreparation");
                    mWebViewUpdateService.prepareWebViewInSystemServer();
                    Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
                }

                Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "StartSystemUI");
                try {
                    // 启动SystemUI
                    startSystemUi(context);
                } catch (Throwable e) {
                    reportWtf("starting System UI", e);
                }
                Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
              
              
                // 执行一系列服务的systemReady方法
                Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "MakeNetworkScoreReady");
                try {
                    if (networkScoreF != null) networkScoreF.systemReady();
                } catch (Throwable e) {
                    reportWtf("making Network Score Service ready", e);
                }
                Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
                Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "MakeNetworkManagementServiceReady");
                try {
                    if (networkManagementF != null) networkManagementF.systemReady();
                } catch (Throwable e) {
                    reportWtf("making Network Managment Service ready", e);
                }
                Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
                Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "MakeNetworkStatsServiceReady");
                try {
                    if (networkStatsF != null) networkStatsF.systemReady();
                } catch (Throwable e) {
                    reportWtf("making Network Stats Service ready", e);
                }
                Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
                Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "MakeNetworkPolicyServiceReady");
                try {
                    if (networkPolicyF != null) networkPolicyF.systemReady();
                } catch (Throwable e) {
                    reportWtf("making Network Policy Service ready", e);
                }
                Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
                Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "MakeConnectivityServiceReady");
                try {
                    if (connectivityF != null) connectivityF.systemReady();
                } catch (Throwable e) {
                    reportWtf("making Connectivity Service ready", e);
                }
                Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);

              
                // Watchdog开始工作
                Watchdog.getInstance().start();

                //各种系统服务启动
                Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
                Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "PhaseThirdPartyAppsCanStart");
                mSystemServiceManager.startBootPhase(
                        SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);

                try {
                    if (locationF != null) locationF.systemRunning();
                } catch (Throwable e) {
                    reportWtf("Notifying Location Service running", e);
                }
                try {
                    if (countryDetectorF != null) countryDetectorF.systemRunning();
                } catch (Throwable e) {
                    reportWtf("Notifying CountryDetectorService running", e);
                }
                try {
                    if (networkTimeUpdaterF != null) networkTimeUpdaterF.systemRunning();
                } catch (Throwable e) {
                    reportWtf("Notifying NetworkTimeService running", e);
                }
                try {
                    if (commonTimeMgmtServiceF != null) {
                        commonTimeMgmtServiceF.systemRunning();
                    }
                } catch (Throwable e) {
                    reportWtf("Notifying CommonTimeManagementService running", e);
                }
                try {
                    if (atlasF != null) atlasF.systemRunning();
                } catch (Throwable e) {
                    reportWtf("Notifying AssetAtlasService running", e);
                }
                try {
                    // TODO(BT) Pass parameter to input manager
                    if (inputManagerF != null) inputManagerF.systemRunning();
                } catch (Throwable e) {
                    reportWtf("Notifying InputManagerService running", e);
                }
                try {
                    if (telephonyRegistryF != null) telephonyRegistryF.systemRunning();
                } catch (Throwable e) {
                    reportWtf("Notifying TelephonyRegistry running", e);
                }
                try {
                    if (mediaRouterF != null) mediaRouterF.systemRunning();
                } catch (Throwable e) {
                    reportWtf("Notifying MediaRouterService running", e);
                }

                try {
                    if (mmsServiceF != null) mmsServiceF.systemRunning();
                } catch (Throwable e) {
                    reportWtf("Notifying MmsService running", e);
                }

                try {
                    if (networkScoreF != null) networkScoreF.systemRunning();
                } catch (Throwable e) {
                    reportWtf("Notifying NetworkScoreService running", e);
                }
                Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
            }
        });
}
```

这个过程会

- 启动监听Native Crash
- 启动WebView，并且会创建进程，这是zygote正式创建的第一个进程
- 启动SystemUI
- 启动WatchDog
- 执行一系列服务的systemRunning方法启动

### after goingCallback

```java

        //回调所有SystemService的onStartUser()方法
        mSystemServiceManager.startUser(currentUserId);

        synchronized (this) {
            // Only start up encryption-aware persistent apps; once user is
            // unlocked we'll come back around and start unaware apps
            // 启动 persistent app
            startPersistentApps(PackageManager.MATCH_DIRECT_BOOT_AWARE);

            // Start up initial activity.
            mBooting = true;
            // Enable home activity for system user, so that the system can always boot
            if (UserManager.isSplitSystemUser()) {
                ComponentName cName = new ComponentName(mContext, SystemUserHomeActivity.class);
                try {
                    AppGlobals.getPackageManager().setComponentEnabledSetting(cName,
                            PackageManager.COMPONENT_ENABLED_STATE_ENABLED, 0,
                            UserHandle.USER_SYSTEM);
                } catch (RemoteException e) {
                    throw e.rethrowAsRuntimeException();
                }
            }
             // 启动 Home Activity
            startHomeActivityLocked(currentUserId, "systemReady");

            try {
                if (AppGlobals.getPackageManager().hasSystemUidErrors()) {
                    Slog.e(TAG, "UIDs on the system are inconsistent, you need to wipe your"
                            + " data partition or your device will be unstable.");
                    mUiHandler.obtainMessage(SHOW_UID_ERROR_UI_MSG).sendToTarget();
                }
            } catch (RemoteException e) {
            }

            if (!Build.isBuildConsistent()) {
                Slog.e(TAG, "Build fingerprint is not consistent, warning user");
                mUiHandler.obtainMessage(SHOW_FINGERPRINT_ERROR_UI_MSG).sendToTarget();
            }

            long ident = Binder.clearCallingIdentity();
            try {
                 //system发送广播USER_STARTED
                Intent intent = new Intent(Intent.ACTION_USER_STARTED);
                intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                        | Intent.FLAG_RECEIVER_FOREGROUND);
                intent.putExtra(Intent.EXTRA_USER_HANDLE, currentUserId);
                broadcastIntentLocked(null, null, intent,
                        null, null, 0, null, null, null, AppOpsManager.OP_NONE,
                        null, false, false, MY_PID, Process.SYSTEM_UID,
                        currentUserId);
                //system发送广播USER_STARTING
                intent = new Intent(Intent.ACTION_USER_STARTING);
                intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
                intent.putExtra(Intent.EXTRA_USER_HANDLE, currentUserId);
                broadcastIntentLocked(null, null, intent,
                        null, new IIntentReceiver.Stub() {
                            @Override
                            public void performReceive(Intent intent, int resultCode, String data,
                                    Bundle extras, boolean ordered, boolean sticky, int sendingUser)
                                    throws RemoteException {
                            }
                        }, 0, null, null,
                        new String[] {INTERACT_ACROSS_USERS}, AppOpsManager.OP_NONE,
                        null, true, false, MY_PID, Process.SYSTEM_UID, UserHandle.USER_ALL);
            } catch (Throwable t) {
                Slog.wtf(TAG, "Failed sending first user broadcasts", t);
            } finally {
                Binder.restoreCallingIdentity(ident);
            }
            // 恢复显示Top Activity
            mStackSupervisor.resumeFocusedStackTopActivityLocked();
            mUserController.sendUserSwitchBroadcastsLocked(-1, currentUserId);
        }
```

该阶段主要功能：

- 回调所有SystemService的onStartUser()方法；
- 启动persistent app（常驻内存应用）；
- 启动home Activity;
- 发送广播USER_STARTED和USER_STARTING；
- 恢复栈顶Activity;
- 发送广播USER_SWITCHED；

## startHomeActivityLocked

```java
 boolean startHomeActivityLocked(int userId, String reason) {
        if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL
                && mTopAction == null) {
            // We are running in factory test mode, but unable to find
            // the factory test app, so just sit around displaying the
            // error message and don't try to start anything.
            return false;
        }
        //home intent有CATEGORY_HOME
        Intent intent = getHomeIntent();
        ActivityInfo aInfo = resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);
        if (aInfo != null) {
            intent.setComponent(new ComponentName(aInfo.applicationInfo.packageName, aInfo.name));
            // Don't do this if the home app is currently being
            // instrumented.
            aInfo = new ActivityInfo(aInfo);
            aInfo.applicationInfo = getAppInfoForUser(aInfo.applicationInfo, userId);
            ProcessRecord app = getProcessRecordLocked(aInfo.processName,
                    aInfo.applicationInfo.uid, true);
            if (app == null || app.instrumentationClass == null) {
                intent.setFlags(intent.getFlags() | Intent.FLAG_ACTIVITY_NEW_TASK);
                 //启动桌面Activity
                mActivityStarter.startHomeActivityLocked(intent, aInfo, reason);
            }
        } else {
            Slog.wtf(TAG, "No home screen found for " + intent, new Throwable());
        }
        return true;
    }
```



## 总结

AMS启动流程

1. SystemServer调用Lifecycle 创建ActivityManagerService实例
2. AMS构造函数初始化变量，创建ActivityManager，ActivityManager:kill，android.ui，CpuTracker线程
3. setSystemProcess：注册AMS、meminfo、cpuinfo等服务到ServiceManager
4. installSystemProviderss，加载SettingsProvider
5. 启动SystemUIService，再调用一系列服务的systemReady()方法
6. 启动HomeActivity(Launcher)

