---
title: Android包管理机制(一)PMS服务启动
date: 2021-03-17 12:00:59
categories: 
- Android系统
tags:
- Android
- PMS
- 系统
---

PackageManagerService(简称PMS)，是Android系统中核心服务之一，管理着所有跟package相关的工作，常见的比如安装、卸载应用。 

## SyetemServer处理

SystemServer启动过程中启动服务

```Java
private void run() {
    try {
        ...
        //创建消息Looper
         Looper.prepareMainLooper();
        //加载了动态库libandroid_servers.so
        System.loadLibrary("android_servers");
        performPendingShutdown();
        // 创建系统的Context
        createSystemContext();
        // 创建SystemServiceManager
        mSystemServiceManager = new SystemServiceManager(mSystemContext);
        mSystemServiceManager.setRuntimeRestarted(mRuntimeRestart);
        LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
        SystemServerInitThreadPool.get();
    } finally {
        traceEnd(); 
    }
    try {
        traceBeginAndSlog("StartServices");
        //启动引导服务
        startBootstrapServices();
        //启动核心服务
        startCoreServices();
        //启动其他服务
        startOtherServices();
        SystemServerInitThreadPool.shutdown();
    } catch (Throwable ex) {
        Slog.e("System", "******************************************");
        Slog.e("System", "************ Failure starting system services", ex);
        throw ex;
    } finally {
        traceEnd();
    }
    ...
}
```

从上面可以看出，系统服务分为了三种类型，分别是引导服务、核心服务和其他服务，本文要了解的PMS属于引导服务。

| 引导服务               | 作用                                                         |
| ---------------------- | ------------------------------------------------------------ |
| Installer              | 系统安装apk时的一个服务类，启动完成Installer服务之后才能启动其他的系统服务 |
| ActivityManagerService | 负责四大组件的启动、切换、调度。                             |
| PowerManagerService    | 计算系统中和Power相关的计算，然后决策系统应该如何反应        |
| LightsService          | 管理和显示背光LED                                            |
| DisplayManagerService  | 用来管理所有显示设备                                         |
| UserManagerService     | 多用户模式管理                                               |
| SensorService          | 为系统提供各种感应器服务                                     |
| PackageManagerService  | 用来对apk进行安装、解析、删除、卸载等等操作                  |



```java
    private void startBootstrapServices() {
        //启动installer服务
        Installer installer = mSystemServiceManager.startService(Installer.class);
        // AMS
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);
        
        //POWERMS
        mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
        Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "InitPowerManagement");
        mActivityManagerService.initPowerManagement();
        Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        mSystemServiceManager.startService(LightsService.class);
        mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);
        mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);

        // Only run "core" apps if we're encrypting the device.
        // 表示加密了设备，这时mOnlyCore的值为true，表示只运行“核心”程序，创建一个极简的启动环境。
        String cryptState = SystemProperties.get("vold.decrypt");

        mIsAlarmBoot = SystemProperties.getBoolean("ro.alarm_boot", false);
        if (ENCRYPTING_STATE.equals(cryptState)) {
            Slog.w(TAG, "Detected encryption in progress - only parsing core apps");
            mOnlyCore = true;
        } else if (ENCRYPTED_STATE.equals(cryptState)) {
            Slog.w(TAG, "Device encrypted - only parsing core apps");
            mOnlyCore = true;
        } else if (mIsAlarmBoot) {
            mOnlyCore = true;
        }

        if (RegionalizationEnvironment.isSupported()) {
            Slog.i(TAG, "Regionalization Service");
            RegionalizationService regionalizationService = new RegionalizationService();
            ServiceManager.addService("regionalization", regionalizationService);
        }

        // Start the package manager.
        //启动PMS
        traceBeginAndSlog("StartPackageManagerService");
        //创建PMS
        mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        // 表示PMS是否首次被启动，这个参数会在WMS创建时使用
        mFirstBoot = mPackageManagerService.isFirstBoot();
        mPackageManager = mSystemContext.getPackageManager();
        Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);

        if (!mOnlyCore) {
            boolean disableOtaDexopt = SystemProperties.getBoolean("config.disable_otadexopt",
                    false);
            if (!disableOtaDexopt) {
                traceBeginAndSlog("StartOtaDexOptService");
                try {
                    OtaDexoptService.main(mSystemContext, mPackageManagerService);
                } catch (Throwable e) {
                    reportWtf("starting OtaDexOptService", e);
                } finally {
                    Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
                }
            }
        }

        traceBeginAndSlog("StartUserManagerService");
        mSystemServiceManager.startService(UserManagerService.LifeCycle.class);
        Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        AttributeCache.init(mSystemContext);
        mActivityManagerService.setSystemProcess();
        startSensorService();
    }
```





