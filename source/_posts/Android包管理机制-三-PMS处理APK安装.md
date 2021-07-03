---
title: Android系统-包管理机制(三)PMS处理APK安装
date: 2021-03-24 19:47:19
categories: 
- Android系统
tags:
- PMS
- 系统
---

在上一篇中，我们知道了最终调用了PMS的installStage方法安装APK，接下来进入PMS处理APK的流程。

`frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java`

```java
void installStage(String packageName, File stagedDir, String stagedCid,
            IPackageInstallObserver2 observer, PackageInstaller.SessionParams sessionParams,
            String installerPackageName, int installerUid, UserHandle user,
            Certificate[][] certificates) {
  
        ......

        final Message msg = mHandler.obtainMessage(INIT_COPY);
        final InstallParams params = new InstallParams(origin, null, observer,
                sessionParams.installFlags, installerPackageName, sessionParams.volumeUuid,
                verificationInfo, user, sessionParams.abiOverride,
                sessionParams.grantedRuntimePermissions, certificates);
        params.setTraceMethod("installStage").setTraceCookie(System.identityHashCode(params));
        msg.obj = params;

        Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "installStage",
                System.identityHashCode(msg.obj));
        Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                System.identityHashCode(msg.obj));

        mHandler.sendMessage(msg);
    }
```

在installStage函数中创建了类型为INIT_COPY的消息，接下来看看对INIT_COPY的处理

## 1.对INIT_COPY的消息的处理

```java
void doHandleMessage(Message msg) {
            switch (msg.what) {
                case INIT_COPY: {
                    HandlerParams params = (HandlerParams) msg.obj;
                    int idx = mPendingInstalls.size();
                    if (DEBUG_INSTALL) Slog.i(TAG, "init_copy idx=" + idx + ": " + params);
                    // If a bind was already initiated we dont really
                    // need to do anything. The pending install
                    // will be processed later on.
                    //mBound用于标识是否绑定了服务，默认值为false
                    if (!mBound) {
                        Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "bindingMCS",
                                System.identityHashCode(mHandler));
                        // If this is the only one pending we might
                        // have to bind to the service again.
                        //如果没有绑定服务，重新绑定，connectToService方法内部如果绑定成功会将mBound置为true
                        if (!connectToService()) {
                            Slog.e(TAG, "Failed to bind to media container service");
                            params.serviceError();
                            Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "bindingMCS",
                                    System.identityHashCode(mHandler));
                            if (params.traceMethod != null) {
                                Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, params.traceMethod,
                                        params.traceCookie);
                            }
                            return;
                        } else {
                            // Once we bind to the service, the first
                            // pending request will be processed.
                            //绑定服务成功，将请求添加到mPendingInstalls中，等待处理
                            mPendingInstalls.add(idx, params);
                        }
                    } else {
                        //服务已经绑定成功，添加到mPendingInstalls中
                        mPendingInstalls.add(idx, params);
                        // Already bound to the service. Just make
                        // sure we trigger off processing the first request.
                        if (idx == 0) {
                            // 发送MCS_BOUND消息
                            mHandler.sendEmptyMessage(MCS_BOUND);
                        }
                    }
                    break;
                }
```

mBound用于标识是否绑定了DefaultContainerService，默认值为false。DefaultContainerService是用于检查和复制可移动文件的服务，这是一个比较耗时的操作，因此DefaultContainerService没有和PMS运行在同一进程中，它运行在com.android.defcontainer进程，通过IMediaContainerService和PMS进行IPC通信。

`frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java#PackageHandler`

```java
private boolean connectToService() {
          if (DEBUG_SD_INSTALL) Log.i(TAG, "Trying to bind to" +
                  " DefaultContainerService");
          Intent service = new Intent().setComponent(DEFAULT_CONTAINER_COMPONENT);
          Process.setThreadPriority(Process.THREAD_PRIORITY_DEFAULT);
          if (mContext.bindServiceAsUser(service, mDefContainerConn,
                  Context.BIND_AUTO_CREATE, UserHandle.SYSTEM)) {//1
              Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
              mBound = true;//2
              return true;
          }
          Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
          return false;
      }
```

bindServiceAsUser方法的处理逻辑和我们调用bindService是类似的，服务建立连接后，会调用onServiceConnected方法(传入了mDefContainerConn)

```java
final private DefaultContainerConnection mDefContainerConn =
            new DefaultContainerConnection();
    class DefaultContainerConnection implements ServiceConnection {
        public void onServiceConnected(ComponentName name, IBinder service) {
            if (DEBUG_SD_INSTALL) Log.i(TAG, "onServiceConnected");
            IMediaContainerService imcs =
                IMediaContainerService.Stub.asInterface(service);
            mHandler.sendMessage(mHandler.obtainMessage(MCS_BOUND, imcs));
        }

        public void onServiceDisconnected(ComponentName name) {
            if (DEBUG_SD_INSTALL) Log.i(TAG, "onServiceDisconnected");
        }
    }
```

在onServiceConnected中，又发送了MCS_BOUND消息，并将IMediaContainerService实例传递出去。



## 2.对MCS_BOUND类型的消息的处理

