---
title: Android包管理机制(二)PackageInstaller安装APK
date: 2021-03-17 12:03:03
categories: 
- Android系统
tags:
- Android
- 系统
---

APK的安装有很多方式，应用商城下载、文件浏览器安装、adb命令安装，我们以文件浏览器为例。

将一个APK文件放到SD卡目录下，文件浏览器显示apk文件。

当我们点击APK文件时，文件浏览器根据文件的后缀(.apk)解析，执行startActivity，调起packageinstaller.PackageInstallerActivity。

```
ActivityManager: START u0 {act=android.intent.action.VIEW dat=file:///storage/emulated/0/com.dewmobile.kuaiya.apk typ=application/vnd.android.package-archive flg=0x88000 cmp=com.android.packageinstaller/.PackageInstallerActivity} from uid 10032 on display 0
```

### PackageInstallerActivity

PackageInstallerActivity这个类的主要作用是显示安装弹窗，对APK进行解析，判断是否允许未知来源安装，判断应用权限，等待用户安装。

`packages/apps/PackageInstaller/src/com/android/packageinstaller/PackageInstallerActivity.java`

```java
public void onClick(View v) {
        if (v == mOk) {
            if (mOkCanInstall || mScrollView == null) {
                if (mSessionId != -1) {
                    mInstaller.setPermissionsResult(mSessionId, true);
                    clearCachedApkIfNeededAndFinish();
                } else {
                    startInstall(); //开始安装
                }
            } else {
                mScrollView.pageScroll(View.FOCUS_DOWN);
            }
        } else if (v == mCancel) {
            // Cancel and finish
            setResult(RESULT_CANCELED);
            if (mSessionId != -1) {
                mInstaller.setPermissionsResult(mSessionId, false);
            }
            clearCachedApkIfNeededAndFinish();
        }
    }
```

然后用户点击安装按钮，就会调用startInstall()函数

```java
   private void startInstall() {
        // Start subactivity to actually install the application
        Intent newIntent = new Intent();
        newIntent.putExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO,
                mPkgInfo.applicationInfo);
        newIntent.setData(mPackageURI);
        newIntent.setClass(this, InstallAppProgress.class);//
        String installerPackageName = getIntent().getStringExtra(
                Intent.EXTRA_INSTALLER_PACKAGE_NAME);
     
        ......
          
        if(localLOGV) Log.i(TAG, "downloaded app uri="+mPackageURI);
        startActivity(newIntent);
        finish();
    }

```

startInstall函数用于跳转到InstallAppProgress这个Activity，关闭PackageInstallerActivity。

### InstallAppProgress

InstallAppProgress主要用于展示安装界面、向PMS发送包信息， 处理回调。

`packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallAppProgress.java`

```java
@Override
    public void onCreate(Bundle icicle) {
        super.onCreate(icicle);
        Intent intent = getIntent();
        mAppInfo = intent.getParcelableExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO);
        mPackageURI = intent.getData();

        final String scheme = mPackageURI.getScheme();
        if (scheme != null && !"file".equals(scheme) && !"package".equals(scheme)) {
            throw new IllegalArgumentException("unexpected scheme " + scheme);
        }

        // 开启安装线程
        mInstallThread = new HandlerThread("InstallThread");
        mInstallThread.start();
        mInstallHandler = new Handler(mInstallThread.getLooper());

        //注册安装监听
        IntentFilter intentFilter = new IntentFilter();
        intentFilter.addAction(BROADCAST_ACTION);
        registerReceiver(
                mBroadcastReceiver, intentFilter, BROADCAST_SENDER_PERMISSION, null /*scheduler*/);

        initView();
    }

```

InstallAppProgress中启动了一个HandlerThread，开启安装线程，注册了一个广播，这个广播用来监听系统返回的安装结果。

主要代码在在initView()函数中

```java
            final PackageInstaller.SessionParams params = new PackageInstaller.SessionParams(
                    PackageInstaller.SessionParams.MODE_FULL_INSTALL);
            params.referrerUri = getIntent().getParcelableExtra(Intent.EXTRA_REFERRER);
            params.originatingUri = getIntent().getParcelableExtra(Intent.EXTRA_ORIGINATING_URI);
            params.originatingUid = getIntent().getIntExtra(Intent.EXTRA_ORIGINATING_UID,
                    UID_UNKNOWN);

            File file = new File(mPackageURI.getPath());
            try {
                //解析安装包，设置安装位置，从AndroidManifest中获取
                PackageLite pkg = PackageParser.parsePackageLite(file, 0);
                params.setAppPackageName(pkg.packageName);
                params.setInstallLocation(pkg.installLocation);
                params.setSize(
                    PackageHelper.calculateInstalledSize(pkg, false, params.abiOverride));
            } catch (PackageParser.PackageParserException e) {
                Log.e(TAG, "Cannot parse package " + file + ". Assuming defaults.");
                Log.e(TAG, "Cannot calculate installed size " + file + ". Try only apk size.");
                params.setSize(file.length());
            } catch (IOException e) {
                Log.e(TAG, "Cannot calculate installed size " + file + ". Try only apk size.");
                params.setSize(file.length());
            }

            mInstallHandler.post(new Runnable() {
                @Override
                public void run() {
                    //执行安装
                    doPackageStage(pm, params);
                }
            });
```