## PMS

### 构造方法

```java
 public static PackageManagerService main(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        // Self-check for initial settings.
        PackageManagerServiceCompilerMapping.checkProperties();
			  //初始化PMS	
        PackageManagerService m = new PackageManagerService(context, installer,
                factoryTest, onlyCore);
        m.enableSystemUserPackages();
        //将package服务注册到ServiceManager大管家
        ServiceManager.addService("package", m);
        return m;
    }
```

PMS的构造方法一共有700多行，在代码中，将PMS的构造流程分为了5个阶段，每个阶段会使用EventLog.writeEvent打印系统日志。

1. BOOT_PROGRESS_PMS_START（开始阶段）
2. BOOT_PROGRESS_PMS_SYSTEM_SCAN_START（扫描系统阶段）
3. BOOT_PROGRESS_PMS_DATA_SCAN_START（扫描Data分区阶段）
4. BOOT_PROGRESS_PMS_SCAN_END（扫描结束阶段）
5. BOOT_PROGRESS_PMS_READY（准备阶段）

### 1、开始阶段

BOOT_PROGRESS_PMS_START

```Java
   public PackageManagerService(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        //打印开始日志
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_START,
                SystemClock.uptimeMillis());

        if (mSdkVersion <= 0) {
            Slog.w(TAG, "**** ro.build.version.sdk not set!");
        }

        mContext = context;

        mPermissionReviewRequired = context.getResources().getBoolean(
                R.bool.config_permissionReviewRequired);

        mFactoryTest = factoryTest;
        mOnlyCore = onlyCore;
        //用于存储屏幕的相关信息
        mMetrics = new DisplayMetrics();
        //创建Settings对象 (1)
        mSettings = new Settings(mPackages);
        // 添加system, phone, log, nfc, bluetooth, shell这六种shareUserId到mSettings；
        mSettings.addSharedUserLPw("android.uid.system", Process.SYSTEM_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.phone", RADIO_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.log", LOG_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.nfc", NFC_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.bluetooth", BLUETOOTH_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.shell", SHELL_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
     
     
     		......

        mInstaller = installer;
        //创建Dex优化工具类
        mPackageDexOptimizer = new PackageDexOptimizer(installer, mInstallLock, context,
                "*dexopt*");
        mMoveCallbacks = new MoveCallbacks(FgThread.get().getLooper());

        mOnPermissionChangeListeners = new OnPermissionChangeListeners(
                FgThread.get().getLooper());

        getDefaultDisplayMetrics(context, mMetrics);
        //得到全局系统配置信息
        SystemConfig systemConfig = SystemConfig.getInstance();
     		//获取全局的groupId 
        mGlobalGids = systemConfig.getGlobalGids();
     		//获取系统权限
        mSystemPermissions = systemConfig.getSystemPermissions();
        mAvailableFeatures = systemConfig.getAvailableFeatures();

        mProtectedPackages = new ProtectedPackages(mContext);

        //安装APK时需要的锁，保护所有对installd的访问。
        synchronized (mInstallLock) {
        //更新APK时需要的锁，保护内存中已经解析的包信息等内容
        synchronized (mPackages) {
            //创建后台线程ServiceThread
            mHandlerThread = new ServiceThread(TAG,
                    Process.THREAD_PRIORITY_BACKGROUND, true /*allowIo*/);
            mHandlerThread.start();
            //创建PackageHandler绑定到ServiceThread的消息队列
            mHandler = new PackageHandler(mHandlerThread.getLooper());
            mProcessLoggingHandler = new ProcessLoggingHandler();
            //将PackageHandler添加到Watchdog的检测集中
            Watchdog.getInstance().addThread(mHandler, WATCHDOG_TIMEOUT);

            mDefaultPermissionPolicy = new DefaultPermissionGrantPolicy(this);
  					
            //在Data分区创建一些目录
            File dataDir = Environment.getDataDirectory();
            mAppInstallDir = new File(dataDir, "app");
            mAppLib32InstallDir = new File(dataDir, "app-lib");
            mEphemeralInstallDir = new File(dataDir, "app-ephemeral");
            mAsecInternalPath = new File(dataDir, "app-asec").getPath();
            mDrmAppPrivateInstallDir = new File(dataDir, "app-private");
            mRegionalizationAppInstallDir = new File(dataDir, "app-regional");

            //创建多用户管理服务
            sUserManager = new UserManagerService(context, this, mPackages);

            mFoundPolicyFile = SELinuxMMAC.readInstallPolicy();
   					
          	//解析packages.xml等文件的信息，保存到Settings的对应字段中。packages.xml中记录系统中所有安装的应用信息，包括基本信息、签名和权限。如果packages.xml有安装的应用信息，readLPw方法会返回true，mFirstBoot的值为false，说明PMS不是首次被启动。
            mFirstBoot = !mSettings.readLPw(sUserManager.getUsers(false));

```