```java
case MCS_BOUND: {
        if (DEBUG_INSTALL) Slog.i(TAG, "mcs_bound");
        if (msg.obj != null) {
          mContainerService = (IMediaContainerService) msg.obj; //1
          Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "bindingMCS",
                              System.identityHashCode(mHandler));
        }
        if (mContainerService == null) { //2
          if (!mBound) { //3
            // Something seriously wrong since we are not bound and we are not
            // waiting for connection. Bail out.
            Slog.e(TAG, "Cannot bind to media container service");
            for (HandlerParams params : mPendingInstalls) {
              // Indicate service bind error
              params.serviceError();
              Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                                  System.identityHashCode(params));
              if (params.traceMethod != null) {
                Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER,
                                    params.traceMethod, params.traceCookie);
              }
              return;
            }
            mPendingInstalls.clear();
          } else {
            Slog.w(TAG, "Waiting to connect to media container service");//4
          }
        } else if (mPendingInstalls.size() > 0) { //5
          HandlerParams params = mPendingInstalls.get(0);
          if (params != null) {
            Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                                System.identityHashCode(params));
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "startCopy");
            if (params.startCopy()) { //6
              // We are done...  look for more work or to
              // go idle.
              if (DEBUG_SD_INSTALL) Log.i(TAG,
                                          "Checking for more work or unbind...");
              // Delete pending install
              if (mPendingInstalls.size() > 0) {
                //如果APK安装成功，删除本次安装请求
                mPendingInstalls.remove(0);
              }
              if (mPendingInstalls.size() == 0) {
                if (mBound) {
                  //如果没有安装请求了，发送解绑服务的请求
                  if (DEBUG_SD_INSTALL) Log.i(TAG,
                                              "Posting delayed MCS_UNBIND");
                  removeMessages(MCS_UNBIND);
                  Message ubmsg = obtainMessage(MCS_UNBIND);
                  // Unbind after a little delay, to avoid
                  // continual thrashing.
                  sendMessageDelayed(ubmsg, 10000);
                }
              } else {
                // There are more pending requests in queue.
                // Just post MCS_BOUND message to trigger processing
                // of next pending install.
                //如果还有其他的安装请求，接着发送MCS_BOUND消息继续处理剩余的安装请求 
                if (DEBUG_SD_INSTALL) Log.i(TAG,
                                            "Posting MCS_BOUND for next work");
                mHandler.sendEmptyMessage(MCS_BOUND);
              }
            }
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
          }
        } else {
          // Should never happen ideally.
          Slog.w(TAG, "Empty queue");
        }
        break;
      }
```

在注释1处获取MediaContainerService服务对象，在这里就表示是接收到了onServiceConnected中发出的MCS_BOUND消息。如果msg.obj != null，表示从INIT_COPY消息处理中发出的。

注释2成立表示INIT_COPY消息处理中发出MCS_BOUND，注释3判断是否已经绑定服务，如果!mBound为true，而在发出MCS_BOUND消息之前的各种操作中就已经判断是否绑定服务，没有绑定的话就已经去connectToService()，这里说明发生异常情况，开始处理异常情况。如果!mBound为false，说明已经调用了connectToService方法去绑定服务，注释4打印日志提示等待服务绑定。

如果注释2不成立就进入注释5处判断，如果安装请求数不大于0，就会打印日志，提示空队列。

否则取出安装请求队列第一个请求HandlerParams ，如果HandlerParams 不为null就会调用注释6处的HandlerParams的startCopy方法，用于开始复制APK的流程。

## 3.复制APK

HandlerParams是PMS中的抽象类，它的实现类为PMS的内部类InstallParams。HandlerParams的startCopy方法如下所示。

`frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java#HandlerParams`

```java
final boolean startCopy() {
            boolean res;
            try {
                if (DEBUG_INSTALL) Slog.i(TAG, "startCopy " + mUser + ": " + this);
                //startCopy方法尝试的次数，超过了4次，就放弃这个安装请求
                if (++mRetries > MAX_RETRIES) {
                    Slog.w(TAG, "Failed to invoke remote methods on default container service. Giving up");
                    mHandler.sendEmptyMessage(MCS_GIVE_UP);
                    handleServiceError();
                    return false;
                } else {
                    //开始处理复制
                    handleStartCopy();
                    res = true;
                }
            } catch (RemoteException e) {
                if (DEBUG_INSTALL) Slog.i(TAG, "Posting install MCS_RECONNECT");
                mHandler.sendEmptyMessage(MCS_RECONNECT);
                res = false;
            }
            handleReturnCode();
            return res;
        }
```

handleStartCopy()的实现在InstallParams类中

`frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java#InstallParams`