```java
 private void doPackageStage(PackageManager pm, PackageInstaller.SessionParams params) {
        //初始化安装器
        final PackageInstaller packageInstaller = pm.getPackageInstaller();
        PackageInstaller.Session session = null;
        try {
            final String packageLocation = mPackageURI.getPath();
            final File file = new File(packageLocation);
            //获取sessionId
            final int sessionId = packageInstaller.createSession(params);
            final byte[] buffer = new byte[65536];
     				//获取session
            session = packageInstaller.openSession(sessionId);

            final InputStream in = new FileInputStream(file);
            final long sizeBytes = file.length();
            // 根据这个session回话, 获取一个OutPutstream, 然后将文件写入到这个OutPutStream中
            final OutputStream out = session.openWrite("PackageInstaller", 0, sizeBytes);
            try {
                int c;
                while ((c = in.read(buffer)) != -1) {
                    out.write(buffer, 0, c);
                    if (sizeBytes > 0) {
                        final float fraction = ((float) c / (float) sizeBytes);
                        //安装中
                        session.addProgress(fraction);
                    }
                }
                session.fsync(out);
            } finally {
                IoUtils.closeQuietly(in);
                IoUtils.closeQuietly(out);
            }

            // Create a PendingIntent and use it to generate the IntentSender
            Intent broadcastIntent = new Intent(BROADCAST_ACTION);
            PendingIntent pendingIntent = PendingIntent.getBroadcast(
                    InstallAppProgress.this /*context*/,
                    sessionId,
                    broadcastIntent,
                    PendingIntent.FLAG_UPDATE_CURRENT);
            //最后调用session的commit方法进行提交
            session.commit(pendingIntent.getIntentSender());
        } catch (IOException e) {
            onPackageInstalled(PackageInstaller.STATUS_FAILURE);
        } finally {
            IoUtils.closeQuietly(session);
        }
    }
```

最终广播接受安装结果

```java
private final BroadcastReceiver mBroadcastReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            final int statusCode = intent.getIntExtra(
                    PackageInstaller.EXTRA_STATUS, PackageInstaller.STATUS_FAILURE);
            if (statusCode == PackageInstaller.STATUS_PENDING_USER_ACTION) {
                context.startActivity((Intent)intent.getParcelableExtra(Intent.EXTRA_INTENT));
            } else {
                onPackageInstalled(statusCode);
            }
        }
    };
```

应用安装的前期工作，用户可以看到的部分基本就这些，接下来的工作就都由Java框架层处理。

### Java框架层

上面最终调用PackageInstallerSession的commit方法，将安装APK的信息发给框架层。

`frameworks/base/services/core/java/com/android/server/pm/PackageInstallerSession.java`

```java
		@Override
    public void commit(IntentSender statusReceiver) {
        ......
        mActiveCount.incrementAndGet();
        final PackageInstallObserverAdapter adapter = new PackageInstallObserverAdapter(mContext,
                statusReceiver, sessionId, mIsInstallerDeviceOwner, userId);
        mHandler.obtainMessage(MSG_COMMIT, adapter.getBinder()).sendToTarget();
    }
```

commit函数中向Handler发送一个类型为MSG_COMMIT的消息，adapter.getBinder()会得到IPackageInstallObserver2观察者，处理消息逻辑如下：

`frameworks/base/services/core/java/com/android/server/pm/PackageInstallerSession.java`

```java
private final Handler.Callback mHandlerCallback = new Handler.Callback() {
        @Override
        public boolean handleMessage(Message msg) {
            // Cache package manager data without the lock held
            final PackageInfo pkgInfo = mPm.getPackageInfo(
                    params.appPackageName, PackageManager.GET_SIGNATURES /*flags*/, userId);
            final ApplicationInfo appInfo = mPm.getApplicationInfo(
                    params.appPackageName, 0, userId);

            synchronized (mLock) {
                if (msg.obj != null) {
                    mRemoteObserver = (IPackageInstallObserver2) msg.obj;
                }

                try {
                    commitLocked(pkgInfo, appInfo);
                } catch (PackageManagerException e) {
                    final String completeMsg = ExceptionUtils.getCompleteMessage(e);
                    Slog.e(TAG, "Commit of session " + sessionId + " failed: " + completeMsg);
                    destroyInternal();
                    dispatchSessionFinished(e.error, completeMsg, null);
                }

                return true;
            }
        }
    };
```



```java
private void commitLocked(PackageInfo pkgInfo, ApplicationInfo appInfo)
          throws PackageManagerException {
     ...
      mPm.installStage(mPackageName, stageDir, stageCid, localObserver, params,
              installerPackageName, installerUid, user, mCertificates);
  }
```

在commitLocked方法中，调用PMS的installStage方法，这样逻辑就进入PMS中了。

### 总结

PackageInstaller安装APK的过程，简单来说就两步：

1. 将APK的信息通过IO流的形式写入到PackageInstaller.Session中。
2. 调用PackageInstaller.Session的commit方法，将APK的信息交由PMS处理。