在开始阶段，创建了很多PMS中的关键对象并赋值给PMS中的成员变量

#### mSettings

用于保存所有包的动态设置。

```java
Settings(Object lock) {
    this(Environment.getDataDirectory(), lock);
}

Settings(File dataDir, Object lock) {
    mLock = lock;

    mRuntimePermissionsPersistence = new RuntimePermissionPersistence(mLock);

    mSystemDir = new File(dataDir, "system");
    mSystemDir.mkdirs(); //创建/data/system
    FileUtils.setPermissions(mSystemDir.toString(),
           FileUtils.S_IRWXU|FileUtils.S_IRWXG
           |FileUtils.S_IROTH|FileUtils.S_IXOTH,
           -1, -1);
    mSettingsFilename = new File(mSystemDir, "packages.xml");
    mBackupSettingsFilename = new File(mSystemDir, "packages-backup.xml");
    mPackageListFilename = new File(mSystemDir, "packages.list");
    FileUtils.setPermissions(mPackageListFilename, 0640, SYSTEM_UID, PACKAGE_INFO_GID);

    mStoppedPackagesFilename = new File(mSystemDir, "packages-stopped.xml");
    mBackupStoppedPackagesFilename = new File(mSystemDir, "packages-stopped-backup.xml");
}
```

此处mSystemDir是指目录`/data/system`，在该目录有以下5个文件：

| 文件                        | 功能                     |
| :-------------------------- | :----------------------- |
| packages.xml                | 记录所有安装app的信息    |
| packages-backup.xml         | 备份文件                 |
| packages-stopped.xml        | 记录系统被强制停止的文件 |
| packages-stopped-backup.xml | 备份文件                 |
| packages.list               | 记录应用的数据信息       |

#### mInstaller

Installer继承自SystemService，和PMS、AMS一样是系统的服务(引导服务)，PMS很多的操作都是由Installer来完成的，比如APK的安装和卸载。在Installer内部，通过IInstalld和installd进行Binder通信，由位于nativie层的installd来完成具体的操作。

#### systemConfig

用于得到全局系统配置信息。比如系统的权限就可以通过SystemConfig来获取。

#### mPackageDexOptimizer

Dex优化的工具类。

#### mHandler（PackageHandler类型）

PackageHandler继承自Handler，PMS通过PackageHandler驱动APK的复制和安装工作。
PackageHandler处理的消息队列如果过于繁忙，有可能导致系统卡住， 因此将它添加到Watchdog的监测集中。
Watchdog主要有两个用途，一个是定时检测系统关键服务（AMS和WMS等）是否可能发生死锁，还有一个是定时检测线程的消息队列是否长时间处于工作状态（可能阻塞等待了很长时间）。如果出现上述问题，Watchdog会将日志保存起来，必要时还会杀掉自己所在的进程，也就是SystemServer进程。

#### sUserManager（UserManagerService类型）

多用户管理服务。

### 2、扫描系统阶段