```java
 public void handleStartCopy() throws RemoteException {
            int ret = PackageManager.INSTALL_SUCCEEDED;
   
   					//确定APK的安装位置。
   					//onSd：安装到SD卡， onInt：内部存储即Data分区，ephemeral：安装到临时存储（Instant Apps安装）
            final boolean onSd = (installFlags & PackageManager.INSTALL_EXTERNAL) != 0;
            final boolean onInt = (installFlags & PackageManager.INSTALL_INTERNAL) != 0;
            final boolean ephemeral = (installFlags & PackageManager.INSTALL_EPHEMERAL) != 0;
            PackageInfoLite pkgLite = null;

            if (onInt && onSd) {
                // // APK不能同时安装在SD卡和Data分区
                Slog.w(TAG, "Conflicting flags specified for installing on both internal and external");
                ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
            } else if (onSd && ephemeral) {
                ////安装标志冲突，Instant Apps不能安装到SD卡中
                Slog.w(TAG,  "Conflicting flags specified for installing ephemeral on external");
                ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
            } else {
                //获取APK的少量的信息
                pkgLite = mContainerService.getMinimalPackageInfo(origin.resolvedPath, installFlags,
                        packageAbiOverride);
              	......
            }

            if (ret == PackageManager.INSTALL_SUCCEEDED) {
                //判断安装的位置
                int loc = pkgLite.recommendedInstallLocation;
                if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_LOCATION) {
                    ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
                } else if (loc == PackageHelper.RECOMMEND_FAILED_ALREADY_EXISTS) {
                    ret = PackageManager.INSTALL_FAILED_ALREADY_EXISTS;
                } else if (loc == PackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE) {
                    ret = PackageManager.INSTALL_FAILED_INSUFFICIENT_STORAGE;
                } else if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_APK) {
                    ret = PackageManager.INSTALL_FAILED_INVALID_APK;
                } else if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_URI) {
                    ret = PackageManager.INSTALL_FAILED_INVALID_URI;
                } else if (loc == PackageHelper.RECOMMEND_MEDIA_UNAVAILABLE) {
                    ret = PackageManager.INSTALL_FAILED_MEDIA_UNAVAILABLE;
                } else {
                    // Override with defaults if needed.
                    loc = installLocationPolicy(pkgLite);
                }
            }
						
   					//根据InstallParams创建InstallArgs对象
            final InstallArgs args = createInstallArgs(this);
            mArgs = args;

            if (ret == PackageManager.INSTALL_SUCCEEDED) {
                  .....
                    
                  ret = args.copyApk(mContainerService, true);
                }
            }

            mRet = ret;
        }
```

首先是通过IMediaContainerService跨进程调用DefaultContainerService的getMinimalPackageInfo方法，该方法轻量解析APK并得到APK的少量信息，轻量解析的原因是这里不需要得到APK的全部信息，APK的少量信息会封装到PackageInfoLite中。然后确认APK的安装位置，最后创建InstallArgs对象。

InstallArgs 是一个抽象类，定义了APK的安装逻辑，比如复制和重命名APK等，它有3个子类，MoveInstallArgs、AsecInstallArgs、FileInstallArgs，都被定义在PMS中

```java
private InstallArgs createInstallArgs(InstallParams params) {
        if (params.move != null) {
            return new MoveInstallArgs(params);
        } else if (installOnExternalAsec(params.installFlags) || params.isForwardLocked()) {
            return new AsecInstallArgs(params);
        } else {
            return new FileInstallArgs(params);
        }
    }
```

其中FileInstallArgs用于处理安装到非ASEC的存储空间的APK，也就是内部存储空间（Data分区），AsecInstallArgs用于处理安装到ASEC中（mnt/asec）即SD卡中的APK。MoveInstallArgs用于处理已安装APK的移动的逻辑。

这里以FileInstallArgs为例，调用copyApk会直接调到FileInstallArgs的doCopyApk方法。

```java
private int doCopyApk(IMediaContainerService imcs, boolean temp) throws RemoteException {
            
            try {
                final boolean isEphemeral = (installFlags & PackageManager.INSTALL_EPHEMERAL) != 0;
                //创建临时文件存储目录
                final File tempDir =
                        mInstallerService.allocateStageDirLegacy(volumeUuid, isEphemeral);//1
                codeFile = tempDir;
                resourceFile = tempDir;
            } catch (IOException e) {
                Slog.w(TAG, "Failed to create copy file: " + e);
                return PackageManager.INSTALL_FAILED_INSUFFICIENT_STORAGE;
            }

            final IParcelFileDescriptorFactory target = new IParcelFileDescriptorFactory.Stub() {
                @Override
                public ParcelFileDescriptor open(String name, int mode) throws RemoteException {
                    if (!FileUtils.isValidExtFilename(name)) {
                        throw new IllegalArgumentException("Invalid filename: " + name);
                    }
                    try {
                        final File file = new File(codeFile, name);
                        final FileDescriptor fd = Os.open(file.getAbsolutePath(),
                                O_RDWR | O_CREAT, 0644);
                        Os.chmod(file.getAbsolutePath(), 0644);
                        return new ParcelFileDescriptor(fd);
                    } catch (ErrnoException e) {
                        throw new RemoteException("Failed to open: " + e.getMessage());
                    }
                }
            };

            int ret = PackageManager.INSTALL_SUCCEEDED;
            ret = imcs.copyPackage(origin.file.getAbsolutePath(), target);//2
            if (ret != PackageManager.INSTALL_SUCCEEDED) {
                Slog.e(TAG, "Failed to copy package");
                return ret;
            }

            return ret;
        }
```

doCopyApk函数的工作就是在/data/app下创建临时文件夹，以sessionId为临时文件夹名 ，例如/data/app/{sessionId}.tmp/base.apk将APK复制到这个文件夹下，一般命名为base.apk

## 4.安装APK

我们返回startCopy方法中，复制完成后调用handleReturnCode()，这个方法会调用到InstallParams的handleReturnCode方法

```java
				@Override
        void handleReturnCode() {
            // If mArgs is null, then MCS couldn't be reached. When it
            // reconnects, it will try again to install. At that point, this
            // will succeed.
            if (mArgs != null) {
				Slog.d(TAG,"welen:" + "handleReturnCode");
                processPendingInstall(mArgs, mRet);
            }
        }
```

```java
private void processPendingInstall(final InstallArgs args, final int currentStatus) {
        // Queue up an async operation since the package installation may take a little while.
        mHandler.post(new Runnable() {
            public void run() {
				Slog.d(TAG,"welen:" + "processPendingInstall");
                mHandler.removeCallbacks(this);
                 // Result object to be returned
                PackageInstalledInfo res = new PackageInstalledInfo();
                res.setReturnCode(currentStatus);
                res.uid = -1;
                res.pkg = null;
                res.removedInfo = null;
                if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
                    //安装前准备
                    args.doPreInstall(res.returnCode);
                    synchronized (mInstallLock) {
                        installPackageTracedLI(args, res);
                    }
                    // 安装后处理
                    args.doPostInstall(res.returnCode, res.uid);
                }
              ......
```

在processPendingInstall中进行安装前检查，确认安装环境，如果不可靠会清除复制的APK文件，安装完成后doPostInstall，如果安装不成功，删除掉安装相关的目录与文件。

接下来看下 **installPackageTracedLI()**，内部调用**installPackageLI**

```java
private void installPackageTracedLI(InstallArgs args, PackageInstalledInfo res) {
        try {
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "installPackage");
            installPackageLI(args, res);
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
    }
```

**installPackageLI**

```java
private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {
  		  ......
        
   			PackageParser pp = new PackageParser();
        pp.setSeparateProcesses(mSeparateProcesses);
        pp.setDisplayMetrics(mMetrics);

        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "parsePackage");
        final PackageParser.Package pkg;
        try {
            // 解析APK
            pkg = pp.parsePackage(tmpPackageFile, parseFlags);
        } catch (PackageParserException e) {
            res.setError("Failed parse during installPackageLI", e);
            return;
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
  
  			......
          
  			pp = null;
        String oldCodePath = null;
        boolean systemApp = false;
  			synchronized (mPackages) {
           // 判断APK是否存在，存在replace = true
           if ((installFlags & PackageManager.INSTALL_REPLACE_EXISTING) != 0) {
                String oldName = mSettings.mRenamedPackages.get(pkgName);
                if (pkg.mOriginalPackages != null
                        && pkg.mOriginalPackages.contains(oldName)
                        && mPackages.containsKey(oldName)) {
                    // This package is derived from an original package,
                    // and this device has been updating from that original
                    // name.  We must continue using the original name, so
                    // rename the new package here.
                    pkg.setPackageName(oldName);
                    pkgName = pkg.packageName;
                    replace = true;
                    if (DEBUG_INSTALL) Slog.d(TAG, "Replacing existing renamed package: oldName="
                            + oldName + " pkgName=" + pkgName);
                } else if (mPackages.containsKey(pkgName)) {
                    // This package, under its official name, already exists
                    // on the device; we should replace it.
                    replace = true;
                    if (DEBUG_INSTALL) Slog.d(TAG, "Replace existing pacakge: " + pkgName);
                }
            }
          PackageSetting ps = mSettings.mPackages.get(pkgName);
          //查看Settings中是否存有要安装的APK的信息，如果有就获取签名信息
          if (ps != null) {
                //检查签名信息
                if (shouldCheckUpgradeKeySetLP(ps, scanFlags)) {
                    //新APK与老APK签名不一致
                    if (!checkUpgradeKeySetLP(ps, pkg)) {
                        res.setError(INSTALL_FAILED_UPDATE_INCOMPATIBLE, "Package "
                                + pkg.packageName + " upgrade keys do not match the "
                                + "previously installed version");
                        return;
                    }
                } else {
                    try {
                        //确保签名一致
                        verifySignaturesLP(ps, pkg);
                    } catch (PackageManagerException e) {
                        res.setError(e.error, e.getMessage());
                        return;
                    }
                }
            }
          
          
          int N = pkg.permissions.size();
          //遍历每个权限，对权限进行处理
          for (int i = N-1; i >= 0; i--) {
                PackageParser.Permission perm = pkg.permissions.get(i);
                BasePermission bp = mSettings.mPermissions.get(perm.info.name);
            ......
          }
        }
  			
  			f (systemApp) {
            if (onExternal) {
                // //系统APP不能在SD卡上替换安装
                res.setError(INSTALL_FAILED_INVALID_INSTALL_LOCATION,
                        "Cannot install updates to system apps on sdcard");
                return;
            } else if (ephemeral) {
                // 系统APP不能被Instant App替换
                res.setError(INSTALL_FAILED_EPHEMERAL_INVALID,
                        "Cannot update a system app with an ephemeral app");
                return;
            }
        }
  	
  			......
          
        //重命名临时文件
  			if (!args.doRename(res.returnCode, pkg, oldCodePath)) {
            res.setError(INSTALL_FAILED_INSUFFICIENT_STORAGE, "Failed rename");
            return;
        }

        startIntentFilterVerifications(args.user.getIdentifier(), replace, pkg);

        try (PackageFreezer freezer = freezePackageForInstall(pkgName, installFlags,
                "installPackageLI")) {
            if (replace) {
                //替换安装
                replacePackageLIF(pkg, parseFlags, scanFlags | SCAN_REPLACING, args.user,
                        installerPackageName, res);
            } else {
                //安装新APK
                installNewPackageLIF(pkg, parseFlags, scanFlags | SCAN_DELETE_DATA_ON_FAILURES,
                        args.user, installerPackageName, volumeUuid, res);
            }
        }
        synchronized (mPackages) {
            final PackageSetting ps = mSettings.mPackages.get(pkgName);
            if (ps != null) {
                //更新应用程序所属的用户
                res.newUsers = ps.queryInstalledUsers(sUserManager.getUserIds(), true);
            }
          ......
        }

}
```