```java
//打印扫描系统阶段日志
EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SYSTEM_SCAN_START,
                    startTime);

// Set flag to monitor and not change apk file paths when
// scanning install directories.
final int scanFlags = SCAN_NO_PATHS | SCAN_DEFER_DEX | SCAN_BOOTING | SCAN_INITIAL;

final String bootClassPath = System.getenv("BOOTCLASSPATH");
final String systemServerClassPath = System.getenv("SYSTEMSERVERCLASSPATH");

if (bootClassPath == null) {
  Slog.w(TAG, "No BOOTCLASSPATH found!");
}

if (systemServerClassPath == null) {
  Slog.w(TAG, "No SYSTEMSERVERCLASSPATH found!");
}

final List<String> allInstructionSets = InstructionSets.getAllInstructionSets();
final String[] dexCodeInstructionSets =
  getDexCodeInstructionSets(
  allInstructionSets.toArray(new String[allInstructionSets.size()]));

/**
             * Ensure all external libraries have had dexopt run on them.
             */
if (mSharedLibraries.size() > 0) {
  // NOTE: For now, we're compiling these system "shared libraries"
  // (and framework jars) into all available architectures. It's possible
  // to compile them only when we come across an app that uses them (there's
  // already logic for that in scanPackageLI) but that adds some complexity.
  for (String dexCodeInstructionSet : dexCodeInstructionSets) {
    for (SharedLibraryEntry libEntry : mSharedLibraries.values()) {
      final String lib = libEntry.path;
      if (lib == null) {
        continue;
      }

      try {
        // Shared libraries do not have profiles so we perform a full
        // AOT compilation (if needed).
        int dexoptNeeded = DexFile.getDexOptNeeded(
          lib, dexCodeInstructionSet,
          getCompilerFilterForReason(REASON_SHARED_APK),
          false /* newProfile */);
        if (dexoptNeeded != DexFile.NO_DEXOPT_NEEDED) {
          mInstaller.dexopt(lib, Process.SYSTEM_UID, dexCodeInstructionSet,
                            dexoptNeeded, DEXOPT_PUBLIC /*dexFlags*/,
                            getCompilerFilterForReason(REASON_SHARED_APK),
                            StorageManager.UUID_PRIVATE_INTERNAL,
                            SKIP_SHARED_LIBRARY_CHECK);
        }
      } catch (FileNotFoundException e) {
        Slog.w(TAG, "Library not found: " + lib);
      } catch (IOException | InstallerException e) {
        Slog.w(TAG, "Cannot dexopt " + lib + "; is it an APK or JAR? "
               + e.getMessage());
      }
    }
  }
}

//在/system中创建framework目录
File frameworkDir = new File(Environment.getRootDirectory(), "framework");

final VersionInfo ver = mSettings.getInternalVersion();
mIsUpgrade = !Build.FINGERPRINT.equals(ver.fingerprint);

// when upgrading from pre-M, promote system app permissions from install to runtime
mPromoteSystemApps =
  mIsUpgrade && ver.sdkVersion <= Build.VERSION_CODES.LOLLIPOP_MR1;

// When upgrading from pre-N, we need to handle package extraction like first boot,
// as there is no profiling data available.
mIsPreNUpgrade = mIsUpgrade && ver.sdkVersion < Build.VERSION_CODES.N;

mIsPreNMR1Upgrade = mIsUpgrade && ver.sdkVersion < Build.VERSION_CODES.N_MR1;

// save off the names of pre-existing system packages prior to scanning; we don't
// want to automatically grant runtime permissions for new system apps
if (mPromoteSystemApps) {
  Iterator<PackageSetting> pkgSettingIter = mSettings.mPackages.values().iterator();
  while (pkgSettingIter.hasNext()) {
    PackageSetting ps = pkgSettingIter.next();
    if (isSystemApp(ps)) {
      mExistingSystemPackages.add(ps.name);
    }
  }
}

// Collect vendor overlay packages. (Do this before scanning any apps.)
// For security and version matching reason, only consider
// overlay packages if they reside in the right directory.
String overlayThemeDir = SystemProperties.get(VENDOR_OVERLAY_THEME_PROPERTY);
 //扫描/vendor/overlay目录下的文件
if (!overlayThemeDir.isEmpty()) {
  scanDirTracedLI(new File(VENDOR_OVERLAY_DIR, overlayThemeDir), mDefParseFlags
                  | PackageParser.PARSE_IS_SYSTEM
                  | PackageParser.PARSE_IS_SYSTEM_DIR
                  | PackageParser.PARSE_TRUSTED_OVERLAY, scanFlags | SCAN_TRUSTED_OVERLAY, 0);
}
scanDirTracedLI(new File(VENDOR_OVERLAY_DIR), mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM
                | PackageParser.PARSE_IS_SYSTEM_DIR
                | PackageParser.PARSE_TRUSTED_OVERLAY, scanFlags | SCAN_TRUSTED_OVERLAY, 0);

// Find base frameworks (resource packages without code).
//收集包名：/system/framework
scanDirTracedLI(frameworkDir, mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM
                | PackageParser.PARSE_IS_SYSTEM_DIR
                | PackageParser.PARSE_IS_PRIVILEGED,
                scanFlags | SCAN_NO_DEX, 0);

// Collected privileged system packages.
//收集私有的系统包名：/system/priv-app
final File privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app");
scanDirTracedLI(privilegedAppDir, mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM
                | PackageParser.PARSE_IS_SYSTEM_DIR
                | PackageParser.PARSE_IS_PRIVILEGED, scanFlags, 0);

// Collect ordinary system packages.
//收集一般的系统包名：/system/app
final File systemAppDir = new File(Environment.getRootDirectory(), "app");
scanDirTracedLI(systemAppDir, mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM
                | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

// Collect all vendor packages.
//收集所有的供应商包名：/vendor/app
File vendorAppDir = new File("/vendor/app");
try {
  vendorAppDir = vendorAppDir.getCanonicalFile();
} catch (IOException e) {
  // failed to look up canonical path, continue with original one
}
scanDirTracedLI(vendorAppDir, mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM
                | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

// Collect all OEM packages.
//收集所有OEM包名：/oem/app
final File oemAppDir = new File(Environment.getOemDirectory(), "app");
scanDirTracedLI(oemAppDir, mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM
                | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

// Collect all Regionalization packages form Carrier's res packages.
if (RegionalizationEnvironment.isSupported()) {
  Log.d(TAG, "Load Regionalization vendor apks");
  final List<File> RegionalizationDirs =
    RegionalizationEnvironment.getAllPackageDirectories();
  for (File f : RegionalizationDirs) {
    File RegionalizationSystemDir = new File(f, "system");
    // Collect packages in <Package>/system/priv-app
    scanDirLI(new File(RegionalizationSystemDir, "priv-app"),
              PackageParser.PARSE_IS_SYSTEM | PackageParser.PARSE_IS_SYSTEM_DIR
              | PackageParser.PARSE_IS_PRIVILEGED, scanFlags, 0);
    // Collect packages in <Package>/system/app
    scanDirLI(new File(RegionalizationSystemDir, "app"),
              PackageParser.PARSE_IS_SYSTEM | PackageParser.PARSE_IS_SYSTEM_DIR,
              scanFlags, 0);
    // Collect overlay in <Package>/system/vendor
    scanDirLI(new File(RegionalizationSystemDir, "vendor/overlay"),
              PackageParser.PARSE_IS_SYSTEM | PackageParser.PARSE_IS_SYSTEM_DIR,
              scanFlags | SCAN_TRUSTED_OVERLAY, 0);
  }
}

// Prune any system packages that no longer exist.
// 这个列表代表有可能有升级包的系统App
final List<String> possiblyDeletedUpdatedSystemApps = new ArrayList<String>();
if (!mOnlyCore) {
  Iterator<PackageSetting> psit = mSettings.mPackages.values().iterator();
  while (psit.hasNext()) {
    PackageSetting ps = psit.next();

    /*
    * If this is not a system app, it can't be a
    * disable system app.
    */
    if ((ps.pkgFlags & ApplicationInfo.FLAG_SYSTEM) == 0) {
      continue;
    }

    /*
    * If the package is scanned, it's not erased.
    */
    final PackageParser.Package scannedPkg = mPackages.get(ps.name);
    if (scannedPkg != null) {
      /*
      * If the system app is both scanned and in the
      * disabled packages list, then it must have been
      * added via OTA. Remove it from the currently
      * scanned package so the previously user-installed
      * application can be scanned.
      */
      if (mSettings.isDisabledSystemPackageLPr(ps.name)) {  //1
        logCriticalInfo(Log.WARN, "Expecting better updated system app for "
                        + ps.name + "; removing system app.  Last known codePath="
                        + ps.codePathString + ", installStatus=" + ps.installStatus
                        + ", versionCode=" + ps.versionCode + "; scanned versionCode="
                        + scannedPkg.mVersionCode);
        //将这个系统App的PackageSetting从PMS的mPackages中移除
        removePackageLI(scannedPkg, true);
        //将升级包的路径添加到mExpectingBetter列表中
        mExpectingBetter.put(ps.name, ps.codePath);
      }

      continue;
    }

    if (!mSettings.isDisabledSystemPackageLPr(ps.name)) {
      psit.remove();
      logCriticalInfo(Log.WARN, "System package " + ps.name
                      + " no longer exists; it's data will be wiped");
      // Actual deletion of code and data will be handled by later
      // reconciliation step
    } else {
      final PackageSetting disabledPs = mSettings.getDisabledSystemPkgLPr(ps.name);
      //这个系统App升级包信息在mDisabledSysPackages中,但是没有发现这个升级包存在
      if (disabledPs.codePath == null || !disabledPs.codePath.exists()) {//2
        possiblyDeletedUpdatedSystemApps.add(ps.name);
      }
    }
  }
}

//look for any incomplete package installations
//清理所有安装不完整的包
ArrayList<PackageSetting> deletePkgsList = mSettings.getListOfIncompleteInstallPackagesLPr();
for (int i = 0; i < deletePkgsList.size(); i++) {
  // Actual deletion of code and data will be handled by later
  // reconciliation step
  final String packageName = deletePkgsList.get(i).name;
  logCriticalInfo(Log.WARN, "Cleaning up incompletely installed app: " + packageName);
  synchronized (mPackages) {
    mSettings.removePackageLPw(packageName);
  }
}

//delete tmp files
//删除临时文件
deleteTempPackageFiles();

// Remove any shared userIDs that have no associated packages
mSettings.pruneSharedUsersLPw();
```