installPackageLI方法的代码这里截取主要的部分，主要做了几件事：

1. 创建PackageParser解析APK。
2. 检查APK是否存在，如果存在就获取此前没被改名前的包名并赋值给PackageParser.Package类型的pkg，将标志位replace置为true表示是替换安装。
3. 如果Settings中保存有要安装的APK的信息，说明此前安装过该APK，则需要校验APK的签名信息，确保安全的进行替换。
4. 将临时文件重新命名，比如前面提到的/data/app/{sessionId}.tmp/base.apk，重命名为/data/app/包名-1/base.apk。这个新命名的包名会带上一个数字后缀1，每次升级一个已有的App，这个数字会不断的累加(/data/app/com.test.settings-2/base.apk)。
5. 系统APP的更新安装会有两个限制，一个是系统APP不能在SD卡上替换安装，另一个是系统APP不能被Instant App替换。
6. 根据replace来做区分，如果是替换安装就会调用replacePackageLIF方法，其方法内部还会对系统APP和非系统APP进行区分处理，如果是新安装APK会调用installNewPackageLIF方法。

我们看一下安装新APK的逻辑

```java
 /*
     * Install a non-existing package.
     */
    private void installNewPackageLIF(PackageParser.Package pkg, final int policyFlags,
            int scanFlags, UserHandle user, String installerPackageName, String volumeUuid,
            PackageInstalledInfo res) {
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "installNewPackage");

        // Remember this for later, in case we need to rollback this install
        String pkgName = pkg.packageName;

        try {
            //扫描APK
            PackageParser.Package newPackage = scanPackageTracedLI(pkg, policyFlags, scanFlags,
                    System.currentTimeMillis(), user);
						//更新Settings信息
            updateSettingsLI(newPackage, installerPackageName, null, res, user);

            if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
              	//安装成功后，为新安装的应用程序准备数据
                prepareAppDataAfterInstallLIF(newPackage);

            } else {
                //安装失败则删除APK
                deletePackageLIF(pkgName, UserHandle.ALL, false, null,
                        PackageManager.DELETE_KEEP_DATA, res.removedInfo, true, null);
            }
        } catch (PackageManagerException e) {
            res.setError("Package couldn't be installed in " + pkg.codePath, e);
        }

        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    }
```

### 4.1 scanPackageTracedLI

```java
private PackageParser.Package scanPackageTracedLI(PackageParser.Package pkg,
            final int policyFlags, int scanFlags, long currentTime, UserHandle user)
                    throws PackageManagerException {
     
        final PackageParser.Package scannedPkg;
        try {
            // Scan the parent
            scannedPkg = scanPackageLI(pkg, policyFlags, scanFlags, currentTime, user);
            // Scan the children
            final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
            for (int i = 0; i < childCount; i++) {
                PackageParser.Package childPkg = pkg.childPackages.get(i);
                scanPackageLI(childPkg, policyFlags,
                        scanFlags, currentTime, user);
            }
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }

        if ((scanFlags & SCAN_CHECK_ONLY) != 0) {
            return scanPackageTracedLI(pkg, policyFlags, scanFlags, currentTime, user);
        }
        return scannedPkg;
    }
```

```java
private PackageParser.Package scanPackageLI(PackageParser.Package pkg, final int policyFlags,
            int scanFlags, long currentTime, UserHandle user) throws PackageManagerException {
        boolean success = false;
        try {
            final PackageParser.Package res = scanPackageDirtyLI(pkg, policyFlags, scanFlags,
                    currentTime, user);
            success = true;
            return res;
        } finally {
            if (!success && (scanFlags & SCAN_DELETE_DATA_ON_FAILURES) != 0) {
                destroyAppDataLIF(pkg, UserHandle.USER_ALL,
                        StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE);
                destroyAppProfilesLIF(pkg, UserHandle.USER_ALL);
            }
        }
    }
```

#### 4.1.1 scanPackageDirtyLI

最终调用到`scanPackageDirtyLI`方法，这个函数也是非常长，干了很多事，无论通过什么方式安装APK，最终都会调用到这里进行APK的实际安装过程。

主要就是将App中的provider，service，receiver，activity全都保存到PMS的成员集合类中。