系统扫描阶段的主要工作有以下3点：

1. 创建/system的子目录，比如/system/framework、/system/priv-app和/system/app等等
2. 扫描系统文件，比如/vendor/overlay、/system/framework、/system/app等等目录下的文件。
3. 对扫描到的系统文件做后续处理。

主要来说第3点，一次OTA升级对于一个系统App会有三种情况：

- 这个系统APP无更新。
- 这个系统APP有更新。
- 新的OTA版本中，这个系统APP已经被删除。

当系统App升级，PMS会将该系统App的升级包设置数据（PackageSetting）存储到Settings的mDisabledSysPackages列表中（具体见PMS的replaceSystemPackageLIF方法），mDisabledSysPackages的类型为`ArrayMap<String, PackageSetting>`。mDisabledSysPackages中的信息会被PMS保存到packages.xml中的`<updated-package>`标签下（具体见Settings的writeDisabledSysPackageLPr方法）。

注释1处说明这个系统App有升级包，那么就将该系统App的PackageSetting从mDisabledSysPackages列表中移除，并将系统App的升级包的路径添加到mExpectingBetter列表中，mExpectingBetter的类型为`ArrayMap<String, File>`等待后续处理。

注释2处如果这个系统App的升级包信息存储在mDisabledSysPackages列表中，但是没有发现这个升级包存在，则将它加入到possiblyDeletedUpdatedSystemApps列表中，意为“系统App的升级包可能被删除”，之所以是“可能”，是因为系统还没有扫描Data分区，只能暂放到possiblyDeletedUpdatedSystemApps列表中，等到扫描完Data分区后再做处理。

### 3、扫描Data分区阶段

```java
//如果设备没有加密，那么就开始扫描Data分区	
if (!mOnlyCore) {
    //打印扫描Data分区阶段日志
    EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_DATA_SCAN_START,
                        SystemClock.uptimeMillis());
    //扫描/data/app目录下的文件 
    scanDirTracedLI(mAppInstallDir, 0, scanFlags | SCAN_REQUIRE_KNOWN, 0);
    //扫描/data/app-private目录下的文件
    scanDirTracedLI(mDrmAppPrivateInstallDir, mDefParseFlags
                    | PackageParser.PARSE_FORWARD_LOCK,
                    scanFlags | SCAN_REQUIRE_KNOWN, 0);
    //扫描/data/app-ephemeral目录下的文件
    scanDirLI(mEphemeralInstallDir, mDefParseFlags
              | PackageParser.PARSE_IS_EPHEMERAL,
              scanFlags | SCAN_REQUIRE_KNOWN, 0);

    /**
    * Remove disable package settings for any updated system
    * apps that were removed via an OTA. If they're not a
    * previously-updated app, remove them completely.
    * Otherwise, just revoke their system-level permissions.
    * 处理possiblyDeletedUpdatedSystemApps列表
    * 
    */
    for (String deletedAppName : possiblyDeletedUpdatedSystemApps) {
      PackageParser.Package deletedPkg = mPackages.get(deletedAppName);
      mSettings.removeDisabledSystemPackageLPw(deletedAppName);

      String msg;
      if (deletedPkg == null) {
        //1 如果这个系统App的包信息不在PMS的变量mPackages中，说明是残留的App信息，后续会删除它的数据
        msg = "Updated system package " + deletedAppName
          + " no longer exists; it's data will be wiped";
        // Actual deletion of code and data will be handled by later
        // reconciliation step
      } else {
        //2 如果这个系统App在mPackages中，说明是存在于Data分区，不属于系统App，那么移除其系统权限。
        msg = "Updated system app + " + deletedAppName
          + " no longer present; removing system privileges for "
          + deletedAppName;

        deletedPkg.applicationInfo.flags &= ~ApplicationInfo.FLAG_SYSTEM;

        PackageSetting deletedPs = mSettings.mPackages.get(deletedAppName);
        deletedPs.pkgFlags &= ~ApplicationInfo.FLAG_SYSTEM;
      }
      logCriticalInfo(Log.WARN, msg);
    }

    /**
    * Make sure all system apps that we expected to appear on
    * the userdata partition actually showed up. If they never
    * appeared, crawl back and revive the system version.
    */
     //遍历mExpectingBetter列表
    for (int i = 0; i < mExpectingBetter.size(); i++) {
      final String packageName = mExpectingBetter.keyAt(i);
      if (!mPackages.containsKey(packageName)) {
        //得到系统App的升级包路径
        final File scanFile = mExpectingBetter.valueAt(i);

        logCriticalInfo(Log.WARN, "Expected better " + packageName
                        + " but never showed up; reverting to system");
 
        
        //3 根据系统App所在的目录设置扫描的解析参数
        int reparseFlags = mDefParseFlags;
        if (FileUtils.contains(privilegedAppDir, scanFile)) {
          reparseFlags = PackageParser.PARSE_IS_SYSTEM
            | PackageParser.PARSE_IS_SYSTEM_DIR
            | PackageParser.PARSE_IS_PRIVILEGED;
        } else if (FileUtils.contains(systemAppDir, scanFile)) {
          reparseFlags = PackageParser.PARSE_IS_SYSTEM
            | PackageParser.PARSE_IS_SYSTEM_DIR;
        } else if (FileUtils.contains(vendorAppDir, scanFile)) {
          reparseFlags = PackageParser.PARSE_IS_SYSTEM
            | PackageParser.PARSE_IS_SYSTEM_DIR;
        } else if (FileUtils.contains(oemAppDir, scanFile)) {
          reparseFlags = PackageParser.PARSE_IS_SYSTEM
            | PackageParser.PARSE_IS_SYSTEM_DIR;
        } else {
          Slog.e(TAG, "Ignoring unexpected fallback path " + scanFile);
          continue;
        }

        //4 将packageName对应的包设置数据（PackageSetting）添加到mSettings的mPackages中
        mSettings.enableSystemPackageLPw(packageName);

        try {
          //5 扫描系统App的升级包
          scanPackageTracedLI(scanFile, reparseFlags, scanFlags, 0, null);
        } catch (PackageManagerException e) {
          Slog.e(TAG, "Failed to parse original system package: "
                 + e.getMessage());
        }
      }
    }
  }
 //清除mExpectingBetter列表
  mExpectingBetter.clear();
```