```java
private PackageParser.Package scanPackageDirtyLI(PackageParser.Package pkg,
            final int policyFlags, final int scanFlags, long currentTime, UserHandle user)
            throws PackageManagerException {
        final File scanFile = new File(pkg.codePath);
        ......

            // // 处理provider
            int N = pkg.providers.size();
            StringBuilder r = null;
            int i;
            for (i=0; i<N; i++) {
                PackageParser.Provider p = pkg.providers.get(i);
                p.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        p.info.processName, pkg.applicationInfo.uid);
                mProviders.addProvider(p);
            }
            // 处理service
            N = pkg.services.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Service s = pkg.services.get(i);
                s.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        s.info.processName, pkg.applicationInfo.uid);
                mServices.addService(s);
               
            }
          
 
            // 处理receiver
            N = pkg.receivers.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Activity a = pkg.receivers.get(i);
                a.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        a.info.processName, pkg.applicationInfo.uid);
                mReceivers.addActivity(a, "receiver");
              
            }
            

            // 处理activity
            N = pkg.activities.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Activity a = pkg.activities.get(i);
                a.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        a.info.processName, pkg.applicationInfo.uid);
                mActivities.addActivity(a, "activity");
            }
            

            // 处理permissionGroup权限组
            N = pkg.permissionGroups.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.PermissionGroup pg = pkg.permissionGroups.get(i);
                PackageParser.PermissionGroup cur = mPermissionGroups.get(pg.info.name);
                final String curPackageName = cur == null ? null : cur.info.packageName;
                final boolean isPackageUpdate = pg.info.packageName.equals(curPackageName);
                if (cur == null || isPackageUpdate) {
                    mPermissionGroups.put(pg.info.name, pg);
                }
            }
          

            // 处理permission
            N = pkg.permissions.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Permission p = pkg.permissions.get(i);

                p.info.flags &= ~PermissionInfo.FLAG_INSTALLED;

                if (pkg.applicationInfo.targetSdkVersion > Build.VERSION_CODES.LOLLIPOP_MR1) {
                    p.group = mPermissionGroups.get(p.info.group);
                    // Warn for a permission in an unknown group.
                    if (p.info.group != null && p.group == null) {
                        Slog.w(TAG, "Permission " + p.info.name + " from package "
                                + p.info.packageName + " in an unknown group " + p.info.group);
                    }
                }

                ArrayMap<String, BasePermission> permissionMap =
                        p.tree ? mSettings.mPermissionTrees
                                : mSettings.mPermissions;
                BasePermission bp = permissionMap.get(p.info.name);

                // Allow system apps to redefine non-system permissions
                if (bp != null && !Objects.equals(bp.sourcePackage, p.info.packageName)) {
                    final boolean currentOwnerIsSystem = (bp.perm != null
                            && isSystemApp(bp.perm.owner));
                    if (isSystemApp(p.owner)) {
                        if (bp.type == BasePermission.TYPE_BUILTIN && bp.perm == null) {
                            // It's a built-in permission and no owner, take ownership now
                            bp.packageSetting = pkgSetting;
                            bp.perm = p;
                            bp.uid = pkg.applicationInfo.uid;
                            bp.sourcePackage = p.info.packageName;
                            p.info.flags |= PermissionInfo.FLAG_INSTALLED;
                        } else if (!currentOwnerIsSystem) {
                            String msg = "New decl " + p.owner + " of permission  "
                                    + p.info.name + " is system; overriding " + bp.sourcePackage;
                            reportSettingsProblem(Log.WARN, msg);
                            bp = null;
                        }
                    }
                }

                if (bp == null) {
                    bp = new BasePermission(p.info.name, p.info.packageName,
                            BasePermission.TYPE_NORMAL);
                    permissionMap.put(p.info.name, bp);
                }

                if (bp.perm == null) {
                    if (bp.sourcePackage == null
                            || bp.sourcePackage.equals(p.info.packageName)) {
                        BasePermission tree = findPermissionTreeLP(p.info.name);
                        if (tree == null
                                || tree.sourcePackage.equals(p.info.packageName)) {
                            bp.packageSetting = pkgSetting;
                            bp.perm = p;
                            bp.uid = pkg.applicationInfo.uid;
                            bp.sourcePackage = p.info.packageName;
                            p.info.flags |= PermissionInfo.FLAG_INSTALLED;
                            if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                                if (r == null) {
                                    r = new StringBuilder(256);
                                } else {
                                    r.append(' ');
                                }
                                r.append(p.info.name);
                            }
                        } else {
                            Slog.w(TAG, "Permission " + p.info.name + " from package "
                                    + p.info.packageName + " ignored: base tree "
                                    + tree.name + " is from package "
                                    + tree.sourcePackage);
                        }
                    } else {
                        Slog.w(TAG, "Permission " + p.info.name + " from package "
                                + p.info.packageName + " ignored: original from "
                                + bp.sourcePackage);
                    }
                } else if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                    if (r == null) {
                        r = new StringBuilder(256);
                    } else {
                        r.append(' ');
                    }
                    r.append("DUP:");
                    r.append(p.info.name);
                }
                if (bp.perm == p) {
                    bp.protectionLevel = p.info.protectionLevel;
                }
            }

            // 处理instrumentation
            N = pkg.instrumentation.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Instrumentation a = pkg.instrumentation.get(i);
                a.info.packageName = pkg.applicationInfo.packageName;
                a.info.sourceDir = pkg.applicationInfo.sourceDir;
                a.info.publicSourceDir = pkg.applicationInfo.publicSourceDir;
                a.info.splitSourceDirs = pkg.applicationInfo.splitSourceDirs;
                a.info.splitPublicSourceDirs = pkg.applicationInfo.splitPublicSourceDirs;
                a.info.dataDir = pkg.applicationInfo.dataDir;
                a.info.deviceProtectedDataDir = pkg.applicationInfo.deviceProtectedDataDir;
                a.info.credentialProtectedDataDir = pkg.applicationInfo.credentialProtectedDataDir;

                a.info.nativeLibraryDir = pkg.applicationInfo.nativeLibraryDir;
                a.info.secondaryNativeLibraryDir = pkg.applicationInfo.secondaryNativeLibraryDir;
                mInstrumentation.put(a.getComponentName(), a);
               
            }
           
            // 处理protectedBroadcast
            if (pkg.protectedBroadcasts != null) {
                N = pkg.protectedBroadcasts.size();
                for (i=0; i<N; i++) {
                    mProtectedBroadcasts.add(pkg.protectedBroadcasts.get(i));
                }
            }

            // 设置时间戳为扫描文件的时间戳
            pkgSetting.setTimeStamp(scanFileTime);
           ......
        return pkg;
    }
```