扫描Data分区阶段主要做了以下几件事：

1. 扫描/data/app和/data/app-private目录下的文件。
2. 遍历possiblyDeletedUpdatedSystemApps列表，注释1处如果这个系统App的包信息不在PMS的变量mPackages中，说明是残留的App信息，后续会删除它的数据。注释2处如果这个系统App的包信息在mPackages中，说明是存在于Data分区，不属于系统App，那么移除其系统权限。
3. 遍历mExpectingBetter列表，注释3处根据系统App所在的目录设置扫描的解析参数，注释4处的方法内部会将packageName对应的包设置数据（PackageSetting）添加到mSettings的mPackages中。注释5处扫描系统App的升级包，最后清除mExpectingBetter列表。

### 4、扫描结束阶段

```java
//打印扫描结束阶段日志
EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SCAN_END,
                    SystemClock.uptimeMillis());

            // If the platform SDK has changed since the last time we booted,
            // we need to re-grant app permission to catch any new ones that
            // appear.  This is really a hack, and means that apps can in some
            // cases get permissions that the user didn't initially explicitly
            // allow...  it would be nice to have some better way to handle
            // this situation.
            //当sdk版本不一致时，需要更新权限
            int updateFlags = UPDATE_PERMISSIONS_ALL;
            if (ver.sdkVersion != mSdkVersion) {
                Slog.i(TAG, "Platform changed from " + ver.sdkVersion + " to "
                        + mSdkVersion + "; regranting permissions for internal storage");
                updateFlags |= UPDATE_PERMISSIONS_REPLACE_PKG | UPDATE_PERMISSIONS_REPLACE_ALL;
            }
            updatePermissionsLPw(null, null, StorageManager.UUID_PRIVATE_INTERNAL, updateFlags);
            ver.sdkVersion = mSdkVersion;

            // If this is the first boot or an update from pre-M, and it is a normal
            // boot, then we need to initialize the default preferred apps across
            // all defined users.
   					//如果是第一次启动或者是Android M升级后的第一次启动，需要初始化所有用户定义的默认首选App
            if (!onlyCore && (mPromoteSystemApps || mFirstBoot)) {
                for (UserInfo user : sUserManager.getUsers(true)) {
                    mSettings.applyDefaultPreferredAppsLPw(this, user.id);
                    applyFactoryDefaultBrowserLPw(user.id);
                    primeDomainVerificationsLPw(user.id);
                }
            }

            //在引导过程中尽早为系统用户准备存储，
            //因为核心系统应用程序，如设置Provider和SystemUI无法等待用户启动
            final int storageFlags;
            if (StorageManager.isFileEncryptedNativeOrEmulated()) {
                storageFlags = StorageManager.FLAG_STORAGE_DE;
            } else {
                storageFlags = StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE;
            }
            reconcileAppsDataLI(StorageManager.UUID_PRIVATE_INTERNAL, UserHandle.USER_SYSTEM,
                    storageFlags);

            // If this is first boot after an OTA, and a normal boot, then
            // we need to clear code cache directories.
            // Note that we do *not* clear the application profiles. These remain valid
            // across OTAs and are used to drive profile verification (post OTA) and
            // profile compilation (without waiting to collect a fresh set of profiles).
 						// OTA后的第一次启动，会清除代码缓存目录
            if (mIsUpgrade && !onlyCore) {
                Slog.i(TAG, "Build fingerprint changed; clearing code caches");
                for (int i = 0; i < mSettings.mPackages.size(); i++) {
                    final PackageSetting ps = mSettings.mPackages.valueAt(i);
                    if (Objects.equals(StorageManager.UUID_PRIVATE_INTERNAL, ps.volumeUuid)) {
                        // No apps are running this early, so no need to freeze
                        clearAppDataLIF(ps.pkg, UserHandle.USER_ALL,
                                StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE
                                        | Installer.FLAG_CLEAR_CODE_CACHE_ONLY);
                    }
                }
                ver.fingerprint = Build.FINGERPRINT;
            }

            checkDefaultBrowser();

            //当权限和其他默认项都完成更新，则清理相关信息
            mExistingSystemPackages.clear();
            mPromoteSystemApps = false;

            // All the changes are done during package scanning.
            ver.databaseVersion = Settings.CURRENT_DATABASE_VERSION;

            // 把Settings的内容保存到packages.xml中
            mSettings.writeLPr();

```

扫描结束结束阶段主要做了以下几件事：

1. 如果当前平台SDK版本和上次启动时的SDK版本不同，重新更新APK的授权。
2. 如果是第一次启动或者是Android M升级后的第一次启动，需要初始化所有用户定义的默认首选App。
3. OTA升级后的第一次启动，会清除代码缓存目录。
4. 把Settings的内容保存到packages.xml中，这样此后PMS再次创建时会读到此前保存的Settings的内容。

### 5、准备阶段

```java
//打印准备阶段日志
EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_READY,
                    SystemClock.uptimeMillis());

if (!mOnlyCore) {
  	mRequiredVerifierPackage = getRequiredButNotReallyRequiredVerifierLPr();
  	mRequiredInstallerPackage = getRequiredInstallerLPr();
    mRequiredUninstallerPackage = getRequiredUninstallerLPr();
    mIntentFilterVerifierComponent = getIntentFilterVerifierComponentNameLPr();
    mIntentFilterVerifier = new IntentVerifierProxy(mContext,
                                                    mIntentFilterVerifierComponent);
    mServicesSystemSharedLibraryPackageName = getRequiredSharedLibraryLPr(
      PackageManager.SYSTEM_SHARED_LIBRARY_SERVICES);
    mSharedSystemSharedLibraryPackageName = getRequiredSharedLibraryLPr(
      PackageManager.SYSTEM_SHARED_LIBRARY_SHARED);
} else {
    mRequiredVerifierPackage = null;
    if (mOnlyPowerOffAlarm) {
      mRequiredInstallerPackage = getRequiredInstallerLPr();
    } else {
      mRequiredInstallerPackage = null;
    }
    mRequiredUninstallerPackage = null;
    mIntentFilterVerifierComponent = null;
    mIntentFilterVerifier = null;
    mServicesSystemSharedLibraryPackageName = null;
    mSharedSystemSharedLibraryPackageName = null;
}

//创建PackageInstallerService，用于管理安装会话的服务，它会为每次安装过程分配一个SessionId
mInstallerService = new PackageInstallerService(context, this);

//进行一次垃圾收集
untime.getRuntime().gc();

// The initial scanning above does many calls into installd while
// holding the mPackages lock, but we're mostly interested in yelling
// once we have a booted system.
mInstaller.setWarnIfHeld(mPackages);

// 将PackageManagerInternalImpl（PackageManager的本地服务）添加到LocalServices中，LocalServices用于存储运行在当前的进程中的本地服务
LocalServices.addService(PackageManagerInternal.class, new PackageManagerInternalImpl());
```

## 总结

PMS属于引导服务，由SyetemServer启动

PMS启动又分为5个阶段

1. BOOT_PROGRESS_PMS_START（开始阶段）
2. BOOT_PROGRESS_PMS_SYSTEM_SCAN_START（扫描系统阶段）
3. BOOT_PROGRESS_PMS_DATA_SCAN_START（扫描Data分区阶段）
4. BOOT_PROGRESS_PMS_SCAN_END（扫描结束阶段）
5. BOOT_PROGRESS_PMS_READY（准备阶段）