### 4.2 updateSettingsLI

更新Settings信息

```java
private void updateSettingsLI(PackageParser.Package newPackage, String installerPackageName,
            int[] allUsers, PackageInstalledInfo res, UserHandle user) {
        // Update the parent package setting
        updateSettingsInternalLI(newPackage, installerPackageName, allUsers, res.origUsers,
                res, user);
        // Update the child packages setting
        final int childCount = (newPackage.childPackages != null)
                ? newPackage.childPackages.size() : 0;
        for (int i = 0; i < childCount; i++) {
            PackageParser.Package childPackage = newPackage.childPackages.get(i);
            PackageInstalledInfo childRes = res.addedChildPackages.get(childPackage.packageName);
            updateSettingsInternalLI(childPackage, installerPackageName, allUsers,
                    childRes.origUsers, childRes, user);
        }
    }
```

```java
   private void updateSettingsInternalLI(PackageParser.Package newPackage,
            String installerPackageName, int[] allUsers, int[] installedForUsers,
            PackageInstalledInfo res, UserHandle user) {
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "updateSettings");

        String pkgName = newPackage.packageName;
        synchronized (mPackages) {
            //write settings. the installStatus will be incomplete at this stage.
            //note that the new package setting would have already been
            //added to mPackages. It hasn't been persisted yet.
            mSettings.setInstallStatus(pkgName, PackageSettingBase.PKG_INSTALL_INCOMPLETE);
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "writeSettings");
            mSettings.writeLPr();
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }

        synchronized (mPackages) {
            updatePermissionsLPw(newPackage.packageName, newPackage,
                    UPDATE_PERMISSIONS_REPLACE_PKG | (newPackage.permissions.size() > 0
                            ? UPDATE_PERMISSIONS_ALL : 0));
            // For system-bundled packages, we assume that installing an upgraded version
            // of the package implies that the user actually wants to run that new code,
            // so we enable the package.
            PackageSetting ps = mSettings.mPackages.get(pkgName);
            final int userId = user.getIdentifier();
            if (ps != null) {
                if (isSystemApp(newPackage)) {
                    if (DEBUG_INSTALL) {
                        Slog.d(TAG, "Implicitly enabling system package on upgrade: " + pkgName);
                    }
                    // Enable system package for requested users
                    if (res.origUsers != null) {
                        for (int origUserId : res.origUsers) {
                            if (userId == UserHandle.USER_ALL || userId == origUserId) {
                                ps.setEnabled(COMPONENT_ENABLED_STATE_DEFAULT,
                                        origUserId, installerPackageName);
                            }
                        }
                    }
                    // Also convey the prior install/uninstall state
                    if (allUsers != null && installedForUsers != null) {
                        for (int currentUserId : allUsers) {
                            final boolean installed = ArrayUtils.contains(
                                    installedForUsers, currentUserId);
                            if (DEBUG_INSTALL) {
                                Slog.d(TAG, "    user " + currentUserId + " => " + installed);
                            }
                            ps.setInstalled(installed, currentUserId);
                        }
                        // these install state changes will be persisted in the
                        // upcoming call to mSettings.writeLPr().
                    }
                }
                // It's implied that when a user requests installation, they want the app to be
                // installed and enabled.
                if (userId != UserHandle.USER_ALL) {
                    ps.setInstalled(true, userId);
                    ps.setEnabled(COMPONENT_ENABLED_STATE_DEFAULT, userId, installerPackageName);
                }
            }
            res.name = pkgName;
            res.uid = newPackage.applicationInfo.uid;
            res.pkg = newPackage;
            mSettings.setInstallStatus(pkgName, PackageSettingBase.PKG_INSTALL_COMPLETE);
            mSettings.setInstallerPackageName(pkgName, installerPackageName);
            res.setReturnCode(PackageManager.INSTALL_SUCCEEDED);
            //to update install status
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "writeSettings");
            mSettings.writeLPr();
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }

        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    }
```

`mSettings.writeLPr();`会将APK的相关信息转为XML写到data/system/packages.xml中，并在data/system/packages.list增加APP信息。

```java
 void writeLPr() {

        try {
            FileOutputStream fstr = new FileOutputStream(mSettingsFilename);
            BufferedOutputStream str = new BufferedOutputStream(fstr);

            //XmlSerializer serializer = XmlUtils.serializerInstance();
            XmlSerializer serializer = new FastXmlSerializer();
            serializer.setOutput(str, StandardCharsets.UTF_8.name());
            serializer.startDocument(null, true);
            serializer.setFeature("http://xmlpull.org/v1/doc/features.html#indent-output", true);

            serializer.startTag(null, "packages");

            for (int i = 0; i < mVersion.size(); i++) {
                final String volumeUuid = mVersion.keyAt(i);
                final VersionInfo ver = mVersion.valueAt(i);

                serializer.startTag(null, TAG_VERSION);
                XmlUtils.writeStringAttribute(serializer, ATTR_VOLUME_UUID, volumeUuid);
                XmlUtils.writeIntAttribute(serializer, ATTR_SDK_VERSION, ver.sdkVersion);
                XmlUtils.writeIntAttribute(serializer, ATTR_DATABASE_VERSION, ver.databaseVersion);
                XmlUtils.writeStringAttribute(serializer, ATTR_FINGERPRINT, ver.fingerprint);
                serializer.endTag(null, TAG_VERSION);
            }
          
            ......
          
            mKeySetManagerService.writeKeySetManagerServiceLPr(serializer);

            serializer.endTag(null, "packages");

            serializer.endDocument();

            str.flush();
            FileUtils.sync(fstr);
            str.close();

            // New settings successfully written, old ones are no longer
            // needed.
            mBackupSettingsFilename.delete();
            FileUtils.setPermissions(mSettingsFilename.toString(),
                    FileUtils.S_IRUSR|FileUtils.S_IWUSR
                    |FileUtils.S_IRGRP|FileUtils.S_IWGRP,
                    -1, -1);

            writeKernelMappingLPr();
            /// package.list
            writePackageListLPr();
            writeAllUsersPackageRestrictionsLPr();
            writeAllRuntimePermissionsLPr();
            return;
    }
```

`data/system/packages.xml`

```xml
<package name="com.My.app.test" codePath="/data/app/com.My.app.test-1" nativeLibraryPath="/data/app/com.My.app.test-1/lib" primaryCpuAbi="armeabi-v7a" publicFlags="944291396" privateFlags="0" ft="177d1e84078" it="177d1e863ca" ut="177d1e863ca" version="1" userId="10072">
        <sigs count="1">
            <cert index="1" />
        </sigs>
        <perms>
            <item name="android.permission.BLUETOOTH" granted="true" flags="0" />
            <item name="android.permission.INTERNET" granted="true" flags="0" />
            <item name="android.permission.BLUETOOTH_ADMIN" granted="true" flags="0" />
            <item name="android.permission.ACCESS_NETWORK_STATE" granted="true" flags="0" />
            <item name="android.permission.ACCESS_WIFI_STATE" granted="true" flags="0" />
        </perms>
        <proper-signing-keyset identifier="9" />
    </package>
```

`data/system/packages.list`

```
com.My.app.test 10072 0 /data/user/0/com.My.app.test default 3002,3003,3001
```



### 4.3 prepareAppDataAfterInstallLIF

```java
private void prepareAppDataAfterInstallLIF(PackageParser.Package pkg) {
        final PackageSetting ps;
        ......
        prepareAppDataLIF(pkg, user.id, flags);
          
        }
    }
```



```java
private void prepareAppDataLIF(PackageParser.Package pkg, int userId, int flags) {
        if (pkg == null) {
            Slog.wtf(TAG, "Package was null!", new Throwable());
            return;
        }
        prepareAppDataLeafLIF(pkg, userId, flags);
        final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
        for (int i = 0; i < childCount; i++) {
            prepareAppDataLeafLIF(pkg.childPackages.get(i), userId, flags);
        }
    }
```

```java
 private void prepareAppDataLeafLIF(PackageParser.Package pkg, int userId, int flags) {
        
        final String volumeUuid = pkg.volumeUuid;
        final String packageName = pkg.packageName;
        final ApplicationInfo app = pkg.applicationInfo;
        final int appId = UserHandle.getAppId(app.uid);

        Preconditions.checkNotNull(app.seinfo);

        ......
        mInstaller.createAppData(volumeUuid, packageName, userId, flags,
                    appId, app.seinfo, app.targetSdkVersion);
        
        .....
        prepareAppDataContentsLeafLIF(pkg, userId, flags);
    }
```

```java
private void prepareAppDataContentsLeafLIF(PackageParser.Package pkg, int userId, int flags) {
        final String volumeUuid = pkg.volumeUuid;
        final String packageName = pkg.packageName;
        final ApplicationInfo app = pkg.applicationInfo;

        if ((flags & StorageManager.FLAG_STORAGE_CE) != 0) {
            // Create a native library symlink only if we have native libraries
            // and if the native libraries are 32 bit libraries. We do not provide
            // this symlink for 64 bit libraries.
            if (app.primaryCpuAbi != null && !VMRuntime.is64BitAbi(app.primaryCpuAbi)) {
                final String nativeLibPath = app.nativeLibraryDir;
                try {
                    mInstaller.linkNativeLibraryDirectory(volumeUuid, packageName,
                            nativeLibPath, userId);
                } catch (InstallerException e) {
                    Slog.e(TAG, "Failed to link native for " + packageName + ": " + e);
                }
            }
        }
    }
```

在整个流程中可以看到主要就两段：`mInstaller.createAppData`和 `mInstaller.linkNativeLibraryDirectory`，mInstaller我们之前有了解过，Installer继承自SystemService，和PMS、AMS一样是系统的服务(引导服务)，PMS很多的操作都是由Installer来完成的。它会与位于nativie层的installd守护进程通过socket通信来完成具体的操作。关于守护进程installd就在下一篇了解。

参考：

[BATcoder - 刘望舒](http://liuwangshu.cn/tags/Android%E5%8C%85%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/)

[Gityuan](http://gityuan.com/2016/11/06/packagemanager/